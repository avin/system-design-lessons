# Урок 37: Rate Limiter как система

## Введение

Rate Limiting (ограничение скорости запросов) — критически важный компонент любой production-системы. Он защищает ваши сервисы от перегрузки, DDoS-атак, злоупотреблений API и помогает справедливо распределять ресурсы между пользователями.

В этом уроке мы спроектируем полноценную систему Rate Limiting, которая может работать как:
- Встроенный компонент приложения
- Отдельный сервис (middleware)
- Распределённая система для кластера серверов

**Зачем нужен Rate Limiter:**
- Защита от DDoS и злоупотреблений
- Предотвращение перегрузки бэкенда
- Контроль затрат на внешние API
- Монетизация через тарифные планы (100 req/hour для free, 10000 для premium)
- Справедливое распределение ресурсов

## Требования к системе Rate Limiter

### Функциональные требования

1. **Ограничение запросов** по различным правилам:
   - По IP-адресу
   - По user ID (авторизованные пользователи)
   - По API key
   - По комбинации параметров (IP + endpoint)

2. **Гибкие правила**:
   - Разные лимиты для разных endpoints
   - Разные лимиты для разных тарифных планов
   - Временные окна (per second, per minute, per hour, per day)

3. **Информативные ответы**:
   - HTTP 429 Too Many Requests
   - Заголовки с информацией о лимитах
   - Время до сброса счётчика

### Нефункциональные требования

1. **Низкая латентность**: < 10ms на проверку
2. **Высокая доступность**: 99.99%
3. **Точность**: Допустимая погрешность ~5% (eventual consistency)
4. **Масштабируемость**: Миллионы запросов в секунду
5. **Fault tolerance**: Fail open (при отказе Rate Limiter — пропускаем запросы)

## Алгоритмы Rate Limiting

### 1. Token Bucket (Корзина токенов)

Самый популярный и гибкий алгоритм.

**Принцип:**
- Есть корзина с токенами (фиксированная ёмкость)
- Токены добавляются с постоянной скоростью (refill rate)
- Каждый запрос потребляет 1 токен
- Если токенов нет — запрос отклоняется

```
┌─────────────────────────────────┐
│   Token Bucket (capacity: 10)   │
│                                 │
│   🪙 🪙 🪙 🪙 🪙                │
│   🪙 🪙                         │  ← текущее состояние: 7 токенов
│                                 │
│   Refill: +5 tokens/minute      │
└─────────────────────────────────┘

Запрос приходит → есть токен → ✅ пропустить, удалить токен
Запрос приходит → нет токенов → ❌ отклонить (429)
```

**Реализация:**

```javascript
class TokenBucket {
  constructor(capacity, refillRate, refillInterval = 1000) {
    this.capacity = capacity;           // максимум токенов
    this.tokens = capacity;             // текущее количество
    this.refillRate = refillRate;       // токенов за интервал
    this.refillInterval = refillInterval; // интервал в мс
    this.lastRefill = Date.now();
  }

  refill() {
    const now = Date.now();
    const timePassed = now - this.lastRefill;
    const tokensToAdd = (timePassed / this.refillInterval) * this.refillRate;

    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }

  tryConsume(tokens = 1) {
    this.refill();

    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }

    return false;
  }

  getAvailableTokens() {
    this.refill();
    return Math.floor(this.tokens);
  }
}

// Использование
const bucket = new TokenBucket(10, 5, 60000); // 10 токенов, +5 каждую минуту

if (bucket.tryConsume()) {
  console.log('Request allowed');
} else {
  console.log('Rate limit exceeded');
}
```

**Плюсы:**
- Позволяет короткие всплески (burst) трафика
- Плавное пополнение токенов
- Интуитивно понятен

**Минусы:**
- Требует хранить state (количество токенов и timestamp)

### 2. Leaky Bucket (Дырявое ведро)

Похож на Token Bucket, но работает как очередь FIFO.

