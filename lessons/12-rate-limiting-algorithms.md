# Rate Limiting и алгоритмы подсчёта

## Что такое Rate Limiting?

**Rate Limiting** — ограничение количества requests, которые клиент может сделать за определенное время.

**Пример**: "Максимум 100 requests в минуту с одного IP".

```
Request 1-100: ✅ Allowed
Request 101:   ❌ 429 Too Many Requests
```

## Зачем нужен Rate Limiting?

### 1. Защита от DDoS атак

```
Атакующий отправляет 1,000,000 requests/sec
  ↓
Rate Limiter блокирует после 100 req/sec
  ↓
Сервер защищен
```

### 2. Предотвращение abuse

```
Пользователь пытается bruteforce пароль
  ↓
Rate Limiter: max 5 login attempts в минуту
  ↓
Атака невозможна
```

### 3. Контроль costs

```
API с платными вызовами (например, SMS API)
  ↓
Rate Limiter предотвращает случайное превышение бюджета
```

### 4. Справедливое распределение ресурсов

```
100 пользователей
Без rate limiting: 1 пользователь может занять все ресурсы
С rate limiting: каждый получает fair share
```

### 5. Монетизация через API tiers

```
Free tier:    100 req/hour
Basic tier:   1,000 req/hour
Premium tier: 10,000 req/hour
```

## Алгоритмы Rate Limiting

### 1. Token Bucket

**Самый популярный алгоритм**.

#### Как работает

Представьте ведро с токенами:
- Ведро имеет capacity (например, 100 токенов)
- Каждый request забирает 1 токен
- Токены пополняются с постоянной скоростью (например, 10 токенов/сек)
- Если токенов нет → request отклоняется

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/sec

t=0s:  100 tokens
Request → 99 tokens
Request → 98 tokens
...
Request → 0 tokens
Request → ❌ DENIED (no tokens)

t=1s: Refilled +10 → 10 tokens
```

#### Реализация

```javascript
class TokenBucket {
  constructor({ capacity, refillRate }) {
    this.capacity = capacity;
    this.tokens = capacity;
    this.refillRate = refillRate; // tokens per second
    this.lastRefill = Date.now() / 1000;
  }

  allowRequest() {
    const now = Date.now() / 1000;
    const elapsed = now - this.lastRefill;
    const tokensToAdd = elapsed * this.refillRate;

    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;

    if (this.tokens >= 1) {
      this.tokens -= 1;
      return true;
    }

    return false;
  }
}

// Usage
const bucket = new TokenBucket({ capacity: 100, refillRate: 10 });

if (bucket.allowRequest()) {
  console.log('Request allowed');
} else {
  console.log('Rate limit exceeded');
}
```

#### Преимущества

✅ Позволяет bursts (до capacity)
✅ Smooth rate limiting
✅ Память: O(1) на пользователя

#### Недостатки

❌ Может разрешить большой burst если bucket полон

#### Когда использовать

- API rate limiting
- Network traffic shaping
- Нужно разрешить короткие bursts

### 2. Leaky Bucket

**Как вода вытекает из ведра с дыркой**.

#### Как работает

- Requests добавляются в queue (ведро)
- Requests обрабатываются с постоянной скоростью (leak rate)
- Если queue полон → request отклоняется

```
Queue capacity: 10 requests
Processing rate: 2 req/sec

Requests arrive:
[R1, R2, R3, R4, R5]
  ↓ (queue)
Process 2 req/sec
  ↓
[R1, R2] processed at t=1s
[R3, R4] processed at t=2s
[R5] processed at t=3s
```


#### Реализация

```javascript
class LeakyBucket {
  constructor({ capacity, leakRate }) {
    this.capacity = capacity;
    this.queue = [];
    this.leakRate = leakRate; // requests per second
    this.lastLeak = Date.now() / 1000;
  }

