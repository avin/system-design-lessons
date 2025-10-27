# Урок 29: Backend for Frontend (BFF)


## Введение

Backend for Frontend (BFF) — это архитектурный паттерн, где для каждого типа клиента (web, mobile, IoT) создаётся отдельный backend, оптимизированный под нужды этого клиента.

**Проблема: общий API для всех клиентов**

```
        ┌─────────────────────┐
        │   General API       │
        │   Gateway           │
        └──────────┬──────────┘
                   │
     ┌─────────────┼─────────────┐
     │             │             │
  Mobile         Web          IoT
  (нужны        (нужны      (нужны
   только        все         только
   базовые      детали)      метрики)
   данные)
```

**Проблемы:**
- Over-fetching: mobile получает лишние данные
- Under-fetching: mobile делает много запросов
- Разные требования к формату данных
- Разная логика авторизации

**Решение: BFF**

```
Mobile App → Mobile BFF → Services
Web App    → Web BFF    → Services
IoT Device → IoT BFF    → Services
```

## Когда использовать BFF

### ✅ Используйте BFF когда:

1. **Разные требования клиентов**
   - Mobile: минимальный трафик, оптимизация батареи
   - Web: богатый UI, много данных
   - IoT: только критичные метрики

2. **Разные capabilities**
   - Mobile: offline-first, push notifications
   - Web: WebSockets, streaming
   - Desktop: batch operations, file uploads

3. **Разные команды разработки**
   - Mobile team управляет Mobile BFF
   - Web team управляет Web BFF
   - Автономность команд

### ❌ Не используйте BFF когда:

- Клиенты имеют одинаковые требования
- Маленькая команда (overhead управления несколькими BFF)
- Простое CRUD API
- Нужна строгая консистентность API между клиентами

## Реализация BFF

### Mobile BFF

```javascript
// mobile-bff/server.js
const express = require('express');
const axios = require('axios');

const app = express();

// Оптимизированный endpoint для mobile dashboard
app.get('/api/mobile/dashboard', authenticate, async (req, res) => {
  try {
    // Параллельные запросы к сервисам
    const [user, orders, notifications] = await Promise.all([
      // Только базовая информация о пользователе
      axios.get(`http://user-service/users/${req.user.id}/summary`),

      // Только последние 3 заказа
      axios.get(`http://orders-service/users/${req.user.id}/orders`, {
        params: { limit: 3, fields: 'id,status,total,createdAt' },
      }),

      // Только непрочитанные уведомления
      axios.get(`http://notifications-service/users/${req.user.id}/unread`, {
        params: { limit: 5 },
      }),
    ]);

    // Минимальный response для mobile
    res.json({
      user: {
        id: user.data.id,
        name: user.data.name,
        avatar: user.data.avatarUrl,
      },
      recentOrders: orders.data.map(order => ({
        id: order.id,
        status: order.status,
        total: order.total,
      })),
      unreadCount: notifications.data.count,
      notifications: notifications.data.items.map(n => ({
        id: n.id,
        text: n.message,
        time: n.createdAt,
      })),
    });

  } catch (error) {
    console.error('Mobile dashboard error:', error);
    res.status(500).json({ error: 'Failed to load dashboard' });
  }
});

// Пагинация оптимизирована для mobile (infinite scroll)
app.get('/api/mobile/orders', authenticate, async (req, res) => {
  const { cursor, limit = 10 } = req.query;

  const response = await axios.get(`http://orders-service/users/${req.user.id}/orders`, {
    params: { cursor, limit },
  });

  res.json({
    orders: response.data.items.map(order => ({
      id: order.id,
      status: order.status,
      total: order.total,
      itemCount: order.items.length,
      // Thumbnail только первого товара
      thumbnail: order.items[0]?.image,
    })),
    nextCursor: response.data.nextCursor,
    hasMore: response.data.hasMore,
  });
});

// Push notifications registration
app.post('/api/mobile/push-token', authenticate, async (req, res) => {
  const { token, platform } = req.body;

  await axios.post(`http://notifications-service/devices`, {
    userId: req.user.id,
    token,
    platform,  // 'ios' or 'android'
  });

  res.json({ success: true });
});

