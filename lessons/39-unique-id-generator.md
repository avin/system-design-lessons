# Урок 39: Генератор уникальных идентификаторов

## Введение

Генерация уникальных ID — фундаментальная задача в распределённых системах. Простое авто-инкрементирование (AUTO_INCREMENT в MySQL) не работает в distributed environment из-за необходимости координации между узлами.

В этом уроке мы рассмотрим различные подходы к генерации ID и спроектируем масштабируемый, высокопроизводительный генератор, способный выдавать миллионы уникальных ID в секунду без координации между узлами.

**Примеры использования:**
- ID пользователей, постов, заказов в базах данных
- Request ID для трейсинга (distributed tracing)
- Event ID в event streaming системах
- Sharding keys для партиционирования данных

**Реальные системы:**
- **Twitter Snowflake** — 64-bit ID с timestamp, datacenter ID, machine ID
- **Instagram** — модификация Snowflake с shard ID
- **MongoDB ObjectId** — 12 bytes: timestamp + machine ID + process ID + counter
- **UUID** — 128-bit, stateless генерация

## Требования к системе

### Функциональные требования

1. **Генерация ID**:
   - Уникальность: 100% гарантия уникальности
   - Возврат ID за один вызов (без дополнительных запросов)

2. **Масштабируемость**:
   - Генерация 10,000+ ID/sec на узел
   - Горизонтальное масштабирование (добавление узлов без координации)

### Нефункциональные требования

1. **Производительность**:
   - Latency: < 1ms
   - Throughput: 10K+ req/sec на узел

2. **Доступность**: 99.99%

3. **Свойства ID**:
   - Сортируемость по времени создания (желательно)
   - Компактность (64-bit предпочтительнее 128-bit)
   - Не должны быть предсказуемыми (для безопасности)

## Подходы к генерации ID

### 1. Auto-increment (одна БД)

```sql
CREATE TABLE users (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255)
);

INSERT INTO users (name) VALUES ('Alice');
-- id = 1
INSERT INTO users (name) VALUES ('Bob');
-- id = 2
```

**Плюсы:**
- Простота
- Сортируемость
- Компактность

**Минусы:**
- ❌ Не масштабируется горизонтально
- ❌ Single point of failure
- ❌ Bottleneck при высокой нагрузке

### 2. UUID (Universally Unique Identifier)

```javascript
const { v4: uuidv4, v1: uuidv1 } = require('uuid');

// UUID v4 (random)
const id = uuidv4();
// '9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d'

// UUID v1 (timestamp + MAC address)
const id2 = uuidv1();
// '6fa7ed00-5e89-11ec-bf63-0242ac130002'
```

**Формат UUID v4:**
```
xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx

4 = версия
y = variant (8, 9, a, b)
остальное = random

Размер: 128 bits (16 bytes)
Вероятность коллизии: ~10^-18
```

**Плюсы:**
- ✅ Нет координации между узлами
- ✅ Stateless генерация
- ✅ Очень низкая вероятность коллизий

**Минусы:**
- ❌ 128 bits (много памяти для индексов)
- ❌ Не сортируемы по времени (плохо для DB index)
- ❌ Не human-readable

### 3. Snowflake ID (Twitter)

**Формат (64 bits):**

```
┌───────────────────────────────────────────┬─────────┬─────────┬────────────┐
│           Timestamp (41 bits)             │DC (5b)  │Worker(5)│ Sequence(12)│
│                                           │         │         │            │
│     milliseconds since custom epoch       │         │         │  0-4095    │
└───────────────────────────────────────────┴─────────┴─────────┴────────────┘
  63                                      22 21     17 16     12 11         0

Breakdown:
- Timestamp: 41 bits = ~69 years с custom epoch
- Datacenter ID: 5 bits = 32 datacenters
- Worker ID: 5 bits = 32 workers per datacenter
- Sequence: 12 bits = 4096 ID per millisecond per worker
```

**Реализация:**