**Принцип:**
- Запросы попадают в очередь фиксированного размера
- Запросы обрабатываются с постоянной скоростью (leak rate)
- Если очередь полна — новые запросы отклоняются

```
Входящие запросы ↓↓↓↓↓
         ┌─────────────┐
         │  ⚪ ⚪ ⚪   │
         │  ⚪ ⚪      │  ← очередь (capacity: 5)
         │  ⚪         │
         └──────┬──────┘
                │
                ↓ leak rate: 10 req/sec
         Обработка запросов
```

**Реализация:**

```javascript
class LeakyBucket {
  constructor(capacity, leakRate) {
    this.capacity = capacity;
    this.queue = [];
    this.leakRate = leakRate; // requests per second
    this.lastLeak = Date.now();

    this.startLeaking();
  }

  startLeaking() {
    setInterval(() => {
      const now = Date.now();
      const timePassed = (now - this.lastLeak) / 1000; // seconds
      const requestsToProcess = Math.floor(timePassed * this.leakRate);

      for (let i = 0; i < requestsToProcess && this.queue.length > 0; i++) {
        const request = this.queue.shift();
        this.processRequest(request);
      }

      this.lastLeak = now;
    }, 100); // check every 100ms
  }

  addRequest(request) {
    if (this.queue.length >= this.capacity) {
      return false; // bucket is full
    }

    this.queue.push(request);
    return true;
  }

  processRequest(request) {
    // Handle the request
    console.log('Processing:', request);
  }
}
```

**Плюсы:**
- Гарантирует постоянную скорость обработки
- Защита от всплесков

**Минусы:**
- Требует очередь и фоновую обработку
- Более сложная реализация

### 3. Fixed Window Counter (Фиксированное окно)

Простейший алгоритм: считаем запросы в рамках фиксированного временного окна.

**Принцип:**
- Временная шкала делится на окна (например, по минутам)
- Считаем запросы в текущем окне
- Если превышен лимит — отклоняем

```
Window 1      Window 2      Window 3
10:00-10:01   10:01-10:02   10:02-10:03
┌─────────┐   ┌─────────┐   ┌─────────┐
│ ⚪⚪⚪⚪ │   │ ⚪⚪⚪   │   │ ⚪⚪     │
│ ⚪⚪     │   │ ⚪⚪     │   │         │
│ (6/10)  │   │ (5/10)  │   │ (2/10)  │
└─────────┘   └─────────┘   └─────────┘
```

**Реализация с Redis:**

```javascript
class FixedWindowCounter {
  constructor(redis, limit, windowSize) {
    this.redis = redis;
    this.limit = limit;
    this.windowSize = windowSize; // in seconds
  }

  async tryRequest(key) {
    const now = Date.now();
    const windowKey = this.getWindowKey(key, now);

    const current = await this.redis.incr(windowKey);

    if (current === 1) {
      // First request in this window, set expiration
      await this.redis.expire(windowKey, this.windowSize);
    }

    if (current <= this.limit) {
      return {
        allowed: true,
        remaining: this.limit - current,
        resetAt: this.getWindowEnd(now)
      };
    }

    return {
      allowed: false,
      remaining: 0,
      resetAt: this.getWindowEnd(now)
    };
  }

  getWindowKey(key, timestamp) {
    const windowStart = Math.floor(timestamp / 1000 / this.windowSize) * this.windowSize;
    return `rate_limit:${key}:${windowStart}`;
  }

  getWindowEnd(timestamp) {
    const windowStart = Math.floor(timestamp / 1000 / this.windowSize) * this.windowSize;
    return (windowStart + this.windowSize) * 1000;
  }
}

// Использование
const limiter = new FixedWindowCounter(redis, 100, 60); // 100 req/min

const result = await limiter.tryRequest('user:123');
if (result.allowed) {
  console.log(`Allowed. ${result.remaining} remaining`);
} else {
  console.log(`Rate limited. Reset at ${new Date(result.resetAt)}`);
}
```

