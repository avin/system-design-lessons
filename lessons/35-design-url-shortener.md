# Урок 35: Дизайн URL Shortener (bit.ly)


## Введение

URL Shortener — это сервис, который превращает длинные URL в короткие, удобные для шеринга ссылки.

**Примеры:** bit.ly, tinyurl.com, goo.gl

**Функциональность:**
- Создать короткую ссылку из длинной URL
- Перенаправить на оригинальный URL при клике
- (Опционально) Аналитика кликов, custom aliases, TTL

## Требования

### Функциональные требования

1. **Создание короткой ссылки**
   - Input: длинный URL
   - Output: короткий URL

2. **Редирект**
   - Input: короткий URL
   - Output: HTTP 302 redirect на оригинальный URL

3. **Кастомные alias** (опционально)
   - Пользователь выбирает свой короткий код

4. **TTL** (опционально)
   - Ссылка автоматически истекает через N дней

### Нефункциональные требования

1. **High availability** (99.99%)
2. **Low latency** для редиректа (<100ms)
3. **Scalability** (миллионы новых URL в день)
4. **Уникальность** коротких ссылок

### Back-of-the-envelope расчёты

```
Предположения:
- 100M новых URL/день
- 10:1 read:write ratio (100M writes, 1B reads/день)
- Хранить URL 5 лет

Запись:
- 100M URL/день
- ~1,200 URL/секунду
- Пики: ~2,500 URL/сек

Чтение (редиректы):
- 1B редиректов/день
- ~12,000 редиректов/сек
- Пики: ~25,000 редиректов/сек

Storage:
- 100M URL × 365 дней × 5 лет = 182.5B URL
- Средний размер URL: 500 байт
- Метаданные: 100 байт
- Итого: 600 байт × 182.5B = ~110 TB за 5 лет

Bandwidth:
- Write: 1,200 req/s × 500 байт = 600 KB/s
- Read: 12,000 req/s × 500 байт = 6 MB/s
```

## High-Level Design

### Базовая архитектура

```
Client → API Gateway → URL Shortener Service → Database
                                ↓
                            Cache (Redis)
```

### API Design

```
POST /api/v1/shorten
Request:
{
  "long_url": "https://example.com/very/long/url/with/parameters?foo=bar",
  "custom_alias": "my-link",  // optional
  "expire_at": "2025-12-31T23:59:59Z"  // optional
}

Response:
{
  "short_url": "https://short.ly/abc123",
  "long_url": "https://example.com/very/long/url/with/parameters?foo=bar",
  "created_at": "2024-01-15T10:30:00Z",
  "expire_at": "2025-12-31T23:59:59Z"
}

---

GET /{short_code}
Response: HTTP 302 Redirect
Location: https://example.com/very/long/url/with/parameters?foo=bar
```

## Генерация короткого кода

### Требования к short_code

- **Уникальный**
- **Короткий** (7-8 символов достаточно)
- **URL-safe** (буквы, цифры)

### Варианты генерации

**1. Hash (MD5/SHA256)**

```javascript
const crypto = require('crypto');

function generateShortCode(longUrl) {
  const hash = crypto.createHash('md5').update(longUrl).digest('hex');

  // Взять первые 7 символов
  return hash.substring(0, 7);
}

// Проблема: коллизии!
// Разные URL могут дать одинаковый hash (первые 7 символов)
```

**Решение коллизий:**

```javascript
async function generateUniqueShortCode(longUrl) {
  let shortCode = generateShortCode(longUrl);
  let attempt = 0;

  while (await db.exists(shortCode)) {
    // Коллизия → добавить counter
    shortCode = generateShortCode(longUrl + attempt);
    attempt++;

    if (attempt > 10) {
      throw new Error('Too many collisions');
    }
  }

  return shortCode;
}
```

**2. Base62 Encoding**

