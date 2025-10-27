# Урок 33: Репликация данных


## Введение

Репликация — это процесс копирования данных на несколько серверов для повышения availability, fault tolerance и performance.

**Зачем нужна репликация?**

- **High Availability**: если один сервер упал, другие продолжают работать
- **Fault Tolerance**: защита от потери данных
- **Scalability**: распределение read нагрузки
- **Low Latency**: данные ближе к пользователям (geo-replication)
- **Disaster Recovery**: backup в другом ЦОД

Мы уже рассматривали базовые паттерны в [Уроке 15](15-replikaciya-master-slave-master-master.md). Теперь углубимся в продвинутые концепции.

## Виды репликации

### 1. Synchronous Replication

```
Client → Write → Primary
                    ↓ (wait for replica)
                 Replica 1 ✓
                    ↓ (wait for replica)
                 Replica 2 ✓
                    ↓
Client ← ACK ←  Primary
```

**Преимущества:**
- Сильная consistency
- Гарантия durability

**Недостатки:**
- Высокая latency (ждём все реплики)
- Низкая availability (если реплика недоступна)

**Использование:** Financial systems, critical data

### 2. Asynchronous Replication

```
Client → Write → Primary
Client ← ACK ←  Primary (сразу)
                    ↓ (асинхронно)
                 Replica 1 (eventually)
                 Replica 2 (eventually)
```

**Преимущества:**
- Низкая latency
- Высокая availability

**Недостатки:**
- Eventual consistency
- Риск потери данных при падении primary

**Использование:** Большинство систем

### 3. Semi-Synchronous Replication

```
Client → Write → Primary
                    ↓ (wait for 1 replica)
                 Replica 1 ✓
Client ← ACK ←  Primary
                    ↓ (асинхронно)
                 Replica 2 (eventually)
```

**Компромисс:**
- Гарантия на 1+ реплике (durability)
- Разумная latency
- Хорошая availability

**Использование:** MySQL semi-sync, PostgreSQL synchronous_commit

## Single-Leader Replication

### Write Path

```javascript
class SingleLeaderReplication {
  constructor(leader, followers) {
    this.leader = leader;
    this.followers = followers;
    this.replicationLog = [];
  }

  async write(key, value) {
    // Записать на leader
    await this.leader.set(key, value);

    // Добавить в replication log
    const logEntry = {
      lsn: this.replicationLog.length + 1,  // Log Sequence Number
      operation: 'SET',
      key,
      value,
      timestamp: Date.now(),
    };

    this.replicationLog.push(logEntry);

    // Реплицировать на followers (async)
    this.replicateToFollowers(logEntry);

    return { success: true, lsn: logEntry.lsn };
  }

  async replicateToFollowers(logEntry) {
    const replicationPromises = this.followers.map(async (follower) => {
      try {
        await follower.applyLogEntry(logEntry);
        follower.lastAppliedLSN = logEntry.lsn;
      } catch (error) {
        console.error(`Replication to ${follower.id} failed:`, error);
        // Retry logic
      }
    });

    // Не ждём завершения (async replication)
    Promise.all(replicationPromises);
  }

  async read(key, { readFromFollower = false } = {}) {
    if (readFromFollower) {
      // Читать с follower (может быть stale)
      const follower = this.selectFollower();
      return await follower.get(key);
    }

    // Читать с leader (fresh data)
    return await this.leader.get(key);
  }

  selectFollower() {
    // Round-robin
    return this.followers[Math.floor(Math.random() * this.followers.length)];
  }

  async getReplicationLag(follower) {
    const leaderLSN = this.replicationLog.length;
    const followerLSN = follower.lastAppliedLSN || 0;

    return leaderLSN - followerLSN;
  }
}

// Follower
class Follower {
  constructor(id) {
    this.id = id;
    this.data = new Map();
    this.lastAppliedLSN = 0;
  }

  async applyLogEntry(logEntry) {
    // Применить операцию
    if (logEntry.operation === 'SET') {
      this.data.set(logEntry.key, logEntry.value);
    } else if (logEntry.operation === 'DELETE') {
      this.data.delete(logEntry.key);
    }

    this.lastAppliedLSN = logEntry.lsn;

    console.log(`[Follower ${this.id}] Applied LSN ${logEntry.lsn}`);
  }

  async get(key) {
    return this.data.get(key);
  }
}
```

### Replication Lag