**Плюсы:**
- Простая реализация
- Эффективное использование памяти
- Легко понять и debug

**Минусы:**
- **Проблема граничных значений**: в конце одного окна и начале следующего можно отправить 2x запросов

```
Window 1          Window 2
10:00:00-10:01:00 10:01:00-10:02:00
         ↓
   10:00:50        10:01:10
   ⚪⚪⚪⚪⚪⚪⚪⚪⚪⚪  ⚪⚪⚪⚪⚪⚪⚪⚪⚪⚪
   (100 req OK)    (100 req OK)

   За 20 секунд: 200 запросов! (лимит 100/min)
```

### 4. Sliding Window Log (Скользящее окно с логами)

Решение проблемы граничных значений Fixed Window.

**Принцип:**
- Храним timestamp каждого запроса
- Смотрим на окно относительно текущего времени
- Удаляем старые записи

```
Лимит: 5 req/min
Текущее время: 10:05:30

Timestamps в Redis:
10:04:45 ⚪
10:05:10 ⚪
10:05:20 ⚪
10:05:25 ⚪

         ←──── 60 seconds ────→
         10:04:30        10:05:30 (now)
              │              │
              └──────────────┘
              4 запроса в окне → ✅ разрешить (5-й)
```

**Реализация с Redis (Sorted Set):**

```javascript
class SlidingWindowLog {
  constructor(redis, limit, windowSize) {
    this.redis = redis;
    this.limit = limit;
    this.windowSize = windowSize; // in ms
  }

  async tryRequest(key) {
    const now = Date.now();
    const windowStart = now - this.windowSize;
    const redisKey = `rate_limit:log:${key}`;

    // Создаём pipeline для атомарности
    const pipeline = this.redis.pipeline();

    // Удаляем старые записи (за пределами окна)
    pipeline.zremrangebyscore(redisKey, 0, windowStart);

    // Считаем текущие запросы в окне
    pipeline.zcard(redisKey);

    // Добавляем текущий запрос
    pipeline.zadd(redisKey, now, `${now}-${Math.random()}`);

    // Устанавливаем TTL
    pipeline.expire(redisKey, Math.ceil(this.windowSize / 1000) + 1);

    const results = await pipeline.exec();
    const currentCount = results[1][1]; // результат ZCARD

    if (currentCount < this.limit) {
      return {
        allowed: true,
        remaining: this.limit - currentCount - 1,
        resetAt: windowStart + this.windowSize
      };
    } else {
      // Откатываем добавление
      await this.redis.zrem(redisKey, `${now}-${Math.random()}`);

      return {
        allowed: false,
        remaining: 0,
        resetAt: await this.getOldestTimestamp(redisKey) + this.windowSize
      };
    }
  }

  async getOldestTimestamp(key) {
    const oldest = await this.redis.zrange(key, 0, 0, 'WITHSCORES');
    return oldest.length > 0 ? parseInt(oldest[1]) : Date.now();
  }
}

// Использование
const limiter = new SlidingWindowLog(redis, 100, 60000); // 100 req/min
```

**Плюсы:**
- Точный подсчёт, нет проблемы граничных значений
- Справедливое распределение в окне

**Минусы:**
- Большой расход памяти (храним каждый timestamp)
- Не подходит для больших лимитов (10000 req/hour = 10000 записей)

### 5. Sliding Window Counter (Скользящее окно со счётчиком)

Гибрид Fixed Window и Sliding Window — точность + эффективность.

**Принцип:**
- Храним счётчики для текущего и предыдущего окна
- Вычисляем приблизительное количество в скользящем окне на основе пропорции

```
Previous Window   Current Window
10:00-10:01       10:01-10:02
count: 80         count: 30
                         ↑
                    now: 10:01:45 (75% в текущем окне)

Estimated count = 80 × (1 - 0.75) + 30 = 20 + 30 = 50
```

