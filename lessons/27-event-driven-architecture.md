# Урок 27: Event-Driven Architecture


## Введение

Event-Driven Architecture (EDA) — это архитектурный паттерн, где компоненты системы общаются через события. Вместо прямых вызовов сервисов, изменения состояния публикуются как события, на которые подписываются заинтересованные компоненты.

**Request-Driven vs Event-Driven:**

```
REQUEST-DRIVEN (Traditional):
Order Service → (HTTP POST) → Payment Service
             → (HTTP POST) → Inventory Service
             → (HTTP POST) → Email Service

EVENT-DRIVEN:
Order Service → [OrderCreated Event] → Event Bus
                                         ↓
                    ┌────────────────────┼────────────────┐
                    ↓                    ↓                ↓
              Payment Service    Inventory Service   Email Service
```

**Ключевые отличия:**

| Request-Driven | Event-Driven |
|----------------|--------------|
| Synchronous | Asynchronous |
| Tight coupling | Loose coupling |
| Caller знает о получателе | Publisher не знает о subscribers |
| Immediate consistency | Eventual consistency |
| Point-to-point | Pub-Sub |

## Основные концепции

### Event

Событие — это факт о том, что что-то произошло в системе.

```javascript
// Event structure
{
  "eventId": "evt_123456",
  "eventType": "OrderCreated",
  "timestamp": "2024-01-15T10:30:00Z",
  "aggregateId": "order_789",  // О чём событие
  "version": 1,
  "payload": {
    "orderId": "order_789",
    "userId": "user_456",
    "items": [
      { "productId": "prod_1", "quantity": 2, "price": 29.99 },
      { "productId": "prod_2", "quantity": 1, "price": 49.99 }
    ],
    "total": 109.97,
    "status": "pending"
  },
  "metadata": {
    "userId": "user_456",
    "correlationId": "req_abc123",
    "causationId": "cmd_xyz789"
  }
}
```

**Типы событий:**

1. **Domain Events**: бизнес-события (OrderCreated, PaymentCompleted)
2. **Integration Events**: для коммуникации между bounded contexts
3. **System Events**: технические события (ServerStarted, CacheFlushed)

### Event Bus / Event Broker

Центральный компонент для публикации и подписки на события.

**Популярные решения:**
- Apache Kafka
- RabbitMQ
- AWS EventBridge
- Google Pub/Sub
- Azure Event Grid
- NATS

### Producer, Consumer, Subscriber

```javascript
// Producer (Publisher)
async function createOrder(orderData) {
  const order = await db.orders.create(orderData);

  await eventBus.publish('OrderCreated', {
    orderId: order.id,
    userId: order.userId,
    total: order.total,
    items: order.items,
  });

  return order;
}

// Consumer (Subscriber)
eventBus.subscribe('OrderCreated', async (event) => {
  console.log('Processing OrderCreated event:', event.orderId);
  await sendConfirmationEmail(event.userId, event.orderId);
});
```

## Паттерны Event-Driven Architecture

### 1. Event Notification

Простейший паттерн: уведомление о том, что что-то произошло.

```javascript
// Order Service публикует событие
await eventBus.publish('OrderCreated', {
  orderId: 'order_123',
  timestamp: Date.now(),
});

// Email Service подписывается
eventBus.subscribe('OrderCreated', async (event) => {
  // Получить детали заказа (query)
  const order = await orderService.getOrder(event.orderId);
  await sendEmail(order.userEmail, 'Order Confirmation', order);
});
```

**Плюсы:**
- Минимальная информация в событии
- Низкая coupling

**Минусы:**
- Дополнительный запрос для получения деталей
- Зависимость от доступности source service

### 2. Event-Carried State Transfer

Событие содержит все необходимые данные.

```javascript
// Order Service
await eventBus.publish('OrderCreated', {
  orderId: 'order_123',
  userId: 'user_456',
  userEmail: 'user@example.com',
  userName: 'John Doe',
  items: [...],
  total: 109.97,
  shippingAddress: {...},
  timestamp: Date.now(),
});

// Email Service
eventBus.subscribe('OrderCreated', async (event) => {
  // Все данные уже в событии
  await sendEmail(event.userEmail, 'Order Confirmation', {
    orderId: event.orderId,
    userName: event.userName,
    items: event.items,
    total: event.total,
  });
});
```

