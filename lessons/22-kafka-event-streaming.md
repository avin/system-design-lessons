# Урок 22: Kafka и Event Streaming

[← Предыдущий урок: Message Queues](21-message-queues.md) | [Следующий урок: Async Workers →](23-async-workers.md)

## Введение

Apache Kafka — это распределённая платформа для потоковой обработки событий (event streaming), которая отличается от традиционных message queues своей архитектурой и подходом к хранению данных.

**Kafka vs традиционные очереди:**

| Message Queue (RabbitMQ, SQS) | Event Streaming (Kafka) |
|-------------------------------|-------------------------|
| Сообщение удаляется после чтения | События хранятся (retention) |
| Point-to-point или pub-sub | Pub-sub с replay |
| Обработка в реальном времени | Stream processing + batch |
| Ограниченная история | Полная история событий |

**Когда нужен Kafka:**
- Event sourcing и audit log
- Потоковая аналитика в реальном времени
- Интеграция множества сервисов (data pipeline)
- Большой объём событий (миллионы msg/s)
- Нужна история событий

## Основные концепции

### Архитектура Kafka

```
Producers → Topics (Partitions) → Consumer Groups
              ↓
         Kafka Cluster (Brokers)
              ↓
         ZooKeeper/KRaft
```

**Компоненты:**

1. **Topic** — категория событий (логи, заказы, клики)
2. **Partition** — часть топика для параллелизма и масштабирования
3. **Producer** — отправляет события в топик
4. **Consumer** — читает события из топика
5. **Consumer Group** — группа консьюмеров для распределённой обработки
6. **Broker** — сервер Kafka (обычно 3+ в кластере)
7. **ZooKeeper/KRaft** — координация кластера

### Topics и Partitions

```
Topic: "orders"
├── Partition 0: [msg1] [msg2] [msg5] ...
├── Partition 1: [msg3] [msg6] [msg7] ...
└── Partition 2: [msg4] [msg8] [msg9] ...
```

**Ключевые особенности:**
- События в партиции **упорядочены** (FIFO)
- Между партициями порядок **не гарантирован**
- Партиция — единица параллелизма
- Количество партиций определяет максимальный параллелизм консьюмеров

### Offsets

Offset — позиция сообщения в партиции (0, 1, 2, 3...).

```
Partition 0:
Offset: 0    1    2    3    4    5
       [A] → [B] → [C] → [D] → [E] → [F]
                           ↑
                    Consumer position
```

**Consumer offset management:**
- **Committed offset** — последняя обработанная позиция
- **Current offset** — текущая позиция чтения
- Коммит может быть **автоматическим** или **ручным**

### Consumer Groups

```
Topic "orders" (3 partitions)

Consumer Group "email-service":
├── Consumer 1 → Partition 0
├── Consumer 2 → Partition 1
└── Consumer 3 → Partition 2

Consumer Group "analytics-service":
├── Consumer 1 → Partition 0, 1
└── Consumer 2 → Partition 2
```

**Правила:**
- Партиция читается только **одним консьюмером** в группе
- Разные группы читают **независимо** (у каждой свой offset)
- При добавлении/удалении консьюмера → **rebalancing**

## Установка и настройка Kafka

### Docker Compose

```yaml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
```

```bash
docker-compose up -d
```

### Создание топика

```bash
# CLI
kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --partitions 3 \
  --replication-factor 1

# Список топиков
kafka-topics --list --bootstrap-server localhost:9092

# Детали топика
kafka-topics --describe --topic orders --bootstrap-server localhost:9092
```

## Producer (Python)

### Базовый Producer

```javascript
const { Kafka, CompressionTypes } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'orders-app',
  brokers: ['localhost:9092'],
});

async function sendOrder() {
  const producer = kafka.producer();
  await producer.connect();

  // Отправка сообщения
  const order = {
    order_id: '12345',
    user_id: '789',
    total: 99.99,
    items: ['item1', 'item2'],
  };

  try {
    const responses = await producer.send({
      topic: 'orders',
      messages: [{ value: JSON.stringify(order) }],
    });

    responses.forEach((response) => {
      console.log(
        `Sent to partition ${response.partition} at offset ${response.baseOffset}`,
      );
    });
  } catch (error) {
    console.error('Error:', error);
  } finally {
    await producer.disconnect();
  }
}

sendOrder().catch(console.error);
```

### Producer с ключом (для партиционирования)

