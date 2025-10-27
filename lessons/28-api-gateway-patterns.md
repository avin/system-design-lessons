# Урок 28: API Gateway паттерны


## Введение

API Gateway — это единая точка входа для всех клиентских запросов в микросервисной архитектуре. Он выступает как reverse proxy, перенаправляя запросы к соответствующим сервисам и предоставляя дополнительную функциональность.

**Без API Gateway:**
```
Mobile App → Auth Service
          → Orders Service
          → Products Service
          → Payment Service

Web App   → Auth Service
          → Orders Service
          → Products Service
          → Payment Service
```

**С API Gateway:**
```
Mobile App → ┐
             ├→ API Gateway → Auth Service
Web App   → ┘              → Orders Service
                            → Products Service
                            → Payment Service
```

## Основные функции API Gateway

### 1. Routing (Маршрутизация)

```javascript
// Express.js API Gateway
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');

const app = express();

// Routing к различным сервисам
app.use('/api/auth', createProxyMiddleware({
  target: 'http://auth-service:3001',
  changeOrigin: true,
  pathRewrite: { '^/api/auth': '' },
}));

app.use('/api/orders', createProxyMiddleware({
  target: 'http://orders-service:3002',
  changeOrigin: true,
  pathRewrite: { '^/api/orders': '' },
}));

app.use('/api/products', createProxyMiddleware({
  target: 'http://products-service:3003',
  changeOrigin: true,
  pathRewrite: { '^/api/products': '' },
}));

app.listen(8080, () => {
  console.log('API Gateway running on port 8080');
});
```

### 2. Authentication & Authorization

```javascript
const jwt = require('jsonwebtoken');

// Authentication middleware
async function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;

    // Добавить user context в заголовки для backend services
    req.headers['x-user-id'] = decoded.userId;
    req.headers['x-user-email'] = decoded.email;
    req.headers['x-user-roles'] = JSON.stringify(decoded.roles);

    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// Authorization middleware
function authorize(roles) {
  return (req, res, next) => {
    const userRoles = req.user.roles || [];

    const hasRole = roles.some(role => userRoles.includes(role));

    if (!hasRole) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    next();
  };
}

// Использование
app.use('/api/orders', authenticate, createProxyMiddleware({...}));
app.use('/api/admin', authenticate, authorize(['admin']), createProxyMiddleware({...}));
```

### 3. Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');

const redis = new Redis();

// Global rate limit
const globalLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:global:',
  }),
  windowMs: 15 * 60 * 1000,  // 15 минут
  max: 100,  // 100 requests per window
  message: 'Too many requests from this IP',
});

// Per-user rate limit
const userLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:user:',
  }),
  windowMs: 60 * 1000,  // 1 минута
  max: (req) => {
    // Premium users: 100 req/min
    // Free users: 20 req/min
    return req.user?.tier === 'premium' ? 100 : 20;
  },
  keyGenerator: (req) => req.user?.userId || req.ip,
});

app.use('/api', globalLimiter);
app.use('/api', authenticate, userLimiter);
```

### 4. Request/Response Transformation

```javascript
// Request transformation
app.use('/api/legacy', (req, res, next) => {
  // Трансформация запроса для legacy service
  if (req.body) {
    req.body = {
      data: req.body,
      timestamp: Date.now(),
      version: '1.0',
    };
  }
  next();
});

// Response transformation
app.use('/api/products', createProxyMiddleware({
  target: 'http://products-service:3003',
  onProxyRes: (proxyRes, req, res) => {
    let body = [];

    proxyRes.on('data', (chunk) => {
      body.push(chunk);
    });

    proxyRes.on('end', () => {
      body = Buffer.concat(body).toString();

      try {
        const data = JSON.parse(body);

        // Добавить дополнительные поля
        const transformed = {
          data,
          meta: {
            timestamp: Date.now(),
            requestId: req.headers['x-request-id'],
          },
        };

        res.json(transformed);
      } catch (err) {
        res.status(500).json({ error: 'Invalid response' });
      }
    });
  },
}));
```

### 5. Request Aggregation

```javascript
const axios = require('axios');

// Aggregation: комбинирование данных из нескольких сервисов
app.get('/api/dashboard', authenticate, async (req, res) => {
  try {
    const [user, orders, recommendations] = await Promise.all([
      axios.get(`http://user-service/users/${req.user.userId}`),
      axios.get(`http://orders-service/users/${req.user.userId}/orders`),
      axios.get(`http://recommendations-service/users/${req.user.userId}`),
    ]);

    res.json({
      user: user.data,
      recentOrders: orders.data.slice(0, 5),
      recommendations: recommendations.data,
    });

  } catch (error) {
    console.error('Dashboard aggregation error:', error);
    res.status(500).json({ error: 'Failed to load dashboard' });
  }
});
```

### 6. Caching

```javascript
const redis = new Redis();

