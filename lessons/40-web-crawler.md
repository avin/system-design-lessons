# Урок 40: Дизайн Web Crawler

## Введение

Web Crawler (паук, робот) — это программа, которая автоматически обходит веб-страницы, скачивает их содержимое и извлекает ссылки для дальнейшего обхода. Краулеры — основа поисковых систем (Google, Bing), систем мониторинга (проверка broken links), архивации (Internet Archive), анализа данных и SEO-инструментов.

В этом уроке мы спроектируем масштабируемый, вежливый (respectful) и эффективный web crawler, способный обрабатывать миллиарды страниц.

**Примеры использования:**
- **Поисковые системы**: индексация веба (Googlebot, Bingbot)
- **Мониторинг**: отслеживание изменений на сайтах
- **Архивация**: Internet Archive Wayback Machine
- **SEO**: анализ сайтов (Screaming Frog, Ahrefs)
- **Data mining**: сбор данных для анализа

## Требования к системе

### Функциональные требования

1. **Crawling**:
   - Начиная с seed URLs, обходить веб-страницы
   - Извлекать ссылки и добавлять их в очередь
   - Скачивать и сохранять HTML-контент

2. **Politeness**:
   - Соблюдать robots.txt
   - Rate limiting per domain (не более N запросов в секунду)
   - Уважать Crawl-Delay директивы

3. **Фильтрация**:
   - Избегать дубликатов (одна и та же страница)
   - Фильтровать по типу контента (только HTML)
   - Настраиваемые правила (белый/чёрный список доменов)

### Нефункциональные требования

1. **Масштабируемость**:
   - 1 миллиард страниц в месяц (≈ 385 pages/sec)
   - Горизонтальное масштабирование

2. **Надёжность**:
   - Fault tolerance (перезапуск с checkpoint)
   - Обработка ошибок (timeouts, 404, 500, etc.)

3. **Производительность**:
   - Высокая пропускная способность
   - Эффективное использование сети

4. **Вежливость** (respectfulness):
   - Не перегружать целевые сайты
   - Соблюдение robots.txt — 100%

### Back-of-the-envelope расчёты

```
Цель: 1 миллиард страниц в месяц

1B pages / 30 days / 24 hours / 3600 sec ≈ 385 pages/sec

Средний размер HTML: 500 KB
Пропускная способность: 385 × 500 KB = 192 MB/sec ≈ 1.5 Gbps

Хранение:
1B pages × 500 KB = 500 TB (сжатие ~5x) = 100 TB compressed

Метаданные (URL, timestamp, checksum):
1B × 200 bytes = 200 GB
```

## Архитектура высокого уровня

```
                    ┌──────────────┐
                    │  Seed URLs   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ URL Frontier │◄───┐
                    │   (Queue)    │    │
                    └──────┬───────┘    │
                           │            │
                    ┌──────▼────────┐   │
                    │   Fetchers    │   │
                    │  (Workers)    │   │
                    └──────┬────────┘   │
                           │            │
                    ┌──────▼────────┐   │
                    │   DNS Resolver│   │
                    └──────┬────────┘   │
                           │            │
                    ┌──────▼────────┐   │
                    │HTTP Downloader│   │
                    └──────┬────────┘   │
                           │            │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐ ┌──▼───────┐ ┌──▼────────┐
       │Content Store│ │URL Filter│ │Link Extract│
       │   (S3/FS)   │ │(Seen URLs)│ │  (Parser) │
       └─────────────┘ └──────────┘ └──┬────────┘
                                       │
                                       └──────────┘
                                         new URLs
```

## Ключевые компоненты

### 1. URL Frontier (Очередь)

Управление очередью URL для обхода с приоритизацией.

```
┌─────────────────────────────────────────┐
│          URL Frontier                   │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌────────┐│
│  │Priority 1│  │Priority 2│  │Priority3││
│  │ (high)   │  │ (medium) │  │ (low)  ││
│  └────┬─────┘  └────┬─────┘  └────┬───┘│
│       │             │             │     │
│       └─────────────┼─────────────┘     │
│                     │                   │
│        ┌────────────▼────────────┐      │
│        │   Politeness Queue      │      │
│        │  (per-domain queues)    │      │
│        │                         │      │
│        │  domain1.com: [URL1,...]│      │
│        │  domain2.com: [URL2,...]│      │
│        └─────────────────────────┘      │
└─────────────────────────────────────────┘
```

