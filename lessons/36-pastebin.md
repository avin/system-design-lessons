# Урок 36: Pastebin


## Введение

Pastebin — это веб-сервис для обмена текстовыми данными (код, логи, конфигурации). Пользователь вставляет текст, получает уникальную ссылку для шеринга.

**Примеры:** pastebin.com, GitHub Gist, jsbin.com

**Основная функциональность:**
- Создать paste (текст) → получить URL
- Прочитать paste по URL
- Syntax highlighting для кода
- Expiration time (1 hour, 1 day, never)
- Private/Public pastes

## Требования

### Функциональные требования

1. **Создание paste**
   - Input: текст (до 10 MB)
   - Output: уникальная ссылка
   - Опции: язык программирования, expiration, private/public

2. **Чтение paste**
   - Input: paste URL
   - Output: текст с syntax highlighting

3. **Expiration**
   - Автоматическое удаление через N времени

4. **Analytics** (опционально)
   - Просмотры, география

### Нефункциональные требования

1. **Availability**: 99.9%
2. **Durability**: paste не должен потеряться
3. **Low latency** для чтения (<200ms)
4. **Scalability**: миллионы paste в день

### Back-of-the-envelope расчёты

```
Предположения:
- 1M новых paste/день
- 5:1 read:write ratio
- Средний размер paste: 10 KB
- Хранить paste до истечения (50% - 1 день, 30% - 1 неделя, 20% - навсегда)

Запись:
- 1M paste/день
- ~12 paste/секунду
- Пики: ~25 paste/сек

Чтение:
- 5M reads/день
- ~60 reads/сек
- Пики: ~120 reads/сек

Storage (5 лет):
- Active paste (без expiration): 1M × 20% × 365 × 5 = 365M paste
- Средний размер: 10 KB
- Метаданные: 1 KB
- Итого: 11 KB × 365M = ~4 TB

Bandwidth:
- Write: 12 paste/s × 10 KB = 120 KB/s
- Read: 60 reads/s × 10 KB = 600 KB/s
```

## High-Level Design

### Архитектура

```
Client → CDN (static assets)
       ↓
   Load Balancer
       ↓
┌──────┴──────┐
│  API        │
│  Servers    │
└──────┬──────┘
       │
   ┌───┴────┬─────────┬────────┐
   ↓        ↓         ↓        ↓
Cache    Database   Object   Analytics
(Redis)    (SQL)   Storage    (Queue)
                   (S3/Blob)
```

### API Design

```javascript
// POST /api/v1/paste
Request:
{
  "content": "console.log('Hello World');",
  "language": "javascript",
  "title": "My First Paste",
  "expiration": "1d",  // 1h, 1d, 1w, never
  "visibility": "public"  // public, unlisted, private
}

Response:
{
  "paste_id": "abc123xyz",
  "url": "https://paste.bin/abc123xyz",
  "created_at": "2024-01-15T10:30:00Z",
  "expire_at": "2024-01-16T10:30:00Z"
}

// GET /api/v1/paste/{paste_id}
Response:
{
  "paste_id": "abc123xyz",
  "content": "console.log('Hello World');",
  "language": "javascript",
  "title": "My First Paste",
  "created_at": "2024-01-15T10:30:00Z",
  "expire_at": "2024-01-16T10:30:00Z",
  "views": 42
}

// DELETE /api/v1/paste/{paste_id}
Response:
{
  "success": true
}
```

## Database Design

### SQL Schema

```sql
CREATE TABLE pastes (
  id BIGSERIAL PRIMARY KEY,
  paste_id VARCHAR(20) UNIQUE NOT NULL,
  title VARCHAR(255),
  language VARCHAR(50),
  content_url TEXT NOT NULL,  -- S3/Blob Storage URL
  content_hash VARCHAR(64),   -- SHA256 hash для deduplication
  user_id BIGINT,
  visibility VARCHAR(20) DEFAULT 'public',  -- public, unlisted, private
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expire_at TIMESTAMP,
  views BIGINT DEFAULT 0,
  size_bytes INT NOT NULL,

  INDEX idx_paste_id (paste_id),
  INDEX idx_user_id (user_id),
  INDEX idx_expire_at (expire_at),
  INDEX idx_content_hash (content_hash)
);

CREATE TABLE paste_metadata (
  paste_id VARCHAR(20) PRIMARY KEY,
  ip_address VARCHAR(45),
  user_agent TEXT,
  country VARCHAR(2),
  referrer TEXT
);
```