**Плюсы:**
- Нет дополнительных запросов
- Consumer автономен

**Минусы:**
- Большой размер события
- Дублирование данных

### 3. Event Sourcing

Все изменения состояния сохраняются как последовательность событий. Текущее состояние = replay всех событий.

```javascript
// Event Store
const events = [
  { type: 'OrderCreated', orderId: '123', total: 100, status: 'pending' },
  { type: 'PaymentReceived', orderId: '123', amount: 100 },
  { type: 'OrderShipped', orderId: '123', trackingNumber: 'TRACK123' },
  { type: 'OrderDelivered', orderId: '123', deliveredAt: '2024-01-20' },
];

// Восстановление состояния
function rebuildOrder(orderId) {
  let order = null;

  const orderEvents = events.filter(e => e.orderId === orderId);

  for (const event of orderEvents) {
    switch (event.type) {
      case 'OrderCreated':
        order = {
          id: event.orderId,
          total: event.total,
          status: 'pending',
        };
        break;

      case 'PaymentReceived':
        order.status = 'paid';
        order.paidAmount = event.amount;
        break;

      case 'OrderShipped':
        order.status = 'shipped';
        order.trackingNumber = event.trackingNumber;
        break;

      case 'OrderDelivered':
        order.status = 'delivered';
        order.deliveredAt = event.deliveredAt;
        break;
    }
  }

  return order;
}

const currentOrder = rebuildOrder('123');
// { id: '123', total: 100, status: 'delivered', ... }
```

**Преимущества Event Sourcing:**
- Полная audit trail (история всех изменений)
- Temporal queries (состояние на любой момент времени)
- Event replay для debugging
- Возможность добавить новые read models

**Недостатки:**
- Сложность
- Snapshots для производительности
- Schema evolution

### 4. CQRS + Event Sourcing

Разделение Write Model (commands → events) и Read Model (projections).

```javascript
// WRITE SIDE: Command Handler
class OrderCommandHandler {
  async createOrder(command) {
    const events = [
      {
        type: 'OrderCreated',
        orderId: generateId(),
        userId: command.userId,
        items: command.items,
        total: calculateTotal(command.items),
      },
    ];

    // Сохранить события в Event Store
    await eventStore.append('order', events);

    return events[0].orderId;
  }

  async shipOrder(command) {
    // Проверить бизнес-правила
    const order = await this.getOrder(command.orderId);

    if (order.status !== 'paid') {
      throw new Error('Order must be paid before shipping');
    }

    const event = {
      type: 'OrderShipped',
      orderId: command.orderId,
      trackingNumber: command.trackingNumber,
    };

    await eventStore.append('order', [event]);
  }

  async getOrder(orderId) {
    const events = await eventStore.getEvents('order', orderId);
    return rebuildOrder(events);
  }
}

// READ SIDE: Projections
class OrderProjection {
  constructor() {
    // Подписка на события для обновления read model
    eventStore.subscribe('OrderCreated', this.onOrderCreated.bind(this));
    eventStore.subscribe('OrderShipped', this.onOrderShipped.bind(this));
    eventStore.subscribe('OrderDelivered', this.onOrderDelivered.bind(this));
  }

  async onOrderCreated(event) {
    // Сохранить в read-optimized DB (PostgreSQL, MongoDB, Elasticsearch)
    await readDb.orders.insert({
      orderId: event.orderId,
      userId: event.userId,
      total: event.total,
      status: 'pending',
      createdAt: event.timestamp,
    });

    // Денормализованная таблица для быстрых запросов
    await readDb.userOrders.insert({
      userId: event.userId,
      orderId: event.orderId,
      total: event.total,
      createdAt: event.timestamp,
    });
  }

  async onOrderShipped(event) {
    await readDb.orders.update(
      { orderId: event.orderId },
      { status: 'shipped', trackingNumber: event.trackingNumber }
    );
  }

  async onOrderDelivered(event) {
    await readDb.orders.update(
      { orderId: event.orderId },
      { status: 'delivered', deliveredAt: event.deliveredAt }
    );
  }
}

// QUERY SIDE
class OrderQueryService {
  async getOrder(orderId) {
    return await readDb.orders.findOne({ orderId });
  }

  async getUserOrders(userId) {
    return await readDb.userOrders.find({ userId }).sort({ createdAt: -1 });
  }

  async searchOrders(criteria) {
    return await readDb.orders.find(criteria);
  }
}
```

