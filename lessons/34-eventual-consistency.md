# Урок 34: Eventual Consistency


## Введение

Eventual Consistency — это модель consistency в распределённых системах, где гарантируется, что если не будет новых обновлений, то в конечном итоге все реплики сойдутся к одному и тому же значению.

**CAP теорема напоминание:**

В распределённой системе при network partition можно выбрать только 2 из 3:
- **Consistency**: все узлы видят одинаковые данные
- **Availability**: система всегда отвечает
- **Partition tolerance**: система работает при разделении сети

Eventual Consistency выбирает **AP** (Availability + Partition tolerance), жертвуя строгой Consistency.

## Strong Consistency vs Eventual Consistency

### Strong Consistency

```
Client 1 → Write(x=1) → Database
                            ↓ (replicate synchronously)
                         All replicas have x=1
                            ↓
Client 2 → Read(x) → returns 1 (guaranteed)
```

**Гарантия:** Read всегда возвращает последнее записанное значение.

**Цена:** Высокая latency, низкая availability при partition.

### Eventual Consistency

```
Client 1 → Write(x=1) → Database (primary)
                            ↓ (replicate asynchronously)
Client 2 → Read(x) → Replica → returns 0 (stale) ❌
                            ↓ (после некоторого времени)
                         All replicas eventually have x=1
Client 2 → Read(x) → Replica → returns 1 ✅
```

**Гарантия:** Если прекратить обновления, в конечном итоге все читатели увидят одно значение.

**Цена:** Возможны stale reads.

## Уровни Consistency

### 1. Strong Consistency (Linearizability)

```javascript
// Все операции видятся в глобальном порядке
Client 1: Write(x=1) at t1
Client 2: Read(x) at t2 (t2 > t1) → returns 1 (guaranteed)
```

**Пример:** Single-leader с синхронной репликацией, Spanner, etcd.

### 2. Sequential Consistency

```javascript
// Операции каждого клиента упорядочены, но глобальный порядок может отличаться

Client 1: Write(x=1), Write(y=2)
Client 2: Read(y=2), Read(x=1)  // OK
Client 2: Read(y=2), Read(x=0)  // OK (может не видеть x=1 ещё)
```

### 3. Causal Consistency

```javascript
// Если операция B причинно зависит от A, все увидят A перед B

// Причинная связь:
Client 1: Write(x=1)
Client 1: Write(y=2)  // причинно зависит от x=1

// Все клиенты увидят x=1 перед y=2
// Но независимые операции могут быть в любом порядке
```

### 4. Eventual Consistency

```javascript
// В конечном итоге все реплики сойдутся

Client 1: Write(x=1)
Client 2: Read(x) → может вернуть 0 или 1
Client 2: Read(x) после convergence → вернёт 1
```

### 5. Read Your Writes

```javascript
// Клиент всегда видит свои собственные записи

Client 1: Write(x=1)
Client 1: Read(x) → returns 1 (guaranteed)

Client 2: Read(x) → может вернуть 0 (eventual)
```

### 6. Monotonic Reads

```javascript
// Если клиент прочитал значение, последующие чтения не вернут более старое

Client 1: Read(x) → returns 5
Client 1: Read(x) → returns 5 или больше, но НЕ 4
```

## Реализация Eventual Consistency

### 1. Read Your Writes

```javascript
class ReadYourWritesConsistency {
  constructor(leader, replicas) {
    this.leader = leader;
    this.replicas = replicas;
    this.clientWrites = new Map();  // clientId → lastWriteTimestamp
  }

  async write(clientId, key, value) {
    const timestamp = Date.now();

    // Записать на leader
    await this.leader.write(key, { value, timestamp });

    // Сохранить timestamp последней записи клиента
    this.clientWrites.set(clientId, timestamp);

    return { success: true, timestamp };
  }

  async read(clientId, key) {
    const lastWriteTimestamp = this.clientWrites.get(clientId);

    if (lastWriteTimestamp) {
      // Клиент делал запись → читать с leader
      return await this.leader.read(key);
    }

    // Клиент не делал записи → можно читать с replica
    const replica = this.selectReplica();
    return await replica.read(key);
  }

  selectReplica() {
    return this.replicas[Math.floor(Math.random() * this.replicas.length)];
  }
}

// Использование
const consistency = new ReadYourWritesConsistency(leader, replicas);

// Client записал
await consistency.write('client-1', 'profile:name', 'Alice');

// Client читает свою запись → always fresh
const name = await consistency.read('client-1', 'profile:name');  // 'Alice'

// Другой client → может быть stale
const name2 = await consistency.read('client-2', 'profile:name');  // может быть старое значение
```