```javascript
const ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
const BASE = ALPHABET.length;  // 62

function encodeBase62(num) {
  if (num === 0) return ALPHABET[0];

  let encoded = '';

  while (num > 0) {
    encoded = ALPHABET[num % BASE] + encoded;
    num = Math.floor(num / BASE);
  }

  return encoded;
}

function decodeBase62(str) {
  let decoded = 0;

  for (let i = 0; i < str.length; i++) {
    decoded = decoded * BASE + ALPHABET.indexOf(str[i]);
  }

  return decoded;
}

// Использование:
// Генерируем уникальный ID (auto-increment или snowflake)
const id = 123456789;
const shortCode = encodeBase62(id);  // "8M0kX"

console.log(shortCode);  // "8M0kX"
console.log(decodeBase62(shortCode));  // 123456789
```

**Длина short_code:**

```
62^7 = 3.5 триллионов комбинаций
62^6 = 56 миллиардов комбинаций
62^5 = 916 миллионов комбинаций

Для 182.5B URL (5 лет) → 7 символов достаточно
```

**3. Random Generation**

```javascript
function generateRandomShortCode(length = 7) {
  let code = '';

  for (let i = 0; i < length; i++) {
    code += ALPHABET[Math.floor(Math.random() * BASE)];
  }

  return code;
}

async function generateUniqueRandomCode() {
  let shortCode;
  let attempts = 0;
  const MAX_ATTEMPTS = 5;

  do {
    shortCode = generateRandomShortCode();
    attempts++;

    if (attempts > MAX_ATTEMPTS) {
      throw new Error('Failed to generate unique code');
    }
  } while (await db.exists(shortCode));

  return shortCode;
}
```

### Выбор стратегии

| Стратегия | Плюсы | Минусы |
|-----------|-------|--------|
| **Hash** | Один URL → один hash | Коллизии, нужна проверка |
| **Base62 (ID)** | Гарантированно уникальный | Предсказуемый |
| **Random** | Непредсказуемый | Коллизии, нужна проверка |

**Рекомендация:** Base62 с distributed ID generator (Snowflake).

## Database Design

### Schema

```sql
CREATE TABLE urls (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  short_code VARCHAR(10) UNIQUE NOT NULL,
  long_url TEXT NOT NULL,
  user_id BIGINT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expire_at TIMESTAMP,
  clicks BIGINT DEFAULT 0,
  INDEX idx_short_code (short_code),
  INDEX idx_user_id (user_id),
  INDEX idx_expire_at (expire_at)
);

CREATE TABLE analytics (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  short_code VARCHAR(10) NOT NULL,
  clicked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  ip_address VARCHAR(45),
  user_agent TEXT,
  referer TEXT,
  country VARCHAR(2),
  INDEX idx_short_code (short_code),
  INDEX idx_clicked_at (clicked_at)
);
```

### NoSQL Alternative (Cassandra)

```sql
-- URL mapping
CREATE TABLE urls (
  short_code TEXT PRIMARY KEY,
  long_url TEXT,
  user_id BIGINT,
  created_at TIMESTAMP,
  expire_at TIMESTAMP,
  clicks COUNTER
);

-- Analytics (time-series)
CREATE TABLE analytics (
  short_code TEXT,
  clicked_at TIMESTAMP,
  ip_address TEXT,
  user_agent TEXT,
  referer TEXT,
  country TEXT,
  PRIMARY KEY (short_code, clicked_at)
) WITH CLUSTERING ORDER BY (clicked_at DESC);
```

## Реализация сервиса

### URL Shortener Service