## Реализация с Kafka

### Producer

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
});

const producer = kafka.producer({
  idempotent: true,  // Exactly-once semantics
});

await producer.connect();

async function publishEvent(eventType, payload) {
  await producer.send({
    topic: 'domain-events',
    messages: [{
      key: payload.orderId,  // Партиционирование по orderId
      value: JSON.stringify({
        eventId: generateId(),
        eventType,
        timestamp: Date.now(),
        payload,
      }),
      headers: {
        'event-type': eventType,
        'correlation-id': getCorrelationId(),
      },
    }],
  });
}

// Использование
await publishEvent('OrderCreated', {
  orderId: 'order_123',
  userId: 'user_456',
  total: 109.97,
});
```

### Consumer

```javascript
const consumer = kafka.consumer({
  groupId: 'email-service',
  sessionTimeout: 30000,
  heartbeatInterval: 3000,
});

await consumer.connect();
await consumer.subscribe({ topic: 'domain-events', fromBeginning: false });

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const event = JSON.parse(message.value.toString());

    console.log(`Processing event: ${event.eventType}`);

    try {
      // Idempotency check
      if (await isEventProcessed(event.eventId)) {
        console.log('Event already processed, skipping');
        return;
      }

      // Обработка
      await handleEvent(event);

      // Пометка как обработанного
      await markEventAsProcessed(event.eventId);

    } catch (error) {
      console.error('Error processing event:', error);
      // DLQ или retry
      await sendToDLQ(event, error);
    }
  },
});

async function handleEvent(event) {
  switch (event.eventType) {
    case 'OrderCreated':
      await sendConfirmationEmail(event.payload);
      break;

    case 'OrderShipped':
      await sendShippingNotification(event.payload);
      break;

    case 'OrderDelivered':
      await sendDeliveryConfirmation(event.payload);
      break;

    default:
      console.log(`Unknown event type: ${event.eventType}`);
  }
}
```

## Реализация с AWS EventBridge

```javascript
const { EventBridgeClient, PutEventsCommand } = require('@aws-sdk/client-eventbridge');

const client = new EventBridgeClient({ region: 'us-east-1' });

async function publishEvent(eventType, payload) {
  const command = new PutEventsCommand({
    Entries: [{
      Source: 'order-service',
      DetailType: eventType,
      Detail: JSON.stringify(payload),
      EventBusName: 'default',
    }],
  });

  await client.send(command);
}

// Lambda function как subscriber
exports.handler = async (event) => {
  console.log('Received event:', event);

  const payload = JSON.parse(event.detail);

  switch (event['detail-type']) {
    case 'OrderCreated':
      await sendConfirmationEmail(payload);
      break;

    case 'OrderShipped':
      await sendShippingNotification(payload);
      break;
  }
};

// EventBridge Rule (Infrastructure as Code)
// {
//   "source": ["order-service"],
//   "detail-type": ["OrderCreated", "OrderShipped"]
// }
```

## Saga Pattern в Event-Driven Architecture

Распределённая транзакция через последовательность событий.

### Choreography-based Saga

```javascript
// Order Service
async function createOrder(orderData) {
  const order = await db.orders.create({ ...orderData, status: 'pending' });

  await eventBus.publish('OrderCreated', {
    orderId: order.id,
    userId: order.userId,
    total: order.total,
    items: order.items,
  });

  return order;
}