app.listen(3001, () => {
  console.log('Mobile BFF running on port 3001');
});
```

### Web BFF

```javascript
// web-bff/server.js
const express = require('express');
const axios = require('axios');

const app = express();

// Полный dashboard для web
app.get('/api/web/dashboard', authenticate, async (req, res) => {
  try {
    const [user, orders, notifications, analytics, recommendations] = await Promise.all([
      // Полная информация о пользователе
      axios.get(`http://user-service/users/${req.user.id}`),

      // Последние 10 заказов с деталями
      axios.get(`http://orders-service/users/${req.user.id}/orders`, {
        params: { limit: 10 },
      }),

      // Все непрочитанные уведомления
      axios.get(`http://notifications-service/users/${req.user.id}/unread`),

      // Аналитика для графиков
      axios.get(`http://analytics-service/users/${req.user.id}/stats`, {
        params: { period: '30d' },
      }),

      // Персональные рекомендации
      axios.get(`http://recommendations-service/users/${req.user.id}`),
    ]);

    // Подробный response для web
    res.json({
      user: user.data,  // Все данные
      orders: orders.data.map(order => ({
        ...order,
        items: order.items.map(item => ({
          ...item,
          product: item.product,  // Полная информация о продукте
        })),
      })),
      notifications: notifications.data,
      analytics: {
        totalSpent: analytics.data.totalSpent,
        ordersCount: analytics.data.ordersCount,
        spendingByMonth: analytics.data.spendingByMonth,  // Для графика
      },
      recommendations: recommendations.data.slice(0, 6),
    });

  } catch (error) {
    console.error('Web dashboard error:', error);
    res.status(500).json({ error: 'Failed to load dashboard' });
  }
});

// Пагинация для web (page-based)
app.get('/api/web/orders', authenticate, async (req, res) => {
  const { page = 1, pageSize = 20 } = req.query;

  const response = await axios.get(`http://orders-service/users/${req.user.id}/orders`, {
    params: {
      offset: (page - 1) * pageSize,
      limit: pageSize,
    },
  });

  res.json({
    orders: response.data.items,
    pagination: {
      page: parseInt(page),
      pageSize: parseInt(pageSize),
      totalCount: response.data.totalCount,
      totalPages: Math.ceil(response.data.totalCount / pageSize),
    },
  });
});

// WebSocket для real-time updates
const { Server } = require('socket.io');
const http = require('http');

const server = http.createServer(app);
const io = new Server(server, {
  cors: { origin: 'https://app.example.com' },
});

io.use(async (socket, next) => {
  const token = socket.handshake.auth.token;
  // Аутентификация
  try {
    const user = await verifyToken(token);
    socket.user = user;
    next();
  } catch (err) {
    next(new Error('Authentication failed'));
  }
});

io.on('connection', (socket) => {
  console.log(`Web client connected: ${socket.user.id}`);

  // Подписка на user-specific комнату
  socket.join(`user:${socket.user.id}`);
});

// Notification service публикует события
// Мы их транслируем web клиентам
const notificationConsumer = kafka.consumer({ groupId: 'web-bff' });

await notificationConsumer.subscribe({ topic: 'user-notifications' });

await notificationConsumer.run({
  eachMessage: async ({ message }) => {
    const notification = JSON.parse(message.value.toString());

    // Отправить через WebSocket
    io.to(`user:${notification.userId}`).emit('notification', {
      id: notification.id,
      type: notification.type,
      message: notification.message,
      timestamp: notification.timestamp,
    });
  },
});

server.listen(3002, () => {
  console.log('Web BFF running on port 3002');
});
```

### IoT BFF

```javascript
// iot-bff/server.js
const express = require('express');
const mqtt = require('mqtt');

const app = express();

// MQTT для IoT devices
const mqttClient = mqtt.connect('mqtt://mqtt-broker:1883');

mqttClient.on('connect', () => {
  console.log('Connected to MQTT broker');
});

// Minimal API для IoT devices (ограниченная память/процессор)
app.post('/api/iot/metrics', async (req, res) => {
  const { deviceId, metrics } = req.body;

  // Валидация device token
  const device = await validateDeviceToken(req.headers['x-device-token']);

  if (!device || device.id !== deviceId) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  // Сохранить метрики
  await axios.post(`http://metrics-service/devices/${deviceId}/metrics`, {
    temperature: metrics.temp,
    humidity: metrics.hum,
    battery: metrics.bat,
    timestamp: Date.now(),
  });

  // Минимальный response (экономия трафика)
  res.json({ ok: 1 });
});