**Реализация:**

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class URLFrontier {
  constructor() {
    this.redis = redis;

    // Приоритетные очереди
    this.priorityQueues = ['high', 'medium', 'low'];

    // Politeness: отслеживание последнего запроса к домену
    this.lastAccessTime = new Map(); // domain → timestamp
    this.crawlDelay = 1000; // 1 sec между запросами к одному домену
  }

  async enqueue(url, priority = 'medium') {
    const domain = new URL(url).hostname;

    // Добавляем в приоритетную очередь
    const queueKey = `frontier:${priority}:${domain}`;
    await this.redis.lpush(queueKey, url);

    // Добавляем домен в список активных
    await this.redis.sadd(`frontier:domains:${priority}`, domain);
  }

  async dequeue() {
    // Пробуем получить URL из каждой приоритетной очереди
    for (const priority of this.priorityQueues) {
      const url = await this.dequeueFromPriority(priority);
      if (url) return url;
    }

    return null; // очередь пуста
  }

  async dequeueFromPriority(priority) {
    // Получаем список доменов для этого приоритета
    const domains = await this.redis.smembers(`frontier:domains:${priority}`);

    if (domains.length === 0) return null;

    // Находим домен, к которому можем обратиться (politeness)
    for (const domain of domains) {
      const lastAccess = this.lastAccessTime.get(domain) || 0;
      const now = Date.now();

      if (now - lastAccess >= this.crawlDelay) {
        // Можем забрать URL с этого домена
        const queueKey = `frontier:${priority}:${domain}`;
        const url = await this.redis.rpop(queueKey);

        if (url) {
          this.lastAccessTime.set(domain, now);

          // Если очередь домена пуста — удаляем из активных
          const queueLen = await this.redis.llen(queueKey);
          if (queueLen === 0) {
            await this.redis.srem(`frontier:domains:${priority}`, domain);
          }

          return url;
        } else {
          // Очередь пуста, удаляем домен
          await this.redis.srem(`frontier:domains:${priority}`, domain);
        }
      }
    }

    return null; // нет доступных URL (все домены в cooldown)
  }

  async size() {
    let total = 0;

    for (const priority of this.priorityQueues) {
      const domains = await this.redis.smembers(`frontier:domains:${priority}`);

      for (const domain of domains) {
        const queueKey = `frontier:${priority}:${domain}`;
        const len = await this.redis.llen(queueKey);
        total += len;
      }
    }

    return total;
  }
}

// Использование
const frontier = new URLFrontier();

await frontier.enqueue('https://example.com/page1', 'high');
await frontier.enqueue('https://example.com/page2', 'high');
await frontier.enqueue('https://another.com/page1', 'medium');

const url = await frontier.dequeue();
console.log('Next URL to crawl:', url);
```

### 2. URL Filter (Фильтр дубликатов)

Проверка, был ли URL уже обработан.

**Подходы:**

1. **Bloom Filter** (probabilistic):
   - Экономия памяти
   - False positive возможен, false negative — нет

2. **Redis Set** (exact):
   - 100% точность
   - Больше памяти

```javascript
const BloomFilter = require('bloomfilter').BloomFilter;

class URLFilter {
  constructor() {
    // Bloom Filter: 1B URLs, 1% false positive rate
    // bits = -n * ln(p) / (ln(2)^2)
    // n = 1B, p = 0.01
    const bits = Math.ceil(-1e9 * Math.log(0.01) / (Math.log(2) ** 2));
    const numHashes = Math.ceil((bits / 1e9) * Math.log(2));

    this.bloom = new BloomFilter(bits, numHashes);

    // Backup: Redis Set для точной проверки (sample)
    this.redis = redis;
  }

  async isSeen(url) {
    // Быстрая проверка с Bloom Filter
    if (!this.bloom.test(url)) {
      return false; // точно не видели
    }

    // Возможен false positive, проверяем в Redis
    const exists = await this.redis.sismember('seen_urls', url);
    return exists === 1;
  }

  async markSeen(url) {
    this.bloom.add(url);
    await this.redis.sadd('seen_urls', url);
  }

  async clear() {
    this.bloom = new BloomFilter(this.bloom.buckets.length * 32, this.bloom.k);
    await this.redis.del('seen_urls');
  }
}
```

**Альтернатива: URL Checksum**

```javascript
const crypto = require('crypto');

class URLChecksumFilter {
  constructor() {
    this.redis = redis;
  }