eventBus.subscribe('PaymentCompleted', async (event) => {
  await db.orders.update(event.orderId, { status: 'paid' });

  await eventBus.publish('OrderPaid', {
    orderId: event.orderId,
    items: event.items,
  });
});

eventBus.subscribe('PaymentFailed', async (event) => {
  await db.orders.update(event.orderId, { status: 'cancelled' });

  await eventBus.publish('OrderCancelled', {
    orderId: event.orderId,
    reason: 'payment_failed',
  });
});

// Payment Service
eventBus.subscribe('OrderCreated', async (event) => {
  try {
    const payment = await processPayment(event.userId, event.total);

    await eventBus.publish('PaymentCompleted', {
      orderId: event.orderId,
      paymentId: payment.id,
      items: event.items,
    });

  } catch (error) {
    await eventBus.publish('PaymentFailed', {
      orderId: event.orderId,
      error: error.message,
    });
  }
});

// Inventory Service
eventBus.subscribe('OrderPaid', async (event) => {
  try {
    await reserveInventory(event.items);

    await eventBus.publish('InventoryReserved', {
      orderId: event.orderId,
    });

  } catch (error) {
    // Compensation: refund
    await eventBus.publish('InventoryReservationFailed', {
      orderId: event.orderId,
    });
  }
});

eventBus.subscribe('InventoryReservationFailed', async (event) => {
  // Trigger refund
  await eventBus.publish('RefundRequested', {
    orderId: event.orderId,
  });
});

// Payment Service: compensation
eventBus.subscribe('RefundRequested', async (event) => {
  await refundPayment(event.orderId);

  await eventBus.publish('PaymentRefunded', {
    orderId: event.orderId,
  });
});
```

### Orchestration-based Saga

```javascript
class OrderSagaOrchestrator {
  constructor(eventBus) {
    this.eventBus = eventBus;
    this.sagas = new Map();

    // Подписка на события
    eventBus.subscribe('PaymentCompleted', this.onPaymentCompleted.bind(this));
    eventBus.subscribe('PaymentFailed', this.onPaymentFailed.bind(this));
    eventBus.subscribe('InventoryReserved', this.onInventoryReserved.bind(this));
    eventBus.subscribe('InventoryReservationFailed', this.onInventoryFailed.bind(this));
  }

  async startSaga(orderData) {
    const sagaId = generateId();

    const saga = {
      sagaId,
      orderId: null,
      status: 'started',
      steps: [],
    };

    this.sagas.set(sagaId, saga);

    try {
      // Step 1: Create Order
      const order = await this.createOrder(sagaId, orderData);
      saga.orderId = order.id;
      saga.steps.push({ step: 'order_created', status: 'completed' });

      // Step 2: Process Payment
      await this.eventBus.publish('ProcessPayment', {
        sagaId,
        orderId: order.id,
        userId: orderData.userId,
        amount: order.total,
      });

      saga.steps.push({ step: 'payment_requested', status: 'pending' });

    } catch (error) {
      await this.compensate(sagaId);
    }
  }

  async onPaymentCompleted(event) {
    const saga = this.sagas.get(event.sagaId);

    if (!saga) return;

    saga.steps.find(s => s.step === 'payment_requested').status = 'completed';

    // Step 3: Reserve Inventory
    await this.eventBus.publish('ReserveInventory', {
      sagaId: event.sagaId,
      orderId: saga.orderId,
      items: event.items,
    });

    saga.steps.push({ step: 'inventory_requested', status: 'pending' });
  }

  async onPaymentFailed(event) {
    await this.compensate(event.sagaId);
  }

  async onInventoryReserved(event) {
    const saga = this.sagas.get(event.sagaId);

    if (!saga) return;

    saga.steps.find(s => s.step === 'inventory_requested').status = 'completed';

    // Success: mark order as confirmed
    await db.orders.update(saga.orderId, { status: 'confirmed' });

    saga.status = 'completed';
    this.sagas.delete(event.sagaId);
  }

  async onInventoryFailed(event) {
    await this.compensate(event.sagaId);
  }