```javascript
class SnowflakeID {
  constructor(workerId, datacenterId, epoch = 1640000000000) {
    this.workerId = BigInt(workerId);
    this.datacenterId = BigInt(datacenterId);
    this.epoch = BigInt(epoch); // custom epoch (например, 2022-01-01)

    this.sequence = 0n;
    this.lastTimestamp = -1n;

    // Bit lengths
    this.workerIdBits = 5n;
    this.datacenterIdBits = 5n;
    this.sequenceBits = 12n;

    // Max values
    this.maxWorkerId = -1n ^ (-1n << this.workerIdBits); // 31
    this.maxDatacenterId = -1n ^ (-1n << this.datacenterIdBits); // 31
    this.maxSequence = -1n ^ (-1n << this.sequenceBits); // 4095

    // Shifts
    this.workerIdShift = this.sequenceBits; // 12
    this.datacenterIdShift = this.sequenceBits + this.workerIdBits; // 17
    this.timestampShift = this.sequenceBits + this.workerIdBits + this.datacenterIdBits; // 22

    // Validate
    if (this.workerId > this.maxWorkerId || this.workerId < 0n) {
      throw new Error(`Worker ID must be between 0 and ${this.maxWorkerId}`);
    }

    if (this.datacenterId > this.maxDatacenterId || this.datacenterId < 0n) {
      throw new Error(`Datacenter ID must be between 0 and ${this.maxDatacenterId}`);
    }
  }

  generate() {
    let timestamp = this.currentTimestamp();

    // Clock moved backwards
    if (timestamp < this.lastTimestamp) {
      throw new Error(
        `Clock moved backwards. Refusing to generate ID for ${this.lastTimestamp - timestamp}ms`
      );
    }

    if (timestamp === this.lastTimestamp) {
      // Same millisecond - increment sequence
      this.sequence = (this.sequence + 1n) & this.maxSequence;

      if (this.sequence === 0n) {
        // Sequence overflow - wait for next millisecond
        timestamp = this.waitNextMillis(this.lastTimestamp);
      }
    } else {
      // New millisecond - reset sequence
      this.sequence = 0n;
    }

    this.lastTimestamp = timestamp;

    // Generate ID
    const id =
      ((timestamp - this.epoch) << this.timestampShift) |
      (this.datacenterId << this.datacenterIdShift) |
      (this.workerId << this.workerIdShift) |
      this.sequence;

    return id;
  }

  currentTimestamp() {
    return BigInt(Date.now());
  }

  waitNextMillis(lastTimestamp) {
    let timestamp = this.currentTimestamp();

    while (timestamp <= lastTimestamp) {
      timestamp = this.currentTimestamp();
    }

    return timestamp;
  }

  // Decode ID (для debugging)
  decode(id) {
    const idBigInt = BigInt(id);

    const timestamp =
      Number((idBigInt >> this.timestampShift) + this.epoch);
    const datacenterId =
      Number((idBigInt >> this.datacenterIdShift) & this.maxDatacenterId);
    const workerId =
      Number((idBigInt >> this.workerIdShift) & this.maxWorkerId);
    const sequence =
      Number(idBigInt & this.maxSequence);

    return {
      timestamp: new Date(timestamp),
      datacenterId,
      workerId,
      sequence
    };
  }
}

// Использование
const generator = new SnowflakeID(1, 1, 1640000000000);

const id1 = generator.generate();
console.log('ID:', id1.toString());
// 3287362738749652992

const decoded = generator.decode(id1);
console.log('Decoded:', decoded);
// {
//   timestamp: 2025-10-26T10:30:45.123Z,
//   datacenterId: 1,
//   workerId: 1,
//   sequence: 0
// }

// Benchmark
const start = Date.now();
const count = 100000;

for (let i = 0; i < count; i++) {
  generator.generate();
}

const elapsed = Date.now() - start;
console.log(`Generated ${count} IDs in ${elapsed}ms`);
console.log(`Throughput: ${Math.floor(count / elapsed * 1000)} IDs/sec`);
// ~500K - 1M IDs/sec
```

**Плюсы:**
- ✅ 64 bits (компактно)
- ✅ Сортируемы по времени
- ✅ Высокая производительность
- ✅ Нет координации между узлами (каждый worker независим)

