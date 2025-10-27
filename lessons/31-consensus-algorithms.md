# Урок 31: Consensus алгоритмы — Paxos, Raft


## Введение

Consensus (консенсус) — это фундаментальная проблема в распределённых системах: как группа узлов может достичь соглашения о некотором значении, даже если некоторые узлы могут отказать или работать медленно.

**Примеры использования:**
- Выбор лидера (leader election)
- Distributed configuration store (etcd, ZooKeeper)
- Distributed databases (CockroachDB, TiDB)
- Distributed locks
- Replicated state machines

**Проблема:**

```
Три сервера пытаются выбрать лидера:

Server A думает: "Я лидер"
Server B думает: "Я лидер"
Server C думает: "A — лидер"

Кто прав? ❌ Split brain!
```

**Решение: Consensus алгоритм**

Гарантирует, что все серверы согласятся с одним и тем же решением.

## CAP теорема и Consensus

Напомним CAP теорему (из [Урока 5](05-cap-teorema-acid-vs-base.md)):

- **Consistency**: все узлы видят одинаковые данные
- **Availability**: система всегда отвечает
- **Partition tolerance**: система работает при разделении сети

**Consensus алгоритмы выбирают CP:**
- Гарантируют Consistency
- Жертвуют Availability при network partition
- Требуют большинства узлов (quorum)

## Raft

Raft — современный consensus алгоритм, разработанный для понятности (в отличие от Paxos).

### Основные концепции Raft

**Роли узлов:**

1. **Leader** — обрабатывает все клиентские запросы
2. **Follower** — пассивно реплицирует данные
3. **Candidate** — участвует в выборах лидера

**Переходы состояний:**

```
Follower → (timeout) → Candidate → (получил большинство голосов) → Leader
                            ↓
                    (не получил большинство) → Follower
```

### Leader Election

```
Шаг 1: Follower не получает heartbeat от Leader
       Таймаут (150-300ms random)
       Становится Candidate

Шаг 2: Candidate голосует за себя
       Отправляет RequestVote всем узлам
       Увеличивает term (election term)

Шаг 3: Узлы голосуют:
       - Только один голос за term
       - Голосуют за Candidate с более свежим log

Шаг 4: Если Candidate получил большинство (quorum):
       Становится Leader
       Отправляет heartbeat всем Followers
```

**Пример:**

```
5 узлов: A, B, C, D, E

1. Leader A умирает
2. B и C ждут timeout (случайный)
3. B timeout раньше → становится Candidate (term 2)
4. B отправляет RequestVote → C, D, E
5. C, D голосуют за B (E не успел ответить)
6. B получил 3/5 голосов (включая свой) → становится Leader
7. B отправляет heartbeat → Followers
```

### Log Replication

