# Шардирование и стратегии партиционирования

## Что такое шардирование?

**Sharding (Шардирование)** — разделение большой базы данных на меньшие, более управляемые части (shards), каждая из которых хранится на отдельном сервере.

```
Monolithic DB (10TB):
[Server 1: All data]

Sharded DB:
[Shard 1: 2TB] [Shard 2: 2TB] [Shard 3: 2TB] [Shard 4: 2TB] [Shard 5: 2TB]
```

**Цель**: Horizontal scaling — распределение нагрузки и данных между серверами.

## Зачем нужно шардирование?

### 1. Превышение capacity одного сервера

```
Single server limits:
- Storage: 10TB
- RAM: 1TB
- CPU: 64 cores
- IOPS: 50K

Your needs: 100TB, 1M IOPS
→ Нужно шардирование!
```

### 2. Scaling writes

Replication масштабирует только reads, не writes.

```
Master-Slave:
- Writes: только Master (bottleneck)
- Reads: распределены между replicas ✅

Sharding:
- Writes: распределены между shards ✅
- Reads: тоже распределены ✅
```

### 3. Isolation

```
Shard 1: US users
Shard 2: EU users
Shard 3: Asia users

Проблема в US не влияет на EU и Asia
```

### 4. Compliance

```
EU users data → EU datacenter (GDPR)
US users data → US datacenter
```

## Sharding vs Partitioning

**Partitioning** — разделение таблицы в пределах одной БД (vertical/horizontal).
**Sharding** — разделение данных между несколькими БД/серверами.

```
Partitioning:
[Single Server]
  Table users:
    - Partition 1: users created 2020
    - Partition 2: users created 2021
    - Partition 3: users created 2022

Sharding:
[Server 1] users with id 0-1M
[Server 2] users with id 1M-2M
[Server 3] users with id 2M-3M
```

Шардирование — это distributed partitioning.

## Shard Key (Ключ шардирования)

**Shard Key** — колонка (или набор колонок), по которой определяется, в какой shard попадет запись.

```javascript
const shardId = hash(shardKey) % numShards;
```

**Выбор shard key критичен!**

### Характеристики хорошего shard key

1. **High cardinality** (много уникальных значений)
   - ✅ user_id (millions)
   - ❌ country (десятки)

2. **Even distribution** (равномерное распределение)
   - ✅ UUID
   - ❌ timestamp (все новые данные на один shard)

3. **Query isolation** (queries обращаются к одному shard)
   - ✅ user_id (query по user_id идет на один shard)
   - ❌ timestamp (range query может затронуть все shards)

### Плохие примеры shard key

**1. Low cardinality**:
```sql
shard_key = country

Shards:
- Shard 1: US (60% users)
- Shard 2: UK (10% users)
- Shard 3: Others (30% users)

Problem: Неравномерное распределение (hotspot на Shard 1)
```

**2. Monotonic key** (auto-increment):
```sql
shard_key = id (auto-increment)

t=0: INSERT id=1 → Shard 1
t=1: INSERT id=2 → Shard 2
t=2: INSERT id=3 → Shard 3
...
t=1000000: INSERT id=1000001 → Shard 1

Problem: Все новые writes на один shard (rotating, но не параллельные)
```

**3. Timestamp**:
```sql
shard_key = created_at

Problem: Все новые данные идут на один shard
```

## Стратегии шардирования

### 1. Hash-based Sharding

**Принцип**: `shard_id = hash(shard_key) % num_shards`

```javascript
function getShard(userId, numShards = 4) {
  return hash(userId) % numShards;
}

// Examples:
getShard(123); // → 3
getShard(456); // → 1
getShard(789); // → 2
```

**Преимущества**:
✅ Равномерное распределение
✅ Простая реализация
✅ Predictable (один user всегда на одном shard)

**Недостатки**:
❌ Resharding сложен (при изменении num_shards)
❌ Range queries требуют обращения ко всем shards

**Когда использовать**:
- Нужно равномерное распределение
- Queries по shard_key (точечные lookups)
- Количество shards стабильно