async function cacheMiddleware(req, res, next) {
  // Кэшировать только GET запросы
  if (req.method !== 'GET') {
    return next();
  }

  const cacheKey = `cache:${req.originalUrl}`;

  try {
    const cached = await redis.get(cacheKey);

    if (cached) {
      console.log('Cache hit:', cacheKey);
      res.setHeader('X-Cache', 'HIT');
      return res.json(JSON.parse(cached));
    }

    // Перехватываем response
    const originalJson = res.json.bind(res);

    res.json = function(data) {
      // Сохранить в кэш (TTL 5 минут)
      redis.setex(cacheKey, 300, JSON.stringify(data));
      res.setHeader('X-Cache', 'MISS');
      originalJson(data);
    };

    next();

  } catch (err) {
    console.error('Cache error:', err);
    next();
  }
}

app.use('/api/products', cacheMiddleware);
```

### 7. Circuit Breaker

```javascript
const CircuitBreaker = require('opossum');

// Circuit breaker для orders service
const orderServiceBreaker = new CircuitBreaker(
  async (url, options) => {
    return await axios(url, options);
  },
  {
    timeout: 5000,  // 5s timeout
    errorThresholdPercentage: 50,  // Открыть при 50% ошибок
    resetTimeout: 30000,  // Попытаться закрыть через 30s
    volumeThreshold: 10,  // Минимум 10 запросов для оценки
  }
);

orderServiceBreaker.on('open', () => {
  console.log('Circuit breaker opened for orders service');
});

orderServiceBreaker.on('halfOpen', () => {
  console.log('Circuit breaker half-open for orders service');
});

orderServiceBreaker.on('close', () => {
  console.log('Circuit breaker closed for orders service');
});

// Fallback при открытом circuit breaker
orderServiceBreaker.fallback(() => {
  return {
    data: [],
    message: 'Service temporarily unavailable. Showing cached data.',
    cached: true,
  };
});

app.get('/api/orders', authenticate, async (req, res) => {
  try {
    const response = await orderServiceBreaker.fire(
      `http://orders-service/users/${req.user.userId}/orders`,
      { headers: req.headers }
    );

    res.json(response.data);

  } catch (error) {
    res.status(503).json({ error: 'Service unavailable' });
  }
});
```

### 8. Load Balancing

```javascript
const loadBalancers = {
  'orders-service': {
    instances: [
      'http://orders-service-1:3000',
      'http://orders-service-2:3000',
      'http://orders-service-3:3000',
    ],
    currentIndex: 0,
  },
};

function roundRobinLoadBalancer(serviceName) {
  const lb = loadBalancers[serviceName];

  if (!lb) {
    throw new Error(`Unknown service: ${serviceName}`);
  }

  const target = lb.instances[lb.currentIndex];
  lb.currentIndex = (lb.currentIndex + 1) % lb.instances.length;

  return target;
}

app.use('/api/orders', createProxyMiddleware({
  target: 'http://orders-service',  // Будет заменено
  changeOrigin: true,
  router: (req) => {
    return roundRobinLoadBalancer('orders-service');
  },
}));
```

### 9. Request/Response Logging

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'api-gateway.log' }),
  ],
});

app.use((req, res, next) => {
  const start = Date.now();

  // Log request
  logger.info('Incoming request', {
    method: req.method,
    url: req.originalUrl,
    ip: req.ip,
    userAgent: req.get('user-agent'),
    userId: req.user?.userId,
  });

  // Log response
  res.on('finish', () => {
    const duration = Date.now() - start;

    logger.info('Request completed', {
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      duration,
      userId: req.user?.userId,
    });
  });

  next();
});
```

### 10. CORS Handling

```javascript
const cors = require('cors');

app.use(cors({
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://app.example.com',
      'https://mobile.example.com',
    ];

    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
}));
```

## Популярные API Gateway решения

### 1. Kong

```yaml
# Kong declarative config
_format_version: "2.1"

services:
  - name: orders-service
    url: http://orders-service:3000
    routes:
      - name: orders-route
        paths:
          - /api/orders
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
      - name: jwt
        config:
          key_claim_name: kid
      - name: cors
        config:
          origins:
            - https://app.example.com

  - name: products-service
    url: http://products-service:3001
    routes:
      - name: products-route
        paths:
          - /api/products
    plugins:
      - name: response-transformer
        config:
          add:
            headers:
              - "X-Gateway: Kong"
```