### Хранение контента

**Опция 1: В БД (для маленьких paste)**

```sql
ALTER TABLE pastes ADD COLUMN content TEXT;
```

**Опция 2: Object Storage (для больших paste)**

```javascript
// Хранить content в S3/Blob Storage
// В БД только metadata + URL

const AWS = require('aws-sdk');
const s3 = new AWS.S3();

async function uploadPasteContent(pasteId, content) {
  const key = `pastes/${pasteId}`;

  await s3.putObject({
    Bucket: 'pastebin-content',
    Key: key,
    Body: content,
    ContentType: 'text/plain',
  }).promise();

  return `s3://pastebin-content/${key}`;
}

async function getPasteContent(contentUrl) {
  const key = contentUrl.replace('s3://pastebin-content/', '');

  const result = await s3.getObject({
    Bucket: 'pastebin-content',
    Key: key,
  }).promise();

  return result.Body.toString('utf-8');
}
```

**Выбор стратегии:**

| Размер | Хранение | Причина |
|--------|----------|---------|
| < 1 KB | Database (TEXT column) | Быстрый доступ, меньше latency |
| 1 KB - 100 KB | Database или Blob | Зависит от частоты доступа |
| > 100 KB | Blob Storage (S3) | Экономия БД, лучше для редких чтений |

## Реализация сервиса

### Pastebin Service

```javascript
const express = require('express');
const Redis = require('ioredis');
const { Pool } = require('pg');
const AWS = require('aws-sdk');
const crypto = require('crypto');

const app = express();
app.use(express.json({ limit: '10mb' }));

const redis = new Redis();
const db = new Pool({ connectionString: process.env.DATABASE_URL });
const s3 = new AWS.S3();

const ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';

// ID Generator (similar to URL Shortener)
class IDGenerator {
  constructor(workerId) {
    this.workerId = workerId;
    this.sequence = 0;
    this.lastTimestamp = -1;
  }

  generate() {
    let timestamp = Date.now();

    if (timestamp === this.lastTimestamp) {
      this.sequence = (this.sequence + 1) & 0xFFF;

      if (this.sequence === 0) {
        while (timestamp <= this.lastTimestamp) {
          timestamp = Date.now();
        }
      }
    } else {
      this.sequence = 0;
    }

    this.lastTimestamp = timestamp;

    const id = (BigInt(timestamp) << 22n) |
               (BigInt(this.workerId) << 12n) |
               BigInt(this.sequence);

    return Number(id);
  }
}

const idGen = new IDGenerator(parseInt(process.env.WORKER_ID || '1'));

function encodeBase62(num) {
  if (num === 0) return ALPHABET[0];

  let encoded = '';

  while (num > 0) {
    encoded = ALPHABET[num % 62] + encoded;
    num = Math.floor(num / 62);
  }

  return encoded;
}

function calculateExpireAt(expiration) {
  const now = new Date();

  switch (expiration) {
    case '1h':
      return new Date(now.getTime() + 60 * 60 * 1000);
    case '1d':
      return new Date(now.getTime() + 24 * 60 * 60 * 1000);
    case '1w':
      return new Date(now.getTime() + 7 * 24 * 60 * 60 * 1000);
    case 'never':
      return null;
    default:
      return new Date(now.getTime() + 24 * 60 * 60 * 1000);  // default 1 day
  }
}

