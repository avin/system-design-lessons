# Урок 38: Распределённое хранилище ключ-значение

## Введение

Распределённое хранилище ключ-значение (Distributed Key-Value Store) — это фундаментальный building block современных систем. Такие системы, как Redis, DynamoDB, Cassandra, используются везде: от кэширования и сессий до primary database для миллионов запросов в секунду.

В этом уроке мы спроектируем собственное распределённое key-value хранилище, которое будет:
- Масштабироваться горизонтально
- Обеспечивать высокую доступность
- Реплицировать данные для надёжности
- Поддерживать eventual consistency

**Примеры реальных систем:**
- **Redis** — in-memory, single-threaded, persistence опционально
- **DynamoDB** — managed, multi-tenant, horizontal scaling
- **Cassandra** — wide-column, masterless, eventual consistency
- **etcd** — strongly consistent, используется в Kubernetes

## Требования к системе

### Функциональные требования

1. **Базовые операции:**
   - `put(key, value)` — сохранить значение
   - `get(key)` → value — получить значение
   - `delete(key)` — удалить значение

2. **Масштабирование:**
   - Поддержка петабайтов данных
   - Горизонтальное масштабирование (добавление узлов)

3. **Надёжность:**
   - Репликация данных (N копий)
   - Автоматическое восстановление после сбоев

### Нефункциональные требования

1. **Высокая доступность**: 99.99% (< 1 час downtime в год)
2. **Производительность**:
   - Latency: < 10ms (p99)
   - Throughput: 100K+ ops/sec на узел
3. **Консистентность**: Eventual consistency (CAP theorem: AP > CP)
4. **Fault tolerance**: Работа при отказе N-1 узлов (при репликации N)

### Оценка масштаба (Back-of-the-envelope)

- **Data size**: 1 PB (1000 TB)
- **Node capacity**: 2 TB SSD
- **Nodes needed**: 1000 TB / 2 TB = 500 узлов (без репликации)
- **With 3x replication**: 1500 узлов
- **QPS**: 1M ops/sec → ~667 ops/sec на узел (с репликацией)

## Ключевые концепции

### 1. Consistent Hashing

Проблема обычного хэширования:
```
hash(key) % N = server_id

Есть 3 сервера (N=3):
key="user:123" → hash=456 → 456 % 3 = 0 → server 0

Добавили сервер (N=4):
key="user:123" → hash=456 → 456 % 4 = 0 → server 0 ✅ (повезло)
key="user:456" → hash=789 → 789 % 3 = 0, но 789 % 4 = 1 ❌

Почти все ключи переместятся!
```

**Consistent Hashing решение:**

```
         Hash Ring (0 - 2^32-1)
              ┌───────────────┐
        Server C               Server A
         (hash=3B)              (hash=0A)
              │                     │
              │                     │
              │                     │
         Server B ─────────────────
          (hash=2F)

Ключи попадают на первый сервер по часовой:
key1 (hash=05) → Server A (0A)
key2 (hash=1A) → Server B (2F)
key3 (hash=35) → Server C (3B)

Добавили Server D (hash=15):
         ┌───────────────┐
    Server C         Server A
     (hash=3B)        (hash=0A)
         │                 │
    Server D          Server D
     (hash=15)             │
         │            Server B
    Server B ─────────────
     (hash=2F)

key1 (hash=05) → Server A (без изменений)
key2 (hash=1A) → Server D (переехал с B на D)
key3 (hash=35) → Server C (без изменений)

Переместился только 1 ключ из 3!
```

**Реализация:**

