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

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Отправка сообщения
order = {
    'order_id': '12345',
    'user_id': '789',
    'total': 99.99,
    'items': ['item1', 'item2']
}

future = producer.send('orders', value=order)

# Ожидание подтверждения
try:
    record_metadata = future.get(timeout=10)
    print(f"Sent to partition {record_metadata.partition} at offset {record_metadata.offset}")
except Exception as e:
    print(f"Error: {e}")

producer.close()
```

### Producer с ключом (для партиционирования)

```python
# Сообщения с одинаковым ключом попадут в одну партицию
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    key_serializer=lambda k: k.encode('utf-8'),
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Все заказы пользователя 789 — в одной партиции (сохраняется порядок)
for i in range(10):
    producer.send(
        'orders',
        key='user_789',  # Partition = hash(key) % num_partitions
        value={'order_id': f'order_{i}', 'total': i * 10}
    )

producer.flush()
```

### Асинхронный Producer с callback

```python
def on_send_success(record_metadata):
    print(f"Message sent to {record_metadata.topic} "
          f"partition {record_metadata.partition} "
          f"offset {record_metadata.offset}")

def on_send_error(exc):
    print(f"Error sending message: {exc}")

# Async send
producer.send('orders', value=order) \
    .add_callback(on_send_success) \
    .add_errback(on_send_error)
```

### Producer с compression

```python
# Сжатие для экономии bandwidth и storage
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    compression_type='gzip',  # gzip, snappy, lz4, zstd
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)
```

### Idempotent Producer

```python
# Гарантия exactly-once для producer (дедупликация)
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    enable_idempotence=True,  # Предотвращает дубликаты
    acks='all',  # Ожидание подтверждения от всех реплик
    retries=5,
    max_in_flight_requests_per_connection=5
)
```

### Транзакционный Producer

```python
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    transactional_id='my-transactional-id',
    enable_idempotence=True
)

producer.init_transactions()

try:
    producer.begin_transaction()

    # Атомарная отправка нескольких сообщений
    producer.send('orders', value={'order_id': '1'})
    producer.send('inventory', value={'product_id': 'A', 'qty': -1})
    producer.send('payments', value={'amount': 99.99})

    producer.commit_transaction()
except Exception as e:
    producer.abort_transaction()
    print(f"Transaction aborted: {e}")
finally:
    producer.close()
```

## Consumer (Python)

### Базовый Consumer

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'orders',  # Topic
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',  # earliest, latest, none
    enable_auto_commit=True,
    group_id='email-service',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

# Бесконечный цикл чтения
for message in consumer:
    print(f"Partition: {message.partition}, Offset: {message.offset}")
    print(f"Key: {message.key}, Value: {message.value}")

    # Обработка события
    order = message.value
    send_confirmation_email(order['user_id'], order['order_id'])
```

### Manual Commit

```python
consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    enable_auto_commit=False,  # Manual commit
    group_id='inventory-service'
)

for message in consumer:
    try:
        process_order(message.value)

        # Commit после успешной обработки
        consumer.commit()
    except Exception as e:
        print(f"Error processing message: {e}")
        # Не коммитим → сообщение обработается повторно
```

### Batch Processing

```python
from kafka import TopicPartition

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    enable_auto_commit=False,
    max_poll_records=100  # Читать до 100 сообщений за раз
)

while True:
    messages = consumer.poll(timeout_ms=1000)

    if not messages:
        continue

    batch = []
    for topic_partition, records in messages.items():
        for record in records:
            batch.append(record.value)

    # Обработка батча
    if batch:
        process_batch(batch)
        consumer.commit()
```

### Partition Assignment

```python
from kafka import KafkaConsumer, TopicPartition

consumer = KafkaConsumer(
    bootstrap_servers=['localhost:9092'],
    enable_auto_commit=False
)

# Ручное назначение партиций (без consumer group)
partition_0 = TopicPartition('orders', 0)
partition_1 = TopicPartition('orders', 1)

consumer.assign([partition_0, partition_1])

# Seek к конкретному offset
consumer.seek(partition_0, 100)  # Начать с offset 100

for message in consumer:
    print(f"Partition {message.partition}: {message.value}")
```

