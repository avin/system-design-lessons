# Blob Storage и CDN

## Blob Storage

### Что такое Blob Storage?

**Blob (Binary Large Object) Storage** — хранилище для неструктурированных данных: изображения, видео, аудио, документы, backups.

**Характеристики**:
- Object-based (не filesystem hierarchy, хотя может эмулировать)
- Highly scalable (petabytes)
- Durable (99.999999999% durability — "11 nines")
- Relatively cheap
- Доступ через HTTP API

**Примеры**:
- **AWS S3** (Simple Storage Service)
- **Google Cloud Storage**
- **Azure Blob Storage**
- **Cloudflare R2**
- **MinIO** (self-hosted, S3-compatible)

### AWS S3 Basics

**Структура**:
```
Bucket (container)
  └─ Object (file)
       ├─ Key (path/to/file.jpg)
       ├─ Data (binary content)
       └─ Metadata (content-type, etc)
```

**Создание bucket**:
```python
import boto3

s3 = boto3.client('s3')

# Create bucket
s3.create_bucket(Bucket='my-app-images')
```

**Upload объекта**:
```python
# Upload file
s3.upload_file(
    Filename='local/path/image.jpg',
    Bucket='my-app-images',
    Key='uploads/2024/01/image.jpg'
)

# Upload from memory
s3.put_object(
    Bucket='my-app-images',
    Key='uploads/data.json',
    Body=json.dumps(data),
    ContentType='application/json'
)
```

**Download объекта**:
```python
# Download to file
s3.download_file(
    Bucket='my-app-images',
    Key='uploads/2024/01/image.jpg',
    Filename='local/path/downloaded.jpg'
)

# Download to memory
response = s3.get_object(Bucket='my-app-images', Key='uploads/data.json')
content = response['Body'].read()
```

**Генерация presigned URL**:
```python
# Temporary URL для upload
presigned_upload_url = s3.generate_presigned_url(
    'put_object',
    Params={
        'Bucket': 'my-app-images',
        'Key': 'uploads/user-photo.jpg',
        'ContentType': 'image/jpeg'
    },
    ExpiresIn=3600  # 1 hour
)

# Client может upload directly к S3
# PUT to presigned_upload_url

# Temporary URL для download
presigned_download_url = s3.generate_presigned_url(
    'get_object',
    Params={
        'Bucket': 'my-app-images',
        'Key': 'uploads/user-photo.jpg'
    },
    ExpiresIn=3600
)
```

### S3 Storage Classes

**Trade-off**: Цена vs доступность/latency

| Class | Use Case | Cost | Retrieval |
|-------|----------|------|-----------|
| **Standard** | Frequently accessed | $$$ | Instant |
| **Intelligent-Tiering** | Unknown access patterns | Auto | Instant |
| **Infrequent Access (IA)** | Monthly access | $$ | Instant |
| **Glacier Instant** | Quarterly access | $ | Instant |
| **Glacier Flexible** | Annual access | $ | Minutes-hours |
| **Glacier Deep Archive** | 7-10 year retention | ¢ | 12 hours |

**Lifecycle policy**:
```json
{
  "Rules": [{
    "Id": "Move old data to cheaper storage",
    "Status": "Enabled",
    "Transitions": [
      {
        "Days": 30,
        "StorageClass": "STANDARD_IA"
      },
      {
        "Days": 90,
        "StorageClass": "GLACIER"
      }
    ],
    "Expiration": {
      "Days": 365
    }
  }]
}
```

### Versioning

Сохранять все версии объекта.

```python
# Enable versioning
s3.put_bucket_versioning(
    Bucket='my-app-images',
    VersioningConfiguration={'Status': 'Enabled'}
)

# Upload создает новую версию
s3.put_object(Bucket='my-app-images', Key='file.txt', Body='v1')
s3.put_object(Bucket='my-app-images', Key='file.txt', Body='v2')

# List versions
versions = s3.list_object_versions(Bucket='my-app-images', Prefix='file.txt')

# Get specific version
s3.get_object(Bucket='my-app-images', Key='file.txt', VersionId='version-id-1')
```