```javascript
const crypto = require('crypto');

class ConsistentHashing {
  constructor(nodes = [], virtualNodesCount = 150) {
    this.virtualNodesCount = virtualNodesCount;
    this.ring = new Map(); // hash → node
    this.sortedHashes = [];

    nodes.forEach(node => this.addNode(node));
  }

  hash(key) {
    return crypto
      .createHash('md5')
      .update(key)
      .digest('hex')
      .slice(0, 8); // первые 8 символов (32 бита)
  }

  hashToInt(hashStr) {
    return parseInt(hashStr, 16);
  }

  addNode(node) {
    // Добавляем virtual nodes для равномерного распределения
    for (let i = 0; i < this.virtualNodesCount; i++) {
      const virtualKey = `${node.id}:vnode:${i}`;
      const hash = this.hash(virtualKey);
      const hashInt = this.hashToInt(hash);

      this.ring.set(hashInt, node);
      this.sortedHashes.push(hashInt);
    }

    // Сортируем хэши
    this.sortedHashes.sort((a, b) => a - b);

    console.log(`Added node ${node.id} with ${this.virtualNodesCount} virtual nodes`);
  }

  removeNode(nodeId) {
    // Удаляем все virtual nodes этого узла
    const hashesToRemove = [];

    for (const [hash, node] of this.ring.entries()) {
      if (node.id === nodeId) {
        hashesToRemove.push(hash);
      }
    }

    hashesToRemove.forEach(hash => {
      this.ring.delete(hash);
      const index = this.sortedHashes.indexOf(hash);
      if (index > -1) {
        this.sortedHashes.splice(index, 1);
      }
    });

    console.log(`Removed node ${nodeId}`);
  }

  getNode(key) {
    if (this.ring.size === 0) {
      throw new Error('No nodes available');
    }

    const hash = this.hash(key);
    const hashInt = this.hashToInt(hash);

    // Бинарный поиск первого хэша >= hashInt
    let idx = this.binarySearch(hashInt);

    // Если не нашли (hashInt больше всех) — берём первый (ring замыкается)
    if (idx >= this.sortedHashes.length) {
      idx = 0;
    }

    const nodeHash = this.sortedHashes[idx];
    return this.ring.get(nodeHash);
  }

  binarySearch(target) {
    let left = 0;
    let right = this.sortedHashes.length - 1;

    while (left <= right) {
      const mid = Math.floor((left + right) / 2);

      if (this.sortedHashes[mid] === target) {
        return mid;
      } else if (this.sortedHashes[mid] < target) {
        left = mid + 1;
      } else {
        right = mid - 1;
      }
    }

    return left; // первый элемент >= target
  }

  // Получить N узлов для репликации
  getNodes(key, count = 3) {
    if (this.ring.size === 0) return [];

    const nodes = new Set();
    const hash = this.hash(key);
    const hashInt = this.hashToInt(hash);

    let idx = this.binarySearch(hashInt);

    while (nodes.size < count && nodes.size < this.getUniqueNodesCount()) {
      if (idx >= this.sortedHashes.length) {
        idx = 0; // wrap around
      }

      const nodeHash = this.sortedHashes[idx];
      const node = this.ring.get(nodeHash);
      nodes.add(node);

      idx++;
    }

    return Array.from(nodes);
  }

  getUniqueNodesCount() {
    const uniqueNodes = new Set();
    for (const node of this.ring.values()) {
      uniqueNodes.add(node.id);
    }
    return uniqueNodes.size;
  }
}

// Использование
const nodes = [
  { id: 'node-1', host: '10.0.1.1', port: 8001 },
  { id: 'node-2', host: '10.0.1.2', port: 8002 },
  { id: 'node-3', host: '10.0.1.3', port: 8003 }
];

const ring = new ConsistentHashing(nodes);

// Найти узел для ключа
const node = ring.getNode('user:12345');
console.log(`Key "user:12345" → ${node.id}`);

// Найти 3 узла для репликации
const replicaNodes = ring.getNodes('user:12345', 3);
console.log('Replica nodes:', replicaNodes.map(n => n.id));

// Добавить новый узел
ring.addNode({ id: 'node-4', host: '10.0.1.4', port: 8004 });
```

### 2. Replication

Типы репликации:

**1) Single-Leader (Master-Slave)**

```
        Client
          │
          ↓
     ┌─────────┐
     │  Leader │ ← writes идут сюда
     └────┬────┘
          │ replicate
     ┌────┼────┐
     ↓    ↓    ↓
  ┌───┐ ┌───┐ ┌───┐
  │ R1│ │ R2│ │ R3│ ← reads могут идти отсюда
  └───┘ └───┘ └───┘
```

Плюсы: Простота, strong consistency
Минусы: Single point of failure, bottleneck на leader

**2) Multi-Leader**

```
  Client A          Client B
     │                 │
     ↓                 ↓
┌─────────┐       ┌─────────┐
│ Leader 1│◄─────►│ Leader 2│
└─────────┘       └─────────┘
     │                 │
     └────────┬────────┘
              ↓
         Replicas
```

Плюсы: Высокая доступность
Минусы: Конфликты при одновременной записи

**3) Leaderless (используем в нашем дизайне)**

```
      Client
    ┌───┼───┐
    ↓   ↓   ↓
  ┌───┐┌───┐┌───┐
  │ N1││ N2││ N3│ ← все узлы равноправны
  └───┘└───┘└───┘
     ↕    ↕    ↕
    Gossip Protocol
```

Плюсы: Нет single point of failure, масштабируемость
Минусы: Eventual consistency, нужны conflict resolution

### 3. Quorum Reads/Writes

Для обеспечения консистентности в leaderless системе:

```
N = количество реплик
W = количество узлов, подтверждающих запись (write quorum)
R = количество узлов для чтения (read quorum)

Правило: W + R > N

Пример: N=3, W=2, R=2
При записи ждём подтверждения от 2 узлов.
При чтении запрашиваем 2 узла.
Гарантия: хотя бы 1 узел вернёт последнюю версию.
```

```
PUT key=X, value=V1, version=1

┌────┐  PUT   ┌────┐
│ N1 │◄───────┤Client│
└────┘        └────┘
  │ OK           │
  │              │ PUT
  ↓              ↓
┌────┐         ┌────┐
│ N2 │         │ N3 │
└────┘         └────┘
  │ OK           ✗ timeout
  └──────────────┘
     W=2 OK!

GET key=X

┌────┐         ┌────┐
│ N1 │         │ N2 │
└─┬──┘         └─┬──┘
  │ V1, ver=1    │ V1, ver=1
  └──────┬───────┘
         ↓
    Client picks latest: V1
```

## Архитектура системы

### Компоненты

```
┌─────────────────────────────────────────────┐
│               Coordinator                    │
│  (API Gateway, Request Routing)             │
└──────────────┬──────────────────────────────┘
               │
     ┌─────────┼─────────┐
     │         │         │
┌────▼────┐ ┌──▼──────┐ ┌▼────────┐
│ Node 1  │ │ Node 2  │ │ Node 3  │
│ ┌──────┐│ │ ┌──────┐│ │ ┌──────┐│
│ │ Data ││ │ │ Data ││ │ │ Data ││
│ │Engine││ │ │Engine││ │ │Engine││
│ └──────┘│ │ └──────┘│ │ └──────┘│
│ ┌──────┐│ │ ┌──────┐│ │ ┌──────┐│
│ │Gossip││◄┼─┤│Gossip││◄┼─┤│Gossip││
│ └──────┘│ │ └──────┘│ │ └──────┘│
└─────────┘ └─────────┘ └─────────┘
```

### Реализация узла (Node)