**Пример**:
```sql
-- Shard key: user_id
-- 4 shards

Users:
- user_id=1 → hash(1) % 4 = 1 → Shard 1
- user_id=2 → hash(2) % 4 = 2 → Shard 2
- user_id=3 → hash(3) % 4 = 3 → Shard 3
- user_id=4 → hash(4) % 4 = 0 → Shard 0
```

### 2. Range-based Sharding

**Принцип**: Диапазоны значений shard_key.

```
Shard 1: user_id 0 - 1M
Shard 2: user_id 1M - 2M
Shard 3: user_id 2M - 3M
Shard 4: user_id 3M - 4M
```

**Преимущества**:
✅ Range queries эффективны (обращаются к одному/нескольким shards)
✅ Проще добавлять shards (новый диапазон)

**Недостатки**:
❌ Неравномерное распределение (если данные неравномерны)
❌ Hotspots (активные диапазоны перегружены)

**Когда использовать**:
- Range queries часты
- Данные имеют естественные диапазоны (даты, географические зоны)

**Пример**:
```sql
-- Shard by created_at
Shard 1: 2020-01-01 to 2020-12-31
Shard 2: 2021-01-01 to 2021-12-31
Shard 3: 2022-01-01 to 2022-12-31

Query: WHERE created_at BETWEEN '2021-06-01' AND '2021-08-31'
→ Только Shard 2
```

### 3. Geographic Sharding

**Принцип**: Разделение по географическому признаку.

```
Shard US: US users
Shard EU: EU users
Shard Asia: Asia users
```

**Преимущества**:
✅ Low latency (близко к пользователям)
✅ Compliance (GDPR, data residency)
✅ Isolation (региональные проблемы изолированы)

**Недостатки**:
❌ Неравномерное распределение (разное количество пользователей)
❌ Cross-region queries сложны

**Когда использовать**:
- Географически распределенные пользователи
- Compliance требования
- Latency критична

**Пример**:
```javascript
function getShard(userLocation) {
  if (['US', 'CA', 'MX'].includes(userLocation)) {
    return 'shard_americas';
  }
  if (['UK', 'DE', 'FR', /* ... */].includes(userLocation)) {
    return 'shard_europe';
  }
  if (['JP', 'CN', 'IN', /* ... */].includes(userLocation)) {
    return 'shard_asia';
  }
  return 'shard_default';
}
```

### 4. Directory-based Sharding (Lookup Table)

**Принцип**: Lookup table указывает, где находятся данные.

```
Lookup Table:
user_id | shard_id
--------|----------
1       | shard_1
2       | shard_3
3       | shard_2
4       | shard_1
```

**Преимущества**:
✅ Гибкость (можно переместить данные между shards)
✅ Нет привязки к hash function или ranges

**Недостатки**:
❌ Дополнительный lookup (latency)
❌ Lookup table — potential bottleneck и SPOF
❌ Сложность управления

**Когда использовать**:
- Нужна гибкость в перемещении данных
- Небольшое количество ключей (lookup table поместится в память)

### 5. Consistent Hashing

**Принцип**: Как hash-based, но минимизирует resharding.

```
Hash ring:
0 ----100(S1)---- 200(S2)---- 300(S3)---- 2^32

user_id=150 → hash(150) → ближайший shard по часовой стрелке → S2
```

**Преимущества**:
✅ При добавлении/удалении shard перемещается только ~1/N данных
✅ Равномерное распределение (с virtual nodes)

**Недостатки**:
❌ Сложнее реализация
❌ Range queries неэффективны

**Когда использовать**:
- Динамическое изменение количества shards
- Distributed caching (Memcached, Redis)

См. урок 6 для деталей.

## Проблемы шардирования

### 1. Cross-Shard Queries

**Проблема**: Query затрагивает несколько shards.

```sql
-- Users sharded by user_id
-- Query: все пользователи с email domain = 'gmail.com'
SELECT * FROM users WHERE email LIKE '%@gmail.com';

→ Нужно запросить все shards и объединить результаты
```

