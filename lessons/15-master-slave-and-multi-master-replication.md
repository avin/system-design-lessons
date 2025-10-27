# Репликация: Master-Slave и Master-Master

## Что такое репликация?

**Database Replication** — процесс копирования данных с одного сервера (master/primary) на другие серверы (slaves/replicas) для:
- Повышения availability (если master упал, replica берет на себя)
- Масштабирования reads (распределение read queries)
- Backup и disaster recovery
- Географического распределения (низкая latency для пользователей)

```
     [Master]
      ↓    ↓
  [Replica1] [Replica2]
```

## Зачем нужна репликация?

### 1. High Availability

```
[Master] упал → [Replica] promoted to Master
Downtime: секунды вместо часов
```

### 2. Scalability (Read Scaling)

```
Writes → [Master]
Reads  → [Master, Replica1, Replica2, Replica3]

Можем обрабатывать 10x больше reads!
```

### 3. Backup

```
Replica используется для backup без нагрузки на Master
```

### 4. Analytics

```
Аналитика на Replica → не влияет на production Master
```

### 5. Geo-Distribution

```
US Users → US Replica (low latency)
EU Users → EU Replica (low latency)
```

## Master-Slave (Primary-Replica) Replication

### Как работает

```
           [Master]
          (writes)
             ↓
    Replication Log
       ↓         ↓
  [Slave1]   [Slave2]
  (reads)    (reads)
```

**Процесс**:
1. Client пишет в Master
2. Master записывает change в replication log (binlog, WAL)
3. Slaves читают log и применяют изменения
4. Clients читают из Slaves

### Synchronous vs Asynchronous Replication

#### Synchronous Replication

Master ждет подтверждения от Slave(s) перед commit.

```
1. Client → Write to Master
2. Master → Replicate to Slaves
3. Slaves → Apply changes
4. Slaves → Send ACK to Master
5. Master → Commit transaction
6. Master → Return success to Client
```

**Преимущества**:
✅ Strong consistency (Slave всегда имеет свежие данные)
✅ Zero data loss при failover

**Недостатки**:
❌ Высокая latency (ждем slaves)
❌ Availability зависит от slaves (если slave недоступен, writes блокируются)

**Use case**: Финансовые системы, критичные данные

#### Asynchronous Replication

Master не ждет подтверждения от Slaves.

```
1. Client → Write to Master
2. Master → Commit transaction
3. Master → Return success to Client
4. Master → Replicate to Slaves (асинхронно)
5. Slaves → Apply changes (eventually)
```

**Преимущества**:
✅ Низкая latency (не ждем slaves)
✅ Высокая availability (slaves не влияют на writes)

**Недостатки**:
❌ Replication lag (slaves отстают от master)
❌ Возможна потеря данных при failover

**Use case**: Большинство приложений (acceptable trade-off)

#### Semi-Synchronous Replication

Гибрид: ждем ACK от N slaves (например, 1 из 3).

```
Master → At least 1 Slave ACK → Commit
```

**Trade-off** между consistency и performance.

### Replication Lag

**Проблема**: Slave отстает от Master.

```
t=0:  Master: balance = 100, Slave: balance = 100
t=1:  Client writes to Master: balance = 200
t=1:  Master: balance = 200, Slave: balance = 100 (lag!)
t=3:  Slave applies change: balance = 200
```

**Последствия**:
- Read-your-writes inconsistency (пользователь пишет, затем читает старые данные)
- Stale data в UI

**Измерение lag**:

PostgreSQL:
```sql
SELECT
  client_addr,
  state,
  pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
  extract(epoch from (now() - pg_last_xact_replay_timestamp())) AS lag_seconds
FROM pg_stat_replication;
```

MySQL:
```sql
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: lag в секундах
```

**Решения**:

1. **Read from Master для критичных данных**:
```javascript
// После write читаем из Master
const user = await db.master.query('SELECT * FROM users WHERE id = 123');

// Обычные reads из Slave
const users = await db.slave.query('SELECT * FROM users');
```

2. **Session consistency**: Читать из того же server, куда писали (sticky sessions)

3. **Monotonic reads**: Гарантия что не увидите более старые данные

4. **Accept eventual consistency**: Для некритичных данных (likes, views)

