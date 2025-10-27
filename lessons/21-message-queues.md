# Урок 21: Message Queues — RabbitMQ, SQS


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

```javascript
const amqp = require('amqplib');

async function publishDirectLog() {
  // Подключение к RabbitMQ
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  // Объявление exchange и очереди
  await channel.assertExchange('direct_logs', 'direct', { durable: true });
  await channel.assertQueue('error_logs', { durable: true });

  // Binding: связываем очередь с exchange через routing key
  await channel.bindQueue('error_logs', 'direct_logs', 'error');

  // Producer: отправка сообщения
  channel.publish('direct_logs', 'error', Buffer.from('Database connection failed'), {
    persistent: true,
  });

  console.log('Message sent');
  await channel.close();
  await connection.close();
}

publishDirectLog().catch(console.error);
```

### Пример: Consumer с Acknowledgment

```javascript
async function consumeWithAck() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  await channel.assertQueue('task_queue', { durable: true });

  // Prefetch: обрабатывать по 1 сообщению за раз
  await channel.prefetch(1);

  channel.consume(
    'task_queue',
    async (msg) => {
      if (!msg) {
        return;
      }

      const body = msg.content.toString();
      console.log(`Received: ${body}`);

      // Имитация долгой обработки
      const delaySeconds = (body.match(/\./g) || []).length;
      await new Promise((resolve) => setTimeout(resolve, delaySeconds * 1000));

      console.log('Done');

      // Manual acknowledgment
      channel.ack(msg);
    },
    { noAck: false },
  );

  console.log('Waiting for messages...');
}

consumeWithAck().catch(console.error);
```

### Topic Exchange с Wildcards

```javascript
async function setupTopicExchange() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  await channel.assertExchange('logs_topic', 'topic', { durable: true });

  // Отправка логов разных уровней
  channel.publish('logs_topic', 'app.error.database', Buffer.from('DB error occurred'));
  channel.publish('logs_topic', 'app.info.user', Buffer.from('User logged in'));

  // Consumer 1: все ошибки
  await channel.assertQueue('all_errors', { durable: true });
  await channel.bindQueue('all_errors', 'logs_topic', '*.error.*');

  // Consumer 2: все логи приложения
  await channel.assertQueue('app_logs', { durable: true });
  await channel.bindQueue('app_logs', 'logs_topic', 'app.#');

  // Consumer 3: только ошибки БД
  await channel.assertQueue('db_errors', { durable: true });
  await channel.bindQueue('db_errors', 'logs_topic', '*.error.database');

  await channel.close();
  await connection.close();
}

setupTopicExchange().catch(console.error);
```

**Wildcards:**
- `*` — заменяет ровно одно слово
- `#` — заменяет 0 или более слов

### Dead Letter Exchange (DLX)

Обработка "мёртвых" сообщений, которые не смогли обработаться:

```javascript
async function setupDeadLetterExchange() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  // Dead Letter Queue
  await channel.assertExchange('dlx_exchange', 'direct', { durable: true });
  await channel.assertQueue('dead_letters', { durable: true });
  await channel.bindQueue('dead_letters', 'dlx_exchange', 'failed');

  // Основная очередь с DLX
  await channel.assertQueue('main_queue', {
    durable: true,
    arguments: {
      'x-dead-letter-exchange': 'dlx_exchange',
      'x-dead-letter-routing-key': 'failed',
      'x-message-ttl': 60_000, // 60 секунд
    },
  });

  // Consumer с возможностью reject
  channel.consume(
    'main_queue',
    async (msg) => {
      if (!msg) {
        return;
      }

      try {
        await processMessage(msg.content.toString());
        channel.ack(msg);
      } catch (error) {
        console.error('Failed to process message:', error);
        // Reject без requeue → попадёт в DLX
        channel.nack(msg, false, false);
      }
    },
    { noAck: false },
  );
}

setupDeadLetterExchange().catch(console.error);
```

### RabbitMQ Best Practices

