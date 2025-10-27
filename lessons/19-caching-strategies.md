# Кэширование: стратегии и политики вытеснения

## Что такое кэширование?

**Cache (Кэш)** — быстрое хранилище для часто используемых данных. Основан на принципе locality:
- **Temporal locality**: Недавно использованные данные вероятно будут использованы снова
- **Spatial locality**: Данные рядом с использованными вероятно будут использованы

```
Without cache:
Request → Database (50ms) → Response

With cache:
Request → Cache (1ms) → Response ✅
Request → Cache miss → Database (50ms) → Cache → Response
```

**Performance gain**: 50x faster!

## Зачем нужно кэширование?

### 1. Снижение latency

```
Database query: 50ms
Cache lookup: 1ms

Speedup: 50x
```

### 2. Снижение нагрузки на БД

```
Without cache: 10,000 QPS на БД
With cache (90% hit rate): 1,000 QPS на БД

Load reduction: 10x
```

### 3. Снижение costs

```
Database reads: $0.001 per request
Cache reads: $0.0001 per request

Cost reduction: 10x
```

### 4. Scalability

```
Database: vertical scaling (expensive)
Cache: horizontal scaling (cheap)
```

## Cache Layers

### 1. Client-side Cache (Browser)

```
HTTP headers:
Cache-Control: max-age=3600
```

**Преимущества**: Zero latency, zero server load
**Недостатки**: No control после отправки

### 2. CDN Cache

```
CloudFront edge locations кэшируют static assets
```

**Преимущества**: Geographic distribution, DDoS protection
**Недостатки**: Только для static/public content

### 3. Application-level Cache (In-memory)

```javascript
// In-process cache (Map, LRU cache)
const cache = new Map();

async function getUser(userId) {
  if (cache.has(userId)) {
    return cache.get(userId);
  }

  const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);
  cache.set(userId, user);
  return user;
}
```

**Преимущества**: Ultra-fast (nanoseconds)
**Недостатки**: Не shared между серверами, memory limited

### 4. Distributed Cache (Redis, Memcached)

```javascript
const Redis = require('ioredis');

const cache = new Redis({ host: 'localhost', port: 6379 });

async function getUser(userId) {
  // Try cache
  const cached = await cache.get(`user:${userId}`);
  if (cached) {
    return JSON.parse(cached);
  }

  // Cache miss
  const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);
  await cache.setex(`user:${userId}`, 3600, JSON.stringify(user));
  return user;
}
```

**Преимущества**: Shared, scalable, persistent (Redis)
**Недостатки**: Network latency (~1ms)

### 5. Database Query Cache

```sql
-- MySQL query cache (deprecated)
SELECT SQL_CACHE * FROM users WHERE id = 123;
```

**Современный подход**: Application-level cache (Redis) вместо DB cache.

## Cache Patterns (Стратегии)

### 1. Cache-Aside (Lazy Loading)

**Наиболее распространенный pattern**.

```javascript
async function getData(key) {
  // 1. Check cache
  const cached = await cache.get(key);
  if (cached) {
    return JSON.parse(cached); // Cache hit
  }

  // 2. Cache miss → query database
  const data = await database.query(key);

  // 3. Store в cache
  await cache.set(key, JSON.stringify(data), 'EX', 3600);

  return data;
}
```

**Diagram**:
```
App → Cache (check)
  ↓ miss
  → Database (query)
  → Cache (set)
  → Return
```

**Преимущества**:
✅ Only requested data в cache (efficient)
✅ Проще reasoning (явный control)
✅ Cache failure не критичен (fallback на DB)

**Недостатки**:
❌ Cache miss penalty (query + cache write)
❌ Потенциально stale data

**Use cases**: Read-heavy applications

### 2. Read-Through Cache

Похоже на Cache-Aside, но cache сам запрашивает БД.

```javascript
// Cache library handles DB query
const data = await cache.getOrSet(key, async () => database.query(key));
```

**Diagram**:
```
App → Cache (get)
       ↓ miss
     Database (query by cache)
       ↓
     Cache (set)
       ↓
    Return to App
```

