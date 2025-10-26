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

```python
import time

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.time()

    def allow_request(self):
        # Пополнить токены
        now = time.time()
        elapsed = now - self.last_refill
        tokens_to_add = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now

        # Проверить доступность токена
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        else:
            return False

# Usage
bucket = TokenBucket(capacity=100, refill_rate=10)

if bucket.allow_request():
    print("Request allowed")
else:
    print("Rate limit exceeded")
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

```python
from collections import deque
import time

class LeakyBucket:
    def __init__(self, capacity, leak_rate):
        self.capacity = capacity
        self.queue = deque()
        self.leak_rate = leak_rate  # requests per second
        self.last_leak = time.time()

    def allow_request(self):
        # Leak (process) requests
        now = time.time()
        elapsed = now - self.last_leak
        leaks = int(elapsed * self.leak_rate)

        for _ in range(min(leaks, len(self.queue))):
            self.queue.popleft()

        self.last_leak = now

        # Try to add new request
        if len(self.queue) < self.capacity:
            self.queue.append(now)
            return True
        else:
            return False

# Usage
bucket = LeakyBucket(capacity=10, leak_rate=2)
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

```python
import time
from collections import defaultdict

class FixedWindowCounter:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size  # seconds
        self.windows = defaultdict(int)

    def allow_request(self, user_id):
        now = time.time()
        window = int(now / self.window_size)

        key = f"{user_id}:{window}"

        if self.windows[key] < self.limit:
            self.windows[key] += 1
            return True
        else:
            return False

# Usage
limiter = FixedWindowCounter(limit=100, window_size=60)

if limiter.allow_request("user_123"):
    print("Allowed")
else:
    print("Rate limited")
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

```python
import time
from collections import defaultdict

class SlidingWindowLog:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size  # seconds
        self.logs = defaultdict(list)

    def allow_request(self, user_id):
        now = time.time()
        window_start = now - self.window_size

        # Remove old requests
        self.logs[user_id] = [
            timestamp for timestamp in self.logs[user_id]
            if timestamp > window_start
        ]

        # Check limit
        if len(self.logs[user_id]) < self.limit:
            self.logs[user_id].append(now)
            return True
        else:
            return False

# Usage
limiter = SlidingWindowLog(limit=100, window_size=60)
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

```python
import time
from collections import defaultdict

class SlidingWindowCounter:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.windows = defaultdict(lambda: defaultdict(int))

    def allow_request(self, user_id):
        now = time.time()
        current_window = int(now / self.window_size)
        previous_window = current_window - 1

        # Position in current window (0.0 to 1.0)
        window_position = (now % self.window_size) / self.window_size

        # Weighted count
        previous_count = self.windows[user_id].get(previous_window, 0)
        current_count = self.windows[user_id].get(current_window, 0)

        weighted_count = (
            previous_count * (1 - window_position) +
            current_count
        )

        if weighted_count < self.limit:
            self.windows[user_id][current_window] += 1
            return True
        else:
            return False

# Usage
limiter = SlidingWindowCounter(limit=100, window_size=60)
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
```python
counter = {}  # In-memory
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

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379)

def allow_request(user_id, limit=100, window=60):
    key = f"rate_limit:{user_id}"
    current = r.incr(key)

    if current == 1:
        r.expire(key, window)

    return current <= limit

# Usage
if allow_request("user_123"):
    print("Allowed")
else:
    print("Rate limited")
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

```python
import redis

r = redis.Redis()

lua_script = """
... (script выше)
"""

rate_limit_script = r.register_script(lua_script)

def allow_request(user_id, limit=100, window=60):
    result = rate_limit_script(
        keys=[f"rate_limit:{user_id}"],
        args=[limit, window]
    )
    return result == 1
```

### Решение 2: Redis Sliding Window (Sorted Set)

```python
import redis
import time

r = redis.Redis()

def allow_request(user_id, limit=100, window=60):
    key = f"rate_limit:{user_id}"
    now = time.time()
    window_start = now - window

    # Remove old entries
    r.zremrangebyscore(key, 0, window_start)

    # Count requests in window
    count = r.zcard(key)

    if count < limit:
        # Add new request
        r.zadd(key, {str(now): now})
        r.expire(key, window)
        return True
    else:
        return False
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

```python
import redis
import time

r = redis.Redis()

def allow_request(user_id, capacity=100, refill_rate=10):
    key = f"rate_limit:{user_id}"

    lua_script = """
    local key = KEYS[1]
    local capacity = tonumber(ARGV[1])
    local refill_rate = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])

    local bucket = redis.call('hmget', key, 'tokens', 'last_refill')
    local tokens = tonumber(bucket[1]) or capacity
    local last_refill = tonumber(bucket[2]) or now

    -- Refill tokens
    local elapsed = now - last_refill
    local tokens_to_add = elapsed * refill_rate
    tokens = math.min(capacity, tokens + tokens_to_add)

    -- Check availability
    if tokens >= 1 then
        tokens = tokens - 1
        redis.call('hmset', key, 'tokens', tokens, 'last_refill', now)
        redis.call('expire', key, 3600)
        return 1
    else
        return 0
    end
    """

    script = r.register_script(lua_script)
    result = script(
        keys=[key],
        args=[capacity, refill_rate, time.time()]
    )

    return result == 1