**Реализация:**

```javascript
class SlidingWindowCounter {
  constructor(redis, limit, windowSize) {
    this.redis = redis;
    this.limit = limit;
    this.windowSize = windowSize; // in seconds
  }

  async tryRequest(key) {
    const now = Date.now();
    const currentWindow = Math.floor(now / 1000 / this.windowSize);
    const previousWindow = currentWindow - 1;

    const currentKey = `rate_limit:${key}:${currentWindow}`;
    const previousKey = `rate_limit:${key}:${previousWindow}`;

    // Получаем счётчики
    const [currentCount, previousCount] = await Promise.all([
      this.redis.get(currentKey).then(v => parseInt(v) || 0),
      this.redis.get(previousKey).then(v => parseInt(v) || 0)
    ]);

    // Вычисляем позицию в текущем окне (0.0 - 1.0)
    const windowProgress = ((now / 1000) % this.windowSize) / this.windowSize;

    // Приблизительный счёт в скользящем окне
    const estimatedCount =
      previousCount * (1 - windowProgress) + currentCount;

    if (estimatedCount < this.limit) {
      // Инкрементируем текущее окно
      const pipeline = this.redis.pipeline();
      pipeline.incr(currentKey);
      pipeline.expire(currentKey, this.windowSize * 2);
      await pipeline.exec();

      return {
        allowed: true,
        remaining: Math.floor(this.limit - estimatedCount - 1)
      };
    }

    return {
      allowed: false,
      remaining: 0
    };
  }
}
```

**Плюсы:**
- Баланс между точностью и памятью
- Нет проблемы граничных значений
- Минимальная память (2 счётчика)

**Минусы:**
- Приблизительный подсчёт (погрешность до 5%)

## Сравнение алгоритмов

| Алгоритм | Память | Точность | Всплески | Сложность | Use Case |
|----------|--------|----------|----------|-----------|----------|
| Token Bucket | O(1) | Высокая | ✅ Да | Средняя | API с всплесками трафика |
| Leaky Bucket | O(n) | Высокая | ❌ Нет | Высокая | Сглаживание трафика |
| Fixed Window | O(1) | Низкая | ⚠️ На границах | Низкая | Простые случаи |
| Sliding Log | O(n) | Очень высокая | ✅ Да | Средняя | Малые лимиты, критичная точность |
| Sliding Counter | O(1) | Средняя | ✅ Да | Средняя | **Рекомендуется для большинства** |

## Архитектура распределённого Rate Limiter

### Вариант 1: Централизованный (Redis)

Все application servers обращаются к единому Redis.

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  App     │  │  App     │  │  App     │
│  Server  │  │  Server  │  │  Server  │
│    1     │  │    2     │  │    3     │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┼─────────────┘
                   │
                   ↓
            ┌─────────────┐
            │    Redis    │
            │   Cluster   │
            │  (counters) │
            └─────────────┘