```javascript
// Simplified Raft log structure
class RaftLog {
  constructor() {
    this.entries = [];  // [{term, index, command}]
    this.commitIndex = 0;
    this.lastApplied = 0;
  }

  append(term, command) {
    const index = this.entries.length + 1;
    this.entries.push({ term, index, command });
    return index;
  }

  getEntry(index) {
    return this.entries[index - 1];
  }

  getLastEntry() {
    return this.entries[this.entries.length - 1];
  }
}

// Raft Node
class RaftNode {
  constructor(id, peers) {
    this.id = id;
    this.peers = peers;
    this.state = 'follower';  // follower, candidate, leader
    this.currentTerm = 0;
    this.votedFor = null;
    this.log = new RaftLog();

    // Leader state
    this.nextIndex = {};   // Для каждого peer
    this.matchIndex = {};  // Для каждого peer

    this.electionTimeout = this.randomTimeout(150, 300);
    this.heartbeatInterval = 50;  // 50ms
    this.lastHeartbeat = Date.now();

    this.startElectionTimer();
  }

  randomTimeout(min, max) {
    return min + Math.random() * (max - min);
  }

  startElectionTimer() {
    setInterval(() => {
      if (this.state !== 'leader') {
        const elapsed = Date.now() - this.lastHeartbeat;

        if (elapsed > this.electionTimeout) {
          this.startElection();
        }
      }
    }, 10);
  }

  async startElection() {
    console.log(`Node ${this.id}: Starting election for term ${this.currentTerm + 1}`);

    this.state = 'candidate';
    this.currentTerm++;
    this.votedFor = this.id;

    let votes = 1;  // Голос за себя

    const lastEntry = this.log.getLastEntry();

    // Отправить RequestVote всем peers
    const votePromises = this.peers.map(async (peer) => {
      const response = await peer.requestVote({
        term: this.currentTerm,
        candidateId: this.id,
        lastLogIndex: lastEntry?.index || 0,
        lastLogTerm: lastEntry?.term || 0,
      });

      if (response.voteGranted) {
        votes++;
      }

      if (response.term > this.currentTerm) {
        this.becomeFollower(response.term);
      }
    });

    await Promise.all(votePromises);

    // Проверить quorum
    const quorum = Math.floor(this.peers.length / 2) + 1;

    if (votes >= quorum && this.state === 'candidate') {
      this.becomeLeader();
    } else {
      this.becomeFollower(this.currentTerm);
    }
  }

  async requestVote(request) {
    // Если term запроса меньше — отклонить
    if (request.term < this.currentTerm) {
      return { term: this.currentTerm, voteGranted: false };
    }

    // Если term больше — обновить
    if (request.term > this.currentTerm) {
      this.becomeFollower(request.term);
    }

    // Если уже голосовали в этом term — отклонить
    if (this.votedFor !== null && this.votedFor !== request.candidateId) {
      return { term: this.currentTerm, voteGranted: false };
    }

    // Проверить, что лог candidate не менее свежий
    const lastEntry = this.log.getLastEntry();
    const candidateUpToDate =
      request.lastLogTerm > (lastEntry?.term || 0) ||
      (request.lastLogTerm === (lastEntry?.term || 0) &&
       request.lastLogIndex >= (lastEntry?.index || 0));

    if (!candidateUpToDate) {
      return { term: this.currentTerm, voteGranted: false };
    }

    // Голосовать
    this.votedFor = request.candidateId;
    this.lastHeartbeat = Date.now();

    console.log(`Node ${this.id}: Voted for ${request.candidateId} in term ${request.term}`);

    return { term: this.currentTerm, voteGranted: true };
  }

  becomeLeader() {
    console.log(`Node ${this.id}: Became leader for term ${this.currentTerm}`);

    this.state = 'leader';

    // Инициализировать nextIndex и matchIndex
    this.peers.forEach(peer => {
      this.nextIndex[peer.id] = this.log.entries.length + 1;
      this.matchIndex[peer.id] = 0;
    });

    // Начать отправку heartbeats
    this.sendHeartbeats();
  }

  becomeFollower(term) {
    console.log(`Node ${this.id}: Became follower for term ${term}`);

    this.state = 'follower';
    this.currentTerm = term;
    this.votedFor = null;
    this.lastHeartbeat = Date.now();
  }

  async sendHeartbeats() {
    if (this.state !== 'leader') return;

    // Отправить AppendEntries всем followers
    this.peers.forEach(async (peer) => {
      const prevLogIndex = this.nextIndex[peer.id] - 1;
      const prevLogEntry = this.log.getEntry(prevLogIndex);

      const entries = this.log.entries.slice(this.nextIndex[peer.id] - 1);

      const response = await peer.appendEntries({
        term: this.currentTerm,
        leaderId: this.id,
        prevLogIndex,
        prevLogTerm: prevLogEntry?.term || 0,
        entries,
        leaderCommit: this.log.commitIndex,
      });

      if (response.success) {
        this.matchIndex[peer.id] = prevLogIndex + entries.length;
        this.nextIndex[peer.id] = this.matchIndex[peer.id] + 1;
      } else {
        // Лог follower не совпадает — откатить назад
        this.nextIndex[peer.id]--;
      }
    });

    // Обновить commitIndex
    this.updateCommitIndex();

    // Следующий heartbeat
    setTimeout(() => this.sendHeartbeats(), this.heartbeatInterval);
  }

  async appendEntries(request) {
    this.lastHeartbeat = Date.now();

    // Если term запроса меньше — отклонить
    if (request.term < this.currentTerm) {
      return { term: this.currentTerm, success: false };
    }

    // Если term больше — обновить
    if (request.term > this.currentTerm) {
      this.becomeFollower(request.term);
    }

    // Проверить совпадение предыдущего entry
    const prevEntry = this.log.getEntry(request.prevLogIndex);

    if (prevEntry && prevEntry.term !== request.prevLogTerm) {
      return { term: this.currentTerm, success: false };
    }

    // Добавить новые entries
    request.entries.forEach((entry, i) => {
      const index = request.prevLogIndex + i + 1;
      this.log.entries[index - 1] = entry;
    });

    // Обновить commitIndex
    if (request.leaderCommit > this.log.commitIndex) {
      this.log.commitIndex = Math.min(
        request.leaderCommit,
        this.log.entries.length
      );

      this.applyCommittedEntries();
    }

    return { term: this.currentTerm, success: true };
  }

  updateCommitIndex() {
    // Найти N такой, что большинство matchIndex >= N
    const sorted = Object.values(this.matchIndex).sort((a, b) => b - a);
    const quorum = Math.floor(this.peers.length / 2);
    const N = sorted[quorum];

    if (N > this.log.commitIndex) {
      const entry = this.log.getEntry(N);

      if (entry && entry.term === this.currentTerm) {
        this.log.commitIndex = N;
        this.applyCommittedEntries();
      }
    }
  }

  applyCommittedEntries() {
    while (this.log.lastApplied < this.log.commitIndex) {
      this.log.lastApplied++;
      const entry = this.log.getEntry(this.log.lastApplied);

      console.log(`Node ${this.id}: Applying command ${entry.command}`);

      // Применить команду к state machine
      this.applyCommand(entry.command);
    }
  }

  applyCommand(command) {
    // Применить к state machine
    // Например, обновить key-value store
  }

  // Client API
  async propose(command) {
    if (this.state !== 'leader') {
      throw new Error('Not a leader');
    }

    // Добавить в log
    const index = this.log.append(this.currentTerm, command);

    console.log(`Node ${this.id}: Proposed command ${command} at index ${index}`);

    // Реплицировать немедленно
    await this.sendHeartbeats();

    // Ждать commit
    return new Promise((resolve) => {
      const checkCommit = setInterval(() => {
        if (this.log.commitIndex >= index) {
          clearInterval(checkCommit);
          resolve(index);
        }
      }, 10);
    });
  }
}
```

