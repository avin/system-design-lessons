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

```javascript
const Redis = require('ioredis');

const redis = new Redis({
  host: 'localhost',
  port: 6379,
});

async function demoStrings() {
  // Set/Get
  await redis.set('key', 'value');
  const value = await redis.get('key');
  console.log({ value });

  // With TTL
  await redis.setex('key', 3600, 'value'); // Expire in 1 hour

  // Atomic operations
  await redis.set('counter', 0);
  await redis.incr('counter'); // 1
  await redis.incrby('counter', 5); // 6
  await redis.decr('counter'); // 5
}

demoStrings().catch(console.error);
```

**Use cases**: Counters, flags, caching simple values

#### 2. Hashes

```javascript
async function demoHashes() {
  // User profile
  await redis.hset('user:123', {
    name: 'John Doe',
    email: 'john@example.com',
    age: 30,
  });

  // Get single field
  const name = await redis.hget('user:123', 'name'); // 'John Doe'

  // Get all fields
  const user = await redis.hgetall('user:123');
  // { name: 'John Doe', email: 'john@example.com', age: '30' }

  // Increment field
  await redis.hincrby('user:123', 'age', 1); // 31

  console.log({ name, user });
}

demoHashes().catch(console.error);
```

**Use cases**: Objects, user profiles, session data

#### 3. Lists

```javascript
async function demoLists() {
  // Queue (FIFO)
  await redis.rpush('queue', 'task1', 'task2', 'task3'); // Right push
  const task = await redis.lpop('queue'); // Left pop → 'task1'

  // Stack (LIFO)
  await redis.rpush('stack', 'item1', 'item2');
  const item = await redis.rpop('stack'); // Right pop → 'item2'

  // Recent items
  await redis.lpush('recent:user:123', 'post1', 'post2', 'post3');
  const recent = await redis.lrange('recent:user:123', 0, 9); // First 10
  await redis.ltrim('recent:user:123', 0, 99); // Keep only 100

  console.log({ task, item, recent });
}

demoLists().catch(console.error);
```

**Use cases**: Queues, activity feeds, recent items

#### 4. Sets

```javascript
async function demoSets() {
  // Add members
  await redis.sadd('tags:post:123', 'nodejs', 'redis', 'caching');

  // Check membership
  const hasTag = await redis.sismember('tags:post:123', 'nodejs'); // 1 → true

  // Get all members
  const tags = await redis.smembers('tags:post:123'); // ['nodejs', 'redis', 'caching']

  // Set operations
  await redis.sadd('set1', 'a', 'b', 'c');
  await redis.sadd('set2', 'b', 'c', 'd');

  const intersection = await redis.sinter('set1', 'set2'); // ['b', 'c']
  const union = await redis.sunion('set1', 'set2'); // ['a', 'b', 'c', 'd']
  const difference = await redis.sdiff('set1', 'set2'); // ['a']

  console.log({ hasTag: Boolean(hasTag), tags, intersection, union, difference });
}

demoSets().catch(console.error);
```

**Use cases**: Tags, unique items, relationship graphs

#### 5. Sorted Sets (ZSets)

```javascript
async function demoSortedSets() {
  // Leaderboard
  await redis.zadd(
    'leaderboard',
    100,
    'player1',
    200,
    'player2',
    150,
    'player3',
  );

  // Get rank
  const rank = await redis.zrevrank('leaderboard', 'player2'); // 0-indexed

  // Top 10
  const top10Raw = await redis.zrevrange('leaderboard', 0, 9, 'WITHSCORES');
  const top10 = [];
  for (let i = 0; i < top10Raw.length; i += 2) {
    top10.push({ player: top10Raw[i], score: Number(top10Raw[i + 1]) });
  }

  // Increment score
  await redis.zincrby('leaderboard', 50, 'player1'); // 150

  // Get by score range
  const byScore = await redis.zrangebyscore('leaderboard', 100, 200);

  console.log({ rank, top10, byScore });
}

demoSortedSets().catch(console.error);
```

**Use cases**: Leaderboards, priority queues, time-series data

### Advanced Features

#### Transactions

```javascript
async function demoTransactions() {
  // MULTI/EXEC (atomic)
  const tx = redis.multi();
  tx.set('key1', 'value1');
  tx.incr('counter');
  tx.get('key1');
  const results = await tx.exec(); // All or nothing

  console.log(results);
}

demoTransactions().catch(console.error);
```

#### Pub/Sub

```javascript
async function demoPubSub() {
  const subscriber = redis.duplicate();
  await subscriber.connect();

  await subscriber.subscribe('notifications', (message) => {
    console.log('Received:', message);
  });

  // Publisher
  await redis.publish('notifications', 'New message!');

  // Wait briefly to ensure message is received (demo purposes)
  setTimeout(() => subscriber.disconnect(), 500);
}

demoPubSub().catch(console.error);
```

**Use case**: Real-time notifications, chat, event broadcasting

#### Lua Scripting

```javascript
async function demoRateLimit() {
  const rateLimitScript = `
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
`;

  redis.defineCommand('rateLimit', {
    numberOfKeys: 1,
    lua: rateLimitScript,
  });

  const allowed = await redis.rateLimit('rate:user:123', 100, 60); // 100 req/min
  console.log({ allowed: Boolean(allowed) });
}

demoRateLimit().catch(console.error);
```