### 2. Monotonic Reads

```javascript
class MonotonicReadsConsistency {
  constructor(replicas) {
    this.replicas = replicas;
    this.clientReplicas = new Map();  // clientId → replicaId
  }

  async read(clientId, key) {
    // Привязать клиента к одной реплике
    let replicaId = this.clientReplicas.get(clientId);

    if (!replicaId) {
      // Выбрать реплику для клиента
      replicaId = this.selectReplica().id;
      this.clientReplicas.set(clientId, replicaId);
    }

    const replica = this.replicas.find(r => r.id === replicaId);

    return await replica.read(key);
  }

  selectReplica() {
    return this.replicas[Math.floor(Math.random() * this.replicas.length)];
  }
}

// Использование
const consistency = new MonotonicReadsConsistency(replicas);

// Client всегда читает с одной реплики
const value1 = await consistency.read('client-1', 'counter');  // 5
const value2 = await consistency.read('client-1', 'counter');  // 5 или больше, но не 4
```

### 3. Version Vectors для Causal Consistency

```javascript
class CausalConsistency {
  constructor() {
    this.versionVectors = new Map();  // key → VersionVector
  }

  async write(key, value, clientVector) {
    // Получить текущий version vector для ключа
    let vector = this.versionVectors.get(key) || new VersionVector();

    // Merge с client vector (causal dependencies)
    vector = vector.merge(clientVector);

    // Increment для текущего узла
    vector.increment(this.nodeId);

    // Сохранить
    this.versionVectors.set(key, vector);
    await this.storage.set(key, { value, vector });

    return vector;
  }

  async read(key) {
    const data = await this.storage.get(key);

    if (!data) {
      return { value: null, vector: new VersionVector() };
    }

    return data;
  }

  async causallySafeRead(key, clientVector) {
    const data = await this.read(key);

    // Проверить causal consistency
    if (!data.vector.isDescendantOf(clientVector)) {
      // Данные не отражают causal dependencies клиента
      // Ждать или читать с другой реплики
      await this.waitForCausalConsistency(key, clientVector);
      return await this.read(key);
    }

    return data;
  }

  async waitForCausalConsistency(key, requiredVector) {
    // Подождать пока реплика догонит
    const maxWait = 5000;
    const start = Date.now();

    while (Date.now() - start < maxWait) {
      const data = await this.read(key);

      if (data.vector.isDescendantOf(requiredVector)) {
        return;
      }

      await new Promise(resolve => setTimeout(resolve, 100));
    }

    throw new Error('Causal consistency timeout');
  }
}

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
}
```

## Conflict Resolution

### Last Write Wins (LWW)

```javascript
class LWWDatabase {
  constructor() {
    this.data = new Map();
  }

  async write(key, value) {
    const timestamp = Date.now();

    const existing = this.data.get(key);

    if (!existing || timestamp > existing.timestamp) {
      this.data.set(key, { value, timestamp });
      return { success: true, timestamp };
    }

    return { success: false, reason: 'older timestamp' };
  }

  async read(key) {
    const data = this.data.get(key);
    return data?.value;
  }
}

// Проблема LWW: потеря данных

// Replica 1:
await db1.write('user:123:status', 'active');  // timestamp: 1000

// Replica 2 (часы отстают):
await db2.write('user:123:status', 'suspended');  // timestamp: 999

// После merge: 'active' (потеря 'suspended') ❌
```

### Multi-Value (Siblings)