// POST /api/v1/paste
app.post('/api/v1/paste', async (req, res) => {
  const { content, language, title, expiration = '1d', visibility = 'public' } = req.body;

  if (!content) {
    return res.status(400).json({ error: 'content is required' });
  }

  if (content.length > 10 * 1024 * 1024) {  // 10 MB
    return res.status(400).json({ error: 'content too large (max 10 MB)' });
  }

  try {
    const pasteId = encodeBase62(idGen.generate());
    const contentHash = crypto.createHash('sha256').update(content).digest('hex');
    const expireAt = calculateExpireAt(expiration);

    let contentUrl;

    // Deduplication check
    const existing = await db.query(
      'SELECT paste_id FROM pastes WHERE content_hash = $1 AND expire_at > NOW()',
      [contentHash]
    );

    if (existing.rows.length > 0) {
      // Content already exists → return existing paste
      const existingPasteId = existing.rows[0].paste_id;

      return res.json({
        paste_id: existingPasteId,
        url: `${process.env.BASE_URL}/${existingPasteId}`,
        message: 'Duplicate content, returning existing paste',
      });
    }

    // Store content
    if (content.length < 100 * 1024) {
      // Small paste → store in DB
      contentUrl = null;  // content stored in DB
    } else {
      // Large paste → store in S3
      const key = `pastes/${pasteId}`;

      await s3.putObject({
        Bucket: process.env.S3_BUCKET,
        Key: key,
        Body: content,
        ContentType: 'text/plain',
      }).promise();

      contentUrl = `s3://${process.env.S3_BUCKET}/${key}`;
    }

    // Save metadata to DB
    await db.query(`
      INSERT INTO pastes (
        paste_id, title, language, content, content_url, content_hash,
        visibility, expire_at, size_bytes
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
    `, [
      pasteId,
      title,
      language,
      contentUrl ? null : content,  // Store in DB if no S3
      contentUrl,
      contentHash,
      visibility,
      expireAt,
      Buffer.byteLength(content, 'utf-8'),
    ]);

    res.json({
      paste_id: pasteId,
      url: `${process.env.BASE_URL}/${pasteId}`,
      created_at: new Date().toISOString(),
      expire_at: expireAt?.toISOString(),
    });

  } catch (error) {
    console.error('Error creating paste:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// GET /api/v1/paste/{paste_id}
app.get('/api/v1/paste/:paste_id', async (req, res) => {
  const { paste_id } = req.params;

  try {
    // 1. Check cache
    const cached = await redis.get(`paste:${paste_id}`);

    if (cached) {
      const paste = JSON.parse(cached);

      // Increment views (async)
      incrementViews(paste_id);

      return res.json(paste);
    }

    // 2. Query DB
    const result = await db.query(`
      SELECT paste_id, title, language, content, content_url,
             visibility, created_at, expire_at, views, size_bytes
      FROM pastes
      WHERE paste_id = $1
    `, [paste_id]);

    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Paste not found' });
    }

    const paste = result.rows[0];

    // Check expiration
    if (paste.expire_at && new Date(paste.expire_at) < new Date()) {
      return res.status(410).json({ error: 'Paste expired' });
    }

    // Get content
    let content = paste.content;

    if (paste.content_url) {
      // Fetch from S3
      const key = paste.content_url.replace(`s3://${process.env.S3_BUCKET}/`, '');

      const s3Result = await s3.getObject({
        Bucket: process.env.S3_BUCKET,
        Key: key,
      }).promise();

      content = s3Result.Body.toString('utf-8');
    }

    const response = {
      paste_id: paste.paste_id,
      title: paste.title,
      language: paste.language,
      content,
      visibility: paste.visibility,
      created_at: paste.created_at,
      expire_at: paste.expire_at,
      views: paste.views,
      size_bytes: paste.size_bytes,
    };

    // Cache (TTL based on expiration)
    const ttl = paste.expire_at
      ? Math.floor((new Date(paste.expire_at) - new Date()) / 1000)
      : 3600 * 24;  // 24 hours default

    await redis.setex(`paste:${paste_id}`, Math.min(ttl, 3600 * 24), JSON.stringify(response));

    // Increment views
    incrementViews(paste_id);

    res.json(response);

  } catch (error) {
    console.error('Error fetching paste:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// DELETE /api/v1/paste/{paste_id}
app.delete('/api/v1/paste/:paste_id', async (req, res) => {
  const { paste_id } = req.params;

  // TODO: Add authentication (only owner can delete)

  try {
    const result = await db.query(`
      DELETE FROM pastes
      WHERE paste_id = $1
      RETURNING content_url
    `, [paste_id]);

    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Paste not found' });
    }

    const contentUrl = result.rows[0].content_url;

    // Delete from S3 if stored there
    if (contentUrl) {
      const key = contentUrl.replace(`s3://${process.env.S3_BUCKET}/`, '');

      await s3.deleteObject({
        Bucket: process.env.S3_BUCKET,
        Key: key,
      }).promise();
    }

    // Remove from cache
    await redis.del(`paste:${paste_id}`);

    res.json({ success: true });

  } catch (error) {
    console.error('Error deleting paste:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Increment views (async)
async function incrementViews(pasteId) {
  try {
    await db.query(
      'UPDATE pastes SET views = views + 1 WHERE paste_id = $1',
      [pasteId]
    );
  } catch (error) {
    console.error('Error incrementing views:', error);
  }
}

app.listen(3000, () => {
  console.log('Pastebin running on port 3000');
});
```

## Кэширование

### Cache Strategy

```javascript
// Cache популярных paste (80/20 rule)

// Write-Through Cache
async function createPaste(data) {
  const paste = await db.insert(data);

  // Cache immediately (для популярных paste)
  await redis.setex(
    `paste:${paste.paste_id}`,
    3600,
    JSON.stringify(paste)
  );

  return paste;
}

// Cache-Aside для Read
async function getPaste(pasteId) {
  // 1. Check cache
  let paste = await redis.get(`paste:${pasteId}`);

  if (paste) {
    return JSON.parse(paste);
  }

  // 2. Query DB
  paste = await db.query('SELECT * FROM pastes WHERE paste_id = $1', [pasteId]);

  if (!paste) {
    return null;
  }

  // 3. Store in cache
  await redis.setex(`paste:${pasteId}`, 3600, JSON.stringify(paste));

  return paste;
}
```

### Cache Eviction

```javascript
// LRU eviction в Redis
// maxmemory-policy: allkeys-lru

// Redis config
redis-cli CONFIG SET maxmemory 2gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

## Expiration & Cleanup

### Automated Cleanup

```javascript
const cron = require('node-cron');

// Cleanup expired paste (каждый час)
cron.schedule('0 * * * *', async () => {
  console.log('Cleaning up expired pastes...');

  const result = await db.query(`
    DELETE FROM pastes
    WHERE expire_at IS NOT NULL AND expire_at < NOW()
    RETURNING paste_id, content_url
  `);

  console.log(`Deleted ${result.rowCount} expired pastes`);

  // Delete from S3 and cache
  for (const row of result.rows) {
    // Remove from cache
    await redis.del(`paste:${row.paste_id}`);

    // Delete from S3
    if (row.content_url) {
      const key = row.content_url.replace(`s3://${process.env.S3_BUCKET}/`, '');

      await s3.deleteObject({
        Bucket: process.env.S3_BUCKET,
        Key: key,
      }).promise();
    }
  }
});
```

### Lazy Deletion

```javascript
// Проверять expiration при чтении
app.get('/api/v1/paste/:paste_id', async (req, res) => {
  const paste = await getPaste(req.params.paste_id);

  if (!paste) {
    return res.status(404).json({ error: 'Paste not found' });
  }

  // Lazy deletion
  if (paste.expire_at && new Date(paste.expire_at) < new Date()) {
    // Delete immediately
    await deletePaste(paste.paste_id);

    return res.status(410).json({ error: 'Paste expired' });
  }

  res.json(paste);
});
```

## Масштабирование

### 1. Read Replicas

```javascript
// Primary для записи
const primaryDb = new Pool({ connectionString: process.env.PRIMARY_DB_URL });

// Replicas для чтения
const replicaDbs = [
  new Pool({ connectionString: process.env.REPLICA1_DB_URL }),
  new Pool({ connectionString: process.env.REPLICA2_DB_URL }),
];

function getReadDb() {
  return replicaDbs[Math.floor(Math.random() * replicaDbs.length)];
}

// Write
async function createPaste(data) {
  return await primaryDb.query('INSERT INTO pastes ...', data);
}

// Read
async function getPaste(pasteId) {
  const db = getReadDb();
  return await db.query('SELECT * FROM pastes WHERE paste_id = $1', [pasteId]);
}
```

### 2. CDN для статики

```javascript
// Syntax highlighting assets
// Prism.js, highlight.js

// CDN headers
app.use('/static', express.static('public', {
  maxAge: '30d',  // Cache в CDN на 30 дней
  setHeaders: (res) => {
    res.set('Cache-Control', 'public, max-age=2592000');
  },
}));
```

### 3. Sharding по paste_id

```javascript
function getShardId(pasteId) {
  const hash = pasteId.split('').reduce((acc, char) => {
    return acc + char.charCodeAt(0);
  }, 0);

  return hash % NUM_SHARDS;
}

async function getPaste(pasteId) {
  const shardId = getShardId(pasteId);
  const db = dbShards[shardId];

  return await db.query('SELECT * FROM pastes WHERE paste_id = $1', [pasteId]);
}
```

## Дополнительные фичи

### 1. Syntax Highlighting

```javascript
const hljs = require('highlight.js');

app.get('/view/:paste_id', async (req, res) => {
  const paste = await getPaste(req.params.paste_id);

  if (!paste) {
    return res.status(404).send('Paste not found');
  }

  // Syntax highlighting
  const highlighted = paste.language
    ? hljs.highlight(paste.content, { language: paste.language }).value
    : paste.content;

  res.render('paste', {
    title: paste.title,
    content: highlighted,
    language: paste.language,
  });
});
```

### 2. Diff/Version History

```sql
CREATE TABLE paste_versions (
  id BIGSERIAL PRIMARY KEY,
  paste_id VARCHAR(20) NOT NULL,
  version INT NOT NULL,
  content_url TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  UNIQUE(paste_id, version),
  FOREIGN KEY (paste_id) REFERENCES pastes(paste_id) ON DELETE CASCADE
);
```

```javascript
app.post('/api/v1/paste/:paste_id/edit', async (req, res) => {
  const { paste_id } = req.params;
  const { content } = req.body;

  // Get current version
  const currentVersion = await db.query(
    'SELECT MAX(version) as version FROM paste_versions WHERE paste_id = $1',
    [paste_id]
  );

  const newVersion = (currentVersion.rows[0]?.version || 0) + 1;

  // Store new version
  const contentUrl = await uploadToS3(paste_id, content);

  await db.query(`
    INSERT INTO paste_versions (paste_id, version, content_url)
    VALUES ($1, $2, $3)
  `, [paste_id, newVersion, contentUrl]);

  res.json({ version: newVersion });
});
```

### 3. Fork/Clone

```javascript
app.post('/api/v1/paste/:paste_id/fork', async (req, res) => {
  const { paste_id } = req.params;

  const original = await getPaste(paste_id);

  if (!original) {
    return res.status(404).json({ error: 'Paste not found' });
  }

  // Create new paste with same content
  const newPasteId = encodeBase62(idGen.generate());

  await db.query(`
    INSERT INTO pastes (paste_id, title, language, content, content_url, visibility)
    VALUES ($1, $2, $3, $4, $5, $6)
  `, [
    newPasteId,
    `Fork of ${original.title}`,
    original.language,
    original.content,
    original.content_url,
    'public',
  ]);

  res.json({
    paste_id: newPasteId,
    url: `${process.env.BASE_URL}/${newPasteId}`,
  });
});
```

### 4. RAW endpoint

```javascript
app.get('/raw/:paste_id', async (req, res) => {
  const paste = await getPaste(req.params.paste_id);

  if (!paste) {
    return res.status(404).send('Paste not found');
  }

  res.set('Content-Type', 'text/plain');
  res.send(paste.content);
});
```

## Security

### 1. Content Sanitization

```javascript
const sanitizeHtml = require('sanitize-html');

app.post('/api/v1/paste', async (req, res) => {
  let { content } = req.body;

  // Sanitize для предотвращения XSS
  content = sanitizeHtml(content, {
    allowedTags: [],
    allowedAttributes: {},
  });

  // ...
});
```

### 2. Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

const createLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,  // 10 paste за 15 минут
  message: 'Too many pastes created',
});

app.post('/api/v1/paste', createLimiter, async (req, res) => {
  // ...
});
```

### 3. Private Paste Access Control

```javascript
app.get('/api/v1/paste/:paste_id', async (req, res) => {
  const paste = await getPaste(req.params.paste_id);

  if (!paste) {
    return res.status(404).json({ error: 'Paste not found' });
  }

  if (paste.visibility === 'private') {
    // Требовать аутентификацию
    const userId = req.user?.id;

    if (!userId || paste.user_id !== userId) {
      return res.status(403).json({ error: 'Access denied' });
    }
  }

  res.json(paste);
});
```

## Мониторинг

```javascript
const promClient = require('prom-client');

const pasteCreations = new promClient.Counter({
  name: 'paste_creations_total',
  help: 'Total paste creations',
  labelNames: ['language'],
});

const pasteReads = new promClient.Counter({
  name: 'paste_reads_total',
  help: 'Total paste reads',
  labelNames: ['cache_hit'],
});

const pasteSize = new promClient.Histogram({
  name: 'paste_size_bytes',
  help: 'Paste size distribution',
  buckets: [1024, 10240, 102400, 1048576, 10485760],
});

app.post('/api/v1/paste', async (req, res) => {
  pasteCreations.inc({ language: req.body.language || 'unknown' });
  pasteSize.observe(Buffer.byteLength(req.body.content));
  // ...
});

app.get('/api/v1/paste/:paste_id', async (req, res) => {
  const cached = await redis.get(`paste:${req.params.paste_id}`);

  pasteReads.inc({ cache_hit: cached ? 'true' : 'false' });
  // ...
});
```

## Выводы

Pastebin — система для хранения и обмена текстом:

**Ключевые компоненты:**
- ID generation (Base62)
- Content storage (DB + Blob Storage)
- Caching (Redis)
- Expiration handling
- Syntax highlighting

**Design decisions:**
- Маленькие paste → DB
- Большие paste → S3/Blob Storage
- Deduplication через content hash
- Lazy + Scheduled cleanup для expiration

**Масштабирование:**
- Read replicas для reads
- CDN для static assets
- Sharding по paste_id
- Object storage для больших файлов

**Важные детали:**
- Durability (не потерять paste)
- Low latency reads (cache)
- Efficient storage (deduplication)
- Clean expired paste (cron + lazy)

## Проверь себя

1. Когда хранить content в DB, а когда в Blob Storage?
2. Как работает deduplication через content hash?
3. Зачем нужны два способа cleanup (lazy + scheduled)?
4. Как кэшировать paste с учётом expiration?
5. В чём разница между public, unlisted и private paste?
6. Как реализовать syntax highlighting?
7. Как масштабировать систему для миллионов paste?
8. Зачем нужен RAW endpoint?
9. Как защититься от spam paste?
10. Какие trade-offs между consistency и performance?

---
**Предыдущий урок**: [Урок 35: Дизайн URL Shortener (bit.ly)](35-design-url-shortener.md)
**Следующий урок**: [Урок 37: Rate Limiter как система](37-rate-limiter.md)