### Пример использования

```javascript
// Создать кластер из 5 узлов
const nodes = [];

for (let i = 1; i <= 5; i++) {
  nodes.push(new RaftNode(i, []));
}

// Настроить peers
nodes.forEach(node => {
  node.peers = nodes.filter(n => n.id !== node.id);
});

// Дать время на выборы
setTimeout(async () => {
  // Найти лидера
  const leader = nodes.find(n => n.state === 'leader');

  console.log(`Leader is Node ${leader.id}`);

  // Отправить команду
  await leader.propose('SET x = 10');
  await leader.propose('SET y = 20');

  // Проверить, что все узлы применили команды
  setTimeout(() => {
    nodes.forEach(node => {
      console.log(`Node ${node.id}: commitIndex=${node.log.commitIndex}, lastApplied=${node.log.lastApplied}`);
    });
  }, 1000);
}, 2000);
```

## Paxos

Paxos — классический consensus алгоритм, более сложный, но гибкий.

### Basic Paxos

**Роли:**
- **Proposer**: предлагает значение
- **Acceptor**: голосует за значение
- **Learner**: узнаёт выбранное значение

**Две фазы:**

**Phase 1: Prepare**

```
Proposer → Prepare(n) → Acceptors

Acceptor:
  Если n > highest_n_seen:
    highest_n_seen = n
    Ответить: Promise(n, highest_accepted_value)
  Иначе:
    Отклонить
```

**Phase 2: Accept**

```
Proposer получил Promise от большинства:
  Если все Promise без значения:
    value = своё значение
  Иначе:
    value = значение с максимальным n

Proposer → Accept(n, value) → Acceptors

Acceptor:
  Если n >= highest_n_seen:
    accepted_n = n
    accepted_value = value
    Ответить: Accepted(n, value)
  Иначе:
    Отклонить

Proposer получил Accepted от большинства:
  value выбрано!
  Уведомить Learners
```

### Multi-Paxos

Basic Paxos выбирает одно значение. Multi-Paxos выбирает последовательность значений (log).

**Оптимизация:**
- Один Proposer становится Leader
- Leader пропускает Phase 1 для последующих команд
- Похоже на Raft

## etcd (Raft в production)

etcd — распределённый key-value store на базе Raft, используется в Kubernetes.

### Установка etcd

```bash
# Docker
docker run -d \
  --name etcd \
  -p 2379:2379 \
  -p 2380:2380 \
  quay.io/coreos/etcd:latest \
  /usr/local/bin/etcd \
  --name node1 \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://localhost:2379 \
  --listen-peer-urls http://0.0.0.0:2380
```

### Использование etcd