  async compensate(sagaId) {
    const saga = this.sagas.get(sagaId);

    if (!saga) return;

    console.log(`Compensating saga ${sagaId}`);

    // Rollback в обратном порядке
    for (let i = saga.steps.length - 1; i >= 0; i--) {
      const step = saga.steps[i];

      if (step.status === 'completed') {
        switch (step.step) {
          case 'payment_requested':
            await this.eventBus.publish('RefundPayment', {
              orderId: saga.orderId,
            });
            break;

          case 'inventory_requested':
            await this.eventBus.publish('ReleaseInventory', {
              orderId: saga.orderId,
            });
            break;

          case 'order_created':
            await db.orders.update(saga.orderId, { status: 'cancelled' });
            break;
        }
      }
    }

    saga.status = 'compensated';
    this.sagas.delete(sagaId);
  }
}
```

## Обработка ошибок

### 1. Idempotent Event Handlers

```javascript
const processedEvents = new Set();

async function handleEvent(event) {
  // Проверка дубликатов
  if (processedEvents.has(event.eventId)) {
    console.log('Event already processed');
    return;
  }

  try {
    await processEvent(event);
    processedEvents.add(event.eventId);

  } catch (error) {
    console.error('Error processing event:', error);
    throw error;
  }
}

// Или в Redis для distributed system
async function handleEventIdempotent(event) {
  const key = `processed:${event.eventId}`;
  const alreadyProcessed = await redis.get(key);

  if (alreadyProcessed) {
    return;
  }

  await processEvent(event);

  // TTL 24 часа
  await redis.setex(key, 86400, 'true');
}
```

### 2. Dead Letter Queue

```javascript
const MAX_RETRIES = 3;

async function handleEventWithRetry(event) {
  const retries = event.retries || 0;

  try {
    await processEvent(event);

  } catch (error) {
    if (retries < MAX_RETRIES) {
      // Retry с exponential backoff
      const delay = Math.pow(2, retries) * 1000;

      await eventBus.publish('RetryEvent', {
        ...event,
        retries: retries + 1,
      }, { delay });

    } else {
      // Отправить в DLQ
      await eventBus.publish('DeadLetterQueue', {
        originalEvent: event,
        error: error.message,
        timestamp: Date.now(),
      });

      // Алерт для ops team
      await sendAlert('Event processing failed after max retries', event);
    }
  }
}
```

### 3. Event Versioning

```javascript
// Event Schema Evolution
{
  "eventType": "OrderCreated",
  "version": 2,  // Версия схемы
  "payload": {
    "orderId": "order_123",
    "userId": "user_456",
    "items": [...],
    "total": 109.97,
    // Новое поле в v2
    "currency": "USD"
  }
}

// Handler с поддержкой нескольких версий
async function handleOrderCreated(event) {
  switch (event.version) {
    case 1:
      return await handleOrderCreatedV1(event);

    case 2:
      return await handleOrderCreatedV2(event);

    default:
      throw new Error(`Unsupported event version: ${event.version}`);
  }
}

// Или upcasting: конвертация старых событий в новый формат
function upcastEvent(event) {
  if (event.version === 1) {
    return {
      ...event,
      version: 2,
      payload: {
        ...event.payload,
        currency: 'USD',  // Default value
      },
    };
  }

  return event;
}
```

## Мониторинг и Observability

### Event Tracing

```javascript
const opentelemetry = require('@opentelemetry/api');

async function publishEventWithTracing(eventType, payload) {
  const tracer = opentelemetry.trace.getTracer('event-bus');
  const span = tracer.startSpan('publish_event', {
    attributes: {
      'event.type': eventType,
      'event.id': payload.eventId,
    },
  });

  const context = opentelemetry.trace.setSpan(
    opentelemetry.context.active(),
    span
  );

  try {
    await eventBus.publish(eventType, payload, {
      headers: {
        'traceparent': span.spanContext().traceId,
      },
    });

    span.setStatus({ code: opentelemetry.SpanStatusCode.OK });

  } catch (error) {
    span.setStatus({
      code: opentelemetry.SpanStatusCode.ERROR,
      message: error.message,
    });
    throw error;

  } finally {
    span.end();
  }
}
```

### Metrics

```javascript
const promClient = require('prom-client');