1. **Используйте persistent messages и durable queues** для критичных данных
2. **Manual acknowledgments** вместо auto-ack для надёжности
3. **Prefetch count** для контроля нагрузки на консьюмеров
4. **DLX** для обработки проблемных сообщений
5. **Publisher confirms** для гарантии доставки
6. **Heartbeats** для обнаружения потери соединения
7. **Connection pooling** для производительности

```javascript
async function publishWithConfirm() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createConfirmChannel();

  await channel.assertQueue('queue_name', { durable: true });

  channel.on('return', (msg) => {
    console.error('Message could not be routed:', msg.content.toString());
  });

  try {
    channel.publish('', 'queue_name', Buffer.from('message'), {
      mandatory: true, // Вернёт сообщение в случае отсутствия очереди
      persistent: true,
    });

    await channel.waitForConfirms();
    console.log('Message confirmed');
  } finally {
    await channel.close();
    await connection.close();
  }
}

publishWithConfirm().catch(console.error);
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

### Пример: Standard Queue

```javascript
const {
  SQSClient,
  CreateQueueCommand,
  SendMessageCommand,
  ReceiveMessageCommand,
  DeleteMessageCommand,
  ChangeMessageVisibilityCommand,
  GetQueueAttributesCommand,
  SendMessageBatchCommand,
  DeleteMessageBatchCommand,
} = require('@aws-sdk/client-sqs');

const sqs = new SQSClient({ region: 'us-east-1' });

async function standardQueueExample() {
  // Создание очереди
  const queueResponse = await sqs.send(
    new CreateQueueCommand({
      QueueName: 'my-task-queue',
      Attributes: {
        DelaySeconds: '0',
        MessageRetentionPeriod: '86400', // 1 день
        VisibilityTimeout: '30', // 30 секунд
      },
    }),
  );

  const queueUrl = queueResponse.QueueUrl;

  // Отправка сообщения
  const sendResponse = await sqs.send(
    new SendMessageCommand({
      QueueUrl: queueUrl,
      MessageBody: JSON.stringify({
        task: 'send_email',
        to: 'user@example.com',
        subject: 'Welcome!',
      }),
      MessageAttributes: {
        Priority: {
          StringValue: 'high',
          DataType: 'String',
        },
      },
    }),
  );

  console.log(`MessageId: ${sendResponse.MessageId}`);

  // Получение и обработка сообщений
  const messagesResponse = await sqs.send(
    new ReceiveMessageCommand({
      QueueUrl: queueUrl,
      MaxNumberOfMessages: 10, // До 10 сообщений за раз
      WaitTimeSeconds: 20, // Long polling
      MessageAttributeNames: ['All'],
    }),
  );

  for (const message of messagesResponse.Messages || []) {
    const body = JSON.parse(message.Body);
    console.log('Processing:', body);

    // Обработка...

    // Удаление после успешной обработки
    await sqs.send(
      new DeleteMessageCommand({
        QueueUrl: queueUrl,
        ReceiptHandle: message.ReceiptHandle,
      }),
    );
  }
}

standardQueueExample().catch(console.error);
```

### FIFO Queue

```javascript
async function fifoQueueExample() {
  // Создание FIFO очереди
  const fifoQueue = await sqs.send(
    new CreateQueueCommand({
      QueueName: 'orders.fifo', // Обязательно .fifo
      Attributes: {
        FifoQueue: 'true',
        ContentBasedDeduplication: 'true',
        MessageRetentionPeriod: '345600', // 4 дня
      },
    }),
  );

  const fifoUrl = fifoQueue.QueueUrl;

  // Отправка с MessageGroupId (для ordering)
  await sqs.send(
    new SendMessageCommand({
      QueueUrl: fifoUrl,
      MessageBody: 'Order #1234',
      MessageGroupId: 'user_123', // Сообщения в одной группе — строго по порядку
      MessageDeduplicationId: 'order-1234-v1', // Для exactly-once
    }),
  );

  await sqs.send(
    new SendMessageCommand({
      QueueUrl: fifoUrl,
      MessageBody: 'Order #1235',
      MessageGroupId: 'user_123',
      MessageDeduplicationId: 'order-1235-v1',
    }),
  );

  // Второй пользователь — параллельная обработка
  await sqs.send(
    new SendMessageCommand({
      QueueUrl: fifoUrl,
      MessageBody: 'Order #5678',
      MessageGroupId: 'user_456',
      MessageDeduplicationId: 'order-5678-v1',
    }),
  );
}