// Получить конфигурацию для device
app.get('/api/iot/config/:deviceId', async (req, res) => {
  const { deviceId } = req.params;

  const device = await validateDeviceToken(req.headers['x-device-token']);

  if (!device) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  const config = await axios.get(`http://config-service/devices/${deviceId}`);

  // Компактный формат
  res.json({
    i: config.data.reportInterval,  // interval
    t: config.data.thresholds,      // thresholds
    v: config.data.version,         // version
  });
});

// MQTT Commands для devices
mqttClient.on('message', async (topic, message) => {
  const [, deviceId, command] = topic.split('/');

  switch (command) {
    case 'restart':
      // Отправить команду device через MQTT
      mqttClient.publish(`devices/${deviceId}/cmd`, JSON.stringify({
        cmd: 'restart',
      }));
      break;

    case 'update-config':
      const config = JSON.parse(message.toString());
      mqttClient.publish(`devices/${deviceId}/cmd`, JSON.stringify({
        cmd: 'config',
        data: config,
      }));
      break;
  }
});

app.listen(3003, () => {
  console.log('IoT BFF running on port 3003');
});
```

## GraphQL BFF

GraphQL отлично подходит для BFF, позволяя клиентам запрашивать именно те данные, которые нужны.

```javascript
// graphql-bff/server.js
const { ApolloServer, gql } = require('apollo-server-express');
const axios = require('axios');

const typeDefs = gql`
  type User {
    id: ID!
    name: String!
    email: String!
    avatar: String
    orders(limit: Int): [Order]
  }

  type Order {
    id: ID!
    status: String!
    total: Float!
    items: [OrderItem]
    createdAt: String!
  }

  type OrderItem {
    productId: ID!
    productName: String!
    quantity: Int!
    price: Float!
  }

  type Query {
    me: User
    order(id: ID!): Order
    orders(limit: Int, cursor: String): OrderConnection
  }

  type OrderConnection {
    edges: [OrderEdge]
    pageInfo: PageInfo
  }

  type OrderEdge {
    node: Order
    cursor: String
  }

  type PageInfo {
    hasNextPage: Boolean
    endCursor: String
  }
`;

const resolvers = {
  Query: {
    me: async (_, __, { user }) => {
      const response = await axios.get(`http://user-service/users/${user.id}`);
      return response.data;
    },

    order: async (_, { id }, { user }) => {
      const response = await axios.get(`http://orders-service/orders/${id}`, {
        headers: { 'X-User-Id': user.id },
      });
      return response.data;
    },

    orders: async (_, { limit = 10, cursor }, { user }) => {
      const response = await axios.get(`http://orders-service/users/${user.id}/orders`, {
        params: { limit, cursor },
      });

      return {
        edges: response.data.items.map(order => ({
          node: order,
          cursor: order.id,
        })),
        pageInfo: {
          hasNextPage: response.data.hasMore,
          endCursor: response.data.nextCursor,
        },
      };
    },
  },

  User: {
    orders: async (user, { limit = 5 }) => {
      const response = await axios.get(`http://orders-service/users/${user.id}/orders`, {
        params: { limit },
      });
      return response.data.items;
    },
  },

  Order: {
    items: async (order) => {
      // Fetch product details для каждого item
      const productIds = order.items.map(i => i.productId);

      const response = await axios.get(`http://products-service/products`, {
        params: { ids: productIds.join(',') },
      });

      return order.items.map(item => {
        const product = response.data.find(p => p.id === item.productId);
        return {
          ...item,
          productName: product?.name,
        };
      });
    },
  },
};

const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => {
    // Extract user from JWT
    const user = getUserFromToken(req.headers.authorization);
    return { user };
  },
});

const app = express();
server.applyMiddleware({ app });

app.listen(4000, () => {
  console.log('GraphQL BFF running on port 4000');
});
```

**Клиент может запрашивать именно то, что нужно:**

```graphql
# Mobile: минимальные данные
query MobileDashboard {
  me {
    id
    name
    avatar
    orders(limit: 3) {
      id
      status
      total
    }
  }
}