**Преимущества**:
✅ Упрощает application code
✅ Consistent interface

**Недостатки**:
❌ Cache должен знать о data source

### 3. Write-Through Cache

Пишем одновременно в cache и database.

```javascript
async function updateData(key, value) {
  // 1. Write to cache
  await cache.set(key, JSON.stringify(value));

  // 2. Write to database
  await database.update(key, value);

  return value;
}
```

**Diagram**:
```
App → Write
  → Cache (set)
  → Database (update)
```

**Преимущества**:
✅ Cache всегда fresh
✅ No cache miss для writes

**Недостатки**:
❌ Higher write latency (cache + DB)
❌ Wasted writes (если data не читается)

**Use cases**: Write-heavy с immediate read

### 4. Write-Behind (Write-Back) Cache

Пишем в cache, асинхронно flush в database.

```javascript
async function updateData(key, value) {
  // 1. Write to cache (fast)
  cache.set(key, value);

  // 2. Queue для async DB write
  queue.enqueue('flush_to_db', { key, value });

  return value;
}

// Background worker
async function flushToDb({ key, value }) {
  await database.update(key, value);
}
```

**Diagram**:
```
App → Write to Cache (fast)
       ↓
    Queue (async)
       ↓
  Database (batch write)
```

**Преимущества**:
✅ Very low write latency
✅ Batch writes (efficiency)

**Недостатки**:
❌ Риск data loss (если cache crash до flush)
❌ Сложность (queue, workers)

**Use cases**: Write-heavy, eventual consistency OK

### 5. Refresh-Ahead

Proactively обновляем cache до expiration.

```javascript
async function getData(key) {
  const data = await cache.get(key);

  if (data) {
    // Check if close to expiration
    const ttl = await cache.ttl(key);
    if (ttl < 300) {
      // Refresh asynchronously
      queue.enqueue('refresh_cache', { key });
    }

    return data;
  }

  // Cache miss
  const fresh = await database.query(key);
  await cache.set(key, fresh, { ttl: 3600 });
  return fresh;
}

// Background worker
async function refreshCache({ key }) {
  const data = await database.query(key);
  await cache.set(key, data, { ttl: 3600 });
}
```

**Преимущества**:
✅ Предотвращает cache misses
✅ Low latency для users

**Недостатки**:
❌ Wasted refreshes (если data не будет read)
❌ Сложность prediction

**Use cases**: Predictable access patterns

## Eviction Policies (Политики вытеснения)

Когда cache заполнен, какие данные удалить?

### 1. LRU (Least Recently Used)

**Принцип**: Удаляем данные, которые дольше всех не использовались.

```javascript
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map();
  }

  get(key) {
    if (!this.cache.has(key)) {
      return null;
    }

    // Move to end (mark as recently used)
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  set(key, value) {
    if (this.cache.has(key)) {
      // Update and move to end
      this.cache.delete(key);
    }

    this.cache.set(key, value);

    if (this.cache.size > this.capacity) {
      // Evict least recently used (first item)
      const lruKey = this.cache.keys().next().value;
      this.cache.delete(lruKey);
    }
  }
}

// Usage
const cache = new LRUCache(3);
cache.set('a', 1);
cache.set('b', 2);
cache.set('c', 3);
cache.get('a'); // Access 'a'
cache.set('d', 4); // Evicts 'b' (least recently used)
```

**Преимущества**:
✅ Интуитивный
✅ Good для general use cases

**Недостатки**:
❌ O(1) требует сложную структуру данных
❌ Doesn't account для frequency

**Use cases**: Default choice для большинства случаев

### 2. LFU (Least Frequently Used)

**Принцип**: Удаляем данные, которые реже всего использовались.