### Failover (Переключение на Slave)

**Сценарий**: Master упал.

**Автоматический failover**:
```
1. Health check обнаруживает Master down
2. Выбирается новый Master (обычно Slave с минимальным lag)
3. Promote Slave to Master
4. Переключение clients на новый Master
5. Старый Master (когда восстановится) становится Slave
```

**Проблемы**:
- **Split-brain**: Два Master одновременно (если network partition)
- **Data loss**: Если Slave отставал, некоторые writes теряются

**Решение split-brain**:
- Consensus algorithm (Raft, Paxos)
- Quorum-based systems
- STONITH (Shoot The Other Node In The Head) — убить старый master

**Инструменты**:
- **PostgreSQL**: Patroni, repmgr
- **MySQL**: MySQL Router, ProxySQL, Orchestrator
- **Managed solutions**: AWS RDS, Google Cloud SQL (auto-failover)

### Пример: PostgreSQL Streaming Replication

**Master конфигурация** (postgresql.conf):
```
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
```

**Slave setup**:
```bash
# 1. Backup Master
pg_basebackup -h master_host -D /var/lib/postgresql/data -U replication -P

# 2. Создать recovery.conf (или standby.signal в Postgres 12+)
primary_conninfo = 'host=master_host port=5432 user=replication password=...'
```

**Promote Slave to Master**:
```bash
pg_ctl promote -D /var/lib/postgresql/data
```

### Пример: MySQL Master-Slave

**Master конфигурация** (my.cnf):
```ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-format = ROW
```

**Slave конфигурация**:
```ini
[mysqld]
server-id = 2
relay-log = relay-log
read-only = 1
```

**Setup replication**:
```sql
-- На Master: создать replication user
CREATE USER 'replicator'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';

-- На Master: получить binlog position
SHOW MASTER STATUS;
-- File: mysql-bin.000001, Position: 154

-- На Slave: настроить replication
CHANGE MASTER TO
  MASTER_HOST='master_host',
  MASTER_USER='replicator',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;

START SLAVE;

-- Проверить status
SHOW SLAVE STATUS\G
```

## Master-Master (Multi-Master) Replication

### Как работает

```
[Master1] ←→ [Master2]
  (r/w)       (r/w)

Both can accept writes!
```

**Процесс**:
1. Client пишет в Master1
2. Master1 реплицирует в Master2
3. Client пишет в Master2
4. Master2 реплицирует в Master1

### Преимущества

✅ **High availability**: Любой Master может принимать writes
✅ **Load balancing**: Распределение writes между Masters
✅ **Zero downtime failover**: Другой Master уже работает

### Проблемы Multi-Master

#### 1. Write Conflicts

**Проблема**: Одновременные writes на разных Masters.

```
t=0: Master1: account balance = 100
     Master2: account balance = 100

t=1: Client1 → Master1: SET balance = 50  (withdraw 50)
     Client2 → Master2: SET balance = 20  (withdraw 80)

t=2: Replication:
     Master1 receives: SET balance = 20 (final: 20)
     Master2 receives: SET balance = 50 (final: 50)

Conflict! Разные значения на Masters!
```

**Решения**:

1. **Last Write Wins (LWW)**:
```
Timestamp: Master1 write at 10:00:01
           Master2 write at 10:00:02
Result: Master2 wins (later timestamp)
```
❌ Проблема: Потеря данных (Master1 write игнорируется)

2. **Conflict-free Replicated Data Types (CRDTs)**:
```
Вместо SET balance = X
Используем INC/DEC:
Master1: DEC balance 50
Master2: DEC balance 80
Result: balance = 100 - 50 - 80 = -30 (обе операции применены)
```

3. **Application-level conflict resolution**:
```
Приложение определяет, какой write сохранить
```

4. **Avoid conflicts**:
```
Partition writes: Master1 для users 0-50%, Master2 для 50-100%
```

#### 2. Auto-increment Conflicts

**Проблема**:
```
Master1: INSERT id=1
Master2: INSERT id=1  (conflict!)
```

**Решение**:
```
Master1: auto_increment_increment=2, auto_increment_offset=1 (1,3,5,7...)
Master2: auto_increment_increment=2, auto_increment_offset=2 (2,4,6,8...)
```