```javascript
const express = require('express');
const Redis = require('ioredis');
const { Pool } = require('pg');

const app = express();
const redis = new Redis();
const db = new Pool({ connectionString: process.env.DATABASE_URL });

const ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
const BASE = 62;

// Generate distributed ID (simplified Snowflake)
class IDGenerator {
  constructor(workerId) {
    this.workerId = workerId;
    this.sequence = 0;
    this.lastTimestamp = -1;
  }

  generate() {
    let timestamp = Date.now();

    if (timestamp === this.lastTimestamp) {
      this.sequence = (this.sequence + 1) & 0xFFF;  // 12 bits

      if (this.sequence === 0) {
        // Sequence overflow → wait next millisecond
        while (timestamp <= this.lastTimestamp) {
          timestamp = Date.now();
        }
      }
    } else {
      this.sequence = 0;
    }

    this.lastTimestamp = timestamp;

    // 41 bits timestamp + 10 bits worker + 12 bits sequence
    const id = (BigInt(timestamp) << 22n) |
               (BigInt(this.workerId) << 12n) |
               BigInt(this.sequence);

    return Number(id);
  }
}

const idGen = new IDGenerator(1);

function encodeBase62(num) {
  if (num === 0) return ALPHABET[0];

  let encoded = '';

  while (num > 0) {
    encoded = ALPHABET[num % BASE] + encoded;
    num = Math.floor(num / BASE);
  }

  return encoded;
}

// POST /api/v1/shorten
app.post('/api/v1/shorten', async (req, res) => {
  const { long_url, custom_alias, expire_at } = req.body;

  if (!long_url) {
    return res.status(400).json({ error: 'long_url is required' });
  }

  try {
    let shortCode;

    if (custom_alias) {
      // Проверить доступность
      const existing = await db.query(
        'SELECT short_code FROM urls WHERE short_code = $1',
        [custom_alias]
      );

      if (existing.rows.length > 0) {
        return res.status(409).json({ error: 'Alias already taken' });
      }

      shortCode = custom_alias;
    } else {
      // Генерируем уникальный ID
      const id = idGen.generate();
      shortCode = encodeBase62(id);
    }

    // Сохранить в БД
    await db.query(`
      INSERT INTO urls (id, short_code, long_url, expire_at, created_at)
      VALUES ($1, $2, $3, $4, NOW())
    `, [
      custom_alias ? null : idGen.generate(),
      shortCode,
      long_url,
      expire_at || null,
    ]);

    // Закэшировать
    await redis.setex(
      `url:${shortCode}`,
      3600 * 24,  // 24 hours TTL
      long_url
    );

    res.json({
      short_url: `${process.env.BASE_URL}/${shortCode}`,
      long_url,
      short_code: shortCode,
      created_at: new Date().toISOString(),
      expire_at,
    });

  } catch (error) {
    console.error('Error creating short URL:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// GET /{short_code} - Redirect
app.get('/:short_code', async (req, res) => {
  const { short_code } = req.params;

  try {
    // 1. Проверить кэш
    let longUrl = await redis.get(`url:${short_code}`);

    if (longUrl) {
      // Increment clicks (async, не блокируем редирект)
      incrementClicks(short_code, req);

      return res.redirect(302, longUrl);
    }

    // 2. Запрос к БД
    const result = await db.query(`
      SELECT long_url, expire_at
      FROM urls
      WHERE short_code = $1
    `, [short_code]);

    if (result.rows.length === 0) {
      return res.status(404).send('URL not found');
    }

    const url = result.rows[0];

    // Проверить expiration
    if (url.expire_at && new Date(url.expire_at) < new Date()) {
      return res.status(410).send('URL expired');
    }

    longUrl = url.long_url;

    // Закэшировать
    await redis.setex(`url:${short_code}`, 3600 * 24, longUrl);

    // Increment clicks
    incrementClicks(short_code, req);

    res.redirect(302, longUrl);

  } catch (error) {
    console.error('Error redirecting:', error);
    res.status(500).send('Internal server error');
  }
});

// Analytics (async)
async function incrementClicks(shortCode, req) {
  // Increment counter
  await db.query(`
    UPDATE urls
    SET clicks = clicks + 1
    WHERE short_code = $1
  `, [shortCode]);

  // Log analytics
  await db.query(`
    INSERT INTO analytics (short_code, clicked_at, ip_address, user_agent, referer)
    VALUES ($1, NOW(), $2, $3, $4)
  `, [
    shortCode,
    req.ip,
    req.get('user-agent'),
    req.get('referer'),
  ]);
}

app.listen(3000, () => {
  console.log('URL Shortener running on port 3000');
});
```

## Масштабирование

### 1. Кэширование