```javascript
class LFUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map(); // key → { value, freq }
    this.freqMap = new Map(); // frequency → Set of keys
    this.minFreq = 0;
  }

  get(key) {
    if (!this.cache.has(key)) {
      return null;
    }

    const entry = this.cache.get(key);
    this._updateFrequency(key, entry);
    return entry.value;
  }

  set(key, value) {
    if (this.capacity === 0) {
      return;
    }

    if (this.cache.has(key)) {
      const entry = this.cache.get(key);
      entry.value = value;
      this._updateFrequency(key, entry);
      return;
    }

    if (this.cache.size >= this.capacity) {
      const keys = this.freqMap.get(this.minFreq);
      const evictKey = keys.values().next().value;
      keys.delete(evictKey);
      if (keys.size === 0) {
        this.freqMap.delete(this.minFreq);
      }
      this.cache.delete(evictKey);
    }

    const entry = { value, freq: 1 };
    this.cache.set(key, entry);

    if (!this.freqMap.has(1)) {
      this.freqMap.set(1, new Set());
    }
    this.freqMap.get(1).add(key);
    this.minFreq = 1;
  }

  _updateFrequency(key, entry) {
    const { freq } = entry;
    const keys = this.freqMap.get(freq);
    keys.delete(key);
    if (keys.size === 0) {
      this.freqMap.delete(freq);
      if (this.minFreq === freq) {
        this.minFreq += 1;
      }
    }

    entry.freq += 1;

    if (!this.freqMap.has(entry.freq)) {
      this.freqMap.set(entry.freq, new Set());
    }
    this.freqMap.get(entry.freq).add(key);
  }
}
```

**Преимущества**:
✅ Keeps frequently accessed data

**Недостатки**:
❌ Сложная реализация
❌ Старые popular items могут блокировать новые

**Use cases**: Workloads с clear popular items

### 3. FIFO (First In First Out)

**Принцип**: Удаляем самые старые данные.

```javascript
class FIFOCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map();
    this.queue = [];
  }

  get(key) {
    return this.cache.has(key) ? this.cache.get(key) : null;
  }

  set(key, value) {
    if (!this.cache.has(key)) {
      if (this.cache.size >= this.capacity) {
        // Evict oldest
        const evictKey = this.queue.shift();
        this.cache.delete(evictKey);
      }

      this.queue.push(key);
    }

    this.cache.set(key, value);
  }
}
```

**Преимущества**:
✅ Простая реализация

**Недостатки**:
❌ Не учитывает access patterns

**Use cases**: Редко используется (LRU лучше)

### 4. Random Replacement

**Принцип**: Удаляем случайный item.

```javascript
class RandomCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map();
  }

  set(key, value) {
    if (this.cache.size >= this.capacity) {
      // Evict random
      const keys = Array.from(this.cache.keys());
      const evictIndex = Math.floor(Math.random() * keys.length);
      const evictKey = keys[evictIndex];
      this.cache.delete(evictKey);
    }

    this.cache.set(key, value);
  }
}
```

**Преимущества**:
✅ O(1), простая

**Недостатки**:
❌ No optimization

**Use cases**: Redis (approximation of LRU для efficiency)

### 5. TTL (Time To Live)

**Принцип**: Данные expiring автоматически.

```javascript
class TTLCache {
  constructor() {
    this.cache = new Map(); // key → { value, expiryTime }
  }

  set(key, value, ttlSeconds) {
    const expiryTime = Date.now() + ttlSeconds * 1000;
    this.cache.set(key, { value, expiryTime });
  }

  get(key) {
    const entry = this.cache.get(key);
    if (!entry) {
      return null;
    }

    const { value, expiryTime } = entry;
    if (Date.now() > expiryTime) {
      // Expired
      this.cache.delete(key);
      return null;
    }

    return value;
  }
}
```

**Преимущества**:
✅ Predictable expiration
✅ No stale data (guaranteed)

**Недостатки**:
❌ Popular items могут expire

**Use cases**: Sessions, temporary data

### Redis Eviction Policies

Redis поддерживает multiple policies:

```
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru

Policies:
- noeviction: Errors при full cache
- allkeys-lru: Evict least recently used (any key)
- volatile-lru: Evict LRU среди keys с TTL
- allkeys-lfu: Evict least frequently used
- volatile-lfu: Evict LFU среди keys с TTL
- allkeys-random: Evict random key
- volatile-random: Evict random среди keys с TTL
- volatile-ttl: Evict soonest to expire
```

## Cache Invalidation