  getChecksum(url) {
    // Нормализуем URL
    const normalized = this.normalizeURL(url);

    // MD5 hash (16 bytes) вместо полного URL (экономия памяти)
    return crypto.createHash('md5').update(normalized).digest('hex');
  }

  normalizeURL(url) {
    const parsed = new URL(url);

    // Удаляем якоря, utm-метки, сортируем query параметры
    parsed.hash = '';
    parsed.searchParams.delete('utm_source');
    parsed.searchParams.delete('utm_medium');
    parsed.searchParams.delete('utm_campaign');
    parsed.searchParams.sort();

    // Удаляем trailing slash
    let normalized = parsed.toString();
    if (normalized.endsWith('/')) {
      normalized = normalized.slice(0, -1);
    }

    return normalized.toLowerCase();
  }

  async isSeen(url) {
    const checksum = this.getChecksum(url);
    return await this.redis.sismember('url_checksums', checksum) === 1;
  }

  async markSeen(url) {
    const checksum = this.getChecksum(url);
    await this.redis.sadd('url_checksums', checksum);
  }
}
```

### 3. Fetcher (Worker)

Скачивание страниц с обработкой ошибок и retry.

```javascript
const axios = require('axios');
const robotsParser = require('robots-parser');

class Fetcher {
  constructor(userAgent = 'MyCrawler/1.0') {
    this.userAgent = userAgent;
    this.robotsCache = new Map(); // domain → robots.txt parser
    this.timeout = 10000; // 10 seconds
  }

  async fetch(url) {
    try {
      // Проверяем robots.txt
      const allowed = await this.isAllowed(url);

      if (!allowed) {
        console.log(`Blocked by robots.txt: ${url}`);
        return { success: false, reason: 'robots.txt' };
      }

      // Скачиваем страницу
      const response = await axios.get(url, {
        headers: { 'User-Agent': this.userAgent },
        timeout: this.timeout,
        maxRedirects: 5,
        validateStatus: (status) => status < 500 // accept 4xx
      });

      if (response.status !== 200) {
        return {
          success: false,
          reason: `HTTP ${response.status}`,
          url
        };
      }

      // Проверяем Content-Type
      const contentType = response.headers['content-type'] || '';
      if (!contentType.includes('text/html')) {
        return {
          success: false,
          reason: 'Non-HTML content',
          contentType
        };
      }

      return {
        success: true,
        url,
        html: response.data,
        headers: response.headers,
        timestamp: Date.now()
      };
    } catch (error) {
      console.error(`Fetch error for ${url}:`, error.message);

      return {
        success: false,
        reason: error.message,
        url,
        retry: this.isRetryable(error)
      };
    }
  }

  async isAllowed(url) {
    const parsed = new URL(url);
    const domain = parsed.hostname;
    const robotsURL = `${parsed.protocol}//${domain}/robots.txt`;

    // Проверяем кэш
    if (this.robotsCache.has(domain)) {
      const robots = this.robotsCache.get(domain);
      return robots.isAllowed(url, this.userAgent);
    }

    // Загружаем robots.txt
    try {
      const response = await axios.get(robotsURL, {
        timeout: 5000,
        validateStatus: (status) => status === 200 || status === 404
      });

      const robotsTxt = response.status === 200 ? response.data : '';
      const robots = robotsParser(robotsURL, robotsTxt);

      this.robotsCache.set(domain, robots);

      return robots.isAllowed(url, this.userAgent);
    } catch (error) {
      console.error(`Failed to fetch robots.txt for ${domain}:`, error.message);
      // Если не смогли получить robots.txt — разрешаем (осторожно!)
      return true;
    }
  }

  getCrawlDelay(domain) {
    const robots = this.robotsCache.get(domain);

    if (robots) {
      const delay = robots.getCrawlDelay(this.userAgent);
      return delay ? delay * 1000 : 1000; // в миллисекундах
    }

    return 1000; // по умолчанию 1 sec
  }

  isRetryable(error) {
    // Retry на network errors, timeouts, 5xx
    return (
      error.code === 'ECONNRESET' ||
      error.code === 'ETIMEDOUT' ||
      error.code === 'ENOTFOUND' ||
      (error.response && error.response.status >= 500)
    );
  }
}
```

### 4. Link Extractor (Парсинг ссылок)

Извлечение ссылок из HTML.

```javascript
const cheerio = require('cheerio');