**Use case**: Atomic complex operations

#### Pipelining

```javascript
async function demoPipelining() {
  // Batch multiple commands
  const pipeline = redis.pipeline();
  for (let i = 0; i < 1000; i += 1) {
    pipeline.set(`key:${i}`, `value:${i}`);
  }
  await pipeline.exec();

  // Reduces round-trips: 1 instead of 1000
}

demoPipelining().catch(console.error);
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
```javascript
async function demoRedisCluster() {
  const cluster = new Redis.Cluster([
    { host: '127.0.0.1', port: 7000 },
    { host: '127.0.0.1', port: 7001 },
    { host: '127.0.0.1', port: 7002 },
  ]);

  await cluster.set('key', 'value'); // Автоматически роутится на правильный node
  await cluster.quit();
}

demoRedisCluster().catch(console.error);
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

```javascript
const Memcached = require('memcached');
const { promisify } = require('util');

const memcached = new Memcached('127.0.0.1:11211');
const set = promisify(memcached.set).bind(memcached);
const get = promisify(memcached.get).bind(memcached);
const del = promisify(memcached.del).bind(memcached);
const incr = promisify(memcached.incr).bind(memcached);
const decr = promisify(memcached.decr).bind(memcached);
const getMulti = promisify(memcached.getMulti).bind(memcached);

async function demoMemcached() {
  // Set
  await set('key', 'value', 3600); // Expire in 1 hour

  // Get
  const value = await get('key');

  // Delete
  await del('key');

  // Increment / Decrement
  await set('counter', 0, 3600);
  await incr('counter', 1); // 1
  await decr('counter', 1); // 0

  // Multi-get (efficient)
  const values = await getMulti(['key1', 'key2', 'key3']);

  console.log({ value, values });
  memcached.end();
}

demoMemcached().catch(console.error);
```

### Client-side Sharding

Memcached не имеет built-in clustering → client handles sharding.

```javascript
// Consistent hashing client
const shardedClient = new Memcached(
  ['server1:11211', 'server2:11211', 'server3:11211'],
  { retries: 2, remove: true }, // Ketama hashing по умолчанию
);

const shardedSet = promisify(shardedClient.set).bind(shardedClient);
await shardedSet('key', 'value', 3600); // Client роутит на правильный server
shardedClient.end();
```

## Практические паттерны

### 1. Session Store

**Redis**:
```javascript
const { v4: uuid } = require('uuid');

async function createSession(userId) {
  const sessionId = uuid();
  const sessionData = {
    userId,
    createdAt: Date.now(),
  };

  // TTL = 24 hours
  await redis.setex(`session:${sessionId}`, 86400, JSON.stringify(sessionData));
  return sessionId;
}

async function getSession(sessionId) {
  const data = await redis.get(`session:${sessionId}`);
  return data ? JSON.parse(data) : null;
}

async function deleteSession(sessionId) {
  await redis.del(`session:${sessionId}`);
}
```

### 2. Rate Limiting

```javascript
async function isRateLimited(userId, limit = 100, windowSeconds = 60) {
  const windowId = Math.floor(Date.now() / 1000 / windowSeconds);
  const key = `rate:${userId}:${windowId}`;

  const current = await redis.incr(key);
  if (current === 1) {
    await redis.expire(key, windowSeconds);
  }

  return current > limit;
}

// Usage
if (await isRateLimited(userId)) {
  return { error: 'Rate limit exceeded' }; // HTTP 429
}
```

### 3. Leaderboard

```javascript
async function addScore(userId, score) {
  await redis.zadd('leaderboard', score, userId);
}

async function getLeaderboard(limit = 10) {
  const raw = await redis.zrevrange('leaderboard', 0, limit - 1, 'WITHSCORES');
  const result = [];
  for (let i = 0; i < raw.length; i += 2) {
    result.push({ userId: raw[i], score: Number(raw[i + 1]) });
  }
  return result;
}

async function getRank(userId) {
  const rank = await redis.zrevrank('leaderboard', userId);
  return rank != null ? rank + 1 : null;
}

async function getScore(userId) {
  const score = await redis.zscore('leaderboard', userId);
  return score != null ? Number(score) : null;
}
```

### 4. Cache-Aside Pattern

```javascript
async function getUser(userId) {
  const cacheKey = `user:${userId}`;

  // Try cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // Cache miss → query DB
  const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);
  if (user) {
    await redis.setex(cacheKey, 3600, JSON.stringify(user));
  }

  return user;
}

async function updateUser(userId, data) {
  await db.execute('UPDATE users SET ... WHERE id = ?', [userId]);

  // Invalidate cache
  await redis.del(`user:${userId}`);
}
```

### 5. Distributed Lock