  allowRequest() {
    const now = Date.now() / 1000;
    const elapsed = now - this.lastLeak;
    const leaks = Math.floor(elapsed * this.leakRate);

    for (let i = 0; i < leaks && this.queue.length > 0; i += 1) {
      this.queue.shift();
    }

    this.lastLeak = now;

    if (this.queue.length < this.capacity) {
      this.queue.push(now);
      return true;
    }

    return false;
  }
}

// Usage
const bucket = new LeakyBucket({ capacity: 10, leakRate: 2 });
```

#### Преимущества

✅ Smooth constant rate
✅ Защита от bursts

#### Недостатки

❌ Не позволяет bursts
❌ Может добавить latency (queueing)

#### Когда использовать

- Нужен строгий constant rate
- Network traffic shaping
- Защита от burst traffic

### 3. Fixed Window Counter

**Простейший алгоритм**.

#### Как работает

Делим время на фиксированные окна (например, минуты):

```
Window 1 (10:00:00 - 10:00:59): Counter
Window 2 (10:01:00 - 10:01:59): Counter
...

Limit: 100 req/min

10:00:05 → Request 1  (counter = 1)
10:00:10 → Request 2  (counter = 2)
...
10:00:59 → Request 100 (counter = 100)
10:01:00 → Request 101 (counter = 1, new window!)
```

#### Реализация

```javascript
class FixedWindowCounter {
  constructor({ limit, windowSize }) {
    this.limit = limit;
    this.windowSize = windowSize; // seconds
    this.windows = new Map();
  }

  allowRequest(userId) {
    const now = Math.floor(Date.now() / 1000);
    const window = Math.floor(now / this.windowSize);
    const key = `${userId}:${window}`;
    const currentCount = this.windows.get(key) ?? 0;

    if (currentCount < this.limit) {
      this.windows.set(key, currentCount + 1);
      return true;
    }

    return false;
  }
}

// Usage
const limiter = new FixedWindowCounter({ limit: 100, windowSize: 60 });

if (limiter.allowRequest('user_123')) {
  console.log('Allowed');
} else {
  console.log('Rate limited');
}
```

#### Проблема: Burst на границе окон

```
10:00:30 - 10:00:59: 100 requests (allowed)
10:01:00 - 10:01:29: 100 requests (allowed)

За 59 секунд: 200 requests! (2x limit)
```

#### Преимущества

✅ Очень простая реализация
✅ Memory efficient

#### Недостатки

❌ Burst на границе окон (до 2x limit)
❌ Неравномерное распределение

#### Когда использовать

- Простота критична
- Точность не критична
- Logs, analytics

### 4. Sliding Window Log

**Фиксит проблему Fixed Window**.

#### Как работает

Хранит timestamp каждого request и считает requests в последнюю минуту:

```
Limit: 100 req/min
Current time: 10:01:00

Request log:
[10:00:05, 10:00:10, 10:00:30, ..., 10:00:59]
  ↓
Filter: только requests за последние 60 секунд
  ↓
Count: 95 requests

New request → 96 < 100 → ✅ Allowed
```


#### Реализация

```javascript
class SlidingWindowLog {
  constructor({ limit, windowSize }) {
    this.limit = limit;
    this.windowSize = windowSize; // seconds
    this.logs = new Map();
  }

  allowRequest(userId) {
    const now = Date.now() / 1000;
    const windowStart = now - this.windowSize;
    const timestamps = (this.logs.get(userId) ?? []).filter(
      (timestamp) => timestamp > windowStart,
    );

    if (timestamps.length < this.limit) {
      timestamps.push(now);
      this.logs.set(userId, timestamps);
      return true;
    }

    this.logs.set(userId, timestamps);
    return false;
  }
}

// Usage
const limiter = new SlidingWindowLog({ limit: 100, windowSize: 60 });
```


#### Преимущества

✅ Точный rate limiting
✅ Нет burst проблемы

#### Недостатки

❌ Memory intensive: O(N) где N = количество requests
❌ Медленнее (нужно фильтровать список)

#### Когда использовать

- Точность критична
- Количество requests умеренное
- Можно позволить больше памяти

### 5. Sliding Window Counter

**Гибрид Fixed Window + Sliding Window Log**.

#### Как работает

Использует счетчики для окон, но аппроксимирует скользящее окно:

```
Current window: 10:01:00 - 10:01:59
Previous window: 10:00:00 - 10:00:59