Или используйте UUID вместо auto-increment.

#### 3. Циклическая репликация

**Проблема**: Change реплицируется туда-обратно бесконечно.

```
Master1 → write → Master2 → replicate back → Master1 → ...
```

**Решение**: Tracking origin (не реплицировать changes, пришедшие через репликацию).

### Когда использовать Master-Master

✅ **Geo-distributed writes**: Users в разных регионах пишут в локальный Master
✅ **High write availability**: Любой Master может принимать writes
✅ **Active-Active setup**: Оба Masters обрабатывают load

❌ **Если есть write conflicts**: Лучше Master-Slave
❌ **Если нужна strong consistency**: Multi-Master обычно eventual consistency

### Пример: MySQL Master-Master

```ini
# Master1
[mysqld]
server-id = 1
log-bin = mysql-bin
auto_increment_increment = 2
auto_increment_offset = 1

# Master2
[mysqld]
server-id = 2
log-bin = mysql-bin
auto_increment_increment = 2
auto_increment_offset = 2
```

**Setup**:
```sql
-- На обоих: создать replication user
CREATE USER 'replicator'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';

-- Master1 → Master2
CHANGE MASTER TO MASTER_HOST='master2', MASTER_USER='replicator', ...;
START SLAVE;

-- Master2 → Master1
CHANGE MASTER TO MASTER_HOST='master1', MASTER_USER='replicator', ...;
START SLAVE;
```

## Read Replicas: Scaling Reads

### Load Balancer для Reads

```
       [Application]
            ↓
     [Read LB / Router]
      ↙    ↓    ↘
  [R1]  [R2]  [R3]
      ↖    ↓    ↙
        [Master]
```

**Стратегии routing**:

1. **Round Robin**:
```javascript
const replicas = [replica1, replica2, replica3];
let current = 0;

function getReplica() {
  const replica = replicas[current % replicas.length];
  current += 1;
  return replica;
}
```

2. **Least Connections**:
```javascript
function getReplica() {
  return replicas.reduce((best, replica) =>
    replica.activeConnections < best.activeConnections ? replica : best,
  replicas[0]);
}
```

3. **Geographic routing**:
```javascript
function getReplica(userLocation) {
  if (userLocation === 'US') {
    return usReplica;
  }
  if (userLocation === 'EU') {
    return euReplica;
  }
  return defaultReplica;
}
```

### Application-level routing

**Node.js (Sequelize)**:
```javascript
const { Sequelize } = require('sequelize');

const master = new Sequelize('postgres://master.db.example.com/mydb');
const replica = new Sequelize('postgres://replica.db.example.com/mydb');

async function runQuery({ type, sql, params }) {
  if (type === 'write') {
    return master.query(sql, { replacements: params });
  }

  return replica.query(sql, { replacements: params });
}
```

**Manual routing**:
```javascript
// Write
const user = await runQuery({
  type: 'write',
  sql: 'INSERT INTO users (username) VALUES (:username) RETURNING *',
  params: { username: 'john' },
});

// Read
const users = await runQuery({
  type: 'read',
  sql: 'SELECT * FROM users WHERE is_active = true',
});

// Read-your-writes: read from master after write
const freshUser = await master.query(
  'SELECT * FROM users WHERE id = :id',
  { replacements: { id: user[0][0].id } },
);
```

### Proxy Solutions

**ProxySQL** (MySQL):
```sql
-- Routing rules
INSERT INTO mysql_query_rules (rule_id, active, match_digest, destination_hostgroup)
VALUES
  (1, 1, '^SELECT.*FOR UPDATE', 10),  -- Writes to hostgroup 10 (master)
  (2, 1, '^SELECT', 20);               -- Reads to hostgroup 20 (replicas)
```

**PgBouncer** (PostgreSQL):
```ini
[databases]
mydb = host=master.db port=5432

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
```

**AWS RDS Read Replicas**:
```javascript
const { Client } = require('pg');

// Endpoint routing
const MASTER_ENDPOINT = 'mydb.abc123.us-east-1.rds.amazonaws.com';
const REPLICA_ENDPOINT = 'mydb-replica.abc123.us-east-1.rds.amazonaws.com';

// Application uses different connections
const masterConn = new Client({ host: MASTER_ENDPOINT });
await masterConn.connect();

const replicaConn = new Client({ host: REPLICA_ENDPOINT });
await replicaConn.connect();
```