```javascript
async function sendPartitionedOrders() {
  const producer = kafka.producer();
  await producer.connect();

  // Все заказы пользователя 789 — в одной партиции (сохраняется порядок)
  const messages = Array.from({ length: 10 }, (_, i) => ({
    key: 'user_789', // Partition = hash(key) % num_partitions
    value: JSON.stringify({ order_id: `order_${i}`, total: i * 10 }),
  }));

  await producer.send({ topic: 'orders', messages });
  await producer.disconnect();
}

sendPartitionedOrders().catch(console.error);
```

### Асинхронный Producer с callback

```javascript
async function sendWithCallbacks() {
  const producer = kafka.producer();
  await producer.connect();

  const order = {
    order_id: 'callback-1',
    total: 59.99,
  };

  const onSendSuccess = (responses) => {
    responses.forEach((response) => {
      console.log(
        `Message sent to ${response.topicName} partition ${response.partition} offset ${response.baseOffset}`,
      );
    });
  };

  const onSendError = (error) => {
    console.error('Error sending message:', error);
  };

  const sendPromise = producer.send({
    topic: 'orders',
    messages: [{ value: JSON.stringify(order) }],
  });

  sendPromise.then(onSendSuccess).catch(onSendError).finally(async () => {
    await producer.disconnect();
  });

  await sendPromise;
}

sendWithCallbacks().catch(console.error);
```

### Producer с compression

```javascript
async function sendCompressed() {
  const producer = kafka.producer();
  await producer.connect();

  await producer.send({
    topic: 'orders',
    compression: CompressionTypes.GZIP, // gzip, snappy, lz4, zstd
    messages: [
      {
        value: JSON.stringify({ order_id: 'compressed-1', total: 120.5 }),
      },
    ],
  });

  await producer.disconnect();
}

sendCompressed().catch(console.error);
```

### Idempotent Producer

```javascript
async function sendIdempotent() {
  // Гарантия exactly-once для producer (дедупликация)
  const producer = kafka.producer({
    idempotent: true, // Предотвращает дубликаты
    retry: { retries: 5 },
  });

  await producer.connect();

  await producer.send({
    topic: 'orders',
    messages: [{ value: JSON.stringify({ order_id: 'unique', total: 200 }) }],
  });

  await producer.disconnect();
}

sendIdempotent().catch(console.error);
```

### Транзакционный Producer

```javascript
async function sendTransactional() {
  const producer = kafka.producer({
    transactionalId: 'my-transactional-id',
    idempotent: true,
  });

  await producer.connect();

  const transaction = await producer.transaction();

  try {
    // Атомарная отправка нескольких сообщений
    await transaction.send({
      topic: 'orders',
      messages: [{ value: JSON.stringify({ order_id: '1' }) }],
    });

    await transaction.send({
      topic: 'inventory',
      messages: [{ value: JSON.stringify({ product_id: 'A', qty: -1 }) }],
    });

    await transaction.send({
      topic: 'payments',
      messages: [{ value: JSON.stringify({ amount: 99.99 }) }],
    });

    await transaction.commit();
  } catch (error) {
    await transaction.abort();
    console.error('Transaction aborted:', error);
  } finally {
    await producer.disconnect();
  }
}

sendTransactional().catch(console.error);
```

## Consumer (Python)

### Базовый Consumer

```javascript
async function consumeOrders() {
  const consumer = kafka.consumer({
    groupId: 'email-service',
  });

  await consumer.connect();
  await consumer.subscribe({ topic: 'orders', fromBeginning: true });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      console.log(`Partition: ${partition}, Offset: ${message.offset}`);
      const key = message.key ? message.key.toString() : null;
      const value = message.value ? message.value.toString() : null;
      console.log(`Key: ${key}, Value: ${value}`);

      // Обработка события
      const order = value ? JSON.parse(value) : null;
      if (order) {
        await sendConfirmationEmail(order.user_id, order.order_id);
      }
    },
  });
}

consumeOrders().catch(console.error);
```

### Manual Commit