**Минусы:**
- ❌ Требует назначения worker ID каждому узлу
- ❌ Проблемы при изменении системного времени (clock skew)
- ❌ Ограничение 4096 ID/ms/worker

### 4. Sonyflake (модификация Snowflake)

Разница от Snowflake:
- Точность 10ms (вместо 1ms)
- Sequence: 8 bits = 256 ID per 10ms (меньше throughput, но ID компактнее)

```
┌──────────────────────────┬─────────────┬────────┐
│   Timestamp (39 bits)    │ Sequence(8) │Machine │
│   10ms since epoch       │   0-255     │ ID(16) │
└──────────────────────────┴─────────────┴────────┘
```

### 5. Instagram ID

Instagram использует Postgres с sharding и модифицированный auto-increment:

```sql
-- Схема таблицы
CREATE SCHEMA instagram;

CREATE TABLE instagram.photos (
  id BIGINT NOT NULL DEFAULT instagram.next_id('photos_id_seq'),
  user_id BIGINT,
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (id)
);

-- Функция генерации ID
CREATE OR REPLACE FUNCTION instagram.next_id(seq_name TEXT)
RETURNS BIGINT AS $$
DECLARE
  our_epoch BIGINT := 1314220021721;  -- custom epoch
  seq_id BIGINT;
  now_ms BIGINT;
  shard_id INT := 1;  -- ID текущего шарда (настраивается)
  result BIGINT;
BEGIN
  SELECT nextval(seq_name) % 1024 INTO seq_id;  -- 1024 IDs per ms
  SELECT FLOOR(EXTRACT(EPOCH FROM NOW()) * 1000) INTO now_ms;

  result := (now_ms - our_epoch) << 23;
  result := result | (shard_id << 10);
  result := result | seq_id;

  RETURN result;
END;
$$ LANGUAGE plpgsql;
```

**Формат:**
```
┌──────────────────────┬──────────┬──────────┐
│  Timestamp (41 bits) │ Shard(13)│ Seq (10) │
└──────────────────────┴──────────┴──────────┘
```

### 6. MongoDB ObjectId

```javascript
const { ObjectId } = require('mongodb');

const id = new ObjectId();
console.log(id.toString());
// 617a1b2c3d4e5f6a7b8c9d0e

// Формат (12 bytes = 96 bits):
// ┌──────────┬─────────┬─────────┬────────┐
// │Timestamp │ Random  │Process  │Counter │
// │ (4 bytes)│(5 bytes)│(2 bytes)│(3 bytes)│
// └──────────┴─────────┴─────────┴────────┘

const timestamp = id.getTimestamp();
console.log(timestamp); // время создания
```

**Плюсы:**
- Сортируемость по времени
- Без координации

**Минусы:**
- 96 bits (больше чем Snowflake)
- Не гарантия 100% уникальности (random part может повториться)

## Проектирование системы ID Generator

### Архитектура

```
┌─────────────────────────────────────┐
│         Load Balancer               │
│      (Round Robin / Least Conn)     │
└──────────────┬──────────────────────┘
               │
      ┌────────┼────────┐
      │        │        │
┌─────▼───┐ ┌──▼─────┐ ┌▼────────┐
│  ID Gen │ │  ID Gen│ │  ID Gen │
│  Node 1 │ │  Node 2│ │  Node 3 │
│ Worker=1│ │ Worker=2│ │ Worker=3│
└─────────┘ └────────┘ └─────────┘

Каждая нода:
- Имеет уникальный Worker ID
- Генерирует ID локально (без DB)
- Stateless (можно добавлять/удалять)
```

### Реализация сервиса

