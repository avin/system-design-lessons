# Урок 21: Message Queues — RabbitMQ, SQS

[← Предыдущий урок: Distributed Caching](20-distributed-caching.md) | [Следующий урок: Kafka и Event Streaming →](22-kafka-event-streaming.md)

## Введение

Очереди сообщений (Message Queues) — это фундаментальный паттерн асинхронной коммуникации в распределенных системах. Они позволяют компонентам системы обмениваться сообщениями без необходимости быть одновременно доступными или знать друг о друге напрямую.

**Зачем нужны очереди сообщений?**

Представьте сервис регистрации пользователей. При регистрации нужно:
- Сохранить данные в БД (критично)
- Отправить welcome email (можно с задержкой)
- Обновить аналитику (можно с задержкой)
- Уведомить CRM-систему (можно с задержкой)

Без очередей пользователь будет ждать выполнения всех операций. С очередями — только критичных.

## Основные концепции

### Компоненты Message Queue

```
Producer → [Exchange] → [Queue] → Consumer
   ↑                               ↓
   └──────── Acknowledgment ────────┘
```

**Основные роли:**
- **Producer (Publisher)** — отправляет сообщения
- **Queue** — хранит сообщения до обработки
- **Consumer (Subscriber)** — получает и обработывает сообщения
- **Exchange/Router** — маршрутизирует сообщения (в некоторых системах)
- **Broker** — центральный компонент, управляющий очередями

### Модели доставки

| Модель | Описание | Пример использования |
|--------|----------|---------------------|
| **At-most-once** | Сообщение может потеряться | Метрики, неважная аналитика |
| **At-least-once** | Сообщение может дублироваться | Большинство бизнес-сценариев |
| **Exactly-once** | Сообщение обработается ровно 1 раз | Финансовые транзакции |

### Паттерны обмена сообщениями

**1. Point-to-Point (Queue)**
```
Producer → Queue → Consumer
```
Одно сообщение обрабатывается одним консьюмером.

**2. Publish-Subscribe (Topic)**
```
         ┌→ Consumer 1
Producer → Topic → Consumer 2
         └→ Consumer 3
```
Одно сообщение доставляется всем подписчикам.

**3. Request-Reply**
```
Client → Request Queue → Server
Client ← Reply Queue ← Server
```
Асинхронный RPC через очереди.

## RabbitMQ

RabbitMQ — мощный message broker с поддержкой протокола AMQP (Advanced Message Queuing Protocol).

### Архитектура RabbitMQ

```
Producer → Exchange → Binding → Queue → Consumer
              ↓
         Routing Key
```

**Типы Exchange:**

| Тип | Описание | Routing Key |
|-----|----------|-------------|
| **Direct** | Точное совпадение routing key | Обязателен |
| **Fanout** | Broadcast всем очередям | Игнорируется |
| **Topic** | Паттерн-matching (wildcards) | Паттерн: `*.logs.#` |
| **Headers** | Маршрутизация по заголовкам | Игнорируется |

### Пример: Direct Exchange

```python
import pika

# Подключение к RabbitMQ
connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Объявление exchange и очереди
channel.exchange_declare(exchange='direct_logs', exchange_type='direct')
channel.queue_declare(queue='error_logs')

# Binding: связываем очередь с exchange через routing key
channel.queue_bind(
    exchange='direct_logs',
    queue='error_logs',
    routing_key='error'
)

# Producer: отправка сообщения
channel.basic_publish(
    exchange='direct_logs',
    routing_key='error',
    body='Database connection failed',
    properties=pika.BasicProperties(
        delivery_mode=2,  # persistent message
    )
)

print("Message sent")
connection.close()
```

### Пример: Consumer с Acknowledgment

```python
import pika
import time

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)

def callback(ch, method, properties, body):
    print(f"Received: {body.decode()}")

    # Имитация долгой обработки
    time.sleep(body.count(b'.'))

    print("Done")

    # Manual acknowledgment
    ch.basic_ack(delivery_tag=method.delivery_tag)

# Prefetch: обрабатывать по 1 сообщению за раз
channel.basic_qos(prefetch_count=1)

channel.basic_consume(
    queue='task_queue',
    on_message_callback=callback,
    auto_ack=False  # Manual ack
)

print('Waiting for messages...')
channel.start_consuming()
```