**Phil Karlton**: "There are only two hard things in Computer Science: cache invalidation and naming things."

### Strategies

#### 1. TTL (Time-based)

```javascript
await cache.set('user:123', userData, { ttl: 3600 }); // Expire in 1 hour
```

**Преимущества**: Простота
**Недостатки**: Может быть stale до expiration

#### 2. Explicit Invalidation

```javascript
async function updateUser(userId, data) {
  await db.update(userId, data);
  await cache.delete(`user:${userId}`); // Invalidate
}
```

**Преимущества**: Immediate consistency
**Недостатки**: Нужно помнить invalidate везде

#### 3. Write-Through

```javascript
async function updateUserWriteThrough(userId, data) {
  await db.update(userId, data);
  await cache.set(`user:${userId}`, data); // Update cache
}
```

**Преимущества**: Cache always fresh
**Недостатки**: Wasted writes

#### 4. Event-driven Invalidation

```javascript
// Pub/Sub
async function updateUserEventDriven(userId, data) {
  await db.update(userId, data);
  pubsub.publish('user_updated', { userId });
}

// Subscriber (на всех app servers)
pubsub.subscribe('user_updated', async (message) => {
  await cache.delete(`user:${message.userId}`);
});
```

#### 5. Cache Tags

```javascript
// Tagging
cache.set('user:123', data, { tags: ['users', 'user:123'] });
cache.set('post:456', data, { tags: ['posts', 'user:123'] });

// Invalidate by tag
cache.invalidateTag('user:123'); // Invalidates user и все его posts
```

## Cache Stampede (Thundering Herd)

### Проблема

```
Cache expired для popular key
1000 concurrent requests
↓
All miss cache
↓
1000 queries к database (overload!)
```

### Решения

#### 1. Locking

```javascript
class Mutex {
  constructor() {
    this.locked = false;
    this.queue = [];
  }

  async acquire() {
    if (!this.locked) {
      this.locked = true;
      return;
    }

    return new Promise((resolve) => {
      this.queue.push(resolve);
    }).then(() => {
      this.locked = true;
    });
  }

  release() {
    if (this.queue.length > 0) {
      const next = this.queue.shift();
      next();
    } else {
      this.locked = false;
    }
  }
}

const locks = new Map();

async function getUser(userId) {
  const cacheKey = `user:${userId}`;

  // Check cache
  const cached = await cache.get(cacheKey);
  if (cached) {
    return cached;
  }

  // Acquire lock
  const lockKey = `lock:${cacheKey}`;
  if (!locks.has(lockKey)) {
    locks.set(lockKey, new Mutex());
  }

  const mutex = locks.get(lockKey);
  await mutex.acquire();

  try {
    // Double-check cache (другой worker мог заполнить)
    const cachedAgain = await cache.get(cacheKey);
    if (cachedAgain) {
      return cachedAgain;
    }

    // Query database
    const data = await db.query(userId);
    await cache.set(cacheKey, data, { ttl: 3600 });
    return data;
  } finally {
    mutex.release();
  }
}
```

#### 2. Probabilistic Early Expiration

```javascript
async function getUserWithProbabilisticRefresh(userId) {
  const cacheKey = `user:${userId}`;
  const data = await cache.get(cacheKey);

  if (data) {
    const ttl = await cache.ttl(cacheKey);
    // Вероятность refresh увеличивается near expiration
    if (Math.random() < (3600 - ttl) / 3600) {
      // Refresh асинхронно
      queue.enqueue('refresh_cache', { cacheKey });
    }

    return data;
  }

  // Cache miss
  const fresh = await db.query(userId);
  await cache.set(cacheKey, fresh, { ttl: 3600 });
  return fresh;
}
```

#### 3. Serving Stale While Revalidate

```javascript
async function getUserServingStale(userId) {
  const cacheKey = `user:${userId}`;
  const data = await cache.get(cacheKey);

  if (data) {
    const ttl = await cache.ttl(cacheKey);
    if (ttl < 0) {
      // Expired, но serve stale
      queue.enqueue('refresh_cache', { cacheKey });
      return data; // Stale data
    }

    return data;
  }

  // Cache miss
  const fresh = await db.query(userId);
  await cache.set(cacheKey, fresh, { ttl: 3600 });
  return fresh;
}
```

