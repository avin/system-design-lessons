# Урок 25: Микросервисы — когда и зачем


## Введение

Микросервисная архитектура — это подход к построению приложения как набора небольших, автономных сервисов, каждый из которых работает в своём процессе и общается через лёгкие механизмы (обычно HTTP API).

**Монолит vs Микросервисы:**

```
МОНОЛИТ:
┌────────────────────────────┐
│   Single Application       │
│  ┌──────┐ ┌──────┐        │
│  │ Auth │ │Orders│        │
│  └──────┘ └──────┘        │
│  ┌──────┐ ┌──────┐        │
│  │Users │ │ Pay  │        │
│  └──────┘ └──────┘        │
│         ↓                  │
│    Single Database         │
└────────────────────────────┘

МИКРОСЕРВИСЫ:
┌─────────┐  ┌─────────┐  ┌─────────┐
│  Auth   │  │ Orders  │  │ Payment │
│ Service │  │ Service │  │ Service │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
   ┌─┴─┐        ┌─┴─┐        ┌─┴─┐
   │ DB│        │ DB│        │ DB│
   └───┘        └───┘        └───┘
```

## Когда НЕ нужны микросервисы

### Начните с монолита, если:

❌ **Маленькая команда** (< 5 разработчиков)
- Overhead микросервисов превысит пользу
- Координация проще в одной кодовой базе

❌ **MVP или стартап на ранней стадии**
- Требования быстро меняются
- Нужна скорость разработки, а не масштабируемость
- Преждевременная оптимизация

❌ **Простое приложение**
- CRUD операции
- Нет сложных бизнес-процессов
- Умеренная нагрузка

❌ **Недостаточно DevOps экспертизы**
- Микросервисы требуют infrastructure as code
- Мониторинг, трейсинг, service discovery
- CI/CD pipeline для каждого сервиса

### Проблемы микросервисов

1. **Сложность инфраструктуры**
   - Service discovery
   - Load balancing
   - Distributed tracing
   - Centralized logging

2. **Распределённые транзакции**
   - Нет ACID гарантий между сервисами
   - Eventual consistency
   - Saga pattern

3. **Network latency**
   - Больше сетевых вызовов
   - Cascading failures

4. **Операционная сложность**
   - Deployment множества сервисов
   - Версионирование API
   - Backward compatibility

5. **Testing сложнее**
   - Integration тесты требуют поднять все сервисы
   - Contract testing

## Когда микросервисы оправданы

### ✅ Используйте микросервисы когда:

**1. Большая команда (10+ разработчиков)**
- Разные команды работают над разными частями системы
- Автономность команд повышает продуктивность

**2. Независимое масштабирование**
```
┌──────────┐  ← 2 instances
│  Auth    │
└──────────┘

┌──────────┐  ← 10 instances (высокая нагрузка)
│  Orders  │
└──────────┘

┌──────────┐  ← 3 instances
│  Payment │
└──────────┘
```

**3. Разные технологические стеки**
- Auth Service: Node.js (быстрые I/O операции)
- ML Service: Python (богатая экосистема ML)
- Real-time Service: Go (высокая производительность)

**4. Разная частота обновлений**
- Payment Service: редкие обновления (критичный функционал)
- Recommendation Service: частые обновления (A/B тесты)

**5. Изоляция отказов**
- Падение одного сервиса не ломает всё приложение
- Circuit breaker паттерн

**6. Compliance требования**
- PCI DSS для платежей
- HIPAA для медицинских данных
- Изоляция чувствительных данных

## От монолита к микросервисам

### Strangler Fig Pattern

Постепенная миграция вместо rewrite:

```
Шаг 1: Монолит
┌──────────────────┐
│    Monolith      │
│  All Features    │
└──────────────────┘

Шаг 2: Выделение первого сервиса
┌──────────────────┐     ┌──────────┐
│    Monolith      │────→│  Auth    │
│  (minus Auth)    │     │ Service  │
└──────────────────┘     └──────────┘

Шаг 3: Продолжение миграции
┌──────────────┐  ┌──────────┐  ┌──────────┐
│   Monolith   │  │  Auth    │  │  Orders  │
│  (Core only) │  │ Service  │  │ Service  │
└──────────────┘  └──────────┘  └──────────┘
```