## Практические рекомендации

### 1. Monitoring Replication

**Метрики**:
- Replication lag (seconds/bytes)
- Replica health status
- Throughput (rows replicated/sec)
- Errors count

**Alerts**:
```
IF replication_lag > 60 seconds → ALERT
IF replica_status != 'running' → CRITICAL
```

### 2. Capacity Planning

```
Current:
- Master: 1000 QPS writes, 5000 QPS reads
- 1 Replica: 5000 QPS reads

Target: 20,000 QPS reads
Needed replicas: 20,000 / 5,000 = 4 replicas
```

### 3. Failover Testing

**Chaos engineering**: Регулярно тестируйте failover!

```bash
# Simulate Master failure
docker stop postgres-master

# Verify:
# 1. Replica promoted to Master?
# 2. Application connected to new Master?
# 3. Downtime duration?
```

### 4. Backup Strategy

```
Daily: Full backup from Replica (не нагружает Master)
Hourly: Incremental backup (binlog/WAL)
Retention: 30 days
```

### 5. Security

```sql
-- Replication user с минимальными правами
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'replica_ip';

-- SSL для replication traffic
CHANGE MASTER TO MASTER_SSL=1;
```

## Advanced Topics

### Cascading Replication

```
     [Master]
        ↓
    [Replica1] (intermediate)
      ↓     ↓
  [R2]    [R3]
```

Снижает нагрузку на Master (он реплицирует только в Replica1).

**PostgreSQL**:
```
# Replica1 может быть источником для R2, R3
```

### Delayed Replica

Replica намеренно отстает на N часов.

```
Master → Replica (delayed 2 hours)
```

**Use case**: Protection от случайного DROP TABLE (можно восстановить из delayed replica).

**MySQL**:
```sql
CHANGE MASTER TO MASTER_DELAY = 7200;  -- 2 hours
```

### Logical Replication

Репликация на уровне логических changes (SQL statements), не physical bytes.

**Преимущества**:
- Cross-version replication (PostgreSQL 11 → 14)
- Selective replication (только определенные таблицы)
- Different schema на replica

**PostgreSQL**:
```sql
-- On Master: create publication
CREATE PUBLICATION my_publication FOR TABLE users, posts;

-- On Replica: create subscription
CREATE SUBSCRIPTION my_subscription
  CONNECTION 'host=master dbname=mydb'
  PUBLICATION my_publication;
```

## Что почитать дальше

- "High Performance MySQL" by Baron Schwartz — Chapter on Replication
- PostgreSQL Documentation: Replication
- "Designing Data-Intensive Applications" by Martin Kleppmann — Chapter 5

## Проверьте себя

1. В чем разница между synchronous и asynchronous replication?
2. Что такое replication lag и как его измерить?
3. Какие проблемы возникают в Master-Master replication?
4. Как решить write conflicts в Multi-Master?
5. Что такое split-brain и как его предотвратить?
6. Когда использовать Master-Slave vs Master-Master?
7. Как роутить reads на replicas в приложении?
8. Зачем нужна delayed replica?

## Ключевые выводы

- Replication повышает availability, scalability, и disaster recovery
- Master-Slave: writes на Master, reads на Slaves (наиболее распространенный)
- Synchronous replication — strong consistency, но высокая latency
- Asynchronous replication — низкая latency, но replication lag
- Replication lag — основная проблема асинхронной репликации
- Master-Master позволяет writes на оба, но имеет проблемы с conflicts
- Write conflicts решаются через LWW, CRDTs, или app-level logic
- Read replicas масштабируют reads, но не writes
- Failover должен быть автоматизирован (Patroni, Orchestrator)
- Мониторьте replication lag и тестируйте failover регулярно
- Используйте routing (ProxySQL, app-level) для распределения reads

---

**Предыдущий урок**: [Индексы в базах данных](14-database-indexes.md)
**Следующий урок**: [Шардирование и стратегии партиционирования](16-shardirovanie-strategii.md)