```
Cache Hit Rate: 80-90% (популярные ссылки)

Redis:
- Key: url:{short_code}
- Value: long_url
- TTL: 24 hours

Cache-Aside Pattern:
1. Check cache
2. If miss → query DB
3. Store in cache
```

### 2. Шардирование БД

**Стратегия: Hash-based sharding**

```javascript
function getShardId(shortCode) {
  const hash = shortCode.split('').reduce((acc, char) => {
    return acc + char.charCodeAt(0);
  }, 0);

  return hash % NUM_SHARDS;
}

async function getUrl(shortCode) {
  const shardId = getShardId(shortCode);
  const db = dbShards[shardId];

  const result = await db.query(
    'SELECT long_url FROM urls WHERE short_code = $1',
    [shortCode]
  );

  return result.rows[0]?.long_url;
}
```

### 3. Read Replicas

```
Master (Write):
- Создание новых URL

Replicas (Read):
- Редиректы (99% трафика)

Load Balancer:
- Распределение read запросов между репликами
```

### 4. CDN для популярных ссылок

```javascript
// Заголовки для CDN caching
app.get('/:short_code', async (req, res) => {
  const longUrl = await getUrl(req.params.short_code);

  // Cache в CDN на 1 час
  res.set('Cache-Control', 'public, max-age=3600');
  res.redirect(302, longUrl);
});
```

### 5. Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

const createLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 минут
  max: 100,  // 100 requests
  message: 'Too many URL creations',
});

app.post('/api/v1/shorten', createLimiter, async (req, res) => {
  // ...
});
```

## Дополнительные фичи

### 1. Аналитика

```javascript
// GET /api/v1/analytics/{short_code}
app.get('/api/v1/analytics/:short_code', async (req, res) => {
  const { short_code } = req.params;

  const stats = await db.query(`
    SELECT
      COUNT(*) as total_clicks,
      COUNT(DISTINCT ip_address) as unique_visitors,
      DATE(clicked_at) as date,
      COUNT(*) as clicks_per_day
    FROM analytics
    WHERE short_code = $1
    GROUP BY DATE(clicked_at)
    ORDER BY date DESC
    LIMIT 30
  `, [short_code]);

  const topCountries = await db.query(`
    SELECT country, COUNT(*) as clicks
    FROM analytics
    WHERE short_code = $1 AND country IS NOT NULL
    GROUP BY country
    ORDER BY clicks DESC
    LIMIT 10
  `, [short_code]);

  res.json({
    short_code,
    total_clicks: stats.rows.reduce((sum, row) => sum + parseInt(row.clicks_per_day), 0),
    clicks_per_day: stats.rows,
    top_countries: topCountries.rows,
  });
});
```

### 2. QR Code Generation

```javascript
const QRCode = require('qrcode');

app.get('/api/v1/qr/:short_code', async (req, res) => {
  const shortUrl = `${process.env.BASE_URL}/${req.params.short_code}`;

  const qrCode = await QRCode.toDataURL(shortUrl);

  res.json({ qr_code: qrCode });
});
```

### 3. Custom Domain

```javascript
// Пользователь может использовать свой домен
// short.example.com/{code}

app.get('/:short_code', async (req, res) => {
  const domain = req.hostname;

  const result = await db.query(`
    SELECT long_url
    FROM urls
    WHERE short_code = $1 AND (custom_domain = $2 OR custom_domain IS NULL)
  `, [req.params.short_code, domain]);

  if (result.rows.length === 0) {
    return res.status(404).send('URL not found');
  }

  res.redirect(302, result.rows[0].long_url);
});
```

### 4. Link Expiration Cleanup

```javascript
// Cron job для удаления истёкших ссылок
const cron = require('node-cron');

cron.schedule('0 2 * * *', async () => {
  console.log('Cleaning expired URLs...');

  const result = await db.query(`
    DELETE FROM urls
    WHERE expire_at IS NOT NULL AND expire_at < NOW()
    RETURNING short_code
  `);

  console.log(`Deleted ${result.rowCount} expired URLs`);

  // Удалить из кэша
  for (const row of result.rows) {
    await redis.del(`url:${row.short_code}`);
  }
});
```

## Security

### 1. Защита от abuse

```javascript
// Detect malicious URLs
const maliciousPatterns = [
  /phishing/i,
  /malware/i,
  /virus/i,
];