```javascript
class MultiValueDatabase {
  constructor() {
    this.data = new Map();  // key → Set of values with vectors
  }

  async write(key, value, clientVector) {
    let values = this.data.get(key) || new Set();

    const newVector = clientVector.clone();
    newVector.increment(this.nodeId);

    // Удалить obsolete values (которые являются предками нового)
    values = new Set([...values].filter(v => {
      if (newVector.isDescendantOf(v.vector)) {
        return false;  // obsolete
      }
      return true;
    }));

    // Добавить новое значение
    values.add({ value, vector: newVector });

    this.data.set(key, values);

    return newVector;
  }

  async read(key) {
    const values = this.data.get(key);

    if (!values || values.size === 0) {
      return { values: [], conflict: false };
    }

    if (values.size === 1) {
      return { values: [[...values][0].value], conflict: false };
    }

    // Конфликт: несколько concurrent значений (siblings)
    return {
      values: [...values].map(v => v.value),
      conflict: true,
      vectors: [...values].map(v => v.vector),
    };
  }

  async resolveConflict(key, resolvedValue, vectors) {
    // Client вручную разрешил конфликт

    const mergedVector = vectors.reduce((merged, v) => merged.merge(v), new VersionVector());
    mergedVector.increment(this.nodeId);

    this.data.set(key, new Set([{ value: resolvedValue, vector: mergedVector }]));
  }
}

// Использование
const db = new MultiValueDatabase();

// Concurrent writes
const vector1 = new VersionVector();
vector1.increment('client-1');

const vector2 = new VersionVector();
vector2.increment('client-2');

await db.write('cart', ['item-1', 'item-2'], vector1);
await db.write('cart', ['item-3'], vector2);

// Read → conflict!
const result = await db.read('cart');
console.log(result);
// {
//   conflict: true,
//   values: [['item-1', 'item-2'], ['item-3']],
//   vectors: [...]
// }

// Client resolves: merge все items
await db.resolveConflict('cart', ['item-1', 'item-2', 'item-3'], result.vectors);
```

### CRDTs (Conflict-free Replicated Data Types)

**G-Counter (Grow-only Counter):**

```javascript
class GCounter {
  constructor(nodeId) {
    this.nodeId = nodeId;
    this.counts = {};  // { nodeId: count }
  }

  increment(amount = 1) {
    this.counts[this.nodeId] = (this.counts[this.nodeId] || 0) + amount;
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

// Использование
const counter1 = new GCounter('node-1');
const counter2 = new GCounter('node-2');

counter1.increment(5);
counter2.increment(3);

const merged = counter1.merge(counter2);
console.log(merged.value());  // 8 (5 + 3)
```

**PN-Counter (Positive-Negative Counter):**

```javascript
class PNCounter {
  constructor(nodeId) {
    this.nodeId = nodeId;
    this.increments = new GCounter(nodeId);
    this.decrements = new GCounter(nodeId);
  }

  increment(amount = 1) {
    this.increments.increment(amount);
  }

  decrement(amount = 1) {
    this.decrements.increment(amount);
  }

  value() {
    return this.increments.value() - this.decrements.value();
  }

  merge(other) {
    const merged = new PNCounter(this.nodeId);
    merged.increments = this.increments.merge(other.increments);
    merged.decrements = this.decrements.merge(other.decrements);
    return merged;
  }
}

// Использование
const counter1 = new PNCounter('node-1');
const counter2 = new PNCounter('node-2');

counter1.increment(10);
counter1.decrement(3);  // 7

counter2.increment(5);   // 5

const merged = counter1.merge(counter2);
console.log(merged.value());  // 12 (10 - 3 + 5)
```

**LWW-Element-Set (Last-Write-Wins Set):**