**Use cases**:
- Защита от случайного удаления
- Rollback changes
- Compliance (audit trail)

### Access Control

**Bucket policy** (resource-based):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-public-bucket/*"
  }]
}
```

**IAM policy** (user-based):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",
      "s3:PutObject"
    ],
    "Resource": "arn:aws:s3:::my-app-images/*"
  }]
}
```

**ACL** (legacy, avoid):
```python
s3.put_object_acl(
    Bucket='my-app-images',
    Key='image.jpg',
    ACL='public-read'
)
```

### Use Cases для Blob Storage

#### 1. User-generated Content

```python
# User uploads profile photo
@app.route('/upload-photo', methods=['POST'])
def upload_photo():
    file = request.files['photo']

    # Generate unique filename
    filename = f"profiles/{current_user.id}/{uuid.uuid4()}.jpg"

    # Upload to S3
    s3.upload_fileobj(
        file,
        'my-app-images',
        filename,
        ExtraArgs={'ContentType': 'image/jpeg'}
    )

    # Save URL в БД
    user.profile_photo_url = f"https://my-app-images.s3.amazonaws.com/{filename}"
    db.session.commit()

    return {'url': user.profile_photo_url}
```

#### 2. Static Website Hosting

```bash
# S3 может хостить static websites
aws s3 website s3://my-static-site/ --index-document index.html

# Upload site
aws s3 sync ./dist s3://my-static-site/

# Access: http://my-static-site.s3-website-us-east-1.amazonaws.com
```

#### 3. Backups

```python
# Database backup to S3
import gzip

def backup_database():
    # Export database
    backup_file = f"backups/db-{datetime.now().isoformat()}.sql.gz"

    with gzip.open('/tmp/backup.sql.gz', 'wb') as f:
        # pg_dump или mysqldump
        subprocess.run(['pg_dump', 'mydb'], stdout=f)

    # Upload to S3
    s3.upload_file('/tmp/backup.sql.gz', 'my-backups', backup_file)

    # Lifecycle policy удалит старые backups
```

#### 4. Video/Audio Streaming

```python
# Upload video для streaming
s3.upload_file(
    'video.mp4',
    'my-videos',
    'videos/uuid.mp4',
    ExtraArgs={'ContentType': 'video/mp4'}
)

# CloudFront может stream directly из S3
video_url = f"https://d111111abcdef8.cloudfront.net/videos/uuid.mp4"
```

## CDN (Content Delivery Network)

### Что такое CDN?

**CDN** — распределенная сеть серверов (edge locations), которая кэширует и доставляет контент пользователям с географически ближайшего сервера.

```
Without CDN:
User в Tokyo → Origin Server в US → High latency (200ms)

With CDN:
User в Tokyo → CDN Edge в Tokyo → Low latency (10ms)
```

**Примеры**:
- **CloudFront** (AWS)
- **Cloudflare**
- **Fastly**
- **Akamai**
- **Google Cloud CDN**

### Как работает CDN

```
1. User requests https://cdn.example.com/image.jpg
2. CDN edge lookup в cache
3a. Cache HIT → Return image (fast, ~10ms)
3b. Cache MISS → Fetch from origin → Cache → Return (slower first time)
4. Subsequent requests → Cache HIT
```

**Edge Locations**:
```
CloudFront: 400+ edge locations worldwide
Cloudflare: 300+ cities in 120+ countries
```

### Cache Behavior

**TTL (Time To Live)**:
```
# HTTP headers
Cache-Control: max-age=86400  # Cache for 24 hours
Expires: Wed, 21 Oct 2025 07:28:00 GMT
```

**Cache key**:
```
Default: URL path
- /images/photo.jpg

Custom: URL + query params
- /images/photo.jpg?size=large
- /images/photo.jpg?size=small
```

**Invalidation**:
```bash
# CloudFront invalidation
aws cloudfront create-invalidation \
  --distribution-id E123456789 \
  --paths "/images/*" "/css/*"

# Cloudflare purge
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {token}" \
  -d '{"files":["https://example.com/image.jpg"]}'
```

### CloudFront Setup

**1. Create distribution**:
```python
import boto3

cloudfront = boto3.client('cloudfront')

response = cloudfront.create_distribution(
    DistributionConfig={
        'CallerReference': str(time.time()),
        'Origins': {
            'Quantity': 1,
            'Items': [{
                'Id': 'S3-my-app-images',
                'DomainName': 'my-app-images.s3.amazonaws.com',
                'S3OriginConfig': {
                    'OriginAccessIdentity': ''
                }
            }]
        },
        'DefaultCacheBehavior': {
            'TargetOriginId': 'S3-my-app-images',
            'ViewerProtocolPolicy': 'redirect-to-https',
            'AllowedMethods': {
                'Quantity': 2,
                'Items': ['GET', 'HEAD']
            },
            'CachedMethods': {
                'Quantity': 2,
                'Items': ['GET', 'HEAD']
            },
            'ForwardedValues': {
                'QueryString': False,
                'Cookies': {'Forward': 'none'}
            },
            'MinTTL': 0,
            'DefaultTTL': 86400,
            'MaxTTL': 31536000
        },
        'Enabled': True,
        'Comment': 'CDN for images'
    }
)

distribution_domain = response['Distribution']['DomainName']
# Example: d111111abcdef8.cloudfront.net
```

**2. Custom domain**:
```
CNAME: cdn.example.com → d111111abcdef8.cloudfront.net
SSL certificate (ACM)
```

**3. Usage**:
```html
<!-- Before: direct S3 -->
<img src="https://my-app-images.s3.amazonaws.com/uploads/photo.jpg">

<!-- After: through CloudFront -->
<img src="https://cdn.example.com/uploads/photo.jpg">
```

### Cache Strategies

#### 1. Static Assets

```
Images, CSS, JS — rarely change
Cache: max-age=31536000 (1 year)
```

**Versioned filenames**:
```
/css/style.abc123.css  # Hash в filename
/js/app.def456.js

# При deploy генерируются новые hashes
# No need для cache invalidation
```

#### 2. Dynamic Content

```
HTML pages — change frequently
Cache: max-age=60 (1 minute) или no-cache
```

**Edge Side Includes (ESI)**:
```html
<!-- Cache static parts, fetch dynamic parts -->
<html>
<head><!--cached--></head>
<body>
  <div class="content"><!--cached--></div>
  <div class="user-info">
    <!--#include virtual="/api/user-info" -->
  </div>
</body>
</html>
```

#### 3. API Responses

```python
@app.route('/api/products')
def get_products():
    products = db.query("SELECT * FROM products")

    # Cache на CDN
    response = jsonify(products)
    response.headers['Cache-Control'] = 'public, max-age=300'  # 5 min
    return response
```

**Vary header** для персонализации:
```python
response.headers['Vary'] = 'Accept-Language, Accept-Encoding'
```

#### 4. Personalized Content

```
Bypass CDN cache для user-specific content
Cache-Control: private, no-cache
```

Или используйте **Edge Computing** (Cloudflare Workers, Lambda@Edge):
```javascript
// Lambda@Edge: personalize at edge
exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;

  // Check cookie
  const userId = getCookie(headers.cookie, 'user_id');

  // Modify request based on user
  request.uri = `/personalized/${userId}/content`;

  return request;
};
```

### CDN Features

#### 1. Compression

```
# CloudFront auto-compresses
- gzip
- brotli

Response size: 1MB → 100KB (10x меньше)
```

#### 2. HTTP/2 & HTTP/3

```
CloudFront, Cloudflare support HTTP/2, HTTP/3
- Multiplexing
- Header compression
- Server push
```

#### 3. Image Optimization

**Cloudflare Image Resizing**:
```html
<!-- Original: 4000x3000, 5MB -->
<img src="https://example.com/image.jpg">

<!-- Resized: 800x600, 200KB -->
<img src="https://example.com/cdn-cgi/image/width=800,height=600,format=auto/image.jpg">
```

**AWS Lambda@Edge** для image processing:
```javascript
exports.handler = async (event) => {
  const response = event.Records[0].cf.response;

  // Resize, optimize, convert format
  const optimized = await sharp(response.body)
    .resize(800, 600)
    .webp({ quality: 80 })
    .toBuffer();

  response.body = optimized.toString('base64');
  response.headers['content-type'] = [{ value: 'image/webp' }];

  return response;
};
```

#### 4. DDoS Protection

```
CloudFront, Cloudflare защищают от DDoS
- Rate limiting
- WAF (Web Application Firewall)
- Bot detection
```

#### 5. SSL/TLS Termination

```
CDN терминирует SSL
Origin может быть HTTP (внутри trusted network)

User --HTTPS--> CDN --HTTP--> Origin
```

### Cost Optimization

**S3 + CloudFront vs только S3**:

```
S3 direct:
- Data transfer OUT: $0.09/GB
- GET requests: $0.0004/1000

CloudFront:
- Data transfer: $0.085/GB (дешевле!)
- Requests: $0.0075/10,000
- S3 to CloudFront: FREE

Savings: 5-20% + faster delivery
```

**Regional edge caches**:
```
User → Edge Location → Regional Edge Cache → Origin

Reduces load на origin
```

**Origin Shield**:
```
CloudFront Origin Shield — дополнительный caching layer
Минимизирует requests к origin
```

## Практическая архитектура

### Upload Flow

```
[User] → [App Server] → [S3]
         ↓
     [Generate presigned URL]
         ↓
[User] → [S3 directly] (upload)
         ↓
     [Trigger Lambda] (on upload)
         ↓
     [Process image: resize, thumbnails]
         ↓
     [Save metadata to DB]
         ↓
     [Invalidate CDN cache]
```

**Implementation**:
```python
@app.route('/api/upload-url', methods=['POST'])
def get_upload_url():
    # Generate unique filename
    filename = f"uploads/{current_user.id}/{uuid.uuid4()}.jpg"

    # Presigned URL для upload
    upload_url = s3.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': 'my-app-images',
            'Key': filename,
            'ContentType': 'image/jpeg'
        },
        ExpiresIn=300  # 5 minutes
    )

    return {'uploadUrl': upload_url, 'key': filename}

# Client
async function uploadImage(file) {
  // Get presigned URL
  const { uploadUrl, key } = await fetch('/api/upload-url').then(r => r.json());

  // Upload directly to S3
  await fetch(uploadUrl, {
    method: 'PUT',
    body: file,
    headers: { 'Content-Type': 'image/jpeg' }
  });

  // Notify backend
  await fetch('/api/upload-complete', {
    method: 'POST',
    body: JSON.stringify({ key })
  });
}
```

**Lambda trigger** (on S3 upload):
```python
import boto3
from PIL import Image

s3 = boto3.client('s3')

def lambda_handler(event, context):
    # S3 event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    # Download image
    s3.download_file(bucket, key, '/tmp/original.jpg')

    # Create thumbnail
    img = Image.open('/tmp/original.jpg')
    img.thumbnail((200, 200))
    img.save('/tmp/thumbnail.jpg')

    # Upload thumbnail
    thumbnail_key = key.replace('uploads/', 'thumbnails/')
    s3.upload_file('/tmp/thumbnail.jpg', bucket, thumbnail_key)

    # Save metadata to DB
    db.execute(
        "INSERT INTO images (key, thumbnail_key, user_id) VALUES (?, ?, ?)",
        (key, thumbnail_key, extract_user_id(key))
    )

    return {'statusCode': 200}
```

### Delivery Flow

```
[User] requests https://cdn.example.com/uploads/image.jpg
         ↓
  [CloudFront Edge] (cache check)
         ↓ (cache miss)
   [S3 Origin] (fetch image)
         ↓
  [CloudFront] (cache for 24h)
         ↓
    [User] (fast delivery)
```

### Multi-region Setup

```
Primary region: US-East
S3 bucket: my-app-images-us-east

Replica region: EU-West
S3 bucket: my-app-images-eu-west (replication)

CloudFront origins:
- Primary: my-app-images-us-east.s3.amazonaws.com
- Failover: my-app-images-eu-west.s3.amazonaws.com
```

**S3 Cross-Region Replication**:
```json
{
  "Role": "arn:aws:iam::account:role/replication-role",
  "Rules": [{
    "Status": "Enabled",
    "Priority": 1,
    "Filter": {},
    "Destination": {
      "Bucket": "arn:aws:s3:::my-app-images-eu-west",
      "StorageClass": "STANDARD"
    }
  }]
}
```

## Best Practices

### 1. Naming Convention

```
Структура:
/{environment}/{type}/{user-id}/{uuid}.{ext}

Examples:
/production/avatars/123/a1b2c3d4.jpg
/staging/documents/456/report.pdf
```

### 2. Security

```python
# Never make bucket public
# Use presigned URLs для temporary access

# CloudFront signed URLs для private content
from botocore.signers import CloudFrontSigner

def signed_url(resource):
    return cloudfront_signer.generate_presigned_url(
        resource,
        date_less_than=datetime.now() + timedelta(hours=1)
    )
```

### 3. Monitoring

```
Metrics:
- S3: storage size, request count, 4xx/5xx errors
- CloudFront: cache hit ratio, requests, bandwidth
- Costs: storage, data transfer, requests

Alerts:
- Cache hit ratio < 80%
- 4xx errors spike
- Costs > budget
```

### 4. Disaster Recovery

```bash
# S3 versioning
# Cross-region replication
# Lifecycle policies для backups

# Example: restore deleted file
aws s3api list-object-versions --bucket my-bucket --prefix file.jpg
aws s3api get-object --bucket my-bucket --key file.jpg --version-id {version}
```

## Что почитать дальше

- AWS S3 Documentation
- CloudFront Developer Guide
- Cloudflare Documentation
- "High Performance Browser Networking" — CDN chapter

## Проверьте себя

1. В чем разница между Blob Storage и file system?
2. Какие S3 storage classes существуют и когда их использовать?
3. Что такое presigned URL и зачем он нужен?
4. Как работает CDN и что такое edge location?
5. В чем преимущество CDN перед direct access к origin?
6. Как invalidate CDN cache?
7. Что такое cache hit ratio и почему он важен?
8. Как оптимизировать costs для S3 и CloudFront?

## Ключевые выводы

- Blob Storage (S3) — для хранения неструктурированных данных (images, videos)
- S3 storage classes — trade-off между cost и access frequency
- Presigned URLs — temporary access без публичных permissions
- Versioning защищает от случайного удаления
- CDN кэширует контент на edge locations близко к пользователям
- CDN снижает latency (10x+) и costs
- Cache-Control headers управляют TTL
- Versioned filenames для static assets → no cache invalidation needed
- CloudFront + S3 — стандартная комбинация для delivery
- Lambda@Edge — processing на edge (resize, personalization)
- Monitor cache hit ratio (target 80%+)
- Cross-region replication для disaster recovery

---

**Предыдущий урок**: [Денормализация и когда её применять](17-denormalizaciya.md)
**Следующий урок**: [Кэширование: стратегии и политики вытеснения](19-keshirovanie-strategii.md)