### Topic Exchange с Wildcards

```python
# Producer
channel.exchange_declare(exchange='logs_topic', exchange_type='topic')

# Отправка логов разных уровней
channel.basic_publish(
    exchange='logs_topic',
    routing_key='app.error.database',
    body='DB error occurred'
)

channel.basic_publish(
    exchange='logs_topic',
    routing_key='app.info.user',
    body='User logged in'
)

# Consumer 1: все ошибки
channel.queue_bind(
    exchange='logs_topic',
    queue='all_errors',
    routing_key='*.error.*'
)

# Consumer 2: все логи приложения
channel.queue_bind(
    exchange='logs_topic',
    queue='app_logs',
    routing_key='app.#'
)

# Consumer 3: только ошибки БД
channel.queue_bind(
    exchange='logs_topic',
    routing_key='*.error.database'
)
```

**Wildcards:**
- `*` — заменяет ровно одно слово
- `#` — заменяет 0 или более слов

### Dead Letter Exchange (DLX)

Обработка "мёртвых" сообщений, которые не смогли обработаться:

```python
# Основная очередь с DLX
channel.queue_declare(
    queue='main_queue',
    arguments={
        'x-dead-letter-exchange': 'dlx_exchange',
        'x-dead-letter-routing-key': 'failed',
        'x-message-ttl': 60000  # 60 секунд
    }
)

# Dead Letter Queue
channel.exchange_declare(exchange='dlx_exchange', exchange_type='direct')
channel.queue_declare(queue='dead_letters')
channel.queue_bind(
    exchange='dlx_exchange',
    queue='dead_letters',
    routing_key='failed'
)

# Consumer с возможностью reject
def callback(ch, method, properties, body):
    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        # Reject без requeue → попадёт в DLX
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            requeue=False
        )
```

### RabbitMQ Best Practices

1. **Используйте persistent messages и durable queues** для критичных данных
2. **Manual acknowledgments** вместо auto-ack для надёжности
3. **Prefetch count** для контроля нагрузки на консьюмеров
4. **DLX** для обработки проблемных сообщений
5. **Publisher confirms** для гарантии доставки
6. **Heartbeats** для обнаружения потери соединения
7. **Connection pooling** для производительности

```python
# Publisher Confirms
channel.confirm_delivery()

try:
    channel.basic_publish(
        exchange='',
        routing_key='queue_name',
        body='message',
        mandatory=True  # Вернёт ошибку, если нет очереди
    )
    print("Message confirmed")
except pika.exceptions.UnroutableError:
    print("Message could not be routed")
```

## Amazon SQS

AWS Simple Queue Service — полностью управляемый сервис очередей.

### Типы очередей SQS

| Характеристика | Standard Queue | FIFO Queue |
|----------------|----------------|------------|
| **Throughput** | Почти неограниченный | До 3000 msg/s (batch: 30000) |
| **Ordering** | Best-effort | Строгий порядок |
| **Delivery** | At-least-once | Exactly-once |
| **Deduplication** | Нет | Есть (5 минут) |
| **Use case** | Высокая нагрузка | Важен порядок |

### Пример: Standard Queue (Python + boto3)

```python
import boto3
import json

# Создание клиента SQS
sqs = boto3.client('sqs', region_name='us-east-1')

# Создание очереди
queue_response = sqs.create_queue(
    QueueName='my-task-queue',
    Attributes={
        'DelaySeconds': '0',
        'MessageRetentionPeriod': '86400',  # 1 день
        'VisibilityTimeout': '30'  # 30 секунд
    }
)

queue_url = queue_response['QueueUrl']

# Отправка сообщения
response = sqs.send_message(
    QueueUrl=queue_url,
    MessageBody=json.dumps({
        'task': 'send_email',
        'to': 'user@example.com',
        'subject': 'Welcome!'
    }),
    MessageAttributes={
        'Priority': {
            'StringValue': 'high',
            'DataType': 'String'
        }
    }
)

print(f"MessageId: {response['MessageId']}")

# Получение и обработка сообщений
messages = sqs.receive_message(
    QueueUrl=queue_url,
    MaxNumberOfMessages=10,  # До 10 сообщений за раз
    WaitTimeSeconds=20,  # Long polling
    MessageAttributeNames=['All']
)

for message in messages.get('Messages', []):
    body = json.loads(message['Body'])
    print(f"Processing: {body}")

    # Обработка...

    # Удаление после успешной обработки
    sqs.delete_message(
        QueueUrl=queue_url,
        ReceiptHandle=message['ReceiptHandle']
    )
```