```

**Реализация Express Middleware:**

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

class RedisRateLimiter {
  constructor(options = {}) {
    this.redis = redis;
    this.limit = options.limit || 100;
    this.windowSize = options.windowSize || 60; // seconds
    this.keyPrefix = options.keyPrefix || 'rate_limit';
  }

  async checkLimit(identifier) {
    const now = Date.now();
    const currentWindow = Math.floor(now / 1000 / this.windowSize);
    const previousWindow = currentWindow - 1;

    const currentKey = `${this.keyPrefix}:${identifier}:${currentWindow}`;
    const previousKey = `${this.keyPrefix}:${identifier}:${previousWindow}`;

    const [currentCount, previousCount] = await Promise.all([
      this.redis.get(currentKey).then(v => parseInt(v) || 0),
      this.redis.get(previousKey).then(v => parseInt(v) || 0)
    ]);

    const windowProgress = ((now / 1000) % this.windowSize) / this.windowSize;
    const estimatedCount = previousCount * (1 - windowProgress) + currentCount;

    if (estimatedCount < this.limit) {
      await this.redis
        .pipeline()
        .incr(currentKey)
        .expire(currentKey, this.windowSize * 2)
        .exec();

      return {
        allowed: true,
        limit: this.limit,
        remaining: Math.floor(this.limit - estimatedCount - 1),
        resetAt: (currentWindow + 1) * this.windowSize
      };
    }

    return {
      allowed: false,
      limit: this.limit,
      remaining: 0,
      resetAt: (currentWindow + 1) * this.windowSize
    };
  }
}

// Middleware
function rateLimitMiddleware(options) {
  const limiter = new RedisRateLimiter(options);

  return async (req, res, next) => {
    // Определяем идентификатор (IP, user ID, API key)
    const identifier =
      req.user?.id ||
      req.headers['x-api-key'] ||
      req.ip;

    try {
      const result = await limiter.checkLimit(identifier);

      // Добавляем заголовки
      res.set({
        'X-RateLimit-Limit': result.limit,
        'X-RateLimit-Remaining': result.remaining,
        'X-RateLimit-Reset': result.resetAt
      });

      if (result.allowed) {
        next();
      } else {
        res.status(429).json({
          error: 'Too Many Requests',
          message: `Rate limit exceeded. Try again at ${new Date(result.resetAt * 1000).toISOString()}`,
          retryAfter: result.resetAt - Math.floor(Date.now() / 1000)
        });
      }
    } catch (error) {
      // Fail open: при ошибке Redis — пропускаем запрос
      console.error('Rate limiter error:', error);
      next();
    }
  };
}

// Использование
const express = require('express');
const app = express();

// Глобальный rate limit
app.use(rateLimitMiddleware({
  limit: 1000,
  windowSize: 3600 // 1000 requests per hour
}));

// Специфичный rate limit для API
app.use('/api/expensive', rateLimitMiddleware({
  limit: 10,
  windowSize: 60 // 10 requests per minute
}));

app.get('/api/data', (req, res) => {
  res.json({ data: 'some data' });
});
```

**Плюсы:**
- Точный подсчёт across всех серверов
- Простая синхронизация

**Минусы:**
- Single point of failure (нужен Redis Cluster)
- Дополнительная латентность (network call)
- Bottleneck при высокой нагрузке

### Вариант 2: Локальный кэш + eventual consistency

Каждый сервер имеет локальный кэш, периодически синхронизируется.

```
┌──────────────────┐  ┌──────────────────┐
│  App Server 1    │  │  App Server 2    │
│  ┌────────────┐  │  │  ┌────────────┐  │
│  │ Local Cache│  │  │  │ Local Cache│  │
│  │ (in-memory)│  │  │  │ (in-memory)│  │
│  └──────┬─────┘  │  │  └──────┬─────┘  │
└─────────┼────────┘  └─────────┼────────┘
          │                     │
          └──────────┬──────────┘
                     ↓
              ┌─────────────┐
              │    Redis    │
              │ (aggregator)│
              └─────────────┘
```

**Реализация с Node.js:**