Current time: 10:01:15 (15 секунд в текущем окне)

Weighted count:
  Previous window: 60 requests × (45/60) = 45
  Current window:  20 requests × (15/60) = 5
  Total: 50 requests

Limit: 100 → ✅ Allowed
```

#### Реализация

```javascript
class SlidingWindowCounter {
  constructor({ limit, windowSize }) {
    this.limit = limit;
    this.windowSize = windowSize;
    this.windows = new Map();
  }

  allowRequest(userId) {
    const now = Date.now() / 1000;
    const currentWindow = Math.floor(now / this.windowSize);
    const previousWindow = currentWindow - 1;
    const windowPosition = (now % this.windowSize) / this.windowSize;

    const userWindows = this.windows.get(userId) ?? new Map();
    const previousCount = userWindows.get(previousWindow) ?? 0;
    const currentCount = userWindows.get(currentWindow) ?? 0;

    const weightedCount = previousCount * (1 - windowPosition) + currentCount;

    if (weightedCount < this.limit) {
      userWindows.set(currentWindow, currentCount + 1);
      this.windows.set(userId, userWindows);
      return true;
    }

    return false;
  }
}

// Usage
const limiter = new SlidingWindowCounter({ limit: 100, windowSize: 60 });
```

#### Преимущества

✅ Точнее чем Fixed Window
✅ Memory efficient (только 2 окна)
✅ Хороший compromise

#### Недостатки

⚠️ Approximation (не 100% точно)

#### Когда использовать

- **Рекомендуется для большинства случаев**
- Good balance между точностью и performance

### Сравнение алгоритмов

| Алгоритм | Memory | Accuracy | Bursts | Сложность |
|----------|--------|----------|--------|-----------|
| Token Bucket | O(1) | Хорошая | ✅ Да | Низкая |
| Leaky Bucket | O(N) | Отличная | ❌ Нет | Средняя |
| Fixed Window | O(1) | Плохая | ⚠️ 2x на границе | Очень низкая |
| Sliding Log | O(N) | Отличная | ❌ Нет | Средняя |
| Sliding Counter | O(1) | Хорошая | ⚠️ Небольшие | Средняя |

**Рекомендация**: Используйте **Sliding Window Counter** для большинства случаев.

## Распределенный Rate Limiting

### Проблема

С одним сервером легко:
```javascript
const counter = new Map(); // In-memory
```

С несколькими серверами:
```
[S1] counter = {user_123: 50}
[S2] counter = {user_123: 45}
[S3] counter = {user_123: 30}

Total: 125 requests (превышен limit 100!)
```

### Решение 1: Redis Centralized Counter

**Простейшее решение**:

```javascript
const Redis = require('ioredis');

const redis = new Redis({ host: 'localhost', port: 6379 });

async function allowRequest(userId, { limit = 100, window = 60 } = {}) {
  const key = `rate_limit:${userId}`;
  const current = await redis.incr(key);

  if (current === 1) {
    await redis.expire(key, window);
  }

  return current <= limit;
}

(async () => {
  if (await allowRequest('user_123')) {
    console.log('Allowed');
  } else {
    console.log('Rate limited');
  }
})();
```

**Проблема**: Race condition между INCR и EXPIRE.

**Лучше — Lua script (atomic)**:
```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('incr', key)

if current == 1 then
    redis.call('expire', key, window)
end

if current > limit then
    return 0
else
    return 1
end
```

```javascript
const Redis = require('ioredis');

const redis = new Redis();

const luaScript = `
-- (lua-скрипт как выше)
`;

redis.defineCommand('rateLimit', {
  numberOfKeys: 1,
  lua: luaScript,
});