```javascript
const express = require('express');
const level = require('level'); // LevelDB для хранения

class KVNode {
  constructor(id, port, peers = []) {
    this.id = id;
    this.port = port;
    this.peers = peers; // другие узлы
    this.db = level(`./data/${id}`);
    this.metadata = new Map(); // version, timestamp
    this.app = express();

    this.setupRoutes();
    this.startGossip();
  }

  setupRoutes() {
    this.app.use(express.json());

    // Internal API (для координатора)
    this.app.put('/internal/put', async (req, res) => {
      try {
        const { key, value, version, timestamp } = req.body;

        // Проверяем версию (conflict resolution)
        const existing = await this.getWithMetadata(key);

        if (existing && existing.version >= version) {
          return res.json({
            success: false,
            message: 'Older version, rejected'
          });
        }

        // Сохраняем
        await this.db.put(key, JSON.stringify(value));

        this.metadata.set(key, { version, timestamp });

        res.json({ success: true, node: this.id });
      } catch (error) {
        res.status(500).json({ success: false, error: error.message });
      }
    });

    this.app.get('/internal/get', async (req, res) => {
      try {
        const { key } = req.query;
        const data = await this.getWithMetadata(key);

        res.json({
          success: true,
          data,
          node: this.id
        });
      } catch (error) {
        if (error.notFound) {
          res.status(404).json({ success: false, message: 'Not found' });
        } else {
          res.status(500).json({ success: false, error: error.message });
        }
      }
    });

    this.app.delete('/internal/delete', async (req, res) => {
      try {
        const { key, version } = req.body;

        await this.db.del(key);
        this.metadata.delete(key);

        res.json({ success: true, node: this.id });
      } catch (error) {
        res.status(500).json({ success: false, error: error.message });
      }
    });

    // Health check
    this.app.get('/health', (req, res) => {
      res.json({ status: 'healthy', node: this.id });
    });
  }

  async getWithMetadata(key) {
    try {
      const value = await this.db.get(key);
      const meta = this.metadata.get(key) || { version: 1, timestamp: Date.now() };

      return {
        value: JSON.parse(value),
        version: meta.version,
        timestamp: meta.timestamp
      };
    } catch (error) {
      throw error;
    }
  }

  startGossip() {
    // Gossip protocol для обмена metadata
    setInterval(() => {
      this.peers.forEach(peer => {
        this.syncWithPeer(peer);
      });
    }, 5000); // каждые 5 секунд
  }

  async syncWithPeer(peer) {
    // Упрощённая версия Gossip
    try {
      const axios = require('axios');

      // Отправляем наши метаданные
      const ourKeys = Array.from(this.metadata.entries()).map(([key, meta]) => ({
        key,
        version: meta.version,
        timestamp: meta.timestamp
      }));

      await axios.post(`http://${peer.host}:${peer.port}/gossip`, {
        node: this.id,
        keys: ourKeys
      });
    } catch (error) {
      console.error(`Failed to sync with ${peer.host}:${peer.port}:`, error.message);
    }
  }

  start() {
    this.app.listen(this.port, () => {
      console.log(`KV Node ${this.id} listening on port ${this.port}`);
    });
  }
}

// Запуск узла
const node = new KVNode('node-1', 8001, [
  { host: 'localhost', port: 8002 },
  { host: 'localhost', port: 8003 }
]);

node.start();
```

### Реализация координатора

```javascript
const axios = require('axios');

class KVCoordinator {
  constructor(consistentHashing, replicationFactor = 3, quorum = { w: 2, r: 2 }) {
    this.ring = consistentHashing;
    this.replicationFactor = replicationFactor;
    this.quorum = quorum;
    this.app = express();

    this.setupRoutes();
  }