```javascript
async function consumeWithManualCommit() {
  const consumer = kafka.consumer({
    groupId: 'inventory-service',
  });

  await consumer.connect();
  await consumer.subscribe({ topic: 'orders' });

  await consumer.run({
    autoCommit: false, // Manual commit
    eachMessage: async ({ topic, partition, message }) => {
      try {
        await processOrder(JSON.parse(message.value.toString()));

        // Commit после успешной обработки
        await consumer.commitOffsets([
          {
            topic,
            partition,
            offset: (Number(message.offset) + 1).toString(),
          },
        ]);
      } catch (error) {
        console.error('Error processing message:', error);
        // Не коммитим → сообщение обработается повторно
      }
    },
  });
}

consumeWithManualCommit().catch(console.error);
```

### Batch Processing

```javascript
async function consumeInBatches() {
  const consumer = kafka.consumer({ groupId: 'batch-service' });
  await consumer.connect();
  await consumer.subscribe({ topic: 'orders' });

  await consumer.run({
    eachBatchAutoResolve: false,
    eachBatch: async ({ batch, resolveOffset, heartbeat, commitOffsetsIfNecessary }) => {
      const records = batch.messages.map((message) =>
        JSON.parse(message.value.toString()),
      );

      if (records.length > 0) {
        await processBatch(records);
      }

      for (const message of batch.messages) {
        resolveOffset(message.offset);
      }

      await commitOffsetsIfNecessary();
      await heartbeat();
    },
  });
}

consumeInBatches().catch(console.error);
```

### Partition Assignment

```javascript
async function consumeSpecificPartitions() {
  const consumer = kafka.consumer({ groupId: 'manual-assignment' });
  await consumer.connect();

  const assignments = [
    { topic: 'orders', partition: 0 },
    { topic: 'orders', partition: 1 },
  ];

  await consumer.assign(assignments);

  // Seek к конкретному offset
  await consumer.seek({ topic: 'orders', partition: 0, offset: '100' });

  await consumer.run({
    eachMessage: async ({ partition, message }) => {
      console.log(`Partition ${partition}: ${message.value.toString()}`);
    },
  });
}

consumeSpecificPartitions().catch(console.error);
```

### Rebalance Listener

```javascript
async function consumeWithRebalanceHooks() {
  const consumer = kafka.consumer({ groupId: 'service-1' });
  await consumer.connect();
  await consumer.subscribe({ topic: 'orders' });

  await consumer.run({
    partitionsAssigned: async (assignment) => {
      console.log('Partitions assigned:', assignment);
      // Загрузить состояние после rebalance при необходимости
    },
    partitionsRevoked: async (assignment) => {
      console.log('Partitions revoked:', assignment);
      // Сохранить состояние перед rebalance при необходимости
    },
    eachMessage: async ({ message }) => {
      await process(JSON.parse(message.value.toString()));
    },
  });
}

consumeWithRebalanceHooks().catch(console.error);
```

## Kafka Streams

### Простой Stream Processing

```javascript
async function streamTransform() {
  const consumer = kafka.consumer({ groupId: 'stream-worker' });
  const producer = kafka.producer();

  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topic: 'raw-events' });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());

      // Фильтр: только успешные покупки
      if (event.type === 'purchase' && event.status === 'completed') {
        // Трансформация
        const enriched = {
          order_id: event.order_id,
          total: event.total,
          timestamp: event.timestamp,
          category: categorize(event.items),
        };

        // Отправка в другой топик
        await producer.send({
          topic: 'successful-orders',
          messages: [{ value: JSON.stringify(enriched) }],
        });
      }
    },
  });
}

streamTransform().catch(console.error);
```

### Windowing и Aggregation

```javascript
function getWindowKey(timestamp) {
  const date = new Date(timestamp * 1000);
  date.setSeconds(0, 0);
  return date.toISOString();
}

async function windowedAggregation() {
  // Tumbling Window: подсчёт событий за 1-минутные окна
  const windows = new Map();
  const consumer = kafka.consumer({ groupId: 'window-processor' });

  await consumer.connect();
  await consumer.subscribe({ topic: 'page-views' });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      const windowKey = getWindowKey(event.timestamp);
      const current = windows.get(windowKey) || 0;
      windows.set(windowKey, current + 1);

      // Периодический вывод результатов
      if (windows.size > 10) {
        console.log('Page views per minute:');
        [...windows.entries()]
          .sort(([a], [b]) => (a > b ? 1 : -1))
          .forEach(([window, count]) => {
            console.log(`  ${window}: ${count}`);
          });

        // Очистка старых окон
        const cutoff = new Date(Date.now() - 10 * 60 * 1000);
        cutoff.setSeconds(0, 0);
        for (const [key] of windows) {
          if (key < cutoff.toISOString()) {
            windows.delete(key);
          }
        }
      }
    },
  });
}

windowedAggregation().catch(console.error);
```