### 2. AWS API Gateway

```yaml
# AWS SAM template
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  ApiGateway:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: CognitoAuthorizer
        Authorizers:
          CognitoAuthorizer:
            UserPoolArn: !GetAtt UserPool.Arn
      DefinitionBody:
        swagger: '2.0'
        paths:
          /orders:
            get:
              x-amazon-apigateway-integration:
                uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${OrdersFunction.Arn}/invocations
                httpMethod: POST
                type: aws_proxy
              responses:
                '200':
                  description: Success

  OrdersFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: nodejs18.x
      Handler: index.handler
      CodeUri: ./orders
```

### 3. Nginx

```nginx
# nginx.conf
upstream orders_backend {
    least_conn;
    server orders-1:3000;
    server orders-2:3000;
    server orders-3:3000;
}

upstream products_backend {
    server products-1:3001;
    server products-2:3001;
}

limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 80;
    server_name api.example.com;

    # Rate limiting
    limit_req zone=api_limit burst=20 nodelay;

    # CORS
    add_header 'Access-Control-Allow-Origin' 'https://app.example.com' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;

    # Logging
    access_log /var/log/nginx/api-gateway.log combined;

    location /api/orders {
        proxy_pass http://orders_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Circuit breaker
        proxy_next_upstream error timeout http_500 http_502 http_503;
        proxy_next_upstream_tries 3;
    }

    location /api/products {
        proxy_pass http://products_backend;

        # Caching
        proxy_cache products_cache;
        proxy_cache_valid 200 5m;
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

## Паттерны API Gateway

### 1. Backends for Frontends (BFF)

Отдельный gateway для каждого типа клиента.

```
Mobile App → Mobile BFF → Services
Web App → Web BFF → Services
IoT Devices → IoT BFF → Services
```

Подробнее в [Уроке 29](29-backend-for-frontend.md).

### 2. GraphQL Gateway

```javascript
const { ApolloServer } = require('apollo-server-express');
const { buildFederatedSchema } = require('@apollo/federation');

// GraphQL Schema
const typeDefs = `
  type Query {
    user(id: ID!): User
    orders(userId: ID!): [Order]
    products(category: String): [Product]
  }

  type User {
    id: ID!
    name: String!
    email: String!
    orders: [Order]
  }

  type Order {
    id: ID!
    userId: ID!
    total: Float!
    items: [OrderItem]
  }

  type Product {
    id: ID!
    name: String!
    price: Float!
  }
`;

const resolvers = {
  Query: {
    user: async (_, { id }) => {
      const response = await axios.get(`http://user-service/users/${id}`);
      return response.data;
    },

    orders: async (_, { userId }) => {
      const response = await axios.get(`http://orders-service/users/${userId}/orders`);
      return response.data;
    },

    products: async (_, { category }) => {
      const url = category
        ? `http://products-service/products?category=${category}`
        : 'http://products-service/products';
      const response = await axios.get(url);
      return response.data;
    },
  },

  User: {
    orders: async (user) => {
      const response = await axios.get(`http://orders-service/users/${user.id}/orders`);
      return response.data;
    },
  },
};

const server = new ApolloServer({
  schema: buildFederatedSchema([{ typeDefs, resolvers }]),
});

server.applyMiddleware({ app });
```

### 3. API Composition

```javascript
// Композиция данных из нескольких сервисов
app.get('/api/order-details/:orderId', authenticate, async (req, res) => {
  const { orderId } = req.params;

  try {
    // Параллельные запросы
    const [order, user, products] = await Promise.all([
      axios.get(`http://orders-service/orders/${orderId}`),
      axios.get(`http://user-service/users/${req.user.userId}`),
      axios.get(`http://products-service/products`, {
        params: { ids: order.data.items.map(i => i.productId).join(',') },
      }),
    ]);

    // Обогащение данных
    const enrichedOrder = {
      ...order.data,
      user: {
        id: user.data.id,
        name: user.data.name,
        email: user.data.email,
      },
      items: order.data.items.map(item => {
        const product = products.data.find(p => p.id === item.productId);
        return {
          ...item,
          productName: product?.name,
          productImage: product?.image,
        };
      }),
    };

    res.json(enrichedOrder);

  } catch (error) {
    console.error('Order details error:', error);
    res.status(500).json({ error: 'Failed to load order details' });
  }
});
```

### 4. Protocol Translation

```javascript
// REST → gRPC
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDefinition = protoLoader.loadSync('orders.proto');
const ordersProto = grpc.loadPackageDefinition(packageDefinition).orders;