```javascript
const { Etcd3 } = require('etcd3');

const client = new Etcd3({ hosts: 'http://localhost:2379' });

// Put key-value
await client.put('config/db/host').value('localhost');
await client.put('config/db/port').value('5432');

// Get value
const host = await client.get('config/db/host').string();
console.log('DB Host:', host);  // localhost

// Watch for changes
const watcher = await client.watch()
  .key('config/db/host')
  .create();

watcher.on('put', (kv) => {
  console.log('Config changed:', kv.key.toString(), '=', kv.value.toString());
});

// Lease (TTL)
const lease = client.lease(10);  // 10 секунд

await lease.put('session/user123').value('active');

// Автоматически удалится через 10 секунд
```

### Leader Election с etcd

```javascript
class LeaderElection {
  constructor(etcd, name) {
    this.etcd = etcd;
    this.name = name;
    this.lease = null;
    this.isLeader = false;
  }

  async campaign() {
    // Создать lease
    this.lease = this.etcd.lease(10);

    await this.lease.on('lost', () => {
      console.log(`${this.name}: Lost leadership`);
      this.isLeader = false;
      this.campaign();  // Попробовать снова
    });

    try {
      // Попытка стать лидером
      await this.etcd.if('leader', 'Create', '==', 0)
        .then(this.etcd.put('leader').value(this.name).lease(this.lease))
        .commit();

      console.log(`${this.name}: Became leader`);
      this.isLeader = true;

      // Продлевать lease
      await this.lease.grant();

    } catch (err) {
      console.log(`${this.name}: Another node is leader`);

      // Ждать освобождения
      const watcher = await this.etcd.watch().key('leader').create();

      watcher.on('delete', () => {
        watcher.cancel();
        this.campaign();
      });
    }
  }

  async resign() {
    if (this.isLeader && this.lease) {
      await this.lease.revoke();
      this.isLeader = false;
    }
  }
}

// Использование
const election1 = new LeaderElection(client, 'Node-1');
const election2 = new LeaderElection(client, 'Node-2');

await election1.campaign();
await election2.campaign();

// Только один станет лидером
```

### Distributed Lock

```javascript
class DistributedLock {
  constructor(etcd, lockKey, ttl = 10) {
    this.etcd = etcd;
    this.lockKey = lockKey;
    this.ttl = ttl;
    this.lease = null;
  }

  async acquire() {
    this.lease = this.etcd.lease(this.ttl);

    try {
      await this.etcd.if(this.lockKey, 'Create', '==', 0)
        .then(this.etcd.put(this.lockKey).value('locked').lease(this.lease))
        .commit();

      console.log(`Lock acquired: ${this.lockKey}`);

      // Продлевать lease
      await this.lease.grant();

      return true;

    } catch (err) {
      console.log(`Lock already held: ${this.lockKey}`);
      return false;
    }
  }

  async release() {
    if (this.lease) {
      await this.lease.revoke();
      console.log(`Lock released: ${this.lockKey}`);
    }
  }

  async withLock(fn) {
    const acquired = await this.acquire();

    if (!acquired) {
      throw new Error('Failed to acquire lock');
    }

    try {
      return await fn();
    } finally {
      await this.release();
    }
  }
}

// Использование
const lock = new DistributedLock(client, 'locks/resource-1');

await lock.withLock(async () => {
  console.log('Doing critical work...');
  await new Promise(resolve => setTimeout(resolve, 5000));
  console.log('Work done');
});
```

## ZooKeeper (Paxos-подобный)

ZooKeeper использует ZAB (ZooKeeper Atomic Broadcast), похожий на Paxos.

### Установка ZooKeeper

```bash
docker run -d \
  --name zookeeper \
  -p 2181:2181 \
  zookeeper:latest
```

### Использование ZooKeeper

```javascript
const { createClient } = require('node-zookeeper-client');

const client = createClient('localhost:2181');

client.once('connected', async () => {
  console.log('Connected to ZooKeeper');

  // Create node
  client.create('/config', Buffer.from('value'), (err, path) => {
    if (err) {
      console.error('Failed to create node:', err);
    } else {
      console.log('Node created:', path);
    }
  });

  // Get data
  client.getData('/config', (event) => {
    console.log('Data changed:', event);
  }, (err, data, stat) => {
    if (err) {
      console.error('Failed to get data:', err);
    } else {
      console.log('Data:', data.toString());
      console.log('Version:', stat.version);
    }
  });

  // Set data
  client.setData('/config', Buffer.from('new value'), (err, stat) => {
    if (err) {
      console.error('Failed to set data:', err);
    } else {
      console.log('Data updated, version:', stat.version);
    }
  });
});

client.connect();
```

## Raft vs Paxos