# Web: полные данные
query WebDashboard {
  me {
    id
    name
    email
    avatar
    orders(limit: 10) {
      id
      status
      total
      createdAt
      items {
        productName
        quantity
        price
      }
    }
  }
}
```

## Кэширование в BFF

```javascript
const Redis = require('ioredis');
const redis = new Redis();

// Cache decorator
function cache(ttl) {
  return function (target, propertyKey, descriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args) {
      const cacheKey = `bff:${propertyKey}:${JSON.stringify(args)}`;

      // Проверить кэш
      const cached = await redis.get(cacheKey);

      if (cached) {
        console.log('Cache hit:', cacheKey);
        return JSON.parse(cached);
      }

      // Выполнить запрос
      const result = await originalMethod.apply(this, args);

      // Сохранить в кэш
      await redis.setex(cacheKey, ttl, JSON.stringify(result));

      return result;
    };

    return descriptor;
  };
}

class DashboardService {
  @cache(300)  // 5 минут
  async getUserDashboard(userId) {
    const [user, orders, notifications] = await Promise.all([
      axios.get(`http://user-service/users/${userId}`),
      axios.get(`http://orders-service/users/${userId}/orders`),
      axios.get(`http://notifications-service/users/${userId}/unread`),
    ]);

    return {
      user: user.data,
      orders: orders.data,
      notifications: notifications.data,
    };
  }
}
```

## Data Transformation

```javascript
// Backend service возвращает verbose format
const backendResponse = {
  user_identifier: '12345',
  full_name: 'John Doe',
  electronic_mail: 'john@example.com',
  account_creation_timestamp: 1640995200000,
  premium_subscription_active: true,
};

// BFF трансформирует в клиент-friendly format
function transformUser(backendData) {
  return {
    id: backendData.user_identifier,
    name: backendData.full_name,
    email: backendData.electronic_mail,
    joinedAt: new Date(backendData.account_creation_timestamp).toISOString(),
    isPremium: backendData.premium_subscription_active,
  };
}

app.get('/api/users/:id', async (req, res) => {
  const response = await axios.get(`http://user-service/users/${req.params.id}`);
  res.json(transformUser(response.data));
});
```

## Error Handling

```javascript
class BFFError extends Error {
  constructor(message, statusCode, originalError) {
    super(message);
    this.statusCode = statusCode;
    this.originalError = originalError;
  }
}

async function fetchWithErrorHandling(url, options) {
  try {
    return await axios(url, options);
  } catch (error) {
    if (error.response) {
      // Backend service error
      throw new BFFError(
        'Service temporarily unavailable',
        error.response.status,
        error
      );
    } else if (error.request) {
      // Network error
      throw new BFFError(
        'Unable to connect to service',
        503,
        error
      );
    } else {
      throw new BFFError(
        'Unexpected error',
        500,
        error
      );
    }
  }
}

app.use((err, req, res, next) => {
  if (err instanceof BFFError) {
    console.error('BFF Error:', {
      message: err.message,
      statusCode: err.statusCode,
      originalError: err.originalError?.message,
      url: req.originalUrl,
    });

    res.status(err.statusCode).json({
      error: err.message,
      // В development: больше деталей
      ...(process.env.NODE_ENV === 'development' && {
        details: err.originalError?.message,
      }),
    });
  } else {
    next(err);
  }
});
```

## Мониторинг BFF

```javascript
const promClient = require('prom-client');

// Метрики для upstream services
const upstreamRequestDuration = new promClient.Histogram({
  name: 'bff_upstream_request_duration_seconds',
  help: 'Duration of upstream service requests',
  labelNames: ['service', 'endpoint', 'status'],
});

const upstreamRequestErrors = new promClient.Counter({
  name: 'bff_upstream_request_errors_total',
  help: 'Total upstream service errors',
  labelNames: ['service', 'error_type'],
});