### Rebalance Listener

```python
from kafka import KafkaConsumer, ConsumerRebalanceListener, TopicPartition

class RebalanceListener(ConsumerRebalanceListener):
    def __init__(self, consumer):
        self.consumer = consumer

    def on_partitions_revoked(self, revoked):
        print(f"Partitions revoked: {revoked}")
        # Сохранить состояние перед rebalance
        self.consumer.commit()

    def on_partitions_assigned(self, assigned):
        print(f"Partitions assigned: {assigned}")
        # Загрузить состояние после rebalance

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='service-1'
)

listener = RebalanceListener(consumer)
consumer.subscribe(['orders'], listener=listener)

for message in consumer:
    process(message)
```

## Kafka Streams

### Простой Stream Processing

```python
from kafka import KafkaConsumer, KafkaProducer
import json

# Consumer для входного топика
consumer = KafkaConsumer(
    'raw-events',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

# Producer для выходного топика
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Stream processing: фильтрация и трансформация
for message in consumer:
    event = message.value

    # Фильтр: только успешные покупки
    if event['type'] == 'purchase' and event['status'] == 'completed':

        # Трансформация
        enriched = {
            'order_id': event['order_id'],
            'total': event['total'],
            'timestamp': event['timestamp'],
            'category': categorize(event['items'])
        }

        # Отправка в другой топик
        producer.send('successful-orders', value=enriched)
```

### Windowing и Aggregation

```python
from collections import defaultdict
from datetime import datetime, timedelta
import time

# Tumbling Window: подсчёт событий за 1-минутные окна
window_size = 60  # секунд
windows = defaultdict(int)

consumer = KafkaConsumer(
    'page-views',
    bootstrap_servers=['localhost:9092']
)

def get_window_key(timestamp):
    # Округление до минуты
    dt = datetime.fromtimestamp(timestamp)
    window_start = dt.replace(second=0, microsecond=0)
    return window_start.isoformat()

for message in consumer:
    event = message.value
    timestamp = event['timestamp']

    window_key = get_window_key(timestamp)
    windows[window_key] += 1

    # Периодический вывод результатов
    if len(windows) > 10:
        print("Page views per minute:")
        for window, count in sorted(windows.items()):
            print(f"  {window}: {count}")

        # Очистка старых окон
        now = datetime.now()
        cutoff = (now - timedelta(minutes=10)).replace(second=0, microsecond=0)
        windows = {k: v for k, v in windows.items() if k >= cutoff.isoformat()}
```

### Join двух топиков

```python
from collections import defaultdict
import json

# Join: orders + payments
orders_cache = {}

consumer = KafkaConsumer(
    'orders', 'payments',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

for message in consumer:
    topic = message.topic
    data = message.value

    if topic == 'orders':
        order_id = data['order_id']
        orders_cache[order_id] = data

    elif topic == 'payments':
        order_id = data['order_id']

        # Join: если есть соответствующий заказ
        if order_id in orders_cache:
            order = orders_cache[order_id]

            joined = {
                **order,
                'payment_method': data['method'],
                'payment_status': data['status'],
                'paid_at': data['timestamp']
            }

            producer.send('orders-with-payments', value=joined)

            # Очистка cache
            del orders_cache[order_id]
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

```python
# acks=0: не ждать подтверждения (fastest, least reliable)
producer = KafkaProducer(acks=0)

# acks=1: ждать подтверждения от лидера (balanced)
producer = KafkaProducer(acks=1)

# acks=all: ждать подтверждения от всех ISR (slowest, most reliable)
producer = KafkaProducer(acks='all')
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