async function allowRequest(userId, { limit = 100, window = 60 } = {}) {
  const result = await redis.rateLimit(`rate_limit:${userId}`, limit, window);
  return Number(result) === 1;
}
```

### Решение 2: Redis Sliding Window (Sorted Set)

```javascript
const Redis = require('ioredis');

const redis = new Redis();

async function allowRequest(userId, { limit = 100, window = 60 } = {}) {
  const key = `rate_limit:${userId}`;
  const now = Date.now() / 1000;
  const windowStart = now - window;

  await redis.zremrangebyscore(key, 0, windowStart);

  const count = await redis.zcard(key);

  if (count < limit) {
    await redis.zadd(key, now, String(now));
    await redis.expire(key, window);
    return true;
  }

  return false;
}
```

**Lua script для atomicity**:
```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local window_start = now - window

-- Remove old entries
redis.call('zremrangebyscore', key, 0, window_start)

-- Count current requests
local count = redis.call('zcard', key)

if count < limit then
    redis.call('zadd', key, now, now)
    redis.call('expire', key, window)
    return 1
else
    return 0
end
```

### Решение 3: Token Bucket в Redis

```javascript
const Redis = require('ioredis');

const redis = new Redis();

const tokenBucketScript = `
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local bucket = redis.call('hmget', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

local elapsed = now - last_refill
local tokens_to_add = elapsed * refill_rate
tokens = math.min(capacity, tokens + tokens_to_add)

if tokens >= 1 then
    tokens = tokens - 1
    redis.call('hmset', key, 'tokens', tokens, 'last_refill', now)
    redis.call('expire', key, 3600)
    return 1
else
    return 0
end
`;

redis.defineCommand('consumeToken', {
  numberOfKeys: 1,
  lua: tokenBucketScript,
});

async function allowRequest(userId, { capacity = 100, refillRate = 10 } = {}) {
  const now = Math.floor(Date.now() / 1000);
  const result = await redis.consumeToken(
    `rate_limit:${userId}`,
    capacity,
    refillRate,
    now,
  );

  return Number(result) === 1;
}
```

### Performance Considerations

**Latency добавляется**:
- Local counter: ~1µs
- Redis (same datacenter): ~1ms
- Redis (cross-region): ~50-100ms

**Optimization**: Local cache + eventual consistency

```javascript
class HybridRateLimiter {
  constructor(redisClient, { limit, window }) {
    this.redis = redisClient;
    this.limit = limit;
    this.window = window;
    this.localCache = new Map();
  }

  async allowRequest(userId) {
    const now = Date.now() / 1000;
    const cache =
      this.localCache.get(userId) ?? { count: 0, reset: now + this.window };

    if (now > cache.reset) {
      cache.count = 0;
      cache.reset = now + this.window;
    }

    cache.count += 1;
    this.localCache.set(userId, cache);

    if (cache.count > this.limit * 0.9) {
      const key = `rate_limit:${userId}`;
      const count = await this.redis.incr(key);

      if (count === 1) {
        await this.redis.expire(key, this.window);
      }

      if (count > this.limit) {
        return false;
      }
    }

    return cache.count <= this.limit;
  }
}

const hybridLimiter = new HybridRateLimiter(redis, { limit: 100, window: 60 });
```

## HTTP Response Headers

### Standard Headers

**Response при rate limit**:
```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1609459200
Retry-After: 60
Content-Type: application/json

{
  "error": "Rate limit exceeded",
  "message": "Maximum 100 requests per minute allowed"
}
```

**Headers**:
- `X-RateLimit-Limit`: максимум requests
- `X-RateLimit-Remaining`: сколько осталось
- `X-RateLimit-Reset`: timestamp когда сбросится
- `Retry-After`: через сколько секунд можно retry

**GitHub API example**:
```bash
$ curl -I https://api.github.com/users/octocat

HTTP/2 200
x-ratelimit-limit: 60
x-ratelimit-remaining: 58
x-ratelimit-reset: 1609459200
```

### Middleware Implementation

**Express.js**:
```javascript
const redis = require('redis');
const client = redis.createClient();