```

### Performance Considerations

**Latency добавляется**:
- Local counter: ~1µs
- Redis (same datacenter): ~1ms
- Redis (cross-region): ~50-100ms

**Optimization**: Local cache + eventual consistency

```python
import redis
import time
from collections import defaultdict

class HybridRateLimiter:
    def __init__(self, redis_client, limit, window):
        self.redis = redis_client
        self.limit = limit
        self.window = window
        self.local_cache = defaultdict(lambda: {'count': 0, 'reset': time.time()})

    def allow_request(self, user_id):
        now = time.time()

        # Check local cache first (fast path)
        cache = self.local_cache[user_id]
        if now > cache['reset']:
            cache['count'] = 0
            cache['reset'] = now + self.window

        cache['count'] += 1

        # If close to limit, check Redis (authoritative)
        if cache['count'] > self.limit * 0.9:  # 90% of limit
            key = f"rate_limit:{user_id}"
            count = self.redis.incr(key)
            if count == 1:
                self.redis.expire(key, self.window)

            if count > self.limit:
                return False

        return cache['count'] <= self.limit
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

```python
key = f"rate_limit:user:{user_id}"
```

**Use case**: Authenticated API

### 2. Per IP

```python
key = f"rate_limit:ip:{ip_address}"
```

**Use case**: Public endpoints, login pages

**Проблема**: Пользователи за NAT имеют один IP.

### 3. Per API Key

```python
key = f"rate_limit:apikey:{api_key}"
```

**Use case**: Third-party API access

### 4. Per Endpoint

```python
key = f"rate_limit:{user_id}:{endpoint}"

# Example:
# rate_limit:user_123:/api/search  → 10 req/min
# rate_limit:user_123:/api/profile → 100 req/min
```

**Use case**: Разные limits для разных endpoints

### 5. Global

```python
key = "rate_limit:global"
```

**Use case**: Защита от общего overload

### 6. Tiered (по тарифному плану)

```python
def get_limit(user):
    if user.plan == 'free':
        return 100  # req/hour
    elif user.plan == 'basic':
        return 1000
    elif user.plan == 'premium':
        return 10000
    else:
        return 10  # default
```

## Advanced Techniques

### 1. Adaptive Rate Limiting

Динамическое изменение limits на основе нагрузки:

```python
def get_dynamic_limit():
    cpu_usage = psutil.cpu_percent()

    if cpu_usage > 80:
        return 50  # Reduce limit under high load
    elif cpu_usage > 60:
        return 75
    else:
        return 100  # Normal limit
```

### 2. Rate Limiting по cost

Разные requests имеют разный "вес":

```python
def calculate_cost(request):
    if request.endpoint == '/search':
        return 10  # Expensive
    elif request.endpoint == '/profile':
        return 1   # Cheap
    else:
        return 5   # Medium

# Token bucket с cost
if bucket.tokens >= cost:
    bucket.tokens -= cost
    return True
```

### 3. Throttling (замедление вместо блокировки)

```python
def throttle_request(user_id, limit=100, window=60):
    count = get_request_count(user_id, window)

    if count > limit:
        # Добавляем задержку
        delay = min((count - limit) * 0.1, 5)  # max 5 sec
        time.sleep(delay)

    return True  # Всегда allow, но с delay
```

### 4. Whitelist / Blacklist

```python
WHITELISTED_IPS = {'203.0.113.0', '198.51.100.0'}
BLACKLISTED_IPS = {'192.0.2.0'}

def allow_request(ip, user_id):
    if ip in BLACKLISTED_IPS:
        return False

    if ip in WHITELISTED_IPS:
        return True  # No rate limit

    # Normal rate limiting
    return check_rate_limit(user_id)
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

```python
def handle_request():
    try:
        if not rate_limiter.allow_request(user_id):
            return cached_response_if_available()
    except RedisConnectionError:
        # Fallback если Redis недоступен
        return allow_request_local()
```

### 4. Monitoring

Метрики:
- Rate limit hits (сколько requests blocked)
- Top rate-limited users
- Rate limit headroom (насколько близки к limit)
- False positives (legitimate users blocked)

## Популярные библиотеки

### 1. Python: django-ratelimit

```python
from django_ratelimit.decorators import ratelimit

@ratelimit(key='ip', rate='100/h')
def my_view(request):
    return HttpResponse("Success")
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

```python
import redis

r = redis.Redis()

def allow_request(user_id):
    # CL.THROTTLE key max_burst count_per_period period [quantity]
    result = r.execute_command(
        'CL.THROTTLE', f'user:{user_id}',
        15,  # max_burst
        30,  # count_per_period
        60,  # period (seconds)
        1    # quantity (default 1)
    )

    allowed = result[0] == 0
    return allowed
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