### FIFO Queue

```python
# Создание FIFO очереди
fifo_queue = sqs.create_queue(
    QueueName='orders.fifo',  # Обязательно .fifo
    Attributes={
        'FifoQueue': 'true',
        'ContentBasedDeduplication': 'true',
        'MessageRetentionPeriod': '345600'  # 4 дня
    }
)

# Отправка с MessageGroupId (для ordering)
sqs.send_message(
    QueueUrl=fifo_queue['QueueUrl'],
    MessageBody='Order #1234',
    MessageGroupId='user_123',  # Сообщения в одной группе — строго по порядку
    MessageDeduplicationId='order-1234-v1'  # Для exactly-once
)

sqs.send_message(
    QueueUrl=fifo_queue['QueueUrl'],
    MessageBody='Order #1235',
    MessageGroupId='user_123',
    MessageDeduplicationId='order-1235-v1'
)

# Второй пользователь — параллельная обработка
sqs.send_message(
    QueueUrl=fifo_queue['QueueUrl'],
    MessageBody='Order #5678',
    MessageGroupId='user_456',
    MessageDeduplicationId='order-5678-v1'
)
```

### Visibility Timeout

Механизм предотвращения двойной обработки:

```
Consumer получил сообщение → Visibility Timeout начался (30s)
    ↓
Сообщение невидимо для других консьюмеров
    ↓
Если обработка > 30s → можно продлить
    ↓
Delete message → убрать из очереди навсегда
ИЛИ
Timeout истёк → сообщение снова видимо
```

```python
# Продление Visibility Timeout
sqs.change_message_visibility(
    QueueUrl=queue_url,
    ReceiptHandle=receipt_handle,
    VisibilityTimeout=60  # Продлить ещё на 60 секунд
)
```

### Dead Letter Queue (DLQ)

```python
# Создание DLQ
dlq = sqs.create_queue(QueueName='failed-tasks-dlq')
dlq_arn = sqs.get_queue_attributes(
    QueueUrl=dlq['QueueUrl'],
    AttributeNames=['QueueArn']
)['Attributes']['QueueArn']

# Основная очередь с DLQ
main_queue = sqs.create_queue(
    QueueName='main-tasks',
    Attributes={
        'RedrivePolicy': json.dumps({
            'deadLetterTargetArn': dlq_arn,
            'maxReceiveCount': '3'  # После 3 попыток → в DLQ
        })
    }
)
```

### Batch Operations

Для оптимизации стоимости и производительности:

```python
# Batch send (до 10 сообщений)
response = sqs.send_message_batch(
    QueueUrl=queue_url,
    Entries=[
        {
            'Id': '1',
            'MessageBody': 'Task 1'
        },
        {
            'Id': '2',
            'MessageBody': 'Task 2',
            'DelaySeconds': 10
        },
        {
            'Id': '3',
            'MessageBody': 'Task 3'
        }
    ]
)

# Batch delete
sqs.delete_message_batch(
    QueueUrl=queue_url,
    Entries=[
        {'Id': '1', 'ReceiptHandle': receipt1},
        {'Id': '2', 'ReceiptHandle': receipt2},
    ]
)
```

### Long Polling vs Short Polling

```python
# Short Polling (default) - возвращает сразу
messages = sqs.receive_message(
    QueueUrl=queue_url,
    WaitTimeSeconds=0  # Не ждать
)

# Long Polling - ждёт до 20 секунд
messages = sqs.receive_message(
    QueueUrl=queue_url,
    WaitTimeSeconds=20,  # Экономия запросов + снижение latency
    MaxNumberOfMessages=10
)
```

**Преимущества Long Polling:**
- Снижение пустых ответов (меньше затрат)
- Уменьшение latency для получения сообщений
- Меньше запросов к AWS API

### SQS Best Practices

