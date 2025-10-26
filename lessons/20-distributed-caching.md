# Distributed Caching: Redis и Memcached

## Что такое Distributed Cache?

**Distributed Cache** — cache, разделенный между несколькими серверами, доступный всем application servers.

```
Without distributed cache:
[App Server 1] → [Local Cache 1]
[App Server 2] → [Local Cache 2]
[App Server 3] → [Local Cache 3]

Problem: Cache не shared, дублирование данных

With distributed cache:
[App Server 1] ↘
[App Server 2] → [Redis/Memcached Cluster]
[App Server 3] ↗

Benefit: Shared cache, consistent data
```

## Redis vs Memcached

### Сравнительная таблица

| Характеристика | Redis | Memcached |
|----------------|-------|-----------|
| **Data structures** | Rich (strings, lists, sets, hashes, sorted sets) | Strings only |
| **Persistence** | Yes (RDB, AOF) | No |
| **Replication** | Yes (master-slave) | No (client-side sharding) |
| **Transactions** | Yes (MULTI/EXEC) | No |
| **Pub/Sub** | Yes | No |
| **Lua scripting** | Yes | No |
| **Multi-threading** | Single-threaded (6.0+ optional I/O threads) | Multi-threaded |
| **Max value size** | 512MB | 1MB |
| **Use case** | General-purpose cache + data structures | Simple key-value cache |

### Когда использовать Redis

✅ Нужны rich data structures
✅ Нужна persistence
✅ Pub/Sub messaging
✅ Leaderboards (sorted sets)
✅ Session store с TTL
✅ Distributed locking

### Когда использовать Memcached

✅ Simple caching (key-value only)
✅ Multi-threading важен (throughput)
✅ Pure cache (no persistence needed)
✅ Простота важнее features

**Рекомендация**: Redis для большинства случаев (richer features).

## Redis

### Установка

```bash
# Docker
docker run --name redis -p 6379:6379 -d redis

# Ubuntu
sudo apt-get install redis-server

# Start
redis-server

# CLI
redis-cli
```

### Базовые операции

```bash
# Strings
SET user:123:name "John Doe"
GET user:123:name  # "John Doe"

# With expiration (TTL)
SETEX session:abc123 3600 "user_data"  # Expire in 1 hour
TTL session:abc123  # Remaining seconds

# Increment (atomic)
SET counter 0
INCR counter  # 1
INCRBY counter 5  # 6

# Check existence
EXISTS user:123:name  # 1 (true)

# Delete
DEL user:123:name
```

### Data Structures

#### 1. Strings

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Set/Get
r.set('key', 'value')
value = r.get('key')

# With TTL
r.setex('key', 3600, 'value')  # Expire in 1 hour

# Atomic operations
r.set('counter', 0)
r.incr('counter')  # 1
r.incrby('counter', 5)  # 6
r.decr('counter')  # 5
```

**Use cases**: Counters, flags, caching simple values

#### 2. Hashes

```python
# User profile
r.hset('user:123', mapping={
    'name': 'John Doe',
    'email': 'john@example.com',
    'age': 30
})

# Get single field
name = r.hget('user:123', 'name')  # 'John Doe'

# Get all fields
user = r.hgetall('user:123')
# {'name': 'John Doe', 'email': 'john@example.com', 'age': '30'}

# Increment field
r.hincrby('user:123', 'age', 1)  # 31
```

**Use cases**: Objects, user profiles, session data

#### 3. Lists

```python
# Queue (FIFO)
r.rpush('queue', 'task1', 'task2', 'task3')  # Right push
task = r.lpop('queue')  # Left pop → 'task1'

# Stack (LIFO)
r.rpush('stack', 'item1', 'item2')
item = r.rpop('stack')  # Right pop → 'item2'

# Recent items
r.lpush('recent:user:123', 'post1', 'post2', 'post3')
recent = r.lrange('recent:user:123', 0, 9)  # First 10
r.ltrim('recent:user:123', 0, 99)  # Keep only 100
```

**Use cases**: Queues, activity feeds, recent items

#### 4. Sets

```python
# Add members
r.sadd('tags:post:123', 'python', 'redis', 'caching')

# Check membership
r.sismember('tags:post:123', 'python')  # True

# Get all members
tags = r.smembers('tags:post:123')  # {'python', 'redis', 'caching'}