fifoQueueExample().catch(console.error);
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

```javascript
// Продление Visibility Timeout
await sqs.send(
  new ChangeMessageVisibilityCommand({
    QueueUrl: queueUrl,
    ReceiptHandle: receiptHandle,
    VisibilityTimeout: 60, // Продлить ещё на 60 секунд
  }),
);
```

### Dead Letter Queue (DLQ)

```javascript
async function setupDlq() {
  // Создание DLQ
  const dlq = await sqs.send(new CreateQueueCommand({ QueueName: 'failed-tasks-dlq' }));
  const dlqAttributes = await sqs.send(
    new GetQueueAttributesCommand({
      QueueUrl: dlq.QueueUrl,
      AttributeNames: ['QueueArn'],
    }),
  );
  const dlqArn = dlqAttributes.Attributes.QueueArn;

  // Основная очередь с DLQ
  await sqs.send(
    new CreateQueueCommand({
      QueueName: 'main-tasks',
      Attributes: {
        RedrivePolicy: JSON.stringify({
          deadLetterTargetArn: dlqArn,
          maxReceiveCount: '3', // После 3 попыток → в DLQ
        }),
      },
    }),
  );
}

setupDlq().catch(console.error);
```

### Batch Operations

Для оптимизации стоимости и производительности:

```javascript
async function batchOperations(queueUrl, receiptHandles) {
  // Batch send (до 10 сообщений)
  const batchResponse = await sqs.send(
    new SendMessageBatchCommand({
      QueueUrl: queueUrl,
      Entries: [
        {
          Id: '1',
          MessageBody: 'Task 1',
        },
        {
          Id: '2',
          MessageBody: 'Task 2',
          DelaySeconds: 10,
        },
        {
          Id: '3',
          MessageBody: 'Task 3',
        },
      ],
    }),
  );

  console.log('Batch send status:', batchResponse);

  // Batch delete
  await sqs.send(
    new DeleteMessageBatchCommand({
      QueueUrl: queueUrl,
      Entries: receiptHandles.map((handle, index) => ({
        Id: `msg-${index}`,
        ReceiptHandle: handle,
      })),
    }),
  );
}
```

### Long Polling vs Short Polling

```javascript
// Short Polling (default) - возвращает сразу
const shortPolling = await sqs.send(
  new ReceiveMessageCommand({
    QueueUrl: queueUrl,
    WaitTimeSeconds: 0, // Не ждать
  }),
);

// Long Polling - ждёт до 20 секунд
const longPolling = await sqs.send(
  new ReceiveMessageCommand({
    QueueUrl: queueUrl,
    WaitTimeSeconds: 20, // Экономия запросов + снижение latency
    MaxNumberOfMessages: 10,
  }),
);
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

```javascript
async function publishOrderEvent() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  // RabbitMQ Topic Exchange
  await channel.assertExchange('orders', 'topic', { durable: true });

  // Producer: новый заказ
  const order = {
    order_id: '12345',
    user_id: '789',
    total: 99.99,
    items: ['item-1', 'item-2'],
  };

  // Routing key: order.{status}.{priority}
  channel.publish('orders', 'order.new.high', Buffer.from(JSON.stringify(order)), {
    persistent: true,
  });

  // Consumer 1: Инвентарь (все заказы)
  await channel.assertQueue('inventory-service', { durable: true });
  await channel.bindQueue('inventory-service', 'orders', 'order.#');

  // Consumer 2: Уведомления (только срочные)
  await channel.assertQueue('notifications-service', { durable: true });
  await channel.bindQueue('notifications-service', 'orders', 'order.*.high');

  // Consumer 3: Аналитика (все новые)
  await channel.assertQueue('analytics-service', { durable: true });
  await channel.bindQueue('analytics-service', 'orders', 'order.new.*');

  await channel.close();
  await connection.close();
}