class LinkExtractor {
  extract(html, baseURL) {
    const $ = cheerio.load(html);
    const links = [];

    // Извлекаем все <a href="...">
    $('a[href]').each((i, elem) => {
      const href = $(elem).attr('href');

      if (href) {
        try {
          // Резолвим относительные URL
          const absoluteURL = new URL(href, baseURL);

          // Фильтруем протоколы
          if (absoluteURL.protocol === 'http:' || absoluteURL.protocol === 'https:') {
            links.push(absoluteURL.toString());
          }
        } catch (error) {
          // Невалидный URL, игнорируем
        }
      }
    });

    // Удаляем дубликаты
    return [...new Set(links)];
  }

  extractMetadata(html) {
    const $ = cheerio.load(html);

    return {
      title: $('title').text().trim(),
      description: $('meta[name="description"]').attr('content') || '',
      keywords: $('meta[name="keywords"]').attr('content') || '',
      canonical: $('link[rel="canonical"]').attr('href') || ''
    };
  }
}
```

### 5. Content Store (Хранение)

```javascript
const AWS = require('aws-sdk');
const s3 = new AWS.S3();
const zlib = require('zlib');
const { promisify } = require('util');
const gzip = promisify(zlib.gzip);

class ContentStore {
  constructor(bucket) {
    this.bucket = bucket;
  }

  async save(url, html, metadata) {
    const urlHash = crypto.createHash('sha256').update(url).digest('hex');
    const key = `pages/${urlHash.slice(0, 2)}/${urlHash}.html.gz`;

    // Сжимаем HTML
    const compressed = await gzip(html);

    // Сохраняем в S3
    await s3.putObject({
      Bucket: this.bucket,
      Key: key,
      Body: compressed,
      ContentType: 'text/html',
      ContentEncoding: 'gzip',
      Metadata: {
        url: url,
        title: metadata.title || '',
        timestamp: Date.now().toString()
      }
    }).promise();

    // Сохраняем метаданные в БД
    await this.saveMetadata(url, urlHash, key, metadata);

    return { key, hash: urlHash };
  }

  async saveMetadata(url, hash, s3Key, metadata) {
    await db.query(
      `INSERT INTO crawled_pages
       (url_hash, url, s3_key, title, description, crawled_at)
       VALUES ($1, $2, $3, $4, $5, NOW())
       ON CONFLICT (url_hash) DO UPDATE SET
         crawled_at = NOW(),
         title = $4,
         description = $5`,
      [hash, url, s3Key, metadata.title, metadata.description]
    );
  }

  async load(urlHash) {
    // Получаем s3_key из БД
    const result = await db.query(
      'SELECT s3_key FROM crawled_pages WHERE url_hash = $1',
      [urlHash]
    );

    if (result.rows.length === 0) {
      return null;
    }

    const s3Key = result.rows[0].s3_key;

    // Загружаем из S3
    const response = await s3.getObject({
      Bucket: this.bucket,
      Key: s3Key
    }).promise();

    // Разархивируем
    const html = await promisify(zlib.gunzip)(response.Body);

    return html.toString('utf-8');
  }
}
```

## Полная реализация Crawler

```javascript
const EventEmitter = require('events');

class WebCrawler extends EventEmitter {
  constructor(options = {}) {
    super();

    this.frontier = new URLFrontier();
    this.urlFilter = new URLFilter();
    this.fetcher = new Fetcher(options.userAgent);
    this.linkExtractor = new LinkExtractor();
    this.contentStore = new ContentStore(options.s3Bucket);

    this.maxConcurrency = options.maxConcurrency || 10;
    this.activeWorkers = 0;
    this.running = false;

    this.stats = {
      fetched: 0,
      failed: 0,
      skipped: 0
    };
  }

  async start(seedURLs) {
    this.running = true;

    // Добавляем seed URLs
    for (const url of seedURLs) {
      await this.frontier.enqueue(url, 'high');
    }

    console.log(`Starting crawler with ${seedURLs.length} seed URLs`);

    // Запускаем workers
    const workers = [];
    for (let i = 0; i < this.maxConcurrency; i++) {
      workers.push(this.worker(i));
    }

    await Promise.all(workers);

    console.log('Crawler finished');
    console.log('Stats:', this.stats);
  }