1. **Long Polling** вместо Short Polling
2. **Batch operations** для экономии
3. **FIFO** только если нужен строгий порядок
4. **DLQ** для обработки ошибок
5. **Visibility timeout** = время обработки × 6
6. **Message attributes** для метаданных (не в body)
7. **CloudWatch Alarms** на ApproximateAgeOfOldestMessage

## RabbitMQ vs SQS

| Критерий | RabbitMQ | AWS SQS |
|----------|----------|---------|
| **Тип** | Self-hosted или Cloud (CloudAMQP) | Managed service |
| **Протокол** | AMQP, STOMP, MQTT | HTTP/S, AWS SDK |
| **Routing** | Мощный (exchanges, bindings) | Базовый |
| **Ordering** | Через single consumer | FIFO queues |
| **Persistence** | Опционально | Всегда (до 14 дней) |
| **Performance** | До 1M+ msg/s (кластер) | 3000 FIFO, unlimited Standard |
| **Pricing** | Стоимость серверов | Pay-per-request |
| **Complexity** | Выше (нужно управлять) | Ниже (AWS управляет) |
| **Features** | Больше (priority, TTL, DLX) | Базовые |
| **Best for** | Сложная маршрутизация, on-premise | Простые очереди, AWS-экосистема |

## Практические сценарии

### 1. Обработка заказов (E-commerce)

```python
# RabbitMQ Topic Exchange
channel.exchange_declare(exchange='orders', exchange_type='topic')

# Producer: новый заказ
order = {
    'order_id': '12345',
    'user_id': '789',
    'total': 99.99,
    'items': [...]
}

# Routing key: order.{status}.{priority}
channel.basic_publish(
    exchange='orders',
    routing_key='order.new.high',
    body=json.dumps(order)
)

# Consumer 1: Инвентарь (все заказы)
channel.queue_bind(exchange='orders', routing_key='order.#')

# Consumer 2: Уведомления (только срочные)
channel.queue_bind(exchange='orders', routing_key='order.*.high')

# Consumer 3: Аналитика (все новые)
channel.queue_bind(exchange='orders', routing_key='order.new.*')
```

### 2. Email рассылка (SQS)

```python
import boto3
from concurrent.futures import ThreadPoolExecutor

sqs = boto3.client('sqs')
queue_url = 'https://sqs.us-east-1.amazonaws.com/123/email-queue'

# Producer: массовая рассылка
users = get_users_from_db()  # 100,000 пользователей

def send_batch(batch):
    entries = [
        {
            'Id': str(i),
            'MessageBody': json.dumps({
                'to': user['email'],
                'template': 'newsletter',
                'data': {'name': user['name']}
            })
        }
        for i, user in enumerate(batch)
    ]
    sqs.send_message_batch(QueueUrl=queue_url, Entries=entries)

# Отправка батчами по 10
batches = [users[i:i+10] for i in range(0, len(users), 10)]
with ThreadPoolExecutor(max_workers=10) as executor:
    executor.map(send_batch, batches)

# Consumer: отправка email
def process_emails():
    while True:
        messages = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20
        )

        for msg in messages.get('Messages', []):
            data = json.loads(msg['Body'])
            send_email(data['to'], data['template'], data['data'])

            sqs.delete_message(
                QueueUrl=queue_url,
                ReceiptHandle=msg['ReceiptHandle']
            )
```

### 3. Image Processing Pipeline

```python
# RabbitMQ: цепочка обработки
channel.queue_declare(queue='images.upload')
channel.queue_declare(queue='images.resize')
channel.queue_declare(queue='images.watermark')
channel.queue_declare(queue='images.publish')

# Service 1: Upload handler
def on_upload(ch, method, properties, body):
    image = json.loads(body)

    # Сохранить оригинал в S3
    save_to_s3(image['url'], 'originals/')

    # Отправить на resize
    ch.basic_publish(
        exchange='',
        routing_key='images.resize',
        body=json.dumps({'image_id': image['id'], 'sizes': [800, 400, 200]})
    )
    ch.basic_ack(delivery_tag=method.delivery_tag)

# Service 2: Resizer
def on_resize(ch, method, properties, body):
    data = json.loads(body)

    for size in data['sizes']:
        resized = resize_image(data['image_id'], size)
        save_to_s3(resized, f'resized/{size}/')

    # Отправить на watermarking
    ch.basic_publish(
        exchange='',
        routing_key='images.watermark',
        body=body
    )
    ch.basic_ack(delivery_tag=method.delivery_tag)

# Service 3: Watermark
def on_watermark(ch, method, properties, body):
    data = json.loads(body)
    add_watermark(data['image_id'])

    # Финальная публикация
    ch.basic_publish(
        exchange='',
        routing_key='images.publish',
        body=body
    )
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

### 4. Priority Queue (RabbitMQ)

```python
# Очередь с приоритетами
channel.queue_declare(
    queue='tasks',
    arguments={'x-max-priority': 10}
)