### Join двух топиков

```javascript
async function joinOrdersAndPayments() {
  const ordersCache = new Map();
  const consumer = kafka.consumer({ groupId: 'join-service' });
  const producer = kafka.producer();

  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topics: ['orders', 'payments'] });

  await consumer.run({
    eachMessage: async ({ topic, message }) => {
      const data = JSON.parse(message.value.toString());

      if (topic === 'orders') {
        ordersCache.set(data.order_id, data);
      } else if (topic === 'payments') {
        const order = ordersCache.get(data.order_id);
        if (order) {
          const joined = {
            ...order,
            payment_method: data.method,
            payment_status: data.status,
            paid_at: data.timestamp,
          };

          await producer.send({
            topic: 'orders-with-payments',
            messages: [{ value: JSON.stringify(joined) }],
          });

          ordersCache.delete(data.order_id);
        }
      }
    },
  });
}

joinOrdersAndPayments().catch(console.error);
```

## Kafka Connect

Kafka Connect — инструмент для интеграции Kafka с внешними системами (БД, файлы, облачные сервисы).

### Source Connector: MySQL → Kafka

```json
{
  "name": "mysql-source-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",
    "database.server.name": "dbserver1",
    "database.include.list": "inventory",
    "table.include.list": "inventory.orders",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes.inventory"
  }
}
```

Каждое изменение в таблице `orders` → событие в Kafka.

### Sink Connector: Kafka → Elasticsearch

```json
{
  "name": "elasticsearch-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "1",
    "topics": "orders",
    "connection.url": "http://elasticsearch:9200",
    "type.name": "_doc",
    "key.ignore": "true",
    "schema.ignore": "true"
  }
}
```

События из топика `orders` → автоматически индексируются в Elasticsearch.

## Репликация и отказоустойчивость

### Replication Factor

```bash
# Создание топика с replication factor = 3
kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --topic critical-events \
  --partitions 6 \
  --replication-factor 3
```

```
Partition 0:
  Leader: Broker 1
  Replicas: Broker 1, Broker 2, Broker 3
  ISR (In-Sync Replicas): Broker 1, Broker 2, Broker 3

Partition 1:
  Leader: Broker 2
  Replicas: Broker 2, Broker 3, Broker 1
  ISR: Broker 2, Broker 3, Broker 1
```

**Гарантии:**
- Запись подтверждается после репликации на все ISR (при `acks=all`)
- Если лидер упадёт → один из ISR становится новым лидером
- Данные не теряются при падении `replication_factor - 1` брокеров

### Producer ACKs

```javascript
async function sendWithDifferentAcks() {
  const producer = kafka.producer();
  await producer.connect();

  // acks=0: не ждать подтверждения (fastest, least reliable)
  await producer.send({
    topic: 'events',
    acks: 0,
    messages: [{ value: 'fire-and-forget' }],
  });

  // acks=1: ждать подтверждения от лидера (balanced)
  await producer.send({
    topic: 'events',
    acks: 1,
    messages: [{ value: 'leader-only' }],
  });

  // acks=all: ждать подтверждения от всех ISR (slowest, most reliable)
  await producer.send({
    topic: 'events',
    acks: -1,
    messages: [{ value: 'all-replicas' }],
  });

  await producer.disconnect();
}

sendWithDifferentAcks().catch(console.error);
```

| acks | Latency | Durability | Use Case |
|------|---------|------------|----------|
| 0 | Lowest | Lowest | Metrics, non-critical logs |
| 1 | Medium | Medium | General events |
| all | Highest | Highest | Financial transactions |

### Min In-Sync Replicas

```bash
# Конфигурация топика
kafka-configs --alter \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name critical-events \
  --add-config min.insync.replicas=2
```

С `replication.factor=3` и `min.insync.replicas=2`:
- Запись успешна, если подтверждена ≥2 реплик
- Кластер выдержит падение 1 брокера без потери доступности

## Retention и Compaction

### Time-based Retention

```bash
# Хранить события 7 дней
kafka-configs --alter \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name orders \
  --add-config retention.ms=604800000

# Хранить до 10 GB на партицию
kafka-configs --alter \
  --entity-type topics \
  --entity-name orders \
  --add-config retention.bytes=10737418240
```

### Log Compaction