```javascript
class ReplicationMonitor {
  constructor(replication) {
    this.replication = replication;
  }

  async checkLag() {
    const lags = await Promise.all(
      this.replication.followers.map(async (follower) => {
        const lag = await this.replication.getReplicationLag(follower);
        return {
          followerId: follower.id,
          lag,
          healthy: lag < 100,  // Threshold: 100 entries
        };
      })
    );

    return lags;
  }

  async startMonitoring(interval = 5000) {
    setInterval(async () => {
      const lags = await this.checkLag();

      lags.forEach(({ followerId, lag, healthy }) => {
        console.log(`Follower ${followerId}: lag=${lag} entries, healthy=${healthy}`);

        if (!healthy) {
          this.alertHighLag(followerId, lag);
        }
      });
    }, interval);
  }

  alertHighLag(followerId, lag) {
    console.warn(`⚠️  High replication lag on Follower ${followerId}: ${lag} entries`);
    // Send alert to monitoring system
  }
}

const monitor = new ReplicationMonitor(replication);
monitor.startMonitoring();
```

### Failover

```javascript
class AutomaticFailover {
  constructor(replication) {
    this.replication = replication;
    this.healthCheckInterval = 1000;
    this.failoverInProgress = false;
  }

  startHealthChecks() {
    setInterval(async () => {
      const leaderHealthy = await this.checkLeaderHealth();

      if (!leaderHealthy && !this.failoverInProgress) {
        await this.performFailover();
      }
    }, this.healthCheckInterval);
  }

  async checkLeaderHealth() {
    try {
      await this.replication.leader.ping();
      return true;
    } catch (error) {
      console.error('Leader health check failed:', error);
      return false;
    }
  }

  async performFailover() {
    console.log('🔄 Starting automatic failover...');

    this.failoverInProgress = true;

    try {
      // Выбрать follower с наименьшим lag
      const follower = await this.selectBestFollower();

      if (!follower) {
        throw new Error('No healthy follower available');
      }

      // Promote follower to leader
      console.log(`Promoting Follower ${follower.id} to Leader`);

      await this.promoteToLeader(follower);

      // Обновить конфигурацию
      const oldLeader = this.replication.leader;
      this.replication.leader = follower;

      // Удалить promoted follower из followers list
      this.replication.followers = this.replication.followers.filter(
        f => f.id !== follower.id
      );

      // Добавить старый leader как follower (когда восстановится)
      // oldLeader → follower

      console.log('✅ Failover completed successfully');

    } catch (error) {
      console.error('❌ Failover failed:', error);

    } finally {
      this.failoverInProgress = false;
    }
  }

  async selectBestFollower() {
    const lags = await Promise.all(
      this.replication.followers.map(async (follower) => ({
        follower,
        lag: await this.replication.getReplicationLag(follower),
      }))
    );

    // Выбрать с минимальным lag
    lags.sort((a, b) => a.lag - b.lag);

    return lags[0]?.follower;
  }

  async promoteToLeader(follower) {
    // Дождаться применения всех pending log entries
    const leaderLSN = this.replication.replicationLog.length;

    while (follower.lastAppliedLSN < leaderLSN) {
      await new Promise(resolve => setTimeout(resolve, 100));
    }

    // Promote
    follower.isLeader = true;

    console.log(`Follower ${follower.id} is now Leader`);
  }
}

const failover = new AutomaticFailover(replication);
failover.startHealthChecks();
```

## Multi-Leader Replication

### Write Conflicts

```javascript
// Конфликт: два datacenter одновременно обновляют один ключ

// DC1 (New York):
await db.set('user:123:status', 'active');

// DC2 (London):
await db.set('user:123:status', 'suspended');

// Какое значение должно победить? 🤔
```

### Conflict Resolution Strategies

**1. Last Write Wins (LWW)**

```javascript
class LWWConflictResolver {
  resolve(value1, value2) {
    // Использовать timestamp
    if (value1.timestamp > value2.timestamp) {
      return value1;
    } else {
      return value2;
    }
  }
}

// Проблема: можно потерять данные
// User1 в DC1: status = 'active' (timestamp: 1000)
// User2 в DC2: status = 'suspended' (timestamp: 999)
// Результат: 'active' (но 'suspended' было позже по wall-clock!)
```

**2. Version Vectors**