| Критерий | Raft | Paxos |
|----------|------|-------|
| **Понятность** | Легче понять | Сложнее |
| **Leader** | Сильный лидер | Слабый лидер (Multi-Paxos) |
| **Log replication** | Последовательный | Может быть с пробелами |
| **Leader election** | Встроен | Отдельный механизм |
| **Практичность** | Проще реализовать | Сложнее |
| **Гибкость** | Менее гибкий | Более гибкий |
| **Примеры** | etcd, Consul, TiKV | Chubby, Spanner |

## Проблемы и ограничения

### 1. Split Brain

```
Network partition:

[A, B] | [C, D, E]
        ^
    Разделение

Если старый лидер A в меньшинстве:
  - A не может commit новые записи (нет quorum)
  - C, D, E выберут нового лидера

Когда сеть восстановится:
  - A обнаружит новый term
  - A станет follower
  - Log A будет перезаписан
```

### 2. Performance

```javascript
// Latency commit в Raft
// Лидер должен дождаться большинства реплик

// 3 узла: Leader + 1 follower = quorum
// Latency = network RTT к ближайшему follower

// 5 узлов: Leader + 2 followers = quorum
// Latency = network RTT к 2му slowest follower
```

**Оптимизация: Read от followers**

```javascript
// Raft обычно требует read от Leader (linearizability)
// Но можно читать от Follower с ослабленными гарантиями

// Stale reads (может быть устаревшим)
const value = await follower.get('key');

// Read index (Leader подтверждает, что follower актуален)
const readIndex = await leader.getReadIndex();
await follower.waitForIndex(readIndex);
const value = await follower.get('key');
```

### 3. Cluster Membership Changes

```javascript
// Добавление/удаление узлов опасно
// Может нарушить quorum

// Raft: Joint Consensus
// Переход через промежуточное состояние

// Старый кластер: {A, B, C}
// Новый кластер: {A, B, C, D, E}

// Phase 1: Joint consensus {A,B,C} + {A,B,C,D,E}
//   Quorum = большинство в ОБОИХ кластерах

// Phase 2: Переход на новый кластер {A,B,C,D,E}
```

## Best Practices

### 1. Нечётное число узлов

```
3 узла: tolerates 1 failure (quorum = 2)
4 узла: tolerates 1 failure (quorum = 3)  ← не лучше!
5 узлов: tolerates 2 failures (quorum = 3)

Используйте 3, 5, 7 узлов
```

### 2. Географическое распределение

```
Плохо:
  3 узла в одном ЦОД
  ЦОД упал → кластер недоступен

Хорошо:
  3 узла в 3 разных зонах доступности
  1 зона упала → кластер работает

Осторожно:
  3 узла в 3 разных континентах
  Высокая latency commit (network RTT)
```

### 3. Мониторинг

```javascript
// Ключевые метрики
const metrics = {
  // Leader elections
  electionCount: prometheus.Counter,

  // Log replication lag
  replicationLag: prometheus.Gauge,

  // Commit latency
  commitLatency: prometheus.Histogram,

  // Quorum availability
  quorumAvailable: prometheus.Gauge,

  // Leader stability (время без выборов)
  leaderStability: prometheus.Gauge,
};
```

## Выводы

Consensus алгоритмы — основа отказоустойчивых распределённых систем:

**Raft:**
- Проще понять и реализовать
- Сильный лидер
- etcd, Consul

**Paxos:**
- Более гибкий
- Сложнее
- Chubby, Spanner

**Ключевые концепции:**
- Quorum (большинство узлов)
- Leader election
- Log replication
- Linearizability

**Использование:**
- Distributed coordination (etcd, ZooKeeper)
- Distributed databases
- Leader election
- Distributed locks

Consensus обеспечивает consistency и fault tolerance, но жертвует availability при network partition (CAP).

## Проверь себя

1. Что такое consensus и зачем он нужен?
2. В чём разница между Raft и Paxos?
3. Какие роли существуют в Raft?
4. Как происходит Leader Election в Raft?
5. Что такое quorum и зачем он нужен?
6. Как Raft обеспечивает log replication?
7. Почему рекомендуется нечётное число узлов?
8. Что такое split brain и как Raft его предотвращает?
9. Как использовать etcd для leader election?
10. Какие ограничения у consensus алгоритмов?

---
**Предыдущий урок**: [Урок 30: Service Discovery](30-service-discovery.md)
**Следующий урок**: [Урок 32: Distributed Transactions — 2PC, Saga](32-distributed-transactions.md)