```javascript
class HybridRateLimiter {
  constructor(options) {
    this.redis = new Redis();
    this.localCache = new Map();
    this.limit = options.limit || 100;
    this.windowSize = options.windowSize || 60;
    this.syncInterval = options.syncInterval || 5000; // sync every 5s

    this.startSync();
  }

  startSync() {
    setInterval(async () => {
      await this.syncToRedis();
    }, this.syncInterval);
  }

  async checkLimit(identifier) {
    const local = this.getLocalCount(identifier);

    // Быстрая проверка: если локально уже превышен лимит
    if (local >= this.limit) {
      return { allowed: false, remaining: 0 };
    }

    // Проверяем в Redis для точности
    const redis = await this.getRedisCount(identifier);

    if (redis + local < this.limit) {
      this.incrementLocal(identifier);
      return {
        allowed: true,
        remaining: this.limit - redis - local - 1
      };
    }

    return { allowed: false, remaining: 0 };
  }

  getLocalCount(identifier) {
    const key = `local:${identifier}`;
    return this.localCache.get(key) || 0;
  }

  incrementLocal(identifier) {
    const key = `local:${identifier}`;
    const current = this.localCache.get(key) || 0;
    this.localCache.set(key, current + 1);
  }

  async getRedisCount(identifier) {
    const now = Date.now();
    const currentWindow = Math.floor(now / 1000 / this.windowSize);
    const key = `rate_limit:${identifier}:${currentWindow}`;

    const count = await this.redis.get(key);
    return parseInt(count) || 0;
  }

  async syncToRedis() {
    const now = Date.now();
    const currentWindow = Math.floor(now / 1000 / this.windowSize);
    const pipeline = this.redis.pipeline();

    for (const [localKey, count] of this.localCache.entries()) {
      if (count > 0) {
        const identifier = localKey.replace('local:', '');
        const redisKey = `rate_limit:${identifier}:${currentWindow}`;

        pipeline.incrby(redisKey, count);
        pipeline.expire(redisKey, this.windowSize * 2);
      }
    }

    await pipeline.exec();
    this.localCache.clear();
  }
}
```

**Плюсы:**
- Низкая латентность (проверка in-memory)
- Меньше нагрузки на Redis

**Минусы:**
- Eventual consistency (погрешность ~5-10%)
- Возможны всплески до синхронизации

### Вариант 3: Rate Limiter как отдельный сервис

Dedicated микросервис для Rate Limiting.

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│  Client │  │  Client │  │  Client │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  ↓
         ┌────────────────┐
         │  API Gateway   │
         │  (checks RL)   │
         └───────┬────────┘
                 │
                 ↓
      ┌──────────────────────┐
      │ Rate Limiter Service │
      │   (gRPC/HTTP API)    │
      └──────────┬───────────┘
                 │
                 ↓
          ┌─────────────┐
          │    Redis    │
          └─────────────┘
```

**gRPC API для Rate Limiter:**

```protobuf
// rate_limiter.proto
syntax = "proto3";

service RateLimiter {
  rpc CheckLimit(RateLimitRequest) returns (RateLimitResponse);
}

message RateLimitRequest {
  string identifier = 1;
  string resource = 2;  // endpoint или ресурс
  int32 tokens = 3;     // сколько токенов потребить
}

message RateLimitResponse {
  bool allowed = 1;
  int32 limit = 2;
  int32 remaining = 3;
  int64 reset_at = 4;
}
```

**Реализация сервиса:**

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const Redis = require('ioredis');

const redis = new Redis();

class RateLimiterService {
  async checkLimit(call, callback) {
    const { identifier, resource, tokens } = call.request;

    const limiter = new SlidingWindowCounter(redis, 100, 60);
    const result = await limiter.tryRequest(`${resource}:${identifier}`);

    callback(null, {
      allowed: result.allowed,
      limit: result.limit || 100,
      remaining: result.remaining || 0,
      resetAt: result.resetAt || 0
    });
  }
}

// Запуск gRPC сервера
const packageDefinition = protoLoader.loadSync('rate_limiter.proto');
const proto = grpc.loadPackageDefinition(packageDefinition);

const server = new grpc.Server();
server.addService(proto.RateLimiter.service, new RateLimiterService());
server.bindAsync(
  '0.0.0.0:50051',
  grpc.ServerCredentials.createInsecure(),
  () => {
    console.log('Rate Limiter Service running on port 50051');
    server.start();
  }
);
```

## Продвинутые фичи

### 1. Динамические правила