Для топиков с ключами: сохраняется только **последнее** значение для каждого ключа.

```bash
kafka-configs --alter \
  --entity-type topics \
  --entity-name user-profiles \
  --add-config cleanup.policy=compact
```

```
До compaction:
Offset: 0    1    2    3    4    5
Key:   user1 user2 user1 user3 user2 user1
Value:  v1    v2    v3    v4    v5    v6

После compaction:
Offset: 3    4    5
Key:   user3 user2 user1
Value:  v4    v5    v6
```

**Use cases:**
- User profiles (последнее состояние пользователя)
- Configuration changes
- Database changelog (CDC)

## Мониторинг Kafka

### Ключевые метрики

**Producer:**
- `record-send-rate`: msg/s
- `record-error-rate`: ошибки/s
- `request-latency-avg`: средняя задержка

**Consumer:**
- `records-consumed-rate`: msg/s
- `records-lag-max`: максимальное отставание (offset)
- `fetch-latency-avg`: задержка чтения

**Broker:**
- `UnderReplicatedPartitions`: партиции без полной репликации
- `OfflinePartitionsCount`: недоступные партиции
- `BytesInPerSec`, `BytesOutPerSec`: пропускная способность

### Consumer Lag

```bash
# Проверка lag для consumer group
kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group email-service \
  --describe

# Вывод:
# TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# orders   0          1000            1500            500
# orders   1          2000            2000            0
# orders   2          1500            2500            1000
```

Lag = LOG-END-OFFSET - CURRENT-OFFSET

**Проблема:** если lag растёт → консьюмер не успевает обрабатывать события.

**Решения:**
- Добавить больше консьюмеров (до числа партиций)
- Оптимизировать обработку
- Увеличить число партиций

### Prometheus + Grafana

```yaml
# JMX Exporter для Kafka
services:
  kafka:
    environment:
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote
                       -Dcom.sun.management.jmxremote.port=9999
                       -Dcom.sun.management.jmxremote.authenticate=false
                       -Dcom.sun.management.jmxremote.ssl=false"

  jmx-exporter:
    image: sscaling/jmx-prometheus-exporter
    ports:
      - "5556:5556"
    environment:
      SERVICE_PORT: 5556
    volumes:
      - ./kafka-jmx.yml:/etc/jmx-exporter/config.yml

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
```

## Практические паттерны

### 1. Event Sourcing

Все изменения состояния = события в Kafka.

```javascript
async function createOrderEvent(orderData) {
  const producer = kafka.producer();
  await producer.connect();

  const event = {
    event_type: 'OrderCreated',
    order_id: generateId(),
    user_id: orderData.user_id,
    items: orderData.items,
    total: orderData.total,
    timestamp: Date.now() / 1000,
  };

  await producer.send({
    topic: 'order-events',
    messages: [
      {
        key: event.order_id,
        value: JSON.stringify(event),
      },
    ],
  });

  await producer.disconnect();
  return event.order_id;
}

// Event handler: восстановление состояния
async function rebuildOrderState(orderId) {
  const consumer = kafka.consumer({ groupId: `replay-${orderId}-${Date.now()}` });
  await consumer.connect();
  await consumer.subscribe({ topic: 'order-events', fromBeginning: true });

  let order = null;

  await new Promise((resolve) => {
    consumer.run({
      eachMessage: async ({ message }) => {
        const event = JSON.parse(message.value.toString());
        if (event.order_id !== orderId) {
          return;
        }

        if (event.event_type === 'OrderCreated') {
          order = {
            id: event.order_id,
            user_id: event.user_id,
            items: event.items,
            total: event.total,
            status: 'created',
          };
        } else if (event.event_type === 'OrderPaid' && order) {
          order.status = 'paid';
          order.paid_at = event.timestamp;
        } else if (event.event_type === 'OrderShipped' && order) {
          order.status = 'shipped';
          order.tracking_number = event.tracking_number;
          // Последнее событие — можно завершать восстановление
          consumer.stop().then(resolve);
        }
      },
    });
  });

  await consumer.disconnect();
  return order;
}
```

### 2. CQRS (Command Query Responsibility Segregation)

Разделение записи и чтения через Kafka.