# Set operations
r.sadd('set1', 'a', 'b', 'c')
r.sadd('set2', 'b', 'c', 'd')

r.sinter('set1', 'set2')  # Intersection: {'b', 'c'}
r.sunion('set1', 'set2')  # Union: {'a', 'b', 'c', 'd'}
r.sdiff('set1', 'set2')   # Difference: {'a'}
```

**Use cases**: Tags, unique items, relationship graphs

#### 5. Sorted Sets (ZSets)

```python
# Leaderboard
r.zadd('leaderboard', {
    'player1': 100,
    'player2': 200,
    'player3': 150
})

# Get rank
rank = r.zrank('leaderboard', 'player2')  # 2 (0-indexed, highest score)

# Top 10
top10 = r.zrevrange('leaderboard', 0, 9, withscores=True)
# [('player2', 200.0), ('player3', 150.0), ('player1', 100.0)]

# Increment score
r.zincrby('leaderboard', 50, 'player1')  # 150

# Get by score range
r.zrangebyscore('leaderboard', 100, 200)
```

**Use cases**: Leaderboards, priority queues, time-series data

### Advanced Features

#### Transactions

```python
# MULTI/EXEC (atomic)
pipe = r.pipeline()
pipe.set('key1', 'value1')
pipe.incr('counter')
pipe.get('key1')
results = pipe.execute()  # All or nothing
```

#### Pub/Sub

```python
# Publisher
r.publish('notifications', 'New message!')

# Subscriber
pubsub = r.pubsub()
pubsub.subscribe('notifications')

for message in pubsub.listen():
    if message['type'] == 'message':
        print(message['data'])
```

**Use case**: Real-time notifications, chat, event broadcasting

#### Lua Scripting

```python
# Atomic rate limiting
lua_script = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end

if current > limit then
    return 0
else
    return 1