**Решение**:
- **Denormalization**: Дублируйте данные для частых queries
- **Scatter-Gather**: Запросите все shards параллельно и объедините
- **Application-level aggregation**

### 2. Cross-Shard Joins

**Проблема**: JOIN таблиц на разных shards.

```sql
-- Users sharded by user_id
-- Posts sharded by user_id
-- Query: users with their posts

SELECT users.*, posts.*
FROM users
JOIN posts ON users.id = posts.user_id
WHERE users.country = 'US';

If users and posts on different shards → сложно!
```

**Решение**:
- **Co-location**: Shard связанные таблицы по одному ключу
- **Denormalization**: Embed posts в users document (NoSQL)
- **Application-level JOINs**: Fetch отдельно и join в коде

### 3. Distributed Transactions

**Проблема**: Transaction затрагивает несколько shards.

```sql
-- Transfer money между пользователями на разных shards
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;  -- Shard 1
  UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;  -- Shard 2
COMMIT;
```

**Решение**:
- **Two-Phase Commit (2PC)**: Distributed transaction protocol (медленный)
- **Saga Pattern**: Compensating transactions
- **Avoid cross-shard transactions**: Дизайн системы так, чтобы их избежать

### 4. Auto-increment IDs

**Проблема**: Каждый shard генерирует IDs независимо.

```
Shard 1: id=1, id=2, id=3
Shard 2: id=1, id=2, id=3  (конфликт!)
```

**Решения**:

**A. ID с префиксом shard**:
```
Shard 1: 1_1, 1_2, 1_3
Shard 2: 2_1, 2_2, 2_3
```

**B. UUID**:
```javascript
const { randomUUID } = require('crypto');
const id = randomUUID(); // Глобально уникальный
```

**C. Snowflake ID** (Twitter):
```
64-bit ID:
- 41 bits: timestamp
- 10 bits: machine ID
- 12 bits: sequence number

Globally unique, sortable by time
```

**D. Database sequence с offset**:
```
Shard 1: 1, 4, 7, 10, 13... (offset=1, increment=3)
Shard 2: 2, 5, 8, 11, 14... (offset=2, increment=3)
Shard 3: 3, 6, 9, 12, 15... (offset=3, increment=3)
```

### 5. Hotspots (Неравномерная нагрузка)

**Проблема**: Один shard перегружен.

```
Shard 1: 1000 QPS (celebrity users)
Shard 2: 100 QPS
Shard 3: 100 QPS
Shard 4: 100 QPS
```

**Решения**:

**A. Better shard key**: Используйте более равномерный
**B. Split hotspot shard**: Разделите на два
**C. Caching**: Кэшируйте hot data
**D. Rate limiting**: Ограничьте запросы к hot entities

### 6. Resharding (Изменение числа shards)

**Проблема**: Нужно добавить/удалить shard → перемещение данных.

```
Before: 3 shards
After: 4 shards

С hash-based (% num_shards):
75% ключей переместятся!
```

**Решения**:

**A. Consistent Hashing**: Минимизирует перемещение (~25% вместо 75%)

**B. Virtual Shards**:
```
256 virtual shards → 4 physical servers (64 virtual per physical)

Add server:
Move некоторые virtual shards на новый server
```

**C. Double-write during migration**:
```
1. Start writing to both old and new shards
2. Migrate data в background
3. Switch reads to new shards
4. Stop writing to old shards
```

## Реализация шардирования

### Application-level Sharding

Приложение определяет shard для каждой операции.