```javascript
// Write Side: Commands → Events
async function createOrderCommand(orderData, readDb) {
  const producer = kafka.producer();
  await producer.connect();

  const event = {
    event_type: 'OrderCreated',
    order_id: generateId(),
    ...orderData,
  };

  await producer.send({
    topic: 'order-events',
    messages: [{ value: JSON.stringify(event) }],
  });

  await producer.disconnect();
  return { order_id: event.order_id };
}

// Read Side: Consumer → Read DB (Elasticsearch, MongoDB)
async function startOrderReadSide(readDb) {
  const consumer = kafka.consumer({ groupId: 'order-read-side' });
  await consumer.connect();
  await consumer.subscribe({ topic: 'order-events', fromBeginning: true });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());

      if (event.event_type === 'OrderCreated') {
        // Сохранить в read-optimized БД
        await readDb.index({
          index: 'orders',
          id: event.order_id,
          document: event,
        });
      }
    },
  });
}

// Query API: читает из read DB
async function getOrder(orderId, readDb) {
  return readDb.get({ index: 'orders', id: orderId });
}
```

### 3. Change Data Capture (CDC)

Debezium: репликация изменений БД в Kafka.

```javascript
async function consumeCdcChanges() {
  // MySQL table changes → Kafka events
  const consumer = kafka.consumer({ groupId: 'cdc-listener' });
  await consumer.connect();
  await consumer.subscribe({ topic: 'dbserver1.inventory.orders' });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const change = JSON.parse(message.value.toString());

      if (change.op === 'c') {
        console.log('New order:', change.after);
      } else if (change.op === 'u') {
        console.log('Order updated:', {
          before: change.before,
          after: change.after,
        });
      } else if (change.op === 'd') {
        console.log('Order deleted:', change.before);
      }
    },
  });
}

consumeCdcChanges().catch(console.error);
```

### 4. Saga Pattern

Распределённые транзакции через события.

```javascript
// Saga: Order → Payment → Inventory → Shipping

async function createOrderSaga(orderData) {
  const producer = kafka.producer();
  await producer.connect();

  await producer.send({
    topic: 'saga-commands',
    messages: [
      {
        value: JSON.stringify({
          saga_id: generateId(),
          command: 'CreateOrder',
          data: orderData,
        }),
      },
    ],
  });

  await producer.disconnect();
}

async function startOrderSagaService() {
  const consumer = kafka.consumer({ groupId: 'order-saga-service' });
  const producer = kafka.producer();
  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topic: 'saga-commands' });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const cmd = JSON.parse(message.value.toString());

      if (cmd.command === 'CreateOrder') {
        const orderId = await createOrder(cmd.data);

        await producer.send({
          topic: 'saga-commands',
          messages: [
            {
              value: JSON.stringify({
                saga_id: cmd.saga_id,
                command: 'ProcessPayment',
                order_id: orderId,
                amount: cmd.data.total,
              }),
            },
          ],
        });
      }
    },
  });
}

async function startPaymentSagaService() {
  const consumer = kafka.consumer({ groupId: 'payment-saga-service' });
  const producer = kafka.producer();
  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topic: 'saga-commands' });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const cmd = JSON.parse(message.value.toString());

      if (cmd.command === 'ProcessPayment') {
        try {
          await chargeCard(cmd.amount);

          await producer.send({
            topic: 'saga-commands',
            messages: [
              {
                value: JSON.stringify({
                  saga_id: cmd.saga_id,
                  command: 'ReserveInventory',
                  order_id: cmd.order_id,
                }),
              },
            ],
          });
        } catch (error) {
          await producer.send({
            topic: 'saga-commands',
            messages: [
              {
                value: JSON.stringify({
                  saga_id: cmd.saga_id,
                  command: 'CancelOrder',
                  order_id: cmd.order_id,
                  reason: 'PaymentFailed',
                }),
              },
            ],
          });
        }
      }
    },
  });
}

async function startInventorySagaService() {
  const consumer = kafka.consumer({ groupId: 'inventory-saga-service' });
  const producer = kafka.producer();
  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topic: 'saga-commands' });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const cmd = JSON.parse(message.value.toString());

      if (cmd.command === 'ReserveInventory') {
        try {
          await reserveItems(cmd.order_id);

          await producer.send({
            topic: 'saga-commands',
            messages: [
              {
                value: JSON.stringify({
                  saga_id: cmd.saga_id,
                  command: 'ScheduleShipping',
                  order_id: cmd.order_id,
                }),
              },
            ],
          });
        } catch (error) {
          await producer.send({
            topic: 'saga-commands',
            messages: [
              {
                value: JSON.stringify({
                  saga_id: cmd.saga_id,
                  command: 'RefundPayment',
                  order_id: cmd.order_id,
                }),
              },
            ],
          });
        }
      } else if (cmd.command === 'RefundPayment') {
        await refund(cmd.order_id);
        await cancelOrder(cmd.order_id);
      }
    },
  });
}
```