// Wrapper для всех upstream запросов
async function fetchFromService(serviceName, endpoint, options) {
  const start = Date.now();

  try {
    const response = await axios(`http://${serviceName}${endpoint}`, options);

    const duration = (Date.now() - start) / 1000;

    upstreamRequestDuration.observe(
      {
        service: serviceName,
        endpoint,
        status: response.status,
      },
      duration
    );

    return response;

  } catch (error) {
    upstreamRequestErrors.inc({
      service: serviceName,
      error_type: error.code || 'unknown',
    });

    throw error;
  }
}
```

## BFF vs API Gateway

| Критерий | BFF | API Gateway |
|----------|-----|-------------|
| **Цель** | Оптимизация для клиента | Routing и cross-cutting concerns |
| **Логика** | Бизнес-логика композиции | Минимальная логика |
| **Ownership** | Frontend team | Platform/Ops team |
| **Количество** | По одному на клиент | Обычно один |
| **Изменения** | Часто (с frontend) | Редко |
| **Ответственность** | Data aggregation, transformation | Auth, rate limiting, routing |

**Комбинация:**

```
Mobile App → API Gateway → Mobile BFF → Services
Web App    → API Gateway → Web BFF    → Services

API Gateway: Auth, Rate Limiting, SSL
BFF: Data aggregation, Client-specific logic
```

## Best Practices

### 1. Держите BFF thin

```javascript
// ❌ Bad: бизнес-логика в BFF
app.post('/api/orders', async (req, res) => {
  // Вычисление скидок, налогов — это domain logic!
  const discount = calculateDiscount(req.body.items, req.user);
  const tax = calculateTax(req.body.total, req.user.location);
  const total = req.body.total - discount + tax;

  // ...
});

// ✅ Good: только композиция и трансформация
app.post('/api/orders', async (req, res) => {
  // Делегируем бизнес-логику сервису
  const order = await axios.post('http://orders-service/orders', {
    userId: req.user.id,
    items: req.body.items,
  });

  // Только трансформация для клиента
  res.json(transformOrder(order.data));
});
```

### 2. Версионирование

```javascript
// v1: старые mobile клиенты
app.get('/api/v1/mobile/dashboard', async (req, res) => {
  // Старый формат
});

// v2: новые mobile клиенты
app.get('/api/v2/mobile/dashboard', async (req, res) => {
  // Новый формат
});
```

### 3. Graceful degradation

```javascript
app.get('/api/dashboard', async (req, res) => {
  const [userResult, ordersResult, notificationsResult] = await Promise.allSettled([
    fetchUser(req.user.id),
    fetchOrders(req.user.id),
    fetchNotifications(req.user.id),
  ]);

  res.json({
    user: userResult.status === 'fulfilled' ? userResult.value : null,
    orders: ordersResult.status === 'fulfilled' ? ordersResult.value : [],
    notifications: notificationsResult.status === 'fulfilled' ? notificationsResult.value : [],
    errors: {
      user: userResult.status === 'rejected' ? 'Failed to load user data' : null,
      orders: ordersResult.status === 'rejected' ? 'Failed to load orders' : null,
      notifications: notificationsResult.status === 'rejected' ? 'Failed to load notifications' : null,
    },
  });
});
```

## Выводы

Backend for Frontend (BFF) — мощный паттерн для оптимизации клиент-серверной коммуникации:

**Преимущества:**
- Оптимизация под нужды клиента
- Автономность frontend команд
- Гибкость в изменениях
- Снижение over-fetching/under-fetching

**Недостатки:**
- Дублирование кода между BFF
- Operational complexity (несколько сервисов)
- Риск domain logic в BFF

**Когда использовать:**
- Разные требования клиентов
- Автономные команды
- Микросервисная архитектура

BFF особенно эффективен в комбинации с API Gateway и микросервисами.

## Проверь себя

1. Что такое BFF и какую проблему он решает?
2. В чём разница между BFF и API Gateway?
3. Когда стоит использовать отдельные BFF для разных клиентов?
4. Как GraphQL помогает реализовать BFF?
5. Какую логику допустимо размещать в BFF?
6. Как обрабатывать ошибки от upstream сервисов?
7. Зачем нужно кэширование в BFF?
8. Как версионировать BFF API?
9. Что такое graceful degradation?
10. Какие метрики важны для мониторинга BFF?

---
**Предыдущий урок**: [Урок 28: API Gateway паттерны](28-api-gateway-patterns.md)
**Следующий урок**: [Урок 30: Service Discovery](30-service-discovery.md)