```javascript
class VersionVector {
  constructor() {
    this.versions = {};  // { nodeId: counter }
  }

  increment(nodeId) {
    this.versions[nodeId] = (this.versions[nodeId] || 0) + 1;
  }

  merge(other) {
    const merged = new VersionVector();

    const allNodes = new Set([
      ...Object.keys(this.versions),
      ...Object.keys(other.versions),
    ]);

    allNodes.forEach(nodeId => {
      merged.versions[nodeId] = Math.max(
        this.versions[nodeId] || 0,
        other.versions[nodeId] || 0
      );
    });

    return merged;
  }

  isDescendantOf(other) {
    return Object.keys(other.versions).every(nodeId =>
      (this.versions[nodeId] || 0) >= other.versions[nodeId]
    );
  }

  isConcurrent(other) {
    return !this.isDescendantOf(other) && !other.isDescendantOf(this);
  }
}

// Пример
const v1 = new VersionVector();
v1.versions = { A: 1, B: 0 };  // Update from DC A

const v2 = new VersionVector();
v2.versions = { A: 0, B: 1 };  // Update from DC B

console.log(v1.isConcurrent(v2));  // true → конфликт!

// Нужно merge вручную
const merged = v1.merge(v2);
console.log(merged.versions);  // { A: 1, B: 1 }
```

**3. CRDTs (Conflict-free Replicated Data Types)**

```javascript
// G-Counter (Grow-only Counter)
class GCounter {
  constructor(nodeId) {
    this.nodeId = nodeId;
    this.counts = {};  // { nodeId: count }
  }

  increment() {
    this.counts[this.nodeId] = (this.counts[this.nodeId] || 0) + 1;
  }

  value() {
    return Object.values(this.counts).reduce((sum, count) => sum + count, 0);
  }

  merge(other) {
    const merged = new GCounter(this.nodeId);

    const allNodes = new Set([
      ...Object.keys(this.counts),
      ...Object.keys(other.counts),
    ]);

    allNodes.forEach(nodeId => {
      merged.counts[nodeId] = Math.max(
        this.counts[nodeId] || 0,
        other.counts[nodeId] || 0
      );
    });

    return merged;
  }
}

// Пример: счётчик лайков
const counterDC1 = new GCounter('DC1');
const counterDC2 = new GCounter('DC2');

// DC1: 3 лайка
counterDC1.increment();
counterDC1.increment();
counterDC1.increment();

// DC2: 2 лайка
counterDC2.increment();
counterDC2.increment();

// Merge
const merged = counterDC1.merge(counterDC2);
console.log(merged.value());  // 5 (3 + 2) ✅ Нет конфликта!
```

## Leaderless Replication

Нет выделенного leader — пишем и читаем с нескольких реплик.

**Примеры:** Cassandra, DynamoDB, Riak

### Quorum Reads and Writes

```
N = количество реплик
W = write quorum (сколько должны подтвердить запись)
R = read quorum (сколько читаем)

Правило: W + R > N → гарантия чтения свежих данных

Пример: N=3, W=2, R=2
  Write: ждём подтверждения от 2 из 3 реплик
  Read: читаем с 2 из 3 реплик → хотя бы одна будет fresh
```

### Реализация Quorum

```javascript
class LeaderlessReplication {
  constructor(replicas, { N, W, R }) {
    this.replicas = replicas;
    this.N = N;
    this.W = W;
    this.R = R;
  }

  async write(key, value) {
    const version = Date.now();

    // Отправить запись всем репликам
    const writePromises = this.replicas.map(async (replica) => {
      try {
        await replica.write(key, { value, version });
        return { success: true, replica: replica.id };
      } catch (error) {
        return { success: false, replica: replica.id, error };
      }
    });

    const results = await Promise.allSettled(writePromises);

    const successCount = results.filter(
      r => r.status === 'fulfilled' && r.value.success
    ).length;

    if (successCount >= this.W) {
      console.log(`Write successful: ${successCount}/${this.N} replicas`);
      return { success: true, version };
    } else {
      throw new Error(`Write failed: only ${successCount}/${this.W} replicas confirmed`);
    }
  }

  async read(key) {
    // Читать с R реплик
    const selectedReplicas = this.replicas.slice(0, this.R);

    const readPromises = selectedReplicas.map(async (replica) => {
      try {
        const data = await replica.read(key);
        return { success: true, data, replica: replica.id };
      } catch (error) {
        return { success: false, replica: replica.id, error };
      }
    });

    const results = await Promise.all(readPromises);

    const successfulReads = results.filter(r => r.success);

    if (successfulReads.length < this.R) {
      throw new Error(`Read failed: only ${successfulReads.length}/${this.R} replicas responded`);
    }

    // Выбрать значение с наибольшим version (Last Write Wins)
    const latestData = successfulReads.reduce((latest, current) => {
      if (!latest || current.data.version > latest.data.version) {
        return current.data;
      }
      return latest;
    }, null);

    return latestData.value;
  }
}

// Использование
const replicas = [
  new Replica('A'),
  new Replica('B'),
  new Replica('C'),
];

const db = new LeaderlessReplication(replicas, {
  N: 3,
  W: 2,  // Write quorum
  R: 2,  // Read quorum
});

await db.write('user:123', { name: 'Alice' });
const user = await db.read('user:123');
```