**Преимущества:**
- Снижение рисков (постепенный переход)
- Возможность откатиться
- Обучение команды в процессе

### Идентификация границ сервисов

**Domain-Driven Design (DDD):**

```
E-commerce приложение

Bounded Contexts:
1. User Management (Identity & Access)
2. Product Catalog
3. Order Management
4. Inventory
5. Payment Processing
6. Shipping & Fulfillment
7. Notifications
```

**Принципы выделения:**
- **Single Responsibility**: сервис делает одну вещь
- **Bounded Context**: чёткие границы домена
- **Low Coupling**: минимум зависимостей между сервисами
- **High Cohesion**: связанная функциональность внутри сервиса

### Анти-паттерны

❌ **Micro-monolith**: сервисы тесно связаны, деплоятся вместе
❌ **Distributed monolith**: shared database между сервисами
❌ **Nanoservices**: слишком мелкие сервисы (overhead > польза)
❌ **Entity Services**: сервисы по таблицам БД (User Service, Order Service)

## Коммуникация между сервисами

### 1. Synchronous (REST/gRPC)

```javascript
// Order Service вызывает Payment Service
const axios = require('axios');

async function createOrder(orderData) {
  // 1. Создать заказ
  const order = await db.orders.create(orderData);

  // 2. Синхронный вызов Payment Service
  try {
    const payment = await axios.post('http://payment-service/charge', {
      orderId: order.id,
      amount: order.total,
      method: orderData.paymentMethod,
    });

    // 3. Обновить заказ
    await db.orders.update(order.id, {
      paymentId: payment.data.id,
      status: 'paid',
    });

    return order;

  } catch (error) {
    // Rollback или compensation
    await db.orders.update(order.id, { status: 'payment_failed' });
    throw error;
  }
}
```

**Проблемы:**
- Высокая coupling
- Cascading failures
- Latency суммируется

### 2. Asynchronous (Message Queue/Events)

```javascript
// Order Service публикует событие
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ brokers: ['localhost:9092'] });
const producer = kafka.producer();

async function createOrder(orderData) {
  // 1. Создать заказ
  const order = await db.orders.create({
    ...orderData,
    status: 'pending',
  });

  // 2. Опубликовать событие
  await producer.send({
    topic: 'order-created',
    messages: [{
      key: order.id,
      value: JSON.stringify({
        orderId: order.id,
        userId: order.userId,
        total: order.total,
        items: order.items,
        timestamp: Date.now(),
      }),
    }],
  });

  return order;
}

// Payment Service подписывается на событие
const consumer = kafka.consumer({ groupId: 'payment-service' });

await consumer.subscribe({ topic: 'order-created' });

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const order = JSON.parse(message.value.toString());

    try {
      // Обработка платежа
      const payment = await processPayment(order);

      // Публикация результата
      await producer.send({
        topic: 'payment-completed',
        messages: [{
          key: order.orderId,
          value: JSON.stringify({
            orderId: order.orderId,
            paymentId: payment.id,
            status: 'success',
          }),
        }],
      });

    } catch (error) {
      // Публикация ошибки
      await producer.send({
        topic: 'payment-failed',
        messages: [{
          key: order.orderId,
          value: JSON.stringify({
            orderId: order.orderId,
            error: error.message,
          }),
        }],
      });
    }
  },
});
```

**Преимущества:**
- Низкая coupling
- Fault tolerance
- Возможность replay событий

**Недостатки:**
- Eventual consistency
- Сложнее debugging
- Необходимость idempotency

### 3. Service Mesh (Istio, Linkerd)

Автоматическое управление коммуникацией:
- Service discovery
- Load balancing
- Retry logic
- Circuit breaking
- Distributed tracing

Подробнее в [Уроке 26](26-service-mesh.md).

## Data Management

### Database per Service

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Orders    │     │  Inventory  │     │   Payment   │
│   Service   │     │   Service   │     │   Service   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
   ┌───┴───┐           ┌───┴───┐           ┌───┴───┐
   │Orders │           │Inventory│         │Payment│
   │  DB   │           │   DB    │         │  DB   │
   └───────┘           └─────────┘         └───────┘