## Cache Warming

**Preload cache** перед traffic.

```javascript
async function warmCache() {
  // Popular items
  const popularUsers = await db.query('SELECT id FROM users ORDER BY popularity DESC LIMIT 1000');

  for (const user of popularUsers) {
    const data = await db.query(user.id);
    await cache.set(`user:${user.id}`, data, { ttl: 3600 });
  }
}

// Deploy script
(async () => {
  await warmCache();
  // Start serving traffic
})();
```

## Monitoring Cache

### Metrics

```javascript
// Cache hit rate
const hitRate = cacheHits / (cacheHits + cacheMisses);

// Target: > 80%
```

**Other metrics**:
- Cache size (memory usage)
- Eviction rate
- Latency (P50, P95, P99)
- Error rate

### Redis INFO

```bash
redis-cli INFO stats

# Output:
keyspace_hits:1000000
keyspace_misses:100000
# Hit rate = 1000000 / 1100000 = 90.9%
```

### Alerts

```
IF hit_rate < 80% THEN ALERT
IF eviction_rate > 1000/sec THEN ALERT (cache too small)
IF latency P99 > 10ms THEN ALERT
```

## Best Practices

### 1. Cache только expensive operations

```javascript
// Don't cache
const simpleData = { key: 'value' }; // Already fast

// DO cache
const userData = await db.query('SELECT ... JOIN ... WHERE ...'); // Expensive
```

### 2. Set appropriate TTL

```javascript
// Frequently changing: short TTL
await cache.set('stock_price', price, { ttl: 60 }); // 1 min

// Rarely changing: long TTL
await cache.set('user_profile', profile, { ttl: 3600 }); // 1 hour

// Static: very long TTL
await cache.set('product_catalog', catalog, { ttl: 86400 }); // 24 hours
```

### 3. Namespace keys

```javascript
// Good
await cache.set('user:123', data);
await cache.set('post:456', data);

// Bad
await cache.set('123', data); // Ambiguous
```

### 4. Handle cache failures gracefully

```javascript
async function getUserSafe(userId) {
  try {
    const data = await cache.get(`user:${userId}`);
    if (data) {
      return data;
    }
  } catch (error) {
    // Cache down, continue без cache
    console.warn('Cache unavailable', error);
  }

  // Fallback на database
  return db.query(userId);
}
```

### 5. Version cache keys

```javascript
// Schema change → old cache incompatible
// Solution: version в key
await cache.set('user:v2:123', data);
```

## Что почитать дальше

- "Caching at Reddit" — Reddit Engineering Blog
- "Scaling Memcache at Facebook"
- Redis Documentation
- "High Performance MySQL" — Caching chapter

## Проверьте себя

1. В чем разница между Cache-Aside и Read-Through?
2. Когда использовать Write-Through vs Write-Behind?
3. Какая разница между LRU и LFU?
4. Что такое cache stampede и как его предотвратить?
5. Как рассчитать cache hit rate?
6. Что такое TTL и как выбрать правильное значение?
7. Какие стратегии cache invalidation существуют?
8. Как monitoring cache performance?

## Ключевые выводы

- Cache снижает latency в 10-100x и load на БД
- Cache-Aside — наиболее распространенный pattern
- Write-Through — cache always fresh, Write-Behind — low latency
- LRU — default eviction policy для большинства случаев
- TTL — простая invalidation strategy, но может быть stale
- Cache stampede — lock или probabilistic refresh
- Target cache hit rate: 80%+
- Monitor: hit rate, eviction rate, latency
- Namespace keys (user:123, post:456)
- Handle cache failures gracefully (fallback на DB)
- Redis — наиболее популярный distributed cache
- Multi-layer caching: client → CDN → app → distributed cache

---
**Предыдущий урок**: [Blob Storage и CDN](18-blob-storage-and-cdn.md)
**Следующий урок**: [Distributed Caching: Redis и Memcached](20-distributed-caching.md)
