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

```python
def update_data(key, value):
    # 1. Write to cache (fast)
    cache.set(key, value)

    # 2. Queue для async DB write
    queue.enqueue('flush_to_db', key, value)

    return value

# Background worker
def flush_to_db(key, value):
    database.update(key, value)
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

```python
def get_data(key):
    data = cache.get(key)

    if data:
        # Check if close to expiration
        ttl = cache.ttl(key)
        if ttl < 300:  # < 5 minutes
            # Refresh asynchronously
            queue.enqueue('refresh_cache', key)

        return data

    # Cache miss
    data = database.query(key)
    cache.set(key, data, ttl=3600)
    return data

# Background worker
def refresh_cache(key):
    data = database.query(key)
    cache.set(key, data, ttl=3600)
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

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key):
        if key not in self.cache:
            return None

        # Move to end (mark as recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def set(self, key, value):
        if key in self.cache:
            # Update and move to end
            self.cache.move_to_end(key)

        self.cache[key] = value

        if len(self.cache) > self.capacity:
            # Evict least recently used (first item)
            self.cache.popitem(last=False)

# Usage
cache = LRUCache(capacity=3)
cache.set('a', 1)
cache.set('b', 2)
cache.set('c', 3)
cache.get('a')  # Access 'a'
cache.set('d', 4)  # Evicts 'b' (least recently used)
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

```python
from collections import defaultdict
import heapq

class LFUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key → (value, frequency)
        self.freq_map = defaultdict(set)  # frequency → keys
        self.min_freq = 0

    def get(self, key):
        if key not in self.cache:
            return None

        value, freq = self.cache[key]

        # Remove from current frequency
        self.freq_map[freq].remove(key)
        if not self.freq_map[freq] and freq == self.min_freq:
            self.min_freq += 1

        # Increment frequency
        self.cache[key] = (value, freq + 1)
        self.freq_map[freq + 1].add(key)

        return value

    def set(self, key, value):
        if self.capacity == 0:
            return

        if key in self.cache:
            # Update value and increment frequency
            _, freq = self.cache[key]
            self.cache[key] = (value, freq)
            self.get(key)  # Increment freq
            return

        if len(self.cache) >= self.capacity:
            # Evict least frequently used
            evict_key = self.freq_map[self.min_freq].pop()
            del self.cache[evict_key]

        # Add new key
        self.cache[key] = (value, 1)
        self.freq_map[1].add(key)
        self.min_freq = 1
```

**Преимущества**:
✅ Keeps frequently accessed data

**Недостатки**:
❌ Сложная реализация
❌ Старые popular items могут блокировать новые

**Use cases**: Workloads с clear popular items

### 3. FIFO (First In First Out)

**Принцип**: Удаляем самые старые данные.

```python
from collections import deque

class FIFOCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}
        self.queue = deque()

    def get(self, key):
        return self.cache.get(key)

    def set(self, key, value):
        if key not in self.cache:
            if len(self.cache) >= self.capacity:
                # Evict oldest
                evict_key = self.queue.popleft()
                del self.cache[evict_key]

            self.queue.append(key)

        self.cache[key] = value
```

**Преимущества**:
✅ Простая реализация

**Недостатки**:
❌ Не учитывает access patterns

**Use cases**: Редко используется (LRU лучше)

### 4. Random Replacement

**Принцип**: Удаляем случайный item.

```python
import random

class RandomCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}

    def set(self, key, value):
        if len(self.cache) >= self.capacity:
            # Evict random
            evict_key = random.choice(list(self.cache.keys()))
            del self.cache[evict_key]

        self.cache[key] = value
```

**Преимущества**:
✅ O(1), простая

**Недостатки**:
❌ No optimization

**Use cases**: Redis (approximation of LRU для efficiency)

### 5. TTL (Time To Live)

**Принцип**: Данные expiring автоматически.

```python
import time

class TTLCache:
    def __init__(self):
        self.cache = {}  # key → (value, expiry_time)

    def set(self, key, value, ttl):
        expiry = time.time() + ttl
        self.cache[key] = (value, expiry)

    def get(self, key):
        if key not in self.cache:
            return None

        value, expiry = self.cache[key]

        if time.time() > expiry:
            # Expired
            del self.cache[key]
            return None

        return value
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

```python
cache.setex('user:123', 3600, user_data)  # Expire in 1 hour
```

**Преимущества**: Простота
**Недостатки**: Может быть stale до expiration

#### 2. Explicit Invalidation

```python
def update_user(user_id, data):
    db.update(user_id, data)
    cache.delete(f"user:{user_id}")  # Invalidate