### Read Repair

```javascript
class ReadRepair {
  async read(key) {
    const readPromises = this.replicas.map(async (replica) => ({
      replica,
      data: await replica.read(key),
    }));

    const results = await Promise.all(readPromises);

    // Найти самую свежую версию
    const latest = results.reduce((max, current) => {
      if (!max || current.data.version > max.data.version) {
        return current;
      }
      return max;
    });

    // Обновить устаревшие реплики (async)
    results.forEach(async (result) => {
      if (result.data.version < latest.data.version) {
        console.log(`Repairing replica ${result.replica.id}`);
        await result.replica.write(key, latest.data);
      }
    });

    return latest.data.value;
  }
}
```

### Anti-Entropy (Merkle Trees)

```javascript
class MerkleTree {
  constructor(data) {
    this.root = this.buildTree(data);
  }

  buildTree(data) {
    if (data.length === 0) {
      return null;
    }

    if (data.length === 1) {
      return {
        hash: this.hash(data[0]),
        data: data[0],
      };
    }

    const mid = Math.floor(data.length / 2);
    const left = this.buildTree(data.slice(0, mid));
    const right = this.buildTree(data.slice(mid));

    return {
      hash: this.hash(left.hash + right.hash),
      left,
      right,
    };
  }

  hash(value) {
    // Simplified hash function
    return require('crypto').createHash('sha256').update(String(value)).digest('hex');
  }

  compare(otherTree) {
    return this.compareNodes(this.root, otherTree.root);
  }

  compareNodes(node1, node2) {
    if (!node1 || !node2) {
      return [];
    }

    if (node1.hash === node2.hash) {
      return [];  // Идентичны
    }

    if (node1.data && node2.data) {
      // Leaf nodes differ
      return [{ data1: node1.data, data2: node2.data }];
    }

    // Recurse
    return [
      ...this.compareNodes(node1.left, node2.left),
      ...this.compareNodes(node1.right, node2.right),
    ];
  }
}

// Anti-entropy процесс
class AntiEntropy {
  async synchronize(replica1, replica2) {
    const data1 = await replica1.getAllData();
    const data2 = await replica2.getAllData();

    const tree1 = new MerkleTree(data1);
    const tree2 = new MerkleTree(data2);

    const differences = tree1.compare(tree2);

    console.log(`Found ${differences.length} differences`);

    // Синхронизировать различия
    for (const diff of differences) {
      // Использовать version для определения, какое значение новее
      if (diff.data1.version > diff.data2.version) {
        await replica2.write(diff.data1.key, diff.data1);
      } else {
        await replica1.write(diff.data2.key, diff.data2);
      }
    }
  }
}
```

## Geo-Replication

### Multi-Datacenter Setup

```javascript
class GeoReplication {
  constructor(datacenters) {
    this.datacenters = datacenters;  // { 'us-east': db, 'eu-west': db, 'ap-south': db }
  }

  async write(key, value, { primaryDC = 'us-east' } = {}) {
    const primary = this.datacenters[primaryDC];

    // Записать в primary DC (sync)
    await primary.write(key, value);

    // Реплицировать в другие DC (async)
    const otherDCs = Object.entries(this.datacenters).filter(
      ([dc]) => dc !== primaryDC
    );

    otherDCs.forEach(async ([dc, db]) => {
      try {
        await db.write(key, value);
        console.log(`Replicated to ${dc}`);
      } catch (error) {
        console.error(`Replication to ${dc} failed:`, error);
      }
    });

    return { success: true };
  }

  async read(key, { preferredDC } = {}) {
    // Читать из ближайшего DC
    const dc = preferredDC || this.getClosestDC();
    const db = this.datacenters[dc];

    return await db.read(key);
  }

  getClosestDC() {
    // Определить на основе geolocation клиента
    // Упрощённая версия
    return 'us-east';
  }
}

const geo = new GeoReplication({
  'us-east': new Database('us-east'),
  'eu-west': new Database('eu-west'),
  'ap-south': new Database('ap-south'),
});

// US клиент
await geo.write('user:123', { name: 'Alice' }, { primaryDC: 'us-east' });

// EU клиент читает (может быть stale)
const user = await geo.read('user:123', { preferredDC: 'eu-west' });
```