```javascript
class LWWSet {
  constructor() {
    this.additions = new Map();  // element → timestamp
    this.removals = new Map();   // element → timestamp
  }

  add(element) {
    const timestamp = Date.now();
    this.additions.set(element, timestamp);
  }

  remove(element) {
    const timestamp = Date.now();
    this.removals.set(element, timestamp);
  }

  has(element) {
    const addedAt = this.additions.get(element) || 0;
    const removedAt = this.removals.get(element) || 0;

    // Элемент есть, если добавлен позже, чем удалён
    return addedAt > removedAt;
  }

  values() {
    const result = new Set();

    for (const element of this.additions.keys()) {
      if (this.has(element)) {
        result.add(element);
      }
    }

    return result;
  }

  merge(other) {
    const merged = new LWWSet();

    // Merge additions
    const allAddedElements = new Set([
      ...this.additions.keys(),
      ...other.additions.keys(),
    ]);

    allAddedElements.forEach(element => {
      merged.additions.set(
        element,
        Math.max(
          this.additions.get(element) || 0,
          other.additions.get(element) || 0
        )
      );
    });

    // Merge removals
    const allRemovedElements = new Set([
      ...this.removals.keys(),
      ...other.removals.keys(),
    ]);

    allRemovedElements.forEach(element => {
      merged.removals.set(
        element,
        Math.max(
          this.removals.get(element) || 0,
          other.removals.get(element) || 0
        )
      );
    });

    return merged;
  }
}

// Использование: Shopping cart
const cart1 = new LWWSet();
const cart2 = new LWWSet();

cart1.add('item-1');
cart1.add('item-2');

cart2.add('item-3');
cart2.remove('item-1');  // Concurrent removal

const merged = cart1.merge(cart2);
console.log([...merged.values()]);  // ['item-2', 'item-3']
// item-1 removed (если removal timestamp > addition timestamp)
```

## Anti-Entropy и Gossip Protocols

### Gossip Protocol

```javascript
class GossipProtocol {
  constructor(nodeId, peers) {
    this.nodeId = nodeId;
    this.peers = peers;
    this.data = new Map();
    this.versions = new Map();  // key → version
    this.gossipInterval = 1000;  // 1 секунда
  }

  start() {
    setInterval(() => this.gossip(), this.gossipInterval);
  }

  async gossip() {
    // Выбрать случайного peer
    const peer = this.selectRandomPeer();

    if (!peer) return;

    // Отправить своё состояние
    const myState = this.getState();

    const peerState = await peer.exchange(myState);

    // Merge состояние peer
    this.mergeState(peerState);
  }

  selectRandomPeer() {
    if (this.peers.length === 0) return null;
    return this.peers[Math.floor(Math.random() * this.peers.length)];
  }

  getState() {
    return {
      nodeId: this.nodeId,
      data: Object.fromEntries(this.data),
      versions: Object.fromEntries(this.versions),
    };
  }

  async exchange(peerState) {
    // Merge peer state
    this.mergeState(peerState);

    // Вернуть своё состояние
    return this.getState();
  }

  mergeState(peerState) {
    Object.entries(peerState.data).forEach(([key, value]) => {
      const myVersion = this.versions.get(key) || 0;
      const peerVersion = peerState.versions[key] || 0;

      if (peerVersion > myVersion) {
        // Peer имеет более свежую версию
        this.data.set(key, value);
        this.versions.set(key, peerVersion);

        console.log(`[${this.nodeId}] Updated ${key} from ${peerState.nodeId}`);
      }
    });
  }

  set(key, value) {
    const currentVersion = this.versions.get(key) || 0;
    const newVersion = currentVersion + 1;

    this.data.set(key, value);
    this.versions.set(key, newVersion);

    console.log(`[${this.nodeId}] Set ${key}=${value} (version ${newVersion})`);
  }

  get(key) {
    return this.data.get(key);
  }
}

// Создать кластер
const node1 = new GossipProtocol('node-1', []);
const node2 = new GossipProtocol('node-2', []);
const node3 = new GossipProtocol('node-3', []);

node1.peers = [node2, node3];
node2.peers = [node1, node3];
node3.peers = [node1, node2];

// Запустить gossip
node1.start();
node2.start();
node3.start();

// Записать на node1
node1.set('x', 10);

// Через некоторое время node2 и node3 получат x=10 через gossip
setTimeout(() => {
  console.log('Node 2:', node2.get('x'));  // 10
  console.log('Node 3:', node3.get('x'));  // 10
}, 5000);
```

## Практические примеры

### 1. Social Media Feed