publishOrderEvent().catch(console.error);
```

### 2. Email рассылка (SQS)

```javascript
const queueUrl = 'https://sqs.us-east-1.amazonaws.com/123/email-queue';

// Producer: массовая рассылка
async function enqueueEmailBatches() {
  const users = await getUsersFromDb(); // 100,000 пользователей

  const batches = [];
  for (let i = 0; i < users.length; i += 10) {
    batches.push(users.slice(i, i + 10));
  }

  await Promise.all(
    batches.map((batch, batchIndex) =>
      sqs.send(
        new SendMessageBatchCommand({
          QueueUrl: queueUrl,
          Entries: batch.map((user, index) => ({
            Id: `${batchIndex}-${index}`,
            MessageBody: JSON.stringify({
              to: user.email,
              template: 'newsletter',
              data: { name: user.name },
            }),
          })),
        }),
      ),
    ),
  );
}

// Consumer: отправка email
async function processEmails() {
  while (true) {
    const messages = await sqs.send(
      new ReceiveMessageCommand({
        QueueUrl: queueUrl,
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20,
      }),
    );

    for (const msg of messages.Messages || []) {
      const data = JSON.parse(msg.Body);
      await sendEmail(data.to, data.template, data.data);

      await sqs.send(
        new DeleteMessageCommand({
          QueueUrl: queueUrl,
          ReceiptHandle: msg.ReceiptHandle,
        }),
      );
    }
  }
}
```

### 3. Image Processing Pipeline

```javascript
async function setupImagePipeline() {
  const connection = await amqp.connect('amqp://localhost');

  // RabbitMQ: цепочка обработки
  const uploadChannel = await connection.createChannel();
  const resizeChannel = await connection.createChannel();
  const watermarkChannel = await connection.createChannel();

  await uploadChannel.assertQueue('images.upload', { durable: true });
  await resizeChannel.assertQueue('images.resize', { durable: true });
  await watermarkChannel.assertQueue('images.watermark', { durable: true });
  await watermarkChannel.assertQueue('images.publish', { durable: true });

  // Service 1: Upload handler
  uploadChannel.consume('images.upload', async (msg) => {
    if (!msg) {
      return;
    }

    const image = JSON.parse(msg.content.toString());

    // Сохранить оригинал в S3
    await saveToS3(image.url, 'originals/');

    // Отправить на resize
    await uploadChannel.sendToQueue(
      'images.resize',
      Buffer.from(
        JSON.stringify({
          imageId: image.id,
          sizes: [800, 400, 200],
        }),
      ),
      { persistent: true },
    );

    uploadChannel.ack(msg);
  });

  // Service 2: Resizer
  resizeChannel.consume('images.resize', async (msg) => {
    if (!msg) {
      return;
    }

    const data = JSON.parse(msg.content.toString());

    for (const size of data.sizes) {
      const resized = await resizeImage(data.imageId, size);
      await saveToS3(resized, `resized/${size}/`);
    }

    // Отправить на watermarking
    await resizeChannel.sendToQueue('images.watermark', msg.content, { persistent: true });
    resizeChannel.ack(msg);
  });

  // Service 3: Watermark
  watermarkChannel.consume('images.watermark', async (msg) => {
    if (!msg) {
      return;
    }

    const data = JSON.parse(msg.content.toString());
    await addWatermark(data.imageId);

    // Финальная публикация
    await watermarkChannel.sendToQueue('images.publish', msg.content, { persistent: true });
    watermarkChannel.ack(msg);
  });
}