```python
# Commands → Events → State

# Command: создать заказ
def create_order(order_data):
    event = {
        'event_type': 'OrderCreated',
        'order_id': generate_id(),
        'user_id': order_data['user_id'],
        'items': order_data['items'],
        'total': order_data['total'],
        'timestamp': time.time()
    }

    producer.send('order-events', key=event['order_id'], value=event)
    return event['order_id']

# Event handler: восстановление состояния
def rebuild_order_state(order_id):
    consumer = KafkaConsumer(
        'order-events',
        auto_offset_reset='earliest'
    )

    order = None

    for message in consumer:
        event = message.value
        if event['order_id'] != order_id:
            continue

        if event['event_type'] == 'OrderCreated':
            order = {
                'id': event['order_id'],
                'user_id': event['user_id'],
                'items': event['items'],
                'total': event['total'],
                'status': 'created'
            }

        elif event['event_type'] == 'OrderPaid':
            order['status'] = 'paid'
            order['paid_at'] = event['timestamp']

        elif event['event_type'] == 'OrderShipped':
            order['status'] = 'shipped'
            order['tracking_number'] = event['tracking_number']

    return order
```

### 2. CQRS (Command Query Responsibility Segregation)

Разделение записи и чтения через Kafka.

```python
# Write Side: Commands → Events
@app.post("/orders")
def create_order(order_data):
    event = {
        'event_type': 'OrderCreated',
        'order_id': generate_id(),
        **order_data
    }

    producer.send('order-events', value=event)
    return {'order_id': event['order_id']}

# Read Side: Consumer → Read DB (Elasticsearch, MongoDB)
consumer = KafkaConsumer('order-events')

for message in consumer:
    event = message.value

    if event['event_type'] == 'OrderCreated':
        # Сохранить в read-optimized БД
        elasticsearch.index(
            index='orders',
            id=event['order_id'],
            body=event
        )

# Query API: читает из read DB
@app.get("/orders/{order_id}")
def get_order(order_id):
    return elasticsearch.get(index='orders', id=order_id)
```

### 3. Change Data Capture (CDC)

Debezium: репликация изменений БД в Kafka.

```python
# MySQL table changes → Kafka events
consumer = KafkaConsumer('dbserver1.inventory.orders')

for message in consumer:
    change = message.value

    if change['op'] == 'c':  # Create
        print(f"New order: {change['after']}")

    elif change['op'] == 'u':  # Update
        print(f"Order updated:")
        print(f"  Before: {change['before']}")
        print(f"  After: {change['after']}")

    elif change['op'] == 'd':  # Delete
        print(f"Order deleted: {change['before']}")
```

### 4. Saga Pattern

Распределённые транзакции через события.

```python
# Saga: Order → Payment → Inventory → Shipping

# Step 1: Create Order
def create_order_saga(order_data):
    producer.send('saga-commands', value={
        'saga_id': generate_id(),
        'command': 'CreateOrder',
        'data': order_data
    })

# Order Service
consumer = KafkaConsumer('saga-commands')

for message in consumer:
    cmd = message.value

    if cmd['command'] == 'CreateOrder':
        order_id = create_order(cmd['data'])

        # Success → следующий шаг
        producer.send('saga-commands', value={
            'saga_id': cmd['saga_id'],
            'command': 'ProcessPayment',
            'order_id': order_id,
            'amount': cmd['data']['total']
        })

# Payment Service
def process_payment_step(cmd):
    try:
        payment_id = charge_card(cmd['amount'])

        producer.send('saga-commands', value={
            'saga_id': cmd['saga_id'],
            'command': 'ReserveInventory',
            'order_id': cmd['order_id']
        })

    except PaymentError:
        # Failure → compensate
        producer.send('saga-commands', value={
            'saga_id': cmd['saga_id'],
            'command': 'CancelOrder',
            'order_id': cmd['order_id']
        })
```

### 5. Dead Letter Queue

```python
from kafka.errors import KafkaError

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    enable_auto_commit=False
)

producer = KafkaProducer(bootstrap_servers=['localhost:9092'])

max_retries = 3

for message in consumer:
    retries = 0

    while retries < max_retries:
        try:
            process_message(message.value)
            consumer.commit()
            break

        except Exception as e:
            retries += 1
            print(f"Retry {retries}/{max_retries}: {e}")

            if retries == max_retries:
                # Отправить в DLQ
                producer.send(
                    'orders-dlq',
                    value=message.value,
                    headers=[
                        ('original_topic', b'orders'),
                        ('error', str(e).encode('utf-8')),
                        ('retries', str(retries).encode('utf-8'))
                    ]
                )
                consumer.commit()
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