```javascript
class SocialFeed {
  constructor() {
    this.posts = new Map();  // postId → post
    this.userFeeds = new Map();  // userId → Set of postIds
  }

  async createPost(userId, content) {
    const postId = this.generateId();
    const timestamp = Date.now();

    const post = {
      id: postId,
      userId,
      content,
      timestamp,
      likes: 0,
    };

    // Записать пост (eventual consistency OK)
    this.posts.set(postId, post);

    // Добавить в фиды followers (async)
    this.fanOutToFollowers(userId, postId);

    return postId;
  }

  async fanOutToFollowers(userId, postId) {
    const followers = await this.getFollowers(userId);

    // Eventual: followers увидят пост не сразу
    followers.forEach(async (followerId) => {
      const feed = this.userFeeds.get(followerId) || new Set();
      feed.add(postId);
      this.userFeeds.set(followerId, feed);
    });
  }

  async getFeed(userId) {
    const feedPostIds = this.userFeeds.get(userId) || new Set();

    const posts = [...feedPostIds]
      .map(postId => this.posts.get(postId))
      .filter(post => post)  // Могут быть не все посты ещё
      .sort((a, b) => b.timestamp - a.timestamp);

    return posts;
  }

  async likePost(postId) {
    const post = this.posts.get(postId);

    if (post) {
      // Increment counter (eventual consistency)
      post.likes++;
      this.posts.set(postId, post);
    }
  }

  generateId() {
    return `post_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  async getFollowers(userId) {
    // Mock
    return ['user-2', 'user-3', 'user-4'];
  }
}

// Eventual consistency допустима:
// - Follower увидит пост через несколько секунд (OK)
// - Счётчик лайков может быть неточным (OK)
// - Критично: сам пост не потеряется (durability)
```

### 2. Shopping Cart

```javascript
class ShoppingCart {
  constructor() {
    // Используем CRDT для корзины
    this.carts = new Map();  // userId → LWWSet
  }

  addItem(userId, item) {
    let cart = this.carts.get(userId);

    if (!cart) {
      cart = new LWWSet();
      this.carts.set(userId, cart);
    }

    cart.add(JSON.stringify(item));

    return { success: true };
  }

  removeItem(userId, item) {
    const cart = this.carts.get(userId);

    if (cart) {
      cart.remove(JSON.stringify(item));
    }

    return { success: true };
  }

  getCart(userId) {
    const cart = this.carts.get(userId);

    if (!cart) {
      return [];
    }

    return [...cart.values()].map(item => JSON.parse(item));
  }

  merge(userId, otherCart) {
    let cart = this.carts.get(userId);

    if (!cart) {
      cart = new LWWSet();
      this.carts.set(userId, cart);
    }

    cart = cart.merge(otherCart);
    this.carts.set(userId, cart);
  }
}

// Сценарий: пользователь добавляет items с mobile и web одновременно

const cartMobile = new ShoppingCart();
const cartWeb = new ShoppingCart();

// Mobile
cartMobile.addItem('user-1', { id: 'item-1', name: 'Phone' });

// Web (одновременно)
cartWeb.addItem('user-1', { id: 'item-2', name: 'Laptop' });

// Merge (eventual consistency)
const mobileCRDT = cartMobile.carts.get('user-1');
const webCRDT = cartWeb.carts.get('user-1');

const merged = mobileCRDT.merge(webCRDT);

console.log([...merged.values()]);
// ['{"id":"item-1","name":"Phone"}', '{"id":"item-2","name":"Laptop"}']
// Оба items в корзине ✅
```

## Мониторинг Eventual Consistency

```javascript
const promClient = require('prom-client');

const replicationLag = new promClient.Histogram({
  name: 'eventual_consistency_lag_seconds',
  help: 'Time until eventual consistency achieved',
  labelNames: ['operation'],
  buckets: [0.1, 0.5, 1, 2, 5, 10, 30, 60],
});

const staleReads = new promClient.Counter({
  name: 'stale_reads_total',
  help: 'Total stale reads detected',
  labelNames: ['key'],
});

class ConsistencyMonitor {
  async trackWrite(key, value) {
    const writeTimestamp = Date.now();

    // Записать timestamp
    await redis.set(`write_timestamp:${key}`, writeTimestamp);

    // Проверить когда все реплики увидят значение
    this.checkConvergence(key, value, writeTimestamp);
  }