```javascript
class DynamicRateLimiter {
  constructor(redis) {
    this.redis = redis;
    this.rulesCache = new Map();
  }

  async getRules(identifier) {
    // Проверяем кэш
    if (this.rulesCache.has(identifier)) {
      return this.rulesCache.get(identifier);
    }

    // Загружаем из БД/конфига
    const user = await db.query(
      'SELECT tier, custom_limit FROM users WHERE id = $1',
      [identifier]
    );

    const rules = {
      'free': { limit: 100, windowSize: 3600 },
      'pro': { limit: 1000, windowSize: 3600 },
      'enterprise': { limit: 10000, windowSize: 3600 }
    };

    const tier = user.tier || 'free';
    const config = user.custom_limit || rules[tier];

    this.rulesCache.set(identifier, config);
    setTimeout(() => this.rulesCache.delete(identifier), 60000); // cache 1min

    return config;
  }

  async checkLimit(identifier) {
    const rules = await this.getRules(identifier);
    const limiter = new SlidingWindowCounter(
      this.redis,
      rules.limit,
      rules.windowSize
    );

    return limiter.tryRequest(identifier);
  }
}
```

### 2. Distributed Rate Limiting с Consistent Hashing

Для масштабирования Redis используем шардирование.

```javascript
const ConsistentHashing = require('consistent-hashing');

class ShardedRateLimiter {
  constructor(redisNodes) {
    this.ring = new ConsistentHashing(redisNodes);
    this.clients = new Map();

    redisNodes.forEach(node => {
      this.clients.set(node, new Redis({ host: node }));
    });
  }

  getRedisClient(key) {
    const node = this.ring.getNode(key);
    return this.clients.get(node);
  }

  async checkLimit(identifier) {
    const redis = this.getRedisClient(identifier);
    const limiter = new SlidingWindowCounter(redis, 100, 60);

    return limiter.tryRequest(identifier);
  }
}

// Использование
const limiter = new ShardedRateLimiter([
  'redis-1.example.com',
  'redis-2.example.com',
  'redis-3.example.com'
]);
```

### 3. Adaptive Rate Limiting

Автоматическая подстройка лимитов на основе нагрузки системы.

```javascript
class AdaptiveRateLimiter {
  constructor(redis, baseLimit) {
    this.redis = redis;
    this.baseLimit = baseLimit;
    this.currentLimit = baseLimit;

    this.monitorSystemLoad();
  }

  async monitorSystemLoad() {
    setInterval(async () => {
      const metrics = await this.getSystemMetrics();

      // CPU > 80% → снижаем лимит на 50%
      if (metrics.cpu > 80) {
        this.currentLimit = Math.floor(this.baseLimit * 0.5);
      }
      // Error rate > 5% → снижаем на 30%
      else if (metrics.errorRate > 0.05) {
        this.currentLimit = Math.floor(this.baseLimit * 0.7);
      }
      // Всё хорошо → восстанавливаем
      else {
        this.currentLimit = this.baseLimit;
      }

      console.log(`Adaptive limit adjusted to: ${this.currentLimit}`);
    }, 10000); // check every 10s
  }

  async getSystemMetrics() {
    const os = require('os');
    const cpuUsage = os.loadavg()[0] / os.cpus().length * 100;

    // Error rate из метрик
    const errorRate = await this.redis.get('metrics:error_rate')
      .then(v => parseFloat(v) || 0);

    return { cpu: cpuUsage, errorRate };
  }

  async checkLimit(identifier) {
    const limiter = new SlidingWindowCounter(
      this.redis,
      this.currentLimit,
      60
    );

    return limiter.tryRequest(identifier);
  }
}
```

## Мониторинг и алерты

### Метрики для отслеживания