  async worker(id) {
    console.log(`Worker ${id} started`);

    while (this.running) {
      // Получаем следующий URL
      const url = await this.frontier.dequeue();

      if (!url) {
        // Очередь пуста, ждём или завершаем
        if (this.activeWorkers === 0) {
          // Все workers idle — заканчиваем
          this.running = false;
          break;
        }

        await this.sleep(1000);
        continue;
      }

      // Проверяем дубликаты
      if (await this.urlFilter.isSeen(url)) {
        this.stats.skipped++;
        continue;
      }

      this.activeWorkers++;

      try {
        await this.crawlURL(url);
      } catch (error) {
        console.error(`Worker ${id} error:`, error);
      } finally {
        this.activeWorkers--;
      }
    }

    console.log(`Worker ${id} stopped`);
  }

  async crawlURL(url) {
    console.log(`Crawling: ${url}`);

    // Отмечаем как seen
    await this.urlFilter.markSeen(url);

    // Скачиваем страницу
    const result = await this.fetcher.fetch(url);

    if (!result.success) {
      console.log(`Failed to fetch ${url}: ${result.reason}`);

      if (result.retry) {
        // Добавляем обратно в очередь с низким приоритетом
        await this.frontier.enqueue(url, 'low');
      }

      this.stats.failed++;
      return;
    }

    // Извлекаем метаданные
    const metadata = this.linkExtractor.extractMetadata(result.html);

    // Сохраняем контент
    await this.contentStore.save(url, result.html, metadata);

    this.stats.fetched++;

    // Извлекаем ссылки
    const links = this.linkExtractor.extract(result.html, url);

    console.log(`Extracted ${links.length} links from ${url}`);

    // Добавляем в очередь
    for (const link of links) {
      if (!(await this.urlFilter.isSeen(link))) {
        await this.frontier.enqueue(link, 'medium');
      }
    }

    this.emit('page', { url, metadata, linksCount: links.length });
  }