const grpcClient = new ordersProto.OrderService(
  'orders-service:50051',
  grpc.credentials.createInsecure()
);

app.get('/api/orders/:orderId', authenticate, (req, res) => {
  grpcClient.getOrder({ orderId: req.params.orderId }, (error, response) => {
    if (error) {
      return res.status(500).json({ error: error.message });
    }

    res.json(response);
  });
});
```

## Security Best Practices

### 1. API Key Management

```javascript
const apiKeys = new Map();  // В production: Redis/Database

async function validateApiKey(req, res, next) {
  const apiKey = req.headers['x-api-key'];

  if (!apiKey) {
    return res.status(401).json({ error: 'API key required' });
  }

  const keyData = await apiKeys.get(apiKey);

  if (!keyData) {
    return res.status(401).json({ error: 'Invalid API key' });
  }

  if (keyData.expiresAt < Date.now()) {
    return res.status(401).json({ error: 'API key expired' });
  }

  req.apiKeyData = keyData;
  next();
}

app.use('/api/external', validateApiKey);
```

### 2. Request Validation

```javascript
const Joi = require('joi');

function validateRequest(schema) {
  return (req, res, next) => {
    const { error, value } = schema.validate(req.body);

    if (error) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.details.map(d => d.message),
      });
    }

    req.body = value;
    next();
  };
}

const createOrderSchema = Joi.object({
  items: Joi.array().items(Joi.object({
    productId: Joi.string().required(),
    quantity: Joi.number().integer().min(1).required(),
  })).min(1).required(),
  shippingAddress: Joi.object({
    street: Joi.string().required(),
    city: Joi.string().required(),
    zip: Joi.string().required(),
  }).required(),
});

app.post('/api/orders', authenticate, validateRequest(createOrderSchema), async (req, res) => {
  // req.body уже валидирован
});
```

### 3. Input Sanitization

```javascript
const xss = require('xss');

function sanitizeInput(req, res, next) {
  if (req.body) {
    req.body = sanitizeObject(req.body);
  }

  if (req.query) {
    req.query = sanitizeObject(req.query);
  }

  next();
}

function sanitizeObject(obj) {
  const sanitized = {};

  for (const [key, value] of Object.entries(obj)) {
    if (typeof value === 'string') {
      sanitized[key] = xss(value);
    } else if (typeof value === 'object' && value !== null) {
      sanitized[key] = sanitizeObject(value);
    } else {
      sanitized[key] = value;
    }
  }

  return sanitized;
}

app.use(sanitizeInput);
```

## Мониторинг и Observability

### Metrics

```javascript
const promClient = require('prom-client');

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.5, 1, 2, 5, 10],
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;

    httpRequestDuration.observe(
      {
        method: req.method,
        route: req.route?.path || req.path,
        status_code: res.statusCode,
      },
      duration
    );

    httpRequestTotal.inc({
      method: req.method,
      route: req.route?.path || req.path,
      status_code: res.statusCode,
    });
  });

  next();
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});
```

## Выводы

API Gateway — критический компонент микросервисной архитектуры:

**Ключевые функции:**
- Routing и load balancing
- Authentication и authorization
- Rate limiting
- Request/response transformation
- Caching
- Circuit breaking
- Monitoring и logging

**Best Practices:**
- Держите Gateway лёгким (не бизнес-логики)
- Используйте caching агрессивно
- Реализуйте circuit breakers
- Мониторьте все запросы
- Версионируйте API
- Используйте BFF для разных клиентов

API Gateway упрощает клиентскую разработку и централизует cross-cutting concerns.

## Что читать дальше?

- [Урок 29: Backend for Frontend (BFF)](29-backend-for-frontend.md)
- [Урок 30: Service Discovery](30-service-discovery.md)
- [Урок 11: API Gateway и Reverse Proxy](11-api-gateway-i-reverse-proxy.md)

## Проверь себя

1. Какие основные функции выполняет API Gateway?
2. В чём разница между API Gateway и Reverse Proxy?
3. Как реализовать rate limiting в API Gateway?
4. Что такое request aggregation и зачем он нужен?
5. Как работает circuit breaker pattern?
6. Зачем нужен BFF (Backend for Frontend)?
7. Как обеспечить безопасность API Gateway?
8. Какие метрики важны для мониторинга API Gateway?
9. Когда использовать GraphQL Gateway?
10. Какие проблемы может создать API Gateway?

---
**Предыдущий урок**: [Урок 27: Event-Driven Architecture](27-event-driven-architecture.md)
**Следующий урок**: [Урок 29: Backend for Frontend (BFF)](29-backend-for-frontend.md)