```javascript
const express = require('express');
const app = express();

class IDGeneratorService {
  constructor(workerId, datacenterId) {
    this.snowflake = new SnowflakeID(workerId, datacenterId);

    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();

    // Генерация одного ID
    app.get('/id', (req, res) => {
      try {
        const id = this.snowflake.generate();

        res.json({
          id: id.toString(),
          timestamp: Date.now()
        });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Batch генерация (более эффективно)
    app.get('/ids', (req, res) => {
      try {
        const count = Math.min(parseInt(req.query.count) || 1, 1000);
        const ids = [];

        for (let i = 0; i < count; i++) {
          ids.push(this.snowflake.generate().toString());
        }

        res.json({ ids, count: ids.length });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Decode ID (для debugging)
    app.get('/decode/:id', (req, res) => {
      try {
        const decoded = this.snowflake.decode(req.params.id);
        res.json(decoded);
      } catch (error) {
        res.status(400).json({ error: error.message });
      }
    });

    // Health check
    app.get('/health', (req, res) => {
      res.json({
        status: 'healthy',
        workerId: Number(this.snowflake.workerId),
        datacenterId: Number(this.snowflake.datacenterId),
        timestamp: Date.now()
      });
    });

    this.app = app;
  }

  start(port) {
    this.app.listen(port, () => {
      console.log(`ID Generator (Worker ${this.snowflake.workerId}, DC ${this.snowflake.datacenterId}) listening on port ${port}`);
    });
  }
}

// Получаем Worker ID из переменной окружения или аргумента
const workerId = parseInt(process.env.WORKER_ID || process.argv[2] || '1');
const datacenterId = parseInt(process.env.DATACENTER_ID || process.argv[3] || '1');
const port = parseInt(process.env.PORT || '3000');

const service = new IDGeneratorService(workerId, datacenterId);
service.start(port);
```

### Docker Compose для запуска кластера

```yaml
version: '3.8'

services:
  id-gen-1:
    build: .
    environment:
      WORKER_ID: 1
      DATACENTER_ID: 1
      PORT: 3000
    ports:
      - "3001:3000"

  id-gen-2:
    build: .
    environment:
      WORKER_ID: 2
      DATACENTER_ID: 1
      PORT: 3000
    ports:
      - "3002:3000"

  id-gen-3:
    build: .
    environment:
      WORKER_ID: 3
      DATACENTER_ID: 1
      PORT: 3000
    ports:
      - "3003:3000"

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "8080:80"
    depends_on:
      - id-gen-1
      - id-gen-2
      - id-gen-3
```

**nginx.conf:**

```nginx
http {
  upstream id_generators {
    least_conn;  # распределение по наименее загруженному
    server id-gen-1:3000;
    server id-gen-2:3000;
    server id-gen-3:3000;
  }

  server {
    listen 80;

    location / {
      proxy_pass http://id_generators;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }
}
```

## Продвинутые техники

### 1. Range-based ID Allocation

Вместо генерации по одному ID, выделяем диапазоны (range):

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class RangeBasedIDGenerator {
  constructor(nodeId, rangeSize = 1000) {
    this.nodeId = nodeId;
    this.rangeSize = rangeSize;

    this.currentRange = null;
    this.currentIndex = 0;
  }

  async allocateRange() {
    // Атомарно получаем следующий диапазон из Redis
    const rangeStart = await redis.incrby('id_range_counter', this.rangeSize);

    this.currentRange = {
      start: rangeStart - this.rangeSize + 1,
      end: rangeStart
    };

    this.currentIndex = this.currentRange.start;

    console.log(`Allocated range: ${this.currentRange.start} - ${this.currentRange.end}`);
  }

  async generate() {
    if (!this.currentRange || this.currentIndex > this.currentRange.end) {
      await this.allocateRange();
    }

    return this.currentIndex++;
  }
}

// Использование
const generator = new RangeBasedIDGenerator('node-1', 1000);

const id1 = await generator.generate(); // 1
const id2 = await generator.generate(); // 2
// ...
const id1000 = await generator.generate(); // 1000
const id1001 = await generator.generate(); // 1001 (новый range)
```

**Плюсы:**
- Меньше обращений к координатору (Redis)
- Простая реализация

**Минусы:**
- Gaps в ID при рестарте ноды (lost range)
- Не сортируемы по времени

### 2. Hybrid: Snowflake + Database

Комбинация для обеспечения строгой уникальности:

```javascript
class HybridIDGenerator {
  constructor(snowflake, db) {
    this.snowflake = snowflake;
    this.db = db;
  }