async function rateLimitMiddleware(req, res, next) {
  const userId = req.user.id || req.ip;
  const key = `rate_limit:${userId}`;
  const limit = 100;
  const window = 60;

  const count = await client.incr(key);

  if (count === 1) {
    await client.expire(key, window);
  }

  res.set({
    'X-RateLimit-Limit': limit,
    'X-RateLimit-Remaining': Math.max(0, limit - count),
    'X-RateLimit-Reset': Math.floor(Date.now() / 1000) + window
  });

  if (count > limit) {
    res.set('Retry-After', window);
    return res.status(429).json({
      error: 'Too Many Requests',
      message: `Rate limit of ${limit} requests per ${window}s exceeded`
    });
  }

  next();
}

app.use('/api/', rateLimitMiddleware);
```

## Rate Limiting Strategies

### 1. Per User

```javascript
const key = `rate_limit:user:${userId}`;
```

**Use case**: Authenticated API

### 2. Per IP

```javascript
const key = `rate_limit:ip:${ipAddress}`;
```

**Use case**: Public endpoints, login pages

**Проблема**: Пользователи за NAT имеют один IP.

### 3. Per API Key

```javascript
const key = `rate_limit:apikey:${apiKey}`;
```

**Use case**: Third-party API access

### 4. Per Endpoint

```javascript
const key = `rate_limit:${userId}:${endpoint}`;

// Example:
// rate_limit:user_123:/api/search  → 10 req/min
// rate_limit:user_123:/api/profile → 100 req/min
```

**Use case**: Разные limits для разных endpoints

### 5. Global

```javascript
const key = 'rate_limit:global';
```

**Use case**: Защита от общего overload

### 6. Tiered (по тарифному плану)

```javascript
function getLimit(user) {
  if (user.plan === 'free') {
    return 100; // req/hour
  }
  if (user.plan === 'basic') {
    return 1000;
  }
  if (user.plan === 'premium') {
    return 10000;
  }
  return 10; // default
}
```

## Advanced Techniques

### 1. Adaptive Rate Limiting

Динамическое изменение limits на основе нагрузки:

```javascript
const os = require('os');

function getDynamicLimit() {
  const cpuLoad = (os.loadavg()[0] / os.cpus().length) * 100;

  if (cpuLoad > 80) {
    return 50; // Reduce limit under high load
  }
  if (cpuLoad > 60) {
    return 75;
  }
  return 100; // Normal limit
}
```

### 2. Rate Limiting по cost

Разные requests имеют разный "вес":

```javascript
function calculateCost(request) {
  if (request.endpoint === '/search') {
    return 10; // Expensive
  }
  if (request.endpoint === '/profile') {
    return 1; // Cheap
  }
  return 5; // Medium
}

// Token bucket с cost
if (bucket.tokens >= cost) {
  bucket.tokens -= cost;
  return true;
}
```

### 3. Throttling (замедление вместо блокировки)

```javascript
async function throttleRequest(userId, { limit = 100, window = 60 } = {}) {
  const count = await getRequestCount(userId, window);

  if (count > limit) {
    const delay = Math.min((count - limit) * 100, 5000); // max 5 sec
    await new Promise((resolve) => setTimeout(resolve, delay));
  }

  return true; // Всегда allow, но с delay
}
```

### 4. Whitelist / Blacklist

```javascript
const WHITELISTED_IPS = new Set(['203.0.113.0', '198.51.100.0']);
const BLACKLISTED_IPS = new Set(['192.0.2.0']);