```

**Преимущества**: Immediate consistency
**Недостатки**: Нужно помнить invalidate везде

#### 3. Write-Through

```python
def update_user(user_id, data):
    db.update(user_id, data)
    cache.set(f"user:{user_id}", data)  # Update cache
```

**Преимущества**: Cache always fresh
**Недостатки**: Wasted writes

#### 4. Event-driven Invalidation

```python
# Pub/Sub
def update_user(user_id, data):
    db.update(user_id, data)
    pubsub.publish('user_updated', {'user_id': user_id})

# Subscriber (на всех app servers)
pubsub.subscribe('user_updated', lambda msg: cache.delete(f"user:{msg['user_id']}"))
```

#### 5. Cache Tags

```python
# Tagging
cache.set('user:123', data, tags=['users', 'user:123'])
cache.set('post:456', data, tags=['posts', 'user:123'])

# Invalidate by tag
cache.invalidate_tag('user:123')  # Invalidates user и все его posts
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

```python
import threading

locks = {}

def get_user(user_id):
    cache_key = f"user:{user_id}"

    # Check cache
    data = cache.get(cache_key)
    if data:
        return data

    # Acquire lock
    lock_key = f"lock:{cache_key}"
    if lock_key not in locks:
        locks[lock_key] = threading.Lock()

    with locks[lock_key]:
        # Double-check cache (другой thread мог заполнить)
        data = cache.get(cache_key)
        if data:
            return data

        # Query database
        data = db.query(user_id)
        cache.set(cache_key, data, ttl=3600)
        return data
```

#### 2. Probabilistic Early Expiration

```python
import random

def get_user(user_id):
    cache_key = f"user:{user_id}"
    data = cache.get(cache_key)

    if data:
        ttl = cache.ttl(cache_key)
        # Вероятность refresh увеличивается near expiration
        if random.random() < (3600 - ttl) / 3600:
            # Refresh асинхронно
            queue.enqueue('refresh_cache', cache_key)

        return data

    # Cache miss
    data = db.query(user_id)
    cache.set(cache_key, data, ttl=3600)
    return data
```

#### 3. Serving Stale While Revalidate

```python
def get_user(user_id):
    cache_key = f"user:{user_id}"
    data = cache.get(cache_key)

    if data:
        ttl = cache.ttl(cache_key)
        if ttl < 0:
            # Expired, но serve stale
            queue.enqueue('refresh_cache', cache_key)
            return data  # Stale data

        return data

    # Cache miss
    data = db.query(user_id)
    cache.set(cache_key, data, ttl=3600)
    return data
```

## Cache Warming

**Preload cache** перед traffic.

```python
def warm_cache():
    # Popular items
    popular_users = db.query("SELECT id FROM users ORDER BY popularity DESC LIMIT 1000")

    for user in popular_users:
        data = db.query(user.id)
        cache.set(f"user:{user.id}", data, ttl=3600)

# Deploy script
warm_cache()
# Start serving traffic
```

## Monitoring Cache

### Metrics

```python
# Cache hit rate
hit_rate = cache_hits / (cache_hits + cache_misses)

# Target: > 80%
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

```python
# Don't cache
simple_data = {"key": "value"}  # Already fast

# DO cache
user_data = db.query("SELECT ... JOIN ... WHERE ...")  # Expensive
```

### 2. Set appropriate TTL

```python
# Frequently changing: short TTL
cache.setex('stock_price', 60, price)  # 1 min

# Rarely changing: long TTL
cache.setex('user_profile', 3600, profile)  # 1 hour

# Static: very long TTL
cache.setex('product_catalog', 86400, catalog)  # 24 hours
```

### 3. Namespace keys

```python
# Good
cache.set('user:123', data)
cache.set('post:456', data)

# Bad
cache.set('123', data)  # Ambiguous
```

### 4. Handle cache failures gracefully

```python
def get_user(user_id):
    try:
        data = cache.get(f"user:{user_id}")
        if data:
            return data
    except CacheError:
        # Cache down, continue without it
        pass

    # Fallback на database
    return db.query(user_id)
```

### 5. Version cache keys

```python
# Schema change → old cache incompatible
# Solution: version в key
cache.set('user:v2:123', data)
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

**Предыдущий урок**: [Blob Storage и CDN](18-blob-storage-i-cdn.md)
**Следующий урок**: [Distributed Caching: Redis и Memcached](20-distributed-caching.md)