```javascript
class ShardedDatabase {
  constructor() {
    this.shards = new Map([
      [0, connect('shard0.db')],
      [1, connect('shard1.db')],
      [2, connect('shard2.db')],
      [3, connect('shard3.db')],
    ]);
  }

  getShard(userId) {
    const shardId = hash(userId) % this.shards.size;
    return this.shards.get(shardId);
  }

  async insertUser(userId, name, email) {
    const shard = this.getShard(userId);
    await shard.execute('INSERT INTO users (id, name, email) VALUES (?, ?, ?)', [
      userId,
      name,
      email,
    ]);
  }

  async getUser(userId) {
    const shard = this.getShard(userId);
    return shard.execute('SELECT * FROM users WHERE id = ?', [userId]);
  }

  async getAllUsersByCountry(country) {
    const results = [];
    for (const shard of this.shards.values()) {
      const rows = await shard.execute('SELECT * FROM users WHERE country = ?', [
        country,
      ]);
      results.push(...rows);
    }
    return results;
  }
}

// Usage
const db = new ShardedDatabase();
await db.insertUser(123, 'John', 'john@example.com');
const user = await db.getUser(123);
```

### Proxy-based Sharding

Proxy маршрутизирует queries к нужным shards.

```
[Application]
      ↓
   [Proxy]
  ↙  ↓  ↘
[S1][S2][S3]
```

**Примеры**:
- **Vitess** (MySQL sharding)
- **Citus** (PostgreSQL extension)
- **ProxySQL** (MySQL proxy)

**Vitess пример**:
```sql
-- VTGate (proxy) автоматически маршрутизирует
SELECT * FROM users WHERE user_id = 123;
→ VTGate определяет shard и направляет туда
```

### Database-native Sharding

БД сама управляет шардированием.

**MongoDB**:
```javascript
// Enable sharding
sh.enableSharding("mydb")

// Shard collection
sh.shardCollection("mydb.users", { user_id: "hashed" })

// MongoDB автоматически распределяет данные между shards
```

**Cassandra**:
```sql
-- Partition key автоматически определяет placement
CREATE TABLE users (
    user_id uuid,
    name text,
    email text,
    PRIMARY KEY (user_id)
);

-- Cassandra использует consistent hashing
```

**PostgreSQL (Citus)**:
```sql
-- Enable Citus extension
CREATE EXTENSION citus;

-- Distribute table
SELECT create_distributed_table('users', 'user_id');

-- Queries автоматически маршрутизируются
SELECT * FROM users WHERE user_id = 123;
```

## Практические примеры

### Пример 1: Instagram (Users & Posts)

**Shard key**: `user_id`

```javascript
// Schema
const users = {
  shardKey: 'user_id',
  columns: ['user_id', 'username', 'email'],
};

const posts = {
  shardKey: 'user_id', // Co-located с users
  columns: ['post_id', 'user_id', 'image_url', 'created_at'],
};

// Эффективные queries:
// 1. Get user
await db.users.findOne({ user_id: 123 }); // → Shard для user 123

// 2. Get user's posts
await db.posts.find({ user_id: 123 }); // → Тот же shard

// Неэффективные queries:
// 3. Recent posts (global timeline)
await db.posts.find().sort({ created_at: -1 }).limit(100);
// → Нужно запросить все shards (scatter-gather)
```

**Решение для global timeline**: Отдельная таблица (не шардированная или шардированная по времени).

### Пример 2: Uber (Rides)

**Shard key**: `city_id` (geographic sharding)

```javascript
const ridesByCity = {
  shardKey: 'city_id',
  columns: ['ride_id', 'city_id', 'user_id', 'driver_id', 'status'],
};

// Shards:
// Shard SF: San Francisco rides
// Shard NYC: New York rides
// Shard LON: London rides

// Преимущества:
// - Queries изолированы по городу
// - Можно scale города независимо
// - Geo-locality (low latency)
```

### Пример 3: Multi-tenant SaaS

**Shard key**: `tenant_id` (organization)

```javascript
const usersByTenant = {
  shardKey: 'tenant_id',
  columns: ['user_id', 'tenant_id', 'name'],
};

const documentsByTenant = {
  shardKey: 'tenant_id',
  columns: ['document_id', 'tenant_id', 'content'],
};

// Isolation: каждый tenant на своем shard (или несколько tenants на shard)

// Преимущества:
// - Tenant isolation (security)
// - Можно переместить большого tenant на dedicated shard
```

## Monitoring шардирования

### Метрики

**Per-shard metrics**:
```
- QPS per shard
- Disk usage per shard
- CPU/RAM usage per shard
- Replication lag per shard
```