```

**Преимущества:**
- Автономность сервисов
- Свобода выбора технологии БД
- Изоляция данных

**Проблемы:**
- Нет JOIN между сервисами
- Распределённые транзакции
- Дублирование данных

### Saga Pattern

Распределённая транзакция через последовательность локальных транзакций.

**Choreography-based Saga:**

```javascript
// Order Service
async function createOrder(orderData) {
  const order = await db.orders.create({
    ...orderData,
    status: 'pending',
  });

  await eventBus.publish('OrderCreated', { orderId: order.id, ...orderData });
  return order;
}

await eventBus.subscribe('PaymentCompleted', async (event) => {
  await db.orders.update(event.orderId, { status: 'paid' });
  await eventBus.publish('OrderPaid', { orderId: event.orderId });
});

await eventBus.subscribe('PaymentFailed', async (event) => {
  await db.orders.update(event.orderId, { status: 'cancelled' });
  await eventBus.publish('OrderCancelled', { orderId: event.orderId });
});

// Payment Service
await eventBus.subscribe('OrderCreated', async (event) => {
  try {
    const payment = await processPayment(event);
    await eventBus.publish('PaymentCompleted', {
      orderId: event.orderId,
      paymentId: payment.id,
    });
  } catch (error) {
    await eventBus.publish('PaymentFailed', {
      orderId: event.orderId,
      error: error.message,
    });
  }
});

// Inventory Service
await eventBus.subscribe('OrderPaid', async (event) => {
  try {
    await reserveInventory(event.orderId);
    await eventBus.publish('InventoryReserved', { orderId: event.orderId });
  } catch (error) {
    // Compensation: возврат денег
    await eventBus.publish('InventoryReservationFailed', {
      orderId: event.orderId,
    });
  }
});

await eventBus.subscribe('OrderCancelled', async (event) => {
  await releaseInventory(event.orderId);
});
```

**Orchestration-based Saga:**

```javascript
// Order Orchestrator
class OrderSaga {
  async execute(orderData) {
    const sagaId = generateId();

    try {
      // Step 1: Create Order
      const order = await orderService.create(orderData);

      // Step 2: Process Payment
      const payment = await paymentService.charge({
        orderId: order.id,
        amount: order.total,
      });

      // Step 3: Reserve Inventory
      await inventoryService.reserve({
        orderId: order.id,
        items: order.items,
      });

      // Step 4: Ship Order
      await shippingService.schedule({
        orderId: order.id,
        address: order.shippingAddress,
      });

      return { success: true, orderId: order.id };

    } catch (error) {
      // Compensation (rollback)
      await this.compensate(sagaId, error);
      throw error;
    }
  }

  async compensate(sagaId, error) {
    // Откат в обратном порядке
    await shippingService.cancel(sagaId);
    await inventoryService.release(sagaId);
    await paymentService.refund(sagaId);
    await orderService.cancel(sagaId);
  }
}
```

### CQRS (Command Query Responsibility Segregation)

Разделение моделей для чтения и записи:

```javascript
// Write Model: Command Service
class OrderCommandService {
  async createOrder(command) {
    const order = await db.orders.create(command);

    // Publish event
    await eventBus.publish('OrderCreated', {
      orderId: order.id,
      userId: command.userId,
      total: order.total,
      items: order.items,
    });

    return order.id;
  }

  async updateOrderStatus(orderId, status) {
    await db.orders.update(orderId, { status });

    await eventBus.publish('OrderStatusChanged', {
      orderId,
      status,
      timestamp: Date.now(),
    });
  }
}

// Read Model: Query Service
class OrderQueryService {
  constructor() {
    // Подписка на события для обновления read model
    eventBus.subscribe('OrderCreated', this.onOrderCreated.bind(this));
    eventBus.subscribe('OrderStatusChanged', this.onOrderStatusChanged.bind(this));
  }

  async onOrderCreated(event) {
    // Сохранить в read-optimized БД (напр., Elasticsearch)
    await elasticsearch.index({
      index: 'orders',
      id: event.orderId,
      body: {
        orderId: event.orderId,
        userId: event.userId,
        total: event.total,
        items: event.items,
        status: 'pending',
        createdAt: Date.now(),
      },
    });
  }