const eventsPublished = new promClient.Counter({
  name: 'events_published_total',
  help: 'Total events published',
  labelNames: ['event_type'],
});

const eventsProcessed = new promClient.Counter({
  name: 'events_processed_total',
  help: 'Total events processed',
  labelNames: ['event_type', 'status'],
});

const eventProcessingDuration = new promClient.Histogram({
  name: 'event_processing_duration_seconds',
  help: 'Event processing duration',
  labelNames: ['event_type'],
});

// Использование
async function publishEvent(eventType, payload) {
  eventsPublished.inc({ event_type: eventType });
  await eventBus.publish(eventType, payload);
}

async function handleEvent(event) {
  const end = eventProcessingDuration.startTimer({ event_type: event.eventType });

  try {
    await processEvent(event);
    eventsProcessed.inc({ event_type: event.eventType, status: 'success' });

  } catch (error) {
    eventsProcessed.inc({ event_type: event.eventType, status: 'failure' });
    throw error;

  } finally {
    end();
  }
}
```

## Best Practices

### 1. Event Naming

```javascript
// ✅ Good: Past tense, domain-specific
'OrderCreated'
'PaymentCompleted'
'UserRegistered'
'InventoryReserved'

// ❌ Bad: Present/future tense, generic
'CreateOrder'
'CompletePayment'
'DataChanged'
'Updated'
```

### 2. Event Size

```javascript
// ✅ Good: Essential data only
{
  "eventType": "OrderCreated",
  "orderId": "order_123",
  "userId": "user_456",
  "total": 109.97,
  "itemCount": 3
}

// ❌ Bad: Too much data
{
  "eventType": "OrderCreated",
  "orderId": "order_123",
  "user": { /* полный объект пользователя */ },
  "items": [ /* все детали товаров */ ],
  "shippingAddress": { /* полный адрес */ },
  // ... ещё 50 полей
}
```

### 3. Event Ordering

```javascript
// Используйте partition key для сохранения порядка
await producer.send({
  topic: 'domain-events',
  messages: [{
    key: order.userId,  // События одного пользователя → одна партиция
    value: JSON.stringify(event),
  }],
});
```

### 4. Backwards Compatibility

```javascript
// Добавление новых полей — OK (backwards compatible)
// v1
{ "orderId": "123", "total": 100 }

// v2
{ "orderId": "123", "total": 100, "currency": "USD" }

// Удаление полей — NOT OK (breaking change)
// Используйте versioning или deprecation
```

## Выводы

Event-Driven Architecture — мощный паттерн для построения масштабируемых, слабо связанных систем:

**Преимущества:**
- **Loose coupling**: сервисы независимы
- **Scalability**: легко добавлять новых subscribers
- **Auditability**: полная история событий
- **Flexibility**: просто добавить новую функциональность
- **Resilience**: отказ одного subscriber не влияет на других

**Недостатки:**
- **Eventual consistency**: сложнее reasoning
- **Debugging**: distributed tracing необходим
- **Event ordering**: требует внимания
- **Schema evolution**: нужна стратегия версионирования

Event-Driven Architecture особенно полезна в микросервисных системах и при необходимости интеграции множества компонентов.

## Проверь себя

1. В чём разница между Request-Driven и Event-Driven архитектурой?
2. Что такое Event Sourcing и какие у него преимущества?
3. В чём разница между Event Notification и Event-Carried State Transfer?
4. Как реализовать Saga Pattern в Event-Driven системе?
5. Что такое CQRS и как он сочетается с Event Sourcing?
6. Как обеспечить idempotency при обработке событий?
7. Зачем нужен Dead Letter Queue?
8. Как решать проблему event versioning?
9. Какие метрики важны для мониторинга Event-Driven системы?
10. Когда НЕ стоит использовать Event-Driven Architecture?

---
**Предыдущий урок**: [Урок 26: Service Mesh — Istio, Linkerd](26-service-mesh.md)
**Следующий урок**: [Урок 28: API Gateway паттерны](28-api-gateway-patterns.md)