  async generateWithGuarantee(retries = 3) {
    for (let i = 0; i < retries; i++) {
      const id = this.snowflake.generate();

      try {
        // Пытаемся вставить в БД
        await this.db.query(
          'INSERT INTO generated_ids (id, created_at) VALUES ($1, NOW())',
          [id.toString()]
        );

        return id;
      } catch (error) {
        if (error.code === '23505') {
          // Unique constraint violation - collision!
          console.error(`ID collision detected: ${id}, retrying...`);
          continue;
        }

        throw error;
      }
    }

    throw new Error('Failed to generate unique ID after retries');
  }
}
```

### 3. K-ordered ID

ID, который сохраняет порядок в пределах временного окна (k milliseconds):

```javascript
class KOrderedID {
  constructor(workerId, k = 1000) {
    this.workerId = BigInt(workerId);
    this.k = BigInt(k); // временное окно в мс
    this.sequence = 0n;
    this.lastBucket = 0n;
  }

  generate() {
    const now = BigInt(Date.now());
    const bucket = now / this.k; // округление до k-bucket

    if (bucket !== this.lastBucket) {
      this.sequence = 0n;
      this.lastBucket = bucket;
    }

    const id = (bucket << 23n) | (this.workerId << 10n) | this.sequence;
    this.sequence++;

    return id;
  }
}

// IDs, созданные в пределах 1 секунды, будут отсортированы примерно правильно
const generator = new KOrderedID(1, 1000);
```

### 4. Clock Synchronization с NTP

Проблема: если часы сервера отстают/спешат, могут быть коллизии.

```javascript
const ntp = require('ntp-client');

class NTPSyncedSnowflake extends SnowflakeID {
  constructor(workerId, datacenterId) {
    super(workerId, datacenterId);

    this.clockOffset = 0;
    this.syncClock();

    // Периодическая синхронизация
    setInterval(() => this.syncClock(), 3600000); // каждый час
  }

  syncClock() {
    ntp.getNetworkTime('pool.ntp.org', 123, (err, date) => {
      if (err) {
        console.error('NTP sync failed:', err);
        return;
      }

      const localTime = Date.now();
      const ntpTime = date.getTime();
      this.clockOffset = ntpTime - localTime;

      console.log(`Clock offset: ${this.clockOffset}ms`);

      if (Math.abs(this.clockOffset) > 5000) {
        console.warn('Clock drift > 5 seconds! Consider system time sync.');
      }
    });
  }

  currentTimestamp() {
    return BigInt(Date.now() + this.clockOffset);
  }
}
```

## Мониторинг и метрики

```javascript
const prometheus = require('prom-client');

const idGenerationCounter = new prometheus.Counter({
  name: 'id_generation_total',
  help: 'Total number of IDs generated',
  labelNames: ['worker_id', 'datacenter_id']
});

const idGenerationDuration = new prometheus.Histogram({
  name: 'id_generation_duration_seconds',
  help: 'Time to generate ID',
  buckets: [0.00001, 0.0001, 0.001, 0.01]
});

const sequenceOverflows = new prometheus.Counter({
  name: 'id_sequence_overflows_total',
  help: 'Number of times sequence overflowed (had to wait for next ms)'
});

const clockSkewGauge = new prometheus.Gauge({
  name: 'id_clock_skew_ms',
  help: 'Clock skew detected (ms)'
});

// В методе generate()
generate() {
  const start = process.hrtime.bigint();

  try {
    // ... генерация ID ...

    if (this.sequence === 0n && timestamp === this.lastTimestamp) {
      sequenceOverflows.inc();
    }

    idGenerationCounter.inc({
      worker_id: this.workerId.toString(),
      datacenter_id: this.datacenterId.toString()
    });

    return id;
  } finally {
    const end = process.hrtime.bigint();
    const duration = Number(end - start) / 1e9; // в секундах
    idGenerationDuration.observe(duration);
  }
}
```

**Grafana Dashboard Queries:**

```promql
# Throughput (IDs/sec)
rate(id_generation_total[1m])

# P99 latency
histogram_quantile(0.99,
  rate(id_generation_duration_seconds_bucket[5m])
)

# Sequence overflows (признак высокой нагрузки)
rate(id_sequence_overflows_total[5m])