# Отправка с приоритетом
channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Critical task',
    properties=pika.BasicProperties(priority=10)
)

channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Normal task',
    properties=pika.BasicProperties(priority=5)
)

channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Low priority task',
    properties=pika.BasicProperties(priority=1)
)

# Consumer обработает в порядке: 10 → 5 → 1
```

### 5. Delay Queue (SQS)

```python
# Отложенная отправка (до 15 минут в SQS)
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody='Reminder: meeting in 15 min',
    DelaySeconds=900  # 15 минут
)

# Для больших задержек: использовать visibility timeout или SNS + Lambda
```

## Мониторинг и отладка

### RabbitMQ Management UI

```bash
# Включение management plugin
rabbitmq-plugins enable rabbitmq_management

# UI доступен на http://localhost:15672
# Default credentials: guest/guest
```

**Ключевые метрики:**
- Message rate (in/out)
- Queue depth
- Consumer count
- Unacknowledged messages
- Connection count

### RabbitMQ CLI

```bash
# Список очередей
rabbitmqctl list_queues name messages consumers

# Детали очереди
rabbitmqctl list_queues name messages_ready messages_unacknowledged

# Очистка очереди
rabbitmqctl purge_queue queue_name

# Список exchanges
rabbitmqctl list_exchanges name type

# Список bindings
rabbitmqctl list_bindings
```

### SQS CloudWatch Metrics

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

# Получить метрики
response = cloudwatch.get_metric_statistics(
    Namespace='AWS/SQS',
    MetricName='ApproximateNumberOfMessagesVisible',
    Dimensions=[
        {'Name': 'QueueName', 'Value': 'my-queue'}
    ],
    StartTime=datetime.utcnow() - timedelta(hours=1),
    EndTime=datetime.utcnow(),
    Period=300,
    Statistics=['Average', 'Maximum']
)

# Alarm на старые сообщения
cloudwatch.put_metric_alarm(
    AlarmName='SQS-OldMessages',
    MetricName='ApproximateAgeOfOldestMessage',
    Namespace='AWS/SQS',
    Statistic='Maximum',
    Period=300,
    EvaluationPeriods=1,
    Threshold=3600,  # 1 час
    ComparisonOperator='GreaterThanThreshold',
    Dimensions=[
        {'Name': 'QueueName', 'Value': 'my-queue'}
    ]
)
```

## Паттерны отказоустойчивости

### Idempotent Consumer

```python
import redis

redis_client = redis.Redis()

def process_message_idempotently(message_id, message_body):
    # Проверка, не обработано ли уже
    if redis_client.exists(f'processed:{message_id}'):
        print(f"Message {message_id} already processed")
        return

    # Обработка
    result = process(message_body)

    # Пометка как обработанного (с TTL)
    redis_client.setex(
        f'processed:{message_id}',
        86400,  # 24 часа
        'true'
    )

    return result
```

### Circuit Breaker для Consumer

```python
from datetime import datetime, timedelta

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = 'closed'  # closed, open, half-open

    def call(self, func, *args, **kwargs):
        if self.state == 'open':
            if datetime.now() - self.last_failure_time > timedelta(seconds=self.timeout):
                self.state = 'half-open'
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = func(*args, **kwargs)
            if self.state == 'half-open':
                self.state = 'closed'
                self.failures = 0
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure_time = datetime.now()

            if self.failures >= self.failure_threshold:
                self.state = 'open'

            raise e

# Использование
cb = CircuitBreaker()

def consume_messages():
    while True:
        messages = sqs.receive_message(...)
        for msg in messages.get('Messages', []):
            try:
                cb.call(process_message, msg)
                sqs.delete_message(...)
            except Exception as e:
                print(f"Circuit breaker prevented call: {e}")
                time.sleep(60)
```