function isMaliciousUrl(url) {
  return maliciousPatterns.some(pattern => pattern.test(url));
}

app.post('/api/v1/shorten', async (req, res) => {
  const { long_url } = req.body;

  if (isMaliciousUrl(long_url)) {
    return res.status(400).json({ error: 'Malicious URL detected' });
  }

  // ...
});
```

### 2. Prevent enumeration

```javascript
// Не возвращать информацию о существовании URL
app.get('/api/v1/check/:short_code', async (req, res) => {
  // ❌ Bad: раскрывает существование
  const exists = await db.query(
    'SELECT 1 FROM urls WHERE short_code = $1',
    [req.params.short_code]
  );

  if (exists.rows.length > 0) {
    return res.json({ exists: true });
  }

  // ✅ Good: требовать аутентификацию
  // Только владелец может проверить существование
});
```

## Мониторинг

```javascript
const promClient = require('prom-client');

const urlCreations = new promClient.Counter({
  name: 'url_creations_total',
  help: 'Total URL creations',
});

const redirects = new promClient.Counter({
  name: 'redirects_total',
  help: 'Total redirects',
  labelNames: ['status'],
});

const cacheHitRate = new promClient.Counter({
  name: 'cache_hits_total',
  help: 'Cache hits vs misses',
  labelNames: ['result'],
});

app.post('/api/v1/shorten', async (req, res) => {
  urlCreations.inc();
  // ...
});

app.get('/:short_code', async (req, res) => {
  const cached = await redis.get(`url:${req.params.short_code}`);

  if (cached) {
    cacheHitRate.inc({ result: 'hit' });
  } else {
    cacheHitRate.inc({ result: 'miss' });
  }

  redirects.inc({ status: res.statusCode });
  // ...
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});
```

## Trade-offs и решения

| Аспект | Решение | Trade-off |
|--------|---------|-----------|
| **ID generation** | Snowflake + Base62 | Предсказуемые ID vs Уникальность |
| **Consistency** | Eventual (cache) | Latency vs Consistency |
| **Storage** | SQL (PostgreSQL) | ACID vs Scalability |
| **Analytics** | Async logging | Точность vs Performance |
| **Redirect** | 302 (Temporary) | Caching vs Analytics |

## Выводы

URL Shortener — отличный пример для изучения:

**Ключевые компоненты:**
- ID generation (Snowflake, Base62)
- Caching (Redis)
- Database sharding
- Rate limiting
- Analytics

**Масштабирование:**
- Cache для 80%+ requests
- Read replicas для редиректов
- Sharding для хранения
- CDN для популярных ссылок

**Важные детали:**
- Low latency для редиректов (<100ms)
- High availability (99.99%)
- Collision-free ID generation
- Analytics без влияния на redirect performance

## Что читать дальше?

- [Урок 36: Pastebin](36-pastebin.md)
- [Урок 39: Генератор уникальных идентификаторов](39-unique-id-generator.md)
- [Урок 19: Кэширование: стратегии](19-keshirovanie-strategii.md)

## Проверь себя

1. Как генерировать уникальные short codes?
2. Почему Base62 предпочтительнее Base64?
3. Сколько символов нужно для 100B URL?
4. Как работает кэширование редиректов?
5. В чём разница между 301 и 302 редиректами?
6. Как шардировать БД для URL shortener?
7. Как обрабатывать коллизии при генерации?
8. Зачем нужен distributed ID generator?
9. Как реализовать аналитику без влияния на latency?
10. Как защититься от abuse (создание spam ссылок)?

---
**Предыдущий урок**: [Урок 34: Eventual Consistency](34-eventual-consistency.md)
**Следующий урок**: [Урок 36: Pastebin](36-pastebin.md)