  async checkConvergence(key, expectedValue, writeTimestamp) {
    const checkInterval = setInterval(async () => {
      const allConverged = await this.allReplicasHaveValue(key, expectedValue);

      if (allConverged) {
        clearInterval(checkInterval);

        const lag = (Date.now() - writeTimestamp) / 1000;

        replicationLag.observe({ operation: 'write' }, lag);

        console.log(`Convergence achieved in ${lag}s`);
      }
    }, 100);
  }

  async allReplicasHaveValue(key, expectedValue) {
    const checks = replicas.map(async (replica) => {
      const value = await replica.get(key);
      return value === expectedValue;
    });

    const results = await Promise.all(checks);

    return results.every(r => r === true);
  }

  async detectStaleRead(key, readValue) {
    const latestValue = await leader.get(key);

    if (readValue !== latestValue) {
      staleReads.inc({ key });
      console.warn(`Stale read detected for ${key}`);
    }
  }
}
```

## Best Practices

### 1. Выбирайте правильный уровень consistency

```javascript
// Critical data: strong consistency
const balance = await readFromLeader('account:balance');

// Non-critical: eventual consistency
const profileViews = await readFromReplica('profile:views');
```

### 2. Используйте idempotent операции

```javascript
// ❌ Bad: не идемпотентно
function incrementLikes(postId) {
  const post = db.get(postId);
  post.likes++;
  db.set(postId, post);
}

// ✅ Good: идемпотентно с CRDTs
function incrementLikes(postId, nodeId) {
  const counter = db.getCounter(postId);
  counter.increment(nodeId);  // GCounter - идемпотентный
  db.setCounter(postId, counter);
}
```

### 3. Показывайте UI feedback

```javascript
// Optimistic UI update
async function likePost(postId) {
  // Сразу показать в UI
  ui.updateLikes(postId, likes + 1);

  try {
    // Асинхронная запись
    await api.likePost(postId);
  } catch (error) {
    // Rollback в UI
    ui.updateLikes(postId, likes);
    ui.showError('Failed to like post');
  }
}
```

### 4. Документируйте consistency гарантии

```javascript
/**
 * Get user profile
 * Consistency: Eventual (may be up to 5s stale)
 * Use case: Profile viewing (stale data acceptable)
 */
async function getProfile(userId) {
  return await readFromReplica(`profile:${userId}`);
}

/**
 * Get account balance
 * Consistency: Strong (always fresh)
 * Use case: Financial operations (must be accurate)
 */
async function getBalance(accountId) {
  return await readFromLeader(`account:${accountId}:balance`);
}
```

## Выводы

Eventual Consistency — powerful model для высоко доступных распределённых систем:

**Ключевые концепции:**
- **Convergence**: в конечном итоге все реплики сойдутся
- **Conflicts**: необходимы стратегии разрешения
- **CRDTs**: conflict-free data types
- **Gossip**: anti-entropy механизм

**Уровни consistency:**
- Strong → Eventual (trade-off consistency за availability)
- Read Your Writes
- Monotonic Reads
- Causal Consistency

**Когда использовать:**
- Высокая availability критична
- Geo-распределённые системы
- Stale данные допустимы (social feeds, likes, views)

**Когда НЕ использовать:**
- Financial transactions
- Inventory management
- Любые критичные операции где stale data = проблема

Eventual Consistency позволяет строить highly available системы, но требует careful design для handling conflicts и stale data.

## Что читать дальше?

- [Урок 35: Дизайн URL Shortener](35-design-url-shortener.md)
- [Урок 5: CAP теорема, ACID vs BASE](05-cap-teorema-acid-vs-base.md)
- [Урок 33: Репликация данных](33-data-replication.md)

## Проверь себя

1. Что такое Eventual Consistency?
2. В чём разница между Strong и Eventual Consistency?
3. Что такое Read Your Writes consistency?
4. Как работают Version Vectors?
5. Что такое CRDTs и какие проблемы они решают?
6. Как работает Gossip Protocol?
7. В чём разница между LWW и Multi-Value conflict resolution?
8. Когда использовать Eventual вместо Strong Consistency?
9. Как мониторить convergence time?
10. Какие trade-offs у Eventual Consistency?

---
**Предыдущий урок**: [Урок 33: Репликация данных](33-data-replication.md)
**Следующий урок**: [Урок 35: Дизайн URL Shortener (bit.ly)](35-design-url-shortener.md)