  async stop() {
    console.log('Stopping crawler...');
    this.running = false;
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Использование
const crawler = new WebCrawler({
  userAgent: 'MyCrawler/1.0 (+https://example.com/bot)',
  s3Bucket: 'my-crawler-data',
  maxConcurrency: 20
});

crawler.on('page', (data) => {
  console.log(`Crawled: ${data.url} (${data.linksCount} links)`);
});

const seedURLs = [
  'https://example.com',
  'https://en.wikipedia.org/wiki/Main_Page'
];

await crawler.start(seedURLs);
```

## Оптимизации и продвинутые техники

### 1. DNS Caching

```javascript
const dns = require('dns').promises;
const LRU = require('lru-cache');

class DNSResolver {
  constructor() {
    this.cache = new LRU({
      max: 10000,
      maxAge: 3600000 // 1 hour
    });
  }

  async resolve(hostname) {
    if (this.cache.has(hostname)) {
      return this.cache.get(hostname);
    }

    try {
      const addresses = await dns.resolve4(hostname);
      const ip = addresses[0];

      this.cache.set(hostname, ip);

      return ip;
    } catch (error) {
      console.error(`DNS resolution failed for ${hostname}:`, error.message);
      throw error;
    }
  }
}
```

### 2. Приоритизация URL

```javascript
class URLPrioritizer {
  calculatePriority(url, metadata = {}) {
    let score = 0;

    // Глубина в URL path (меньше — лучше)
    const depth = new URL(url).pathname.split('/').filter(Boolean).length;
    score += Math.max(0, 10 - depth);

    // PageRank (если есть)
    if (metadata.pageRank) {
      score += metadata.pageRank * 100;
    }

    // Свежесть (если уже crawlили раньше)
    if (metadata.lastCrawled) {
      const ageHours = (Date.now() - metadata.lastCrawled) / 3600000;
      score += Math.min(ageHours / 24, 10); // max 10 points за старость
    }

    // Домен (приоритетные домены)
    const hostname = new URL(url).hostname;
    if (this.priorityDomains.includes(hostname)) {
      score += 50;
    }

    return score;
  }

  getPriorityBucket(score) {
    if (score >= 100) return 'high';
    if (score >= 50) return 'medium';
    return 'low';
  }
}
```

### 3. Distributed Crawling

Для масштабирования на множество машин:

```javascript
class DistributedCrawler {
  constructor(nodeId, totalNodes) {
    this.nodeId = nodeId;
    this.totalNodes = totalNodes;
  }

  shouldCrawl(url) {
    // Consistent hashing: каждая нода отвечает за subset URL
    const hash = crypto.createHash('md5').update(url).digest();
    const hashInt = hash.readUInt32BE(0);
    const assignedNode = hashInt % this.totalNodes;

    return assignedNode === this.nodeId;
  }

  async enqueue(url) {
    if (this.shouldCrawl(url)) {
      // Локальная очередь
      await this.frontier.enqueue(url);
    } else {
      // Отправляем другой ноде (через message queue)
      const targetNode = this.getTargetNode(url);
      await this.sendToNode(targetNode, url);
    }
  }

  getTargetNode(url) {
    const hash = crypto.createHash('md5').update(url).digest();
    const hashInt = hash.readUInt32BE(0);
    return hashInt % this.totalNodes;
  }

  async sendToNode(nodeId, url) {
    // Публикуем в Kafka/RabbitMQ топик для этой ноды
    await kafka.send({
      topic: `crawler-node-${nodeId}`,
      messages: [{ value: url }]
    });
  }
}
```

### 4. Recrawling Strategy

```javascript
class RecrawlScheduler {
  calculateRecrawlInterval(url, metadata) {
    // Факторы:
    // - Как часто меняется страница (change frequency)
    // - Важность страницы (PageRank)
    // - Тип контента (новости vs документация)

    const changeFreq = metadata.changeFrequency || 'weekly';
    const importance = metadata.pageRank || 0.5;

    const intervals = {
      always: 3600,        // 1 hour
      hourly: 3600,
      daily: 86400,        // 1 day
      weekly: 604800,      // 7 days
      monthly: 2592000,    // 30 days
      yearly: 31536000,    // 365 days
      never: Infinity
    };

    let baseInterval = intervals[changeFreq] || intervals.weekly;

    // Adjust by importance
    baseInterval = baseInterval / (1 + importance);

    return baseInterval;
  }

  async scheduleRecrawl(url, metadata) {
    const interval = this.calculateRecrawlInterval(url, metadata);
    const nextCrawl = Date.now() + interval * 1000;

    await db.query(
      'UPDATE crawled_pages SET next_crawl_at = $1 WHERE url = $2',
      [new Date(nextCrawl), url]
    );
  }
}
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const crawlCounter = new prometheus.Counter({
  name: 'crawler_pages_total',
  help: 'Total pages crawled',
  labelNames: ['status'] // success, failed, skipped
});

const crawlDuration = new prometheus.Histogram({
  name: 'crawler_page_duration_seconds',
  help: 'Time to crawl a page',
  buckets: [0.1, 0.5, 1, 2, 5, 10]
});

const frontierSize = new prometheus.Gauge({
  name: 'crawler_frontier_size',
  help: 'Number of URLs in frontier'
});

// В методе crawlURL
const start = Date.now();

try {
  const result = await this.fetcher.fetch(url);

  if (result.success) {
    crawlCounter.inc({ status: 'success' });
  } else {
    crawlCounter.inc({ status: 'failed' });
  }
} finally {
  crawlDuration.observe((Date.now() - start) / 1000);
}

// Периодически обновляем frontier size
setInterval(async () => {
  const size = await this.frontier.size();
  frontierSize.set(size);
}, 10000);
```

## Best Practices

1. **Respectfulness**:
   - ВСЕГДА соблюдайте robots.txt
   - Используйте rate limiting per domain
   - Идентифицируйте crawler через User-Agent с контактной информацией

2. **Fault Tolerance**:
   - Checkpoint state регулярно
   - Retry transient errors (timeouts, 5xx)
   - Graceful shutdown при SIGTERM

3. **URL Normalization**:
   - Удаляйте tracking parameters (utm_*)
   - Сортируйте query parameters
   - Нормализуйте протокол (http → https если редирект)

4. **Content Deduplication**:
   - Используйте content hash (SimHash, MinHash)
   - Избегайте crawl одинакового контента на разных URL

## Что читать дальше

- **Урок 37**: Rate Limiter — управление скоростью запросов к доменам
- **Урок 41**: Notification System — уведомления о изменениях на сайтах
- **Урок 45**: Search System — индексация данных после crawling

## Проверь себя

1. Почему нужна per-domain очередь (politeness queue)?
2. Как Bloom Filter помогает экономить память при проверке дубликатов?
3. Спроектируйте distributed crawler на 100 машин.
4. Как определить оптимальную частоту recrawl для страницы?
5. Какие проблемы возникают при crawling динамических сайтов (JavaScript-rendered)?

---

[← Урок 39: Генератор уникальных идентификаторов](39-unique-id-generator.md) | [Урок 41: Notification System →](41-notification-system.md)