### 5. Dead Letter Queue

```javascript
async function processWithDlq() {
  const consumer = kafka.consumer({ groupId: 'orders-worker' });
  const producer = kafka.producer();

  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topic: 'orders' });

  const maxRetries = 3;

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      let retries = 0;

      while (retries < maxRetries) {
        try {
          await processMessage(JSON.parse(message.value.toString()));
          await consumer.commitOffsets([
            {
              topic,
              partition,
              offset: (Number(message.offset) + 1).toString(),
            },
          ]);
          break;
        } catch (error) {
          retries += 1;
          console.warn(`Retry ${retries}/${maxRetries}: ${error.message}`);

          if (retries === maxRetries) {
            await producer.send({
              topic: 'orders-dlq',
              messages: [
                {
                  value: message.value,
                  headers: {
                    original_topic: Buffer.from('orders'),
                    error: Buffer.from(error.message),
                    retries: Buffer.from(String(retries)),
                  },
                },
              ],
            });
            await consumer.commitOffsets([
              {
                topic,
                partition,
                offset: (Number(message.offset) + 1).toString(),
              },
            ]);
          }
        }
      }
    },
  });
}

processWithDlq().catch(console.error);
```

## Kafka vs RabbitMQ vs SQS

| Критерий | Kafka | RabbitMQ | SQS |
|----------|-------|----------|-----|
| **Use Case** | Event streaming, history | Task queues | Simple queues |
| **Retention** | Configurable (days/TB) | До ACK | До 14 дней |
| **Replay** | ✅ Да (offset seek) | ❌ Нет | ❌ Нет |
| **Throughput** | Millions msg/s | Hundreds of K msg/s | 3K (FIFO), unlimited (Std) |
| **Ordering** | Per-partition | Per-queue | FIFO queues |
| **Complexity** | High | Medium | Low |
| **Latency** | Low (ms) | Very low (µs) | Medium (100-200ms) |
| **Scalability** | Horizontal (partitions) | Vertical + clustering | AWS managed |
| **Best for** | Logs, analytics, CDC, ES | RPC, task distribution | AWS ecosystem |

**Когда использовать Kafka:**
- Нужна история событий (audit log)
- Event sourcing
- Потоковая аналитика
- Change Data Capture
- Высокая нагрузка (millions msg/s)
- Интеграция многих сервисов (data hub)

## Выводы

Apache Kafka — это не просто очередь сообщений, это **платформа для потоковой обработки событий**:

**Ключевые преимущества:**
- **Durability**: события хранятся, можно перечитать
- **Scalability**: миллионы событий в секунду через партиции
- **Fault-tolerance**: репликация и ISR
- **Decoupling**: сервисы общаются через события
- **Stream processing**: обработка данных в реальном времени

**Основные концепции:**
- Topics & Partitions для масштабирования
- Consumer Groups для параллельной обработки
- Offsets для контроля позиции чтения
- Replication для отказоустойчивости
- Retention & Compaction для управления хранением

Kafka — сложнее, чем традиционные очереди, но даёт уникальные возможности для построения event-driven архитектур.

## Что читать дальше?

- [Урок 23: Async Workers: Celery, BullMQ](23-async-workers.md)
- [Урок 24: WebSockets в production](24-websockets-production.md)
- [Урок 31: Consensus алгоритмы](31-consensus-algorithms.md)

## Проверь себя

1. В чём основное отличие Kafka от RabbitMQ и SQS?
2. Что такое partition и зачем они нужны?
3. Как работает Consumer Group и partition assignment?
4. Что такое offset и как он управляется?
5. В чём разница между `acks=0`, `acks=1` и `acks=all`?
6. Что такое replication factor и ISR?
7. Чем отличается time-based retention от log compaction?
8. Как реализовать exactly-once processing в Kafka?
9. Что такое Consumer Lag и как его мониторить?
10. Когда использовать Kafka вместо традиционных message queues?

---

[← Предыдущий урок: Message Queues](21-message-queues.md) | [Следующий урок: Async Workers →](23-async-workers.md)