  setupRoutes() {
    this.app.use(express.json());

    // Public API
    this.app.put('/kv/:key', async (req, res) => {
      try {
        const { key } = req.params;
        const { value } = req.body;

        const result = await this.put(key, value);

        res.json(result);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    this.app.get('/kv/:key', async (req, res) => {
      try {
        const { key } = req.params;

        const result = await this.get(key);

        if (result.found) {
          res.json({ key, value: result.value });
        } else {
          res.status(404).json({ error: 'Not found' });
        }
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    this.app.delete('/kv/:key', async (req, res) => {
      try {
        const { key } = req.params;

        const result = await this.delete(key);

        res.json(result);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
  }

  async put(key, value) {
    // Получаем N узлов для репликации
    const nodes = this.ring.getNodes(key, this.replicationFactor);

    const version = Date.now(); // упрощённая версия (в реальности: vector clock)
    const timestamp = Date.now();

    // Отправляем запрос на все реплики
    const promises = nodes.map(node =>
      this.putToNode(node, key, value, version, timestamp)
    );

    // Ждём quorum.w подтверждений
    const results = await Promise.allSettled(promises);

    const successCount = results.filter(r => r.status === 'fulfilled' && r.value.success).length;

    if (successCount >= this.quorum.w) {
      return {
        success: true,
        replicas: successCount,
        required: this.quorum.w
      };
    } else {
      throw new Error(`Quorum not met: ${successCount}/${this.quorum.w}`);
    }
  }

  async get(key) {
    const nodes = this.ring.getNodes(key, this.replicationFactor);

    // Отправляем запрос на quorum.r узлов
    const promises = nodes.slice(0, this.quorum.r).map(node =>
      this.getFromNode(node, key)
    );

    const results = await Promise.allSettled(promises);

    const successResults = results
      .filter(r => r.status === 'fulfilled' && r.value.data)
      .map(r => r.value.data);

    if (successResults.length === 0) {
      return { found: false };
    }

    // Выбираем последнюю версию (conflict resolution)
    const latest = successResults.reduce((max, current) => {
      return current.version > max.version ? current : max;
    });

    // Read repair: если нашли разные версии, обновляем отстающие реплики
    this.readRepair(key, latest, nodes);

    return {
      found: true,
      value: latest.value,
      version: latest.version
    };
  }

  async delete(key) {
    const nodes = this.ring.getNodes(key, this.replicationFactor);
    const version = Date.now();

    const promises = nodes.map(node =>
      this.deleteFromNode(node, key, version)
    );

    const results = await Promise.allSettled(promises);
    const successCount = results.filter(r => r.status === 'fulfilled').length;

    return {
      success: successCount >= this.quorum.w,
      replicas: successCount
    };
  }

  async putToNode(node, key, value, version, timestamp) {
    try {
      const response = await axios.put(
        `http://${node.host}:${node.port}/internal/put`,
        { key, value, version, timestamp },
        { timeout: 1000 }
      );

      return response.data;
    } catch (error) {
      console.error(`Failed to PUT to ${node.id}:`, error.message);
      throw error;
    }
  }

  async getFromNode(node, key) {
    try {
      const response = await axios.get(
        `http://${node.host}:${node.port}/internal/get`,
        { params: { key }, timeout: 1000 }
      );

      return response.data;
    } catch (error) {
      console.error(`Failed to GET from ${node.id}:`, error.message);
      throw error;
    }
  }

  async deleteFromNode(node, key, version) {
    try {
      const response = await axios.delete(
        `http://${node.host}:${node.port}/internal/delete`,
        { data: { key, version }, timeout: 1000 }
      );

      return response.data;
    } catch (error) {
      console.error(`Failed to DELETE from ${node.id}:`, error.message);
      throw error;
    }
  }

  async readRepair(key, latestData, nodes) {
    // Background task: обновляем отстающие реплики
    setImmediate(async () => {
      const promises = nodes.map(node =>
        this.putToNode(node, key, latestData.value, latestData.version, latestData.timestamp)
          .catch(err => console.error(`Read repair failed for ${node.id}:`, err.message))
      );

      await Promise.allSettled(promises);
      console.log(`Read repair completed for key: ${key}`);
    });
  }

  start(port) {
    this.app.listen(port, () => {
      console.log(`KV Coordinator listening on port ${port}`);
    });
  }
}

// Запуск
const nodes = [
  { id: 'node-1', host: 'localhost', port: 8001 },
  { id: 'node-2', host: 'localhost', port: 8002 },
  { id: 'node-3', host: 'localhost', port: 8003 }
];

const ring = new ConsistentHashing(nodes);

const coordinator = new KVCoordinator(ring, 3, { w: 2, r: 2 });
coordinator.start(9000);
```

## Продвинутые техники

### 1. Vector Clocks для conflict resolution

Проблема: как определить, какая версия новее при concurrent writes?

```
Client A           Client B
   │                  │
   ├─ PUT X=1 ────────┼─→ Node1: X=1, version=?
   │                  │
   │                  ├─→ PUT X=2 ─→ Node2: X=2, version=?
   │                  │
   Какая версия последняя?
```

**Решение: Vector Clock**

```javascript
class VectorClock {
  constructor() {
    this.clock = {}; // { nodeId: counter }
  }

  increment(nodeId) {
    this.clock[nodeId] = (this.clock[nodeId] || 0) + 1;
  }

  merge(otherClock) {
    const merged = new VectorClock();

    const allNodes = new Set([
      ...Object.keys(this.clock),
      ...Object.keys(otherClock.clock)
    ]);

    allNodes.forEach(nodeId => {
      merged.clock[nodeId] = Math.max(
        this.clock[nodeId] || 0,
        otherClock.clock[nodeId] || 0
      );
    });

    return merged;
  }

  // Сравнение часов
  compare(other) {
    let thisGreater = false;
    let otherGreater = false;

    const allNodes = new Set([
      ...Object.keys(this.clock),
      ...Object.keys(other.clock)
    ]);

    for (const nodeId of allNodes) {
      const thisValue = this.clock[nodeId] || 0;
      const otherValue = other.clock[nodeId] || 0;

      if (thisValue > otherValue) {
        thisGreater = true;
      } else if (thisValue < otherValue) {
        otherGreater = true;
      }
    }

    if (thisGreater && !otherGreater) return 1;  // this > other
    if (otherGreater && !thisGreater) return -1; // this < other
    if (!thisGreater && !otherGreater) return 0; // equal
    return null; // concurrent (conflict!)
  }

  toString() {
    return JSON.stringify(this.clock);
  }
}

// Использование
const clockA = new VectorClock();
clockA.increment('node-1'); // { node-1: 1 }

const clockB = new VectorClock();
clockB.increment('node-2'); // { node-2: 1 }

const comparison = clockA.compare(clockB);
if (comparison === null) {
  console.log('Conflict detected! Need merge strategy');
}
```

### 2. Anti-Entropy via Merkle Trees

Для эффективной синхронизации данных между репликами используем Merkle Tree.

```
          Root
         /    \
      H(1-2)  H(3-4)
      /  \     /  \
    H(1) H(2) H(3) H(4)
     │    │    │    │
    D1   D2   D3   D4

Сравниваем корневой hash:
- Если равны → реплики синхронны
- Если не равны → спускаемся вниз, находим отличия
```

**Реализация:**

```javascript
const crypto = require('crypto');

class MerkleTree {
  constructor(data) {
    this.leaves = data.map(item => this.hash(JSON.stringify(item)));
    this.tree = this.buildTree(this.leaves);
  }

  hash(data) {
    return crypto.createHash('sha256').update(data).digest('hex');
  }

  buildTree(nodes) {
    if (nodes.length === 0) return [];
    if (nodes.length === 1) return nodes;

    const tree = [nodes];
    let currentLevel = nodes;

    while (currentLevel.length > 1) {
      const nextLevel = [];

      for (let i = 0; i < currentLevel.length; i += 2) {
        const left = currentLevel[i];
        const right = currentLevel[i + 1] || left; // если нечётное количество
        const combined = this.hash(left + right);
        nextLevel.push(combined);
      }

      tree.push(nextLevel);
      currentLevel = nextLevel;
    }

    return tree;
  }

  getRoot() {
    return this.tree[this.tree.length - 1][0];
  }

  // Сравнение двух деревьев
  static findDifferences(tree1, tree2, data1, data2) {
    if (tree1.getRoot() === tree2.getRoot()) {
      return []; // одинаковые
    }

    // Упрощённый алгоритм: сравниваем leaves
    const diff = [];

    for (let i = 0; i < Math.max(data1.length, data2.length); i++) {
      const hash1 = tree1.leaves[i];
      const hash2 = tree2.leaves[i];

      if (hash1 !== hash2) {
        diff.push({
          index: i,
          data1: data1[i],
          data2: data2[i]
        });
      }
    }

    return diff;
  }
}

// Использование для anti-entropy
async function syncReplicas(node1, node2) {
  // Получаем данные с обоих узлов
  const data1 = await node1.getAllKeys();
  const data2 = await node2.getAllKeys();

  const tree1 = new MerkleTree(data1);
  const tree2 = new MerkleTree(data2);

  if (tree1.getRoot() === tree2.getRoot()) {
    console.log('Replicas are in sync');
    return;
  }

  // Находим различия
  const diffs = MerkleTree.findDifferences(tree1, tree2, data1, data2);

  console.log(`Found ${diffs.length} differences, syncing...`);

  // Синхронизируем
  for (const diff of diffs) {
    // Выбираем последнюю версию (по timestamp или vector clock)
    // и реплицируем на отстающий узел
  }
}
```

### 3. Handling Network Partitions (Split Brain)

```
Network Partition:

Datacenter 1    │    Datacenter 2
  Node A        │      Node C
  Node B        │      Node D
                │
  PUT X=1       │      PUT X=2
  (version 1)   │      (version 2)

Network restored → Conflict!
```

**Стратегии разрешения:**

```javascript
class ConflictResolver {
  // 1. Last Write Wins (LWW)
  static lastWriteWins(value1, value2) {
    return value1.timestamp > value2.timestamp ? value1 : value2;
  }

  // 2. Custom merge function (для приложения)
  static customMerge(value1, value2, mergeFunction) {
    return mergeFunction(value1, value2);
  }

  // 3. Keep both versions (sibling values)
  static keepBothVersions(value1, value2) {
    return {
      conflict: true,
      versions: [value1, value2]
    };
  }
}

// Использование в Coordinator
async get(key) {
  const results = await this.getFromReplicas(key);

  if (results.length === 1) {
    return results[0];
  }

  // Обнаружен конфликт
  const resolved = ConflictResolver.lastWriteWins(results[0], results[1]);

  // Записываем разрешённую версию обратно
  await this.put(key, resolved.value);

  return resolved;
}
```

## Оптимизации

### 1. Hinted Handoff

При временной недоступности узла записываем hint на другой узел.

```
Replication nodes для key=X: [N1, N2, N3]

N2 недоступен ❌

PUT X=value
  → N1 ✅
  → N2 ❌ (недоступен)
  → N3 ✅
  → N4 (hinted handoff) 💾 "hint for N2"

N2 восстановился ✅

N4 → N2: "Here's the data you missed"
```

**Реализация:**

```javascript
class HintedHandoff {
  constructor(db) {
    this.hints = db; // hints store
  }

  async storeHint(targetNode, key, value, version) {
    const hint = {
      targetNode,
      key,
      value,
      version,
      timestamp: Date.now()
    };

    await this.hints.put(`hint:${targetNode}:${key}`, JSON.stringify(hint));
  }

  async deliverHints(targetNode) {
    const hintsPrefix = `hint:${targetNode}:`;
    const hints = await this.hints.createReadStream({
      gte: hintsPrefix,
      lte: hintsPrefix + '\xff'
    });

    for await (const { key, value } of hints) {
      const hint = JSON.parse(value);

      try {
        await this.sendToNode(targetNode, hint.key, hint.value, hint.version);
        await this.hints.del(key); // удаляем доставленный hint
      } catch (error) {
        console.error(`Failed to deliver hint to ${targetNode}:`, error);
      }
    }
  }

  async sendToNode(node, key, value, version) {
    // отправка данных на узел
  }
}
```

### 2. Read Repair

Исправление несогласованности при чтении:

```javascript
async get(key) {
  const nodes = this.ring.getNodes(key, 3);

  // Читаем с quorum узлов
  const results = await Promise.all(
    nodes.map(node => this.getFromNode(node, key))
  );

  // Находим последнюю версию
  const versions = results.filter(r => r.success).map(r => r.data);
  const latest = versions.reduce((max, curr) =>
    curr.version > max.version ? curr : max
  );

  // Обнаружили несоответствие? Исправляем
  const outdated = results.filter(r =>
    r.success && r.data.version < latest.version
  );

  if (outdated.length > 0) {
    console.log(`Read repair: updating ${outdated.length} replicas`);

    // Background repair
    setImmediate(() => {
      outdated.forEach(r => {
        this.putToNode(r.node, key, latest.value, latest.version, latest.timestamp);
      });
    });
  }

  return latest;
}
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

// Метрики
const kvOperations = new prometheus.Counter({
  name: 'kv_operations_total',
  help: 'Total KV operations',
  labelNames: ['operation', 'status']
});

const kvLatency = new prometheus.Histogram({
  name: 'kv_operation_duration_seconds',
  help: 'KV operation latency',
  labelNames: ['operation'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5]
});

const replicationLag = new prometheus.Gauge({
  name: 'kv_replication_lag_seconds',
  help: 'Replication lag between nodes'
});

// В методах
async put(key, value) {
  const start = Date.now();

  try {
    const result = await this.putInternal(key, value);

    kvOperations.inc({ operation: 'put', status: 'success' });
    kvLatency.observe({ operation: 'put' }, (Date.now() - start) / 1000);

    return result;
  } catch (error) {
    kvOperations.inc({ operation: 'put', status: 'error' });
    throw error;
  }
}
```

## Trade-offs и сравнение

| Аспект | Single-Leader | Multi-Leader | Leaderless |
|--------|---------------|--------------|------------|
| Consistency | Strong | Eventual | Eventual |
| Availability | Medium (SPOF) | High | Very High |
| Write Latency | Low (локально) | Medium | Low-Medium |
| Conflict Resolution | Не нужна | Сложная | Quorum + versioning |
| Use Case | Критична консистентность | Multi-DC, geo-distribution | Высокая доступность, AP |

## Сравнение с реальными системами

| Система | Consistency | Replication | Conflict Resolution | Use Case |
|---------|-------------|-------------|---------------------|----------|
| **Redis** | Strong (single-threaded) | Async (master-slave) | Last write wins | Cache, sessions |
| **Cassandra** | Tunable (quorum) | Leaderless | LWW, timestamps | Time-series, logs |
| **DynamoDB** | Eventual | Multi-leader | Vector clocks | Serverless apps |
| **etcd** | Strong (Raft) | Leader-based | Consensus | Config, coordination |

## Что читать дальше

- **Урок 33**: Data Replication — глубже в стратегии репликации
- **Урок 34**: Eventual Consistency — как работать с eventual consistency
- **Урок 31**: Consensus Algorithms — Raft и Paxos для strong consistency

## Проверь себя

1. Почему Consistent Hashing лучше обычного `hash(key) % N` при добавлении узлов?
2. Как Quorum (W + R > N) гарантирует чтение последней версии?
3. В чём разница между Read Repair и Anti-Entropy?
4. Когда использовать Vector Clock вместо Timestamp?
5. Спроектируйте KV store для сценария: 99.999% availability, eventual consistency OK, geo-distributed.

---

[← Урок 37: Rate Limiter](37-rate-limiter.md) | [Урок 39: Генератор уникальных идентификаторов →](39-unique-id-generator.md)