## Мониторинг репликации

```javascript
const promClient = require('prom-client');

const replicationLag = new promClient.Gauge({
  name: 'replication_lag_seconds',
  help: 'Replication lag in seconds',
  labelNames: ['replica'],
});

const replicationErrors = new promClient.Counter({
  name: 'replication_errors_total',
  help: 'Total replication errors',
  labelNames: ['replica', 'error_type'],
});

const replicationThroughput = new promClient.Counter({
  name: 'replication_bytes_total',
  help: 'Total bytes replicated',
  labelNames: ['replica'],
});

class ReplicationMetrics {
  async updateMetrics(replica) {
    // Lag
    const lag = await this.calculateLag(replica);
    replicationLag.set({ replica: replica.id }, lag);

    // Throughput
    const bytes = await this.getBytesReplicated(replica);
    replicationThroughput.inc({ replica: replica.id }, bytes);
  }

  async calculateLag(replica) {
    const leaderPosition = await this.getLeaderPosition();
    const replicaPosition = await replica.getPosition();

    const lag = (Date.now() - replicaPosition.timestamp) / 1000;

    return lag;
  }
}
```

## Best Practices

### 1. Мониторинг Replication Lag

```javascript
// Alert если lag > 10 секунд
if (replicationLag > 10) {
  sendAlert('High replication lag', { replica, lag: replicationLag });
}
```

### 2. Automatic Failover с осторожностью

```javascript
// Требования для failover:
// 1. Leader недоступен > threshold (30s)
// 2. Follower с минимальным lag
// 3. Manual confirmation для production
```

### 3. Quorum Tuning

```javascript
// High consistency: W=N, R=1 (медленная запись)
// High availability: W=1, R=N (риск stale reads)
// Balanced: W=quorum, R=quorum (W+R > N)

const config = {
  N: 3,
  W: 2,  // Quorum write
  R: 2,  // Quorum read
};
```

### 4. Read от Followers осторожно

```javascript
// ❌ Bad: always read from followers (stale data)
const data = await readFromFollower(key);

// ✅ Good: read from leader для critical data
const data = await readFromLeader(key);

// ✅ Good: read from follower для non-critical + check lag
const follower = selectFollowerWithLowLag();  // lag < 1s
const data = await follower.read(key);
```

## Выводы

Репликация — критический компонент для availability и scalability:

**Типы репликации:**
- **Synchronous**: strong consistency, высокая latency
- **Asynchronous**: low latency, eventual consistency
- **Semi-synchronous**: компромисс

**Архитектуры:**
- **Single-Leader**: простая, но SPOF
- **Multi-Leader**: сложнее, write conflicts
- **Leaderless**: высокая availability, quorums

**Ключевые концепции:**
- Replication lag
- Failover
- Conflict resolution
- Quorums (W + R > N)

**Trade-offs:**
- Consistency vs Latency
- Availability vs Consistency
- Simplicity vs Fault Tolerance

Выбор стратегии репликации зависит от требований к consistency, availability и performance.

## Что читать дальше?

- [Урок 34: Eventual Consistency](34-eventual-consistency.md)
- [Урок 15: Репликация: Master-Slave, Master-Master](15-replikaciya-master-slave-master-master.md)
- [Урок 31: Consensus алгоритмы](31-consensus-algorithms.md)

## Проверь себя

1. В чём разница между синхронной и асинхронной репликацией?
2. Что такое replication lag и как его мониторить?
3. Как работает automatic failover?
4. Что такое write conflicts в multi-leader репликации?
5. Как работают quorum reads and writes?
6. Что такое Read Repair?
7. Как CRDTs решают проблему конфликтов?
8. В чём разница между single-leader и leaderless репликацией?
9. Что такое geo-replication и зачем она нужна?
10. Какие trade-offs между consistency и availability в репликации?

---
**Предыдущий урок**: [Урок 32: Distributed Transactions — 2PC, Saga](32-distributed-transactions.md)
**Следующий урок**: [Урок 34: Eventual Consistency](34-eventual-consistency.md)