# Clock skew monitoring
id_clock_skew_ms
```

## Сравнение подходов

| Подход | Размер | Сортируемость | Производительность | Координация | Use Case |
|--------|--------|---------------|-------------------|-------------|----------|
| Auto-increment | 8 bytes | ✅ | Низкая (DB bottleneck) | Требуется | Single DB |
| UUID v4 | 16 bytes | ❌ | Высокая | Не требуется | Distributed, без сортировки |
| UUID v1 | 16 bytes | ⚠️ Частично | Высокая | Не требуется | Distributed + timestamp |
| Snowflake | 8 bytes | ✅ | Очень высокая | Минимальная (worker ID) | **Рекомендуется** |
| MongoDB ObjectId | 12 bytes | ✅ | Высокая | Не требуется | MongoDB |
| Range-based | 8 bytes | ❌ | Средняя (периодич. координация) | Периодическая | Простые случаи |

## Best Practices

### 1. Выбор custom epoch

```javascript
// Плохо: использование Unix epoch (1970-01-01)
const epoch = 0;

// Хорошо: custom epoch близко к запуску системы
const epoch = 1640000000000; // 2022-01-01
// Даёт больше времени до исчерпания 41 бита timestamp
```

### 2. Мониторинг clock drift

```javascript
setInterval(() => {
  const drift = Date.now() - this.lastTimestamp;

  if (drift < 0) {
    console.error(`Clock moved backwards by ${Math.abs(drift)}ms!`);
    // Alert! Возможна проблема с system time
  }
}, 10000);
```

### 3. Graceful handling при исчерпании sequence

```javascript
if (this.sequence === this.maxSequence) {
  // Достигнут лимит 4096 ID в миллисекунду
  // Опции:
  // 1. Ждать следующую миллисекунду (default)
  // 2. Логировать предупреждение (возможно, нужен больший sequence bits)
  // 3. Fail fast (если критично)

  console.warn(`Sequence overflow at ${Date.now()}`);
}
```

### 4. Testing для уникальности

```javascript
const { describe, it } = require('mocha');
const assert = require('assert');

describe('SnowflakeID', () => {
  it('should generate unique IDs in tight loop', () => {
    const generator = new SnowflakeID(1, 1);
    const ids = new Set();
    const count = 100000;

    for (let i = 0; i < count; i++) {
      const id = generator.generate();
      assert(!ids.has(id), `Duplicate ID found: ${id}`);
      ids.add(id);
    }

    assert.equal(ids.size, count);
  });

  it('should generate sortable IDs', () => {
    const generator = new SnowflakeID(1, 1);
    const ids = [];

    for (let i = 0; i < 1000; i++) {
      ids.push(generator.generate());
    }

    const sorted = [...ids].sort((a, b) => Number(a - b));

    assert.deepEqual(ids, sorted, 'IDs are not sorted by creation time');
  });

  it('should handle multiple workers without collision', async () => {
    const gen1 = new SnowflakeID(1, 1);
    const gen2 = new SnowflakeID(2, 1);
    const ids = new Set();

    for (let i = 0; i < 10000; i++) {
      ids.add(gen1.generate());
      ids.add(gen2.generate());
    }

    assert.equal(ids.size, 20000, 'Collision detected between workers');
  });
});
```

## Что читать дальше

- **Урок 13**: Sharding и Partitioning — использование ID для sharding key
- **Урок 33**: Data Replication — синхронизация ID между репликами
- **Урок 45**: Notification System — использование ID для трейсинга нотификаций

## Проверь себя

1. Почему UUID v4 не подходит для primary key в БД с большими таблицами?
2. Сколько ID в секунду может сгенерировать один Snowflake worker?
3. Что происходит, если системное время сдвинулось назад на 1 час?
4. Спроектируйте схему ID для системы с 100 датацентрами и 1000 серверов в каждом.
5. Как обеспечить строгую уникальность ID при возможных clock skews?

---

[← Урок 38: Распределённое хранилище ключ-значение](38-distributed-kv-store.md) | [Урок 40: Web Crawler →](40-web-crawler.md)