function allowRequest(ip, userId) {
  if (BLACKLISTED_IPS.has(ip)) {
    return false;
  }

  if (WHITELISTED_IPS.has(ip)) {
    return true; // No rate limit
  }

  // Normal rate limiting
  return checkRateLimit(userId);
}
```

## Практические рекомендации

### 1. Выбор алгоритма

```
Simple API: Fixed Window (простота)
Most cases: Sliding Window Counter (balance)
Strict rate: Token Bucket
Prevent bursts: Leaky Bucket
High precision: Sliding Window Log
```

### 2. Выбор window size

```
Login endpoint: 5 requests per 15 minutes
Search endpoint: 100 requests per hour
Standard API: 1000 requests per hour
Heavy API: 10000 requests per day
```

### 3. Graceful degradation

```javascript
async function handleRequest() {
  try {
    if (!(await rateLimiter.allowRequest(userId))) {
      return cachedResponseIfAvailable();
    }
  } catch (error) {
    if (error.message.includes('ECONNREFUSED')) {
      // Fallback если Redis недоступен
      return allowRequestLocal();
    }
    throw error;
  }
}
```

### 4. Monitoring

Метрики:
- Rate limit hits (сколько requests blocked)
- Top rate-limited users
- Rate limit headroom (насколько близки к limit)
- False positives (legitimate users blocked)

## Популярные библиотеки

### 1. Node.js: rate-limiter-flexible

```javascript
const { RateLimiterRedis } = require('rate-limiter-flexible');
const Redis = require('ioredis');

const redisClient = new Redis();

const rateLimiter = new RateLimiterRedis({
  storeClient: redisClient,
  points: 100,
  duration: 60,
});

app.use(async (req, res, next) => {
  try {
    await rateLimiter.consume(req.ip);
    next();
  } catch (error) {
    res.status(429).send('Too Many Requests');
  }
});
```

### 2. Node.js: express-rate-limit

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  standardHeaders: true,
  legacyHeaders: false,
});

app.use('/api/', limiter);
```

### 3. Redis: redis-cell (Token Bucket)

```bash
# Redis module
redis-cli CL.THROTTLE user123 15 30 60 1
# → [allowed?, current_limit, remaining, retry_after, reset_after]
```

```javascript
const Redis = require('ioredis');

const redisCellClient = new Redis();

async function allowRequest(userId) {
  // CL.THROTTLE key max_burst count_per_period period [quantity]
  const [allowed] = await redisCellClient.call(
    'CL.THROTTLE',
    `user:${userId}`,
    15, // max_burst
    30, // count_per_period
    60, // period (seconds)
    1, // quantity (default 1)
  );

  return allowed === 0;
}
```

## Что почитать дальше

- "Rate Limiting Algorithms" by Cloudflare
- [NGINX Rate Limiting](https://www.nginx.com/blog/rate-limiting-nginx/)
- [Redis Rate Limiting Patterns](https://redis.io/topics/ratelimiting)
- [GitHub API Rate Limiting](https://docs.github.com/en/rest/overview/resources-in-the-rest-api#rate-limiting)

## Проверьте себя

1. В чем разница между Token Bucket и Leaky Bucket?
2. Какая проблема у Fixed Window Counter?
3. Почему Sliding Window Counter — good compromise?
4. Как реализовать distributed rate limiting с Redis?
5. Какие HTTP headers используются для rate limiting?
6. Когда стоит использовать rate limiting per user vs per IP?
7. Что такое adaptive rate limiting?
8. Как обрабатывать race conditions в distributed rate limiting?

## Ключевые выводы

- Rate limiting защищает от DDoS, abuse, и обеспечивает fair usage
- Token Bucket — популярный, позволяет bursts
- Sliding Window Counter — рекомендуется для большинства случаев (good balance)
- Fixed Window имеет burst проблему на границах окон
- Distributed rate limiting требует Redis или другое shared storage
- Lua scripts в Redis обеспечивают atomicity
- Всегда возвращайте правильные HTTP headers (X-RateLimit-*, Retry-After)
- Мониторьте rate limit hits и false positives
- Рассмотрите tiered limits (free/premium) для монетизации
- Graceful degradation при недоступности rate limiter

---

**Предыдущий урок**: [API Gateway и Reverse Proxy](11-api-gateway-i-reverse-proxy.md)
**Следующий урок**: [SQL vs NoSQL](13-sql-vs-nosql.md)