```javascript
const prometheus = require('prom-client');

const rateLimitCounter = new prometheus.Counter({
  name: 'rate_limit_requests_total',
  help: 'Total number of rate limit checks',
  labelNames: ['identifier_type', 'resource', 'result']
});

const rateLimitHistogram = new prometheus.Histogram({
  name: 'rate_limit_check_duration_seconds',
  help: 'Rate limit check duration',
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1]
});

class MonitoredRateLimiter {
  async checkLimit(identifier, resource) {
    const start = Date.now();

    try {
      const result = await this.limiter.checkLimit(identifier);

      rateLimitCounter.inc({
        identifier_type: this.getIdentifierType(identifier),
        resource,
        result: result.allowed ? 'allowed' : 'rejected'
      });

      rateLimitHistogram.observe((Date.now() - start) / 1000);

      return result;
    } catch (error) {
      rateLimitCounter.inc({
        identifier_type: this.getIdentifierType(identifier),
        resource,
        result: 'error'
      });

      throw error;
    }
  }

  getIdentifierType(identifier) {
    if (identifier.startsWith('user:')) return 'user';
    if (identifier.startsWith('api_key:')) return 'api_key';
    return 'ip';
  }
}
```

### Grafana Dashboard queries

```promql
# Rate limit rejection rate
rate(rate_limit_requests_total{result="rejected"}[5m])
/ rate(rate_limit_requests_total[5m])

# Top 10 rate limited users
topk(10,
  rate(rate_limit_requests_total{result="rejected"}[5m])
)

# P99 latency of rate limiter
histogram_quantile(0.99,
  rate(rate_limit_check_duration_seconds_bucket[5m])
)
```

## Best Practices

### 1. Fail Open vs Fail Closed

```javascript
async function rateLimitMiddleware(req, res, next) {
  try {
    const result = await limiter.checkLimit(req.ip);

    if (result.allowed) {
      next();
    } else {
      res.status(429).json({ error: 'Too Many Requests' });
    }
  } catch (error) {
    console.error('Rate limiter error:', error);

    // FAIL OPEN: пропускаем при ошибке (рекомендуется)
    next();

    // FAIL CLOSED: блокируем при ошибке (для критичных endpoints)
    // res.status(503).json({ error: 'Service Unavailable' });
  }
}
```

### 2. Информативные заголовки

Стандарт RFC 6585:

```javascript
res.set({
  'X-RateLimit-Limit': '100',
  'X-RateLimit-Remaining': '42',
  'X-RateLimit-Reset': '1640000000',
  'Retry-After': '60' // seconds
});
```

### 3. Graceful degradation

```javascript
class GracefulRateLimiter {
  constructor(redis, options) {
    this.redis = redis;
    this.fallbackLimiter = new InMemoryRateLimiter(options);
    this.redisHealthy = true;

    this.monitorRedis();
  }

  monitorRedis() {
    setInterval(async () => {
      try {
        await this.redis.ping();
        this.redisHealthy = true;
      } catch (error) {
        this.redisHealthy = false;
        console.error('Redis unhealthy, using in-memory fallback');
      }
    }, 5000);
  }

  async checkLimit(identifier) {
    if (this.redisHealthy) {
      try {
        return await this.checkRedis(identifier);
      } catch (error) {
        this.redisHealthy = false;
        return this.fallbackLimiter.checkLimit(identifier);
      }
    } else {
      return this.fallbackLimiter.checkLimit(identifier);
    }
  }
}
```

## Что читать дальше

- **Урок 38**: Распределённое хранилище ключ-значение — deep dive в Redis и его альтернативы
- **Урок 53**: Monitoring и Observability — как отслеживать Rate Limiter в production
- **Урок 40**: Web Crawler — использование rate limiting для внешних API

## Проверь себя

1. Какой алгоритм rate limiting лучше использовать для API с всплесками трафика?
2. В чём проблема Fixed Window Counter на границах окон?
3. Как реализовать rate limiting для распределённой системы из 100 серверов?
4. Что такое "fail open" и когда его использовать?
5. Как посчитать, сколько памяти потребуется Redis для 1M пользователей с лимитом 1000 req/hour (Sliding Window Counter)?

---

[← Урок 36: Pastebin](36-pastebin.md) | [Урок 38: Распределённое хранилище ключ-значение →](38-distributed-kv-store.md)