end
"""

rate_limit = r.register_script(lua_script)
allowed = rate_limit(keys=['rate:user:123'], args=[100, 60])  # 100 req/min
```

**Use case**: Atomic complex operations

#### Pipelining

```python
# Batch multiple commands
pipe = r.pipeline()
for i in range(1000):
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()

# Reduces round-trips: 1 instead of 1000
```

### Redis Persistence

#### RDB (Redis Database Backup)

```
# redis.conf
save 900 1    # Save if 1 key changed in 15 min
save 300 10   # Save if 10 keys changed in 5 min
save 60 10000 # Save if 10k keys changed in 1 min
```

**Преимущества**: Fast recovery, compact
**Недостатки**: Potential data loss (last snapshot to crash)

#### AOF (Append-Only File)

```
# redis.conf
appendonly yes
appendfsync everysec  # Sync every second
```

**Преимущества**: Minimal data loss (1 second max)
**Недостатки**: Larger files, slower recovery

**Hybrid**: RDB + AOF (best of both)

### Redis Replication

```
# Slave config
replicaof master-ip master-port

# Redis 5.0+: replica instead of slave
```

**Master-Slave setup**:
```
[Master] (writes)
   ↓
[Replica 1, 2, 3] (reads)
```

**High Availability**: Redis Sentinel

```bash
# Sentinel config
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
```

Sentinel monitors master, auto-failover при падении.

### Redis Cluster

**Sharding** для horizontal scaling.

```
Cluster (16384 hash slots):
Node 1: slots 0-5460
Node 2: slots 5461-10922
Node 3: slots 10923-16383

Key "user:123" → hash → slot 8457 → Node 2
```

**Setup**:
```bash
redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1
```

**Client**:
```python
from rediscluster import RedisCluster

startup_nodes = [
    {"host": "127.0.0.1", "port": "7000"},
    {"host": "127.0.0.1", "port": "7001"},
    {"host": "127.0.0.1", "port": "7002"}
]

rc = RedisCluster(startup_nodes=startup_nodes, decode_responses=True)
rc.set('key', 'value')  # Автоматически роутится на правильный node
```

## Memcached

### Установка

```bash
# Ubuntu
sudo apt-get install memcached

# Docker
docker run --name memcached -p 11211:11211 -d memcached

# Start
memcached -m 64 -p 11211 -u memcache -l 127.0.0.1
```

### Базовые операции

```python
import memcache

mc = memcache.Client(['127.0.0.1:11211'], debug=0)

# Set
mc.set('key', 'value', time=3600)  # Expire in 1 hour

# Get
value = mc.get('key')

# Delete
mc.delete('key')

# Increment
mc.set('counter', 0)
mc.incr('counter', 1)  # 1
mc.decr('counter', 1)  # 0

# Multi-get (efficient)
values = mc.get_multi(['key1', 'key2', 'key3'])
```

### Client-side Sharding

Memcached не имеет built-in clustering → client handles sharding.

```python
# Consistent hashing client
from pylibmc import Client

mc = Client(
    ['server1:11211', 'server2:11211', 'server3:11211'],
    binary=True,
    behaviors={'ketama': True}  # Consistent hashing
)

mc.set('key', 'value')  # Client роутит на правильный server
```

## Практические паттерны

### 1. Session Store

**Redis**:
```python
import json

def create_session(user_id):
    session_id = str(uuid.uuid4())
    session_data = {
        'user_id': user_id,
        'created_at': time.time()
    }

    # TTL = 24 hours
    r.setex(f"session:{session_id}", 86400, json.dumps(session_data))
    return session_id

def get_session(session_id):
    data = r.get(f"session:{session_id}")
    return json.loads(data) if data else None

def delete_session(session_id):
    r.delete(f"session:{session_id}")
```

### 2. Rate Limiting

```python
def is_rate_limited(user_id, limit=100, window=60):
    key = f"rate:{user_id}:{int(time.time() / window)}"

    current = r.incr(key)
    if current == 1:
        r.expire(key, window)

    return current > limit

# Usage
if is_rate_limited(user_id):
    return {"error": "Rate limit exceeded"}, 429
```

### 3. Leaderboard

```python
def add_score(user_id, score):
    r.zadd('leaderboard', {user_id: score})

def get_leaderboard(limit=10):
    return r.zrevrange('leaderboard', 0, limit - 1, withscores=True)

def get_rank(user_id):
    rank = r.zrevrank('leaderboard', user_id)
    return rank + 1 if rank is not None else None

def get_score(user_id):
    return r.zscore('leaderboard', user_id)
```

### 4. Cache-Aside Pattern

```python
def get_user(user_id):
    cache_key = f"user:{user_id}"

    # Try cache
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss → query DB
    user = db.query("SELECT * FROM users WHERE id = ?", (user_id,))
    if user:
        r.setex(cache_key, 3600, json.dumps(user))

    return user

def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = ?", (user_id,))

    # Invalidate cache
    r.delete(f"user:{user_id}")
```

### 5. Distributed Lock

```python
import time
import uuid

def acquire_lock(lock_name, timeout=10):
    lock_id = str(uuid.uuid4())
    lock_key = f"lock:{lock_name}"

    # SET NX (only if not exists) with expiration
    acquired = r.set(lock_key, lock_id, nx=True, ex=timeout)
    return lock_id if acquired else None

def release_lock(lock_name, lock_id):
    lock_key = f"lock:{lock_name}"

    # Lua script для atomic check-and-delete
    lua_script = """
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
    else
        return 0
    end
    """

    script = r.register_script(lua_script)
    return script(keys=[lock_key], args=[lock_id])

# Usage
lock_id = acquire_lock('resource:123')
if lock_id:
    try:
        # Critical section
        process_resource()
    finally:
        release_lock('resource:123', lock_id)
```

### 6. Queue (с Redis Lists)

```python
# Producer
def enqueue(queue_name, task):
    r.rpush(queue_name, json.dumps(task))

# Consumer (blocking)
def dequeue(queue_name, timeout=0):
    result = r.blpop(queue_name, timeout=timeout)
    if result:
        queue, data = result
        return json.loads(data)
    return None

# Usage
enqueue('tasks', {'type': 'email', 'to': 'user@example.com'})

# Worker
while True:
    task = dequeue('tasks', timeout=5)
    if task:
        process_task(task)
```

## Performance Optimization

### 1. Pipelining

```python
# Bad: 1000 round-trips
for i in range(1000):
    r.set(f'key:{i}', f'value:{i}')

# Good: 1 round-trip
pipe = r.pipeline()
for i in range(1000):
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()
```

### 2. Connection Pooling

```python
# Connection pool (reuse connections)
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=50,
    decode_responses=True
)

r = redis.Redis(connection_pool=pool)
```

### 3. Compression

```python
import gzip
import json

def set_compressed(key, data, ttl=3600):
    json_data = json.dumps(data)
    compressed = gzip.compress(json_data.encode())
    r.setex(key, ttl, compressed)

def get_compressed(key):
    compressed = r.get(key)
    if compressed:
        json_data = gzip.decompress(compressed).decode()
        return json.loads(json_data)
    return None
```

### 4. Hash Tags (Redis Cluster)

```python
# Ensure related keys on same slot
# {user:123} → all with this tag on same slot
r.set('{user:123}:profile', profile_data)
r.set('{user:123}:settings', settings_data)

# Can use multi-key operations
pipe = r.pipeline()
pipe.get('{user:123}:profile')
pipe.get('{user:123}:settings')
pipe.execute()
```

## Monitoring

### Redis INFO

```bash
redis-cli INFO

# Sections:
# Server: version, uptime
# Clients: connected_clients
# Memory: used_memory, used_memory_peak
# Stats: total_commands_processed, keyspace_hits/misses
# Replication: role, connected_slaves
```

### Key Metrics

```python
info = r.info()

# Memory
used_memory = info['used_memory_human']
maxmemory = info['maxmemory_human']

# Hit rate
hits = info['keyspace_hits']
misses = info['keyspace_misses']
hit_rate = hits / (hits + misses) if (hits + misses) > 0 else 0

# Connections
connected_clients = info['connected_clients']

# Commands/sec
total_commands = info['total_commands_processed']
uptime_seconds = info['uptime_in_seconds']
commands_per_sec = total_commands / uptime_seconds
```

### Slow Log

```bash
# Enable slow log
CONFIG SET slowlog-log-slower-than 10000  # 10ms

# View slow queries
SLOWLOG GET 10
```

### Monitoring Tools

- **Redis Commander** (GUI)
- **RedisInsight** (official GUI)
- **Prometheus + Redis Exporter**
- **Datadog, New Relic** (APM)

## Best Practices

### 1. Key Naming Convention

```
# Pattern: {namespace}:{id}:{field}
user:123:profile
user:123:settings
post:456:likes
session:abc123
```

### 2. Avoid Large Keys

```python
# Bad: huge hash
r.hset('all_users', mapping={...})  # Millions of fields

# Good: separate keys
r.set('user:123', data)
r.set('user:456', data)
```

### 3. Set Expiration

```python
# Always set TTL (avoid memory leak)
r.setex('key', 3600, 'value')  # Expire in 1 hour
```

### 4. Use Appropriate Data Structures

```python
# Counters → String with INCR
# Objects → Hash
# Collections → Set or List
# Leaderboards → Sorted Set
```

### 5. Monitor Memory

```bash
# Eviction policy
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru
```

### 6. Secure Redis

```bash
# Bind to localhost
bind 127.0.0.1

# Require password
requirepass strongpassword

# Disable dangerous commands
rename-command FLUSHDB ""
rename-command FLUSHALL ""
```

## Что почитать дальше

- [Redis Documentation](https://redis.io/documentation)
- "Redis in Action" by Josiah Carlson
- [Memcached Documentation](https://memcached.org/)
- "Scaling Memcache at Facebook" paper

## Проверьте себя

1. В чем главная разница между Redis и Memcached?
2. Какие data structures поддерживает Redis?
3. Когда использовать RDB vs AOF persistence?
4. Что такое Redis Sentinel и зачем он нужен?
5. Как работает Redis Cluster sharding?
6. Что такое pipelining и зачем он нужен?
7. Как реализовать distributed lock с Redis?
8. Как мониторить Redis performance?

## Ключевые выводы

- Redis — rich data structures, persistence, pub/sub, scripting
- Memcached — simple key-value, multi-threaded, pure cache
- Redis для большинства use cases (больше features)
- Data structures: Strings, Hashes, Lists, Sets, Sorted Sets
- Persistence: RDB (snapshots) vs AOF (append-only log)
- Replication: Master-Slave для reads, Sentinel для HA
- Redis Cluster: Sharding для horizontal scaling (16384 slots)
- Pipelining снижает latency (batch commands)
- Connection pooling критичен для production
- Monitor: memory usage, hit rate, slow queries
- Security: bind to localhost, requirepass, rename dangerous commands
- Use cases: cache, session store, rate limiting, leaderboards, queues, locks

---

**Предыдущий урок**: [Кэширование: стратегии и политики вытеснения](19-keshirovanie-strategii.md)
**Следующий урок**: [Message Queues: RabbitMQ, SQS](21-message-queues.md)