### Retry with Exponential Backoff

```python
import time
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=1, max_delay=60):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise

                    delay = min(base_delay * (2 ** attempt), max_delay)
                    print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay}s...")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry_with_backoff(max_retries=5)
def send_to_queue(message):
    channel.basic_publish(
        exchange='',
        routing_key='task_queue',
        body=message
    )
```

## Когда использовать что?

### Используйте RabbitMQ когда:

✅ Нужна сложная маршрутизация (Topic/Headers exchange)
✅ Требуется приоритизация сообщений
✅ On-premise развёртывание
✅ Нужны гарантии порядка через single consumer
✅ Продвинутые фичи (TTL, DLX, Federation)
✅ Поддержка нескольких протоколов (AMQP, STOMP, MQTT)

### Используйте SQS когда:

✅ Простые очереди без сложной логики
✅ AWS-экосистема (Lambda, ECS, etc.)
✅ Не хочется управлять инфраструктурой
✅ Нужна неограниченная масштабируемость
✅ Важна стоимость и платите только за использование
✅ FIFO + Exactly-once delivery

### Комбинирование

```python
# SNS (pub-sub) → SQS (queue) → Lambda (processing)
import boto3

sns = boto3.client('sns')
sqs = boto3.client('sqs')

# Создать топик
topic = sns.create_topic(Name='orders')

# Создать очереди для разных сервисов
inventory_queue = sqs.create_queue(QueueName='inventory-service')
email_queue = sqs.create_queue(QueueName='email-service')
analytics_queue = sqs.create_queue(QueueName='analytics-service')

# Подписать очереди на топик
for queue in [inventory_queue, email_queue, analytics_queue]:
    queue_arn = sqs.get_queue_attributes(
        QueueUrl=queue['QueueUrl'],
        AttributeNames=['QueueArn']
    )['Attributes']['QueueArn']

    sns.subscribe(
        TopicArn=topic['TopicArn'],
        Protocol='sqs',
        Endpoint=queue_arn
    )

# Публикация в топик → все очереди получат сообщение
sns.publish(
    TopicArn=topic['TopicArn'],
    Message=json.dumps({'order_id': '12345', 'total': 99.99})
)
```

## Выводы

Message Queues — это фундамент асинхронной архитектуры:

**Ключевые преимущества:**
- **Decoupling** — сервисы не зависят друг от друга
- **Scalability** — независимое масштабирование producers/consumers
- **Reliability** — сообщения не теряются при падении сервисов
- **Load leveling** — сглаживание пиков нагрузки
- **Fault tolerance** — retry и DLQ для обработки ошибок

**RabbitMQ** — мощный и гибкий, но требует управления.
**SQS** — простой и надёжный managed-сервис от AWS.

Выбор зависит от требований к маршрутизации, инфраструктуре и бюджету.

## Что читать дальше?

- [Урок 22: Kafka и Event Streaming](22-kafka-event-streaming.md)
- [Урок 23: Async Workers: Celery, BullMQ](23-async-workers.md)
- [Урок 8: API Design](08-api-design-rest-graphql-grpc.md)

## Проверь себя

1. В чём разница между At-least-once и Exactly-once delivery?
2. Когда использовать Direct vs Topic exchange в RabbitMQ?
3. Что такое Visibility Timeout в SQS и зачем он нужен?
4. Чем отличается Standard Queue от FIFO Queue в SQS?
5. Что произойдёт, если consumer упадёт без acknowledgment сообщения?
6. Как реализовать приоритетную очередь в RabbitMQ и SQS?
7. Зачем нужен Dead Letter Exchange/Queue?
8. В чём преимущество Long Polling перед Short Polling?
9. Как гарантировать idempotency при обработке сообщений?
10. Когда имеет смысл использовать RabbitMQ вместо SQS и наоборот?

---

[← Предыдущий урок: Distributed Caching](20-distributed-caching.md) | [Следующий урок: Kafka и Event Streaming →](22-kafka-event-streaming.md)