  async onOrderStatusChanged(event) {
    await elasticsearch.update({
      index: 'orders',
      id: event.orderId,
      body: {
        doc: {
          status: event.status,
          updatedAt: event.timestamp,
        },
      },
    });
  }

  // Query methods
  async getOrder(orderId) {
    return await elasticsearch.get({
      index: 'orders',
      id: orderId,
    });
  }

  async searchOrders(query) {
    return await elasticsearch.search({
      index: 'orders',
      body: query,
    });
  }
}
```

## API Gateway

Единая точка входа для клиентов:

```
Clients → API Gateway → ┌→ Auth Service
                        ├→ Orders Service
                        ├→ Products Service
                        └→ Payment Service
```

**Функции API Gateway:**
- Routing
- Authentication & Authorization
- Rate Limiting
- Request/Response transformation
- Caching
- Monitoring & Logging

```javascript
// Kong, AWS API Gateway, или custom Express.js

const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');

const app = express();

// Authentication middleware
app.use(async (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  try {
    const user = await verifyToken(token);
    req.user = user;
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
});

// Rate limiting
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
});

app.use('/api', limiter);

// Routing to services
app.use('/api/auth', createProxyMiddleware({
  target: 'http://auth-service:3001',
  changeOrigin: true,
}));

app.use('/api/orders', createProxyMiddleware({
  target: 'http://orders-service:3002',
  changeOrigin: true,
  onProxyReq: (proxyReq, req) => {
    // Добавить user info в заголовки
    proxyReq.setHeader('X-User-Id', req.user.id);
    proxyReq.setHeader('X-User-Email', req.user.email);
  },
}));

app.use('/api/products', createProxyMiddleware({
  target: 'http://products-service:3003',
  changeOrigin: true,
}));

app.listen(8080, () => {
  console.log('API Gateway running on port 8080');
});
```

## Service Discovery

### Client-side discovery (Consul, Eureka)

```javascript
const Consul = require('consul');

const consul = new Consul();

// Register service
await consul.agent.service.register({
  name: 'orders-service',
  id: `orders-service-${process.env.INSTANCE_ID}`,
  address: process.env.HOST,
  port: parseInt(process.env.PORT),
  check: {
    http: `http://${process.env.HOST}:${process.env.PORT}/health`,
    interval: '10s',
  },
});

// Discover service
async function getServiceUrl(serviceName) {
  const services = await consul.health.service({
    service: serviceName,
    passing: true,  // Только healthy instances
  });

  if (services.length === 0) {
    throw new Error(`No healthy instances of ${serviceName}`);
  }

  // Load balancing: random selection
  const service = services[Math.floor(Math.random() * services.length)];
  return `http://${service.Service.Address}:${service.Service.Port}`;
}

// Usage
const paymentServiceUrl = await getServiceUrl('payment-service');
const response = await axios.post(`${paymentServiceUrl}/charge`, data);
```

### Server-side discovery (Kubernetes, AWS ELB)

```yaml
# Kubernetes Service
apiVersion: v1
kind: Service
metadata:
  name: orders-service
spec:
  selector:
    app: orders
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

Клиенты просто обращаются к `http://orders-service`, Kubernetes сам роутит запросы.

## Мониторинг и Observability

### Distributed Tracing (Jaeger, Zipkin)

```javascript
const opentelemetry = require('@opentelemetry/api');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');

// Setup tracer
const provider = new NodeTracerProvider();
const exporter = new JaegerExporter({
  endpoint: 'http://jaeger:14268/api/traces',
});

provider.addSpanProcessor(
  new opentelemetry.SimpleSpanProcessor(exporter)
);

provider.register();

const tracer = opentelemetry.trace.getTracer('orders-service');

// Trace a request
app.post('/orders', async (req, res) => {
  const span = tracer.startSpan('create_order');

  try {
    // Child span: database operation
    const dbSpan = tracer.startSpan('save_to_db', {
      parent: span,
    });

    const order = await db.orders.create(req.body);
    dbSpan.end();

    // Child span: call payment service
    const paymentSpan = tracer.startSpan('charge_payment', {
      parent: span,
    });

    const payment = await axios.post('http://payment-service/charge', {
      orderId: order.id,
      amount: order.total,
    }, {
      headers: {
        // Propagate trace context
        'x-trace-id': span.spanContext().traceId,
        'x-span-id': paymentSpan.spanContext().spanId,
      },
    });

    paymentSpan.end();

    span.setStatus({ code: opentelemetry.SpanStatusCode.OK });
    res.json({ orderId: order.id });

  } catch (error) {
    span.setStatus({
      code: opentelemetry.SpanStatusCode.ERROR,
      message: error.message,
    });
    res.status(500).json({ error: error.message });

  } finally {
    span.end();
  }
});
```