setupImagePipeline().catch(console.error);
```

### 4. Priority Queue (RabbitMQ)

```javascript
async function priorityQueueDemo() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  // Очередь с приоритетами
  await channel.assertQueue('tasks', {
    durable: true,
    arguments: { 'x-max-priority': 10 },
  });

  // Отправка с приоритетом
  channel.sendToQueue('tasks', Buffer.from('Critical task'), {
    priority: 10,
    persistent: true,
  });

  channel.sendToQueue('tasks', Buffer.from('Normal task'), {
    priority: 5,
    persistent: true,
  });

  channel.sendToQueue('tasks', Buffer.from('Low priority task'), {
    priority: 1,
    persistent: true,
  });

  await channel.close();
  await connection.close();

  // Consumer обработает в порядке: 10 → 5 → 1
}

priorityQueueDemo().catch(console.error);
```

### 5. Delay Queue (SQS)

```javascript
// Отложенная отправка (до 15 минут в SQS)
await sqs.send(
  new SendMessageCommand({
    QueueUrl: queueUrl,
    MessageBody: 'Reminder: meeting in 15 min',
    DelaySeconds: 900, // 15 минут
  }),
);

// Для больших задержек: использовать visibility timeout или SNS + Lambda
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

```javascript
const {
  CloudWatchClient,
  GetMetricStatisticsCommand,
  PutMetricAlarmCommand,
} = require('@aws-sdk/client-cloudwatch');

const cloudwatch = new CloudWatchClient({ region: 'us-east-1' });

async function configureCloudWatch(queueName) {
  // Получить метрики
  const metrics = await cloudwatch.send(
    new GetMetricStatisticsCommand({
      Namespace: 'AWS/SQS',
      MetricName: 'ApproximateNumberOfMessagesVisible',
      Dimensions: [{ Name: 'QueueName', Value: queueName }],
      StartTime: new Date(Date.now() - 60 * 60 * 1000), // 1 час назад
      EndTime: new Date(),
      Period: 300,
      Statistics: ['Average', 'Maximum'],
    }),
  );

  console.log(metrics.Datapoints);

  // Alarm на старые сообщения
  await cloudwatch.send(
    new PutMetricAlarmCommand({
      AlarmName: 'SQS-OldMessages',
      MetricName: 'ApproximateAgeOfOldestMessage',
      Namespace: 'AWS/SQS',
      Statistic: 'Maximum',
      Period: 300,
      EvaluationPeriods: 1,
      Threshold: 3600, // 1 час
      ComparisonOperator: 'GreaterThanThreshold',
      Dimensions: [{ Name: 'QueueName', Value: queueName }],
    }),
  );
}

configureCloudWatch('my-queue').catch(console.error);
```

## Паттерны отказоустойчивости

### Idempotent Consumer

```javascript
const Redis = require('ioredis');

const redisClient = new Redis();

async function processMessageIdempotently(messageId, messageBody) {
  // Проверка, не обработано ли уже
  const alreadyProcessed = await redisClient.exists(`processed:${messageId}`);
  if (alreadyProcessed) {
    console.log(`Message ${messageId} already processed`);
    return;
  }

  // Обработка
  const result = await process(messageBody);

  // Пометка как обработанного (с TTL)
  await redisClient.set(`processed:${messageId}`, 'true', 'EX', 86400); // 24 часа

  return result;
}
```

### Circuit Breaker для Consumer