```javascript
async function acquireLock(lockName, timeoutSeconds = 10) {
  const lockId = uuid();
  const lockKey = `lock:${lockName}`;

  // SET NX (only if not exists) with expiration
  const acquired = await redis.set(lockKey, lockId, 'NX', 'EX', timeoutSeconds);
  return acquired ? lockId : null;
}

async function releaseLock(lockName, lockId) {
  const lockKey = `lock:${lockName}`;
  const script = `
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
    else
        return 0
    end
  `;

  return redis.eval(script, 1, lockKey, lockId);
}

// Usage
const lockId = await acquireLock('resource:123');
if (lockId) {
  try {
    // Critical section
    await processResource();
  } finally {
    await releaseLock('resource:123', lockId);
  }
}
```

### 6. Queue (с Redis Lists)

```javascript
// Producer
async function enqueue(queueName, task) {
  await redis.rpush(queueName, JSON.stringify(task));
}

// Consumer (blocking)
async function dequeue(queueName, timeoutSeconds = 0) {
  const result = await redis.blpop(queueName, timeoutSeconds);
  if (!result) {
    return null;
  }

  const [, data] = result;
  return JSON.parse(data);
}

// Usage
await enqueue('tasks', { type: 'email', to: 'user@example.com' });

// Worker
while (true) {
  const task = await dequeue('tasks', 5);
  if (task) {
    await processTask(task);
  }
}
```

## Performance Optimization

### 1. Pipelining

```javascript
async function comparePipelining() {
  // Bad: 1000 round-trips
  for (let i = 0; i < 1000; i += 1) {
    await redis.set(`key:${i}`, `value:${i}`);
  }

  // Good: 1 round-trip
  const pipeline = redis.pipeline();
  for (let i = 0; i < 1000; i += 1) {
    pipeline.set(`key:${i}`, `value:${i}`);
  }
  await pipeline.exec();
}

comparePipelining().catch(console.error);
```

### 2. Connection Pooling

```javascript
const genericPool = require('generic-pool');

// Connection pool (reuse connections)
const redisFactory = {
  create: () =>
    Promise.resolve(
      new Redis({
        host: 'localhost',
        port: 6379,
      }),
    ),
  destroy: (client) => client.quit(),
};

const redisPool = genericPool.createPool(redisFactory, {
  min: 2,
  max: 50,
});

async function withRedis(fn) {
  const client = await redisPool.acquire();
  try {
    return await fn(client);
  } finally {
    redisPool.release(client);
  }
}

await withRedis((client) => client.ping());
```

### 3. Compression

```javascript
const zlib = require('zlib');

async function setCompressed(key, data, ttlSeconds = 3600) {
  const jsonData = JSON.stringify(data);
  const compressed = zlib.gzipSync(Buffer.from(jsonData));
  await redis.set(key, compressed, 'EX', ttlSeconds);
}

async function getCompressed(key) {
  const compressed = await redis.getBuffer(key);
  if (!compressed) {
    return null;
  }

  const jsonData = zlib.gunzipSync(compressed).toString();
  return JSON.parse(jsonData);
}
```

### 4. Hash Tags (Redis Cluster)

```javascript
async function ensureSameSlot() {
  // Ensure related keys on same slot
  // {user:123} → all with this tag on same slot
  await redis.set('{user:123}:profile', profileData);
  await redis.set('{user:123}:settings', settingsData);

  // Can use multi-key operations
  const pipeline = redis.pipeline();
  pipeline.get('{user:123}:profile');
  pipeline.get('{user:123}:settings');
  await pipeline.exec();
}

ensureSameSlot().catch(console.error);
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

```javascript
async function gatherRedisInfo() {
  const rawInfo = await redis.info();
  const stats = Object.fromEntries(
    rawInfo
      .split('\n')
      .filter((line) => line && !line.startsWith('#'))
      .map((line) => line.split(':'))
      .filter((parts) => parts.length === 2),
  );

  // Memory
  const usedMemory = stats.used_memory_human;
  const maxMemory = stats.maxmemory_human;

  // Hit rate
  const hits = Number(stats.keyspace_hits || 0);
  const misses = Number(stats.keyspace_misses || 0);
  const hitRate = hits + misses > 0 ? hits / (hits + misses) : 0;

  // Connections
  const connectedClients = Number(stats.connected_clients || 0);

  // Commands/sec
  const totalCommands = Number(stats.total_commands_processed || 0);
  const uptimeSeconds = Number(stats.uptime_in_seconds || 1);
  const commandsPerSec = totalCommands / uptimeSeconds;

  return { usedMemory, maxMemory, hitRate, connectedClients, commandsPerSec };
}

gatherRedisInfo().then(console.log).catch(console.error);
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

```javascript
async function storeUsers() {
  // Bad: huge hash
  await redis.hset('all_users', largeUserMap); // Millions of fields

  // Good: separate keys
  await redis.set('user:123', data);
  await redis.set('user:456', data);
}

storeUsers().catch(console.error);
```

### 3. Set Expiration

```javascript
async function setWithTtl() {
  // Always set TTL (avoid memory leak)
  await redis.set('key', 'value', 'EX', 3600); // Expire in 1 hour
}

setWithTtl().catch(console.error);
```

### 4. Use Appropriate Data Structures

```javascript
// Counters → String with INCR
// Objects → Hash
// Collections → Set or List
// Leaderboards → Sorted Set
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