**Alerts**:
```
IF shard_qps > 10x avg_qps → HOTSPOT ALERT
IF shard_disk_usage > 80% → ADD CAPACITY
IF shard_imbalance > 30% → REBALANCE NEEDED
```

### Rebalancing

```javascript
// Detect imbalance
const shardSizes = new Map([
  ['shard_1', 1.5],
  ['shard_2', 0.5], // Underutilized
  ['shard_3', 2.0],
  ['shard_4', 0.8],
]);

const total = Array.from(shardSizes.values()).reduce((sum, size) => sum + size, 0);
const avg = total / shardSizes.size; // 1.2TB
const threshold = 0.3; // 30%

for (const [shard, size] of shardSizes.entries()) {
  if (Math.abs(size - avg) / avg > threshold) {
    console.log(`Rebalance needed for ${shard}`);
  }
}
```

## Best Practices

### 1. Выбор shard key

```
Good shard keys:
✅ user_id (high cardinality, even distribution)
✅ tenant_id (для multi-tenant apps)
✅ composite: (region, user_id)

Bad shard keys:
❌ status (low cardinality)
❌ created_at (monotonic, hotspots)
❌ boolean flags
```

### 2. Co-locate связанные данные

```sql
-- Shard users и posts по user_id
users: sharded by user_id
posts: sharded by user_id

-- JOIN локальный (на одном shard)
SELECT users.*, posts.*
FROM users
JOIN posts ON users.id = posts.user_id
WHERE users.id = 123;
```

### 3. Избегайте cross-shard operations

```javascript
// Bad: cross-shard transaction
async function transferCredits(fromUser, toUser, amount) {
  const shard1 = getShard(fromUser);
  const shard2 = getShard(toUser);
  // Distributed transaction (сложно)
}

// Better: credits stored globally или используйте saga pattern
```

### 4. Plan for growth

```javascript
// Start with more shards than needed
// Easier to merge than split

const growthPlan = [
  'Initial: 16 shards для 1M users (overkill, но future-proof)',
  'Growth: 100M users → уже готовы',
];
```

### 5. Monitoring и alerts

```javascript
// Track shard health
for (const shard of shards) {
  monitor(shard.qps);
  monitor(shard.diskUsage);
  monitor(shard.errorRate);
}
```

## Что почитать дальше

- "Designing Data-Intensive Applications" by Martin Kleppmann — Chapter 6
- [Vitess Documentation](https://vitess.io/) — MySQL sharding
- [Citus Documentation](https://docs.citusdata.com/) — PostgreSQL sharding
- Pinterest Engineering Blog: Sharding

## Проверьте себя

1. В чем разница между partitioning и sharding?
2. Какие характеристики делают shard key хорошим?
3. В чем разница между hash-based и range-based sharding?
4. Какие проблемы возникают при cross-shard queries?
5. Как решить проблему auto-increment IDs в шардированной БД?
6. Что такое hotspot и как его избежать?
7. Почему consistent hashing лучше для resharding?
8. Как реализовать sharding: application-level vs proxy vs database-native?

## Ключевые выводы

- Sharding — horizontal scaling через распределение данных между серверами
- Необходим когда single server не справляется (storage, writes, IOPS)
- Shard key критичен: high cardinality, even distribution, query isolation
- Hash-based — равномерное распределение, сложный resharding
- Range-based — эффективные range queries, риск hotspots
- Geographic — low latency, compliance, isolation
- Consistent hashing — minimal data movement при resharding
- Cross-shard queries и JOINs — основная сложность
- Co-locate связанные данные на одном shard
- Используйте UUID или Snowflake ID вместо auto-increment
- Application-level sharding дает контроль, database-native проще
- Monitoring per-shard metrics критичен
- Plan for growth — start с достаточным количеством shards

---

**Предыдущий урок**: [Репликация: Master-Slave и Master-Master](15-replikaciya-master-slave-master-master.md)
**Следующий урок**: [Денормализация и когда её применять](17-denormalizaciya.md)