```javascript
class CircuitBreaker {
  constructor({ failureThreshold = 5, timeoutSeconds = 60 } = {}) {
    this.failureThreshold = failureThreshold;
    this.timeoutSeconds = timeoutSeconds;
    this.failures = 0;
    this.lastFailureTime = null;
    this.state = 'closed'; // closed, open, half-open
  }

  async call(fn) {
    if (this.state === 'open') {
      const diff = (Date.now() - this.lastFailureTime) / 1000;
      if (diff > this.timeoutSeconds) {
        this.state = 'half-open';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await fn();
      if (this.state === 'half-open') {
        this.state = 'closed';
        this.failures = 0;
      }
      return result;
    } catch (error) {
      this.failures += 1;
      this.lastFailureTime = Date.now();

      if (this.failures >= this.failureThreshold) {
        this.state = 'open';
      }

      throw error;
    }
  }
}

// Использование
const cb = new CircuitBreaker();

async function consumeMessages() {
  while (true) {
    const messages = await sqs.send(new ReceiveMessageCommand({ QueueUrl: queueUrl, WaitTimeSeconds: 20 }));

    for (const msg of messages.Messages || []) {
      try {
        await cb.call(() => processMessage(msg));
        await sqs.send(
          new DeleteMessageCommand({
            QueueUrl: queueUrl,
            ReceiptHandle: msg.ReceiptHandle,
          }),
        );
      } catch (error) {
        console.error(`Circuit breaker prevented call: ${error.message}`);
        await new Promise((resolve) => setTimeout(resolve, 60_000));
      }
    }
  }
}
```

### Retry with Exponential Backoff

```javascript
function retryWithBackoff(fn, { maxRetries = 3, baseDelayMs = 1000, maxDelayMs = 60_000 } = {}) {
  return async (...args) => {
    for (let attempt = 0; attempt < maxRetries; attempt += 1) {
      try {
        return await fn(...args);
      } catch (error) {
        if (attempt === maxRetries - 1) {
          throw error;
        }

        const delay = Math.min(baseDelayMs * 2 ** attempt, maxDelayMs);
        console.warn(`Attempt ${attempt + 1} failed: ${error.message}. Retrying in ${delay}ms...`);
        await new Promise((resolve) => setTimeout(resolve, delay));
      }
    }
  };
}

const sendToQueue = retryWithBackoff(async (message) => {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  try {
    await channel.assertQueue('task_queue', { durable: true });
    channel.sendToQueue('task_queue', Buffer.from(message), { persistent: true });
  } finally {
    await channel.close();
    await connection.close();
  }
}, { maxRetries: 5 });
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

```javascript
// SNS (pub-sub) → SQS (queue) → Lambda (processing)
const {
  SNSClient,
  CreateTopicCommand,
  PublishCommand,
  SubscribeCommand,
} = require('@aws-sdk/client-sns');

const snsClient = new SNSClient({ region: 'us-east-1' });

async function fanoutOrders() {
  // Создать топик
  const topic = await snsClient.send(new CreateTopicCommand({ Name: 'orders' }));

  // Создать очереди для разных сервисов
  const inventoryQueue = await sqs.send(new CreateQueueCommand({ QueueName: 'inventory-service' }));
  const emailQueue = await sqs.send(new CreateQueueCommand({ QueueName: 'email-service' }));
  const analyticsQueue = await sqs.send(new CreateQueueCommand({ QueueName: 'analytics-service' }));

  const queues = [inventoryQueue, emailQueue, analyticsQueue];

  // Подписать очереди на топик
  for (const queue of queues) {
    const attrs = await sqs.send(
      new GetQueueAttributesCommand({
        QueueUrl: queue.QueueUrl,
        AttributeNames: ['QueueArn'],
      }),
    );

    await snsClient.send(
      new SubscribeCommand({
        TopicArn: topic.TopicArn,
        Protocol: 'sqs',
        Endpoint: attrs.Attributes.QueueArn,
      }),
    );
  }

  // Публикация в топик → все очереди получат сообщение
  await snsClient.send(
    new PublishCommand({
      TopicArn: topic.TopicArn,
      Message: JSON.stringify({ order_id: '12345', total: 99.99 }),
    }),
  );
}

fanoutOrders().catch(console.error);
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
**Предыдущий урок**: [Distributed Caching: Redis и Memcached](20-distributed-caching.md)
**Следующий урок**: [Урок 22: Kafka и Event Streaming](22-kafka-event-streaming.md)