### Centralized Logging (ELK, Loki)

```javascript
const winston = require('winston');
const { ElasticsearchTransport } = require('winston-elasticsearch');

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'orders-service',
    version: process.env.VERSION,
    instance: process.env.INSTANCE_ID,
  },
  transports: [
    new ElasticsearchTransport({
      level: 'info',
      clientOpts: { node: 'http://elasticsearch:9200' },
      index: 'microservices-logs',
    }),
  ],
});

app.post('/orders', async (req, res) => {
  logger.info('Creating order', {
    userId: req.user.id,
    itemsCount: req.body.items.length,
    total: req.body.total,
  });

  try {
    const order = await createOrder(req.body);

    logger.info('Order created successfully', {
      orderId: order.id,
      userId: req.user.id,
    });

    res.json({ orderId: order.id });

  } catch (error) {
    logger.error('Failed to create order', {
      error: error.message,
      stack: error.stack,
      userId: req.user.id,
    });

    res.status(500).json({ error: error.message });
  }
});
```

## Deployment

### Docker Compose (development)

```yaml
version: '3.8'

services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      - AUTH_SERVICE_URL=http://auth-service:3001
      - ORDERS_SERVICE_URL=http://orders-service:3002

  auth-service:
    build: ./auth-service
    environment:
      - DATABASE_URL=postgresql://user:pass@auth-db:5432/auth

  orders-service:
    build: ./orders-service
    environment:
      - DATABASE_URL=postgresql://user:pass@orders-db:5432/orders
      - KAFKA_BROKERS=kafka:9092

  payment-service:
    build: ./payment-service
    environment:
      - KAFKA_BROKERS=kafka:9092

  kafka:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9092:9092"
```

### Kubernetes (production)

```yaml
# orders-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: orders
          image: myregistry/orders-service:v1.2.3
          ports:
            - containerPort: 3000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: orders-db-secret
                  key: url
            - name: KAFKA_BROKERS
              value: "kafka:9092"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: orders-service
spec:
  selector:
    app: orders
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## Выводы

Микросервисы — мощный инструмент, но не серебряная пуля:

**Преимущества:**
- Автономность команд и сервисов
- Независимое масштабирование
- Технологическое разнообразие
- Изоляция отказов
- Быстрая доставка фич

**Недостатки:**
- Operational complexity
- Распределённые транзакции
- Network overhead
- Debugging сложнее
- Требует DevOps зрелости

**Правило:**
Начинайте с монолита. Переходите к микросервисам когда **боль от монолита** превышает **сложность микросервисов**.

## Что читать дальше?

- [Урок 26: Service Mesh](26-service-mesh.md)
- [Урок 27: Event-Driven Architecture](27-event-driven-architecture.md)
- [Урок 22: Kafka и Event Streaming](22-kafka-event-streaming.md)

## Проверь себя

1. В чём основные отличия монолита от микросервисов?
2. Когда НЕ нужно использовать микросервисы?
3. Что такое Strangler Fig Pattern?
4. В чём разница между Choreography и Orchestration в Saga?
5. Зачем нужен API Gateway?
6. Как работает Service Discovery?
7. Что такое Database per Service pattern и какие у него проблемы?
8. Как обеспечить Observability в микросервисной системе?
9. В чём разница между синхронной и асинхронной коммуникацией?
10. Какие проблемы решает CQRS?

---
**Предыдущий урок**: [Урок 24: WebSockets в production](24-websockets-production.md)
**Следующий урок**: [Урок 26: Service Mesh — Istio, Linkerd](26-service-mesh.md)
