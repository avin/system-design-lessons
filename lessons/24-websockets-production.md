# Урок 24: WebSockets в production


## Введение

В [Уроке 9](09-websockets-sse-long-polling.md) мы изучили основы WebSockets. Теперь разберём как развернуть WebSocket-соединения в production с учётом масштабирования, отказоустойчивости и мониторинга.

**Production challenges:**
- Балансировка нагрузки с sticky sessions
- Масштабирование горизонтально (multiple servers)
- Синхронизация сообщений между серверами
- Переподключение и восстановление состояния
- Аутентификация и авторизация
- Rate limiting
- Мониторинг и отладка

## Архитектура WebSocket-системы

### Простая архитектура (single server)

```
Client ←→ WebSocket Server ←→ Application Logic
```

**Проблемы:**
- Single point of failure
- Ограниченная производительность
- Невозможно горизонтальное масштабирование

### Масштабируемая архитектура

```
                  ┌─→ WS Server 1 ←┐
Clients → ALB →   ├─→ WS Server 2 ←┤ ←→ Redis Pub/Sub
                  └─→ WS Server 3 ←┘
```

**Ключевые компоненты:**
- **Load Balancer** с поддержкой sticky sessions
- **Множество WS серверов** для масштабирования
- **Redis Pub/Sub** или другой message broker для синхронизации
- **Database** для хранения состояния пользователей

## Socket.IO в production

Socket.IO — популярная библиотека с автоматическим fallback и удобным API.

### Настройка сервера

```javascript
// server.js
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const app = express();
const httpServer = createServer(app);

const io = new Server(httpServer, {
  cors: {
    origin: process.env.CORS_ORIGIN || '*',
    credentials: true,
  },
  transports: ['websocket', 'polling'],
  pingTimeout: 60000,
  pingInterval: 25000,
  upgradeTimeout: 30000,
  maxHttpBufferSize: 1e6,  // 1 MB
  allowEIO3: true,  // Обратная совместимость
});

// Redis Adapter для multi-server setup
const pubClient = createClient({ url: 'redis://localhost:6379' });
const subClient = pubClient.duplicate();

Promise.all([pubClient.connect(), subClient.connect()]).then(() => {
  io.adapter(createAdapter(pubClient, subClient));
  console.log('Redis adapter connected');
});

// Middleware: аутентификация
io.use(async (socket, next) => {
  const token = socket.handshake.auth.token;

  if (!token) {
    return next(new Error('Authentication error'));
  }

  try {
    const user = await verifyToken(token);
    socket.user = user;
    next();
  } catch (err) {
    next(new Error('Invalid token'));
  }
});

// Middleware: rate limiting
const rateLimiter = new Map();

io.use((socket, next) => {
  const userId = socket.user.id;
  const now = Date.now();
  const windowMs = 60000;  // 1 минута
  const maxRequests = 100;

  if (!rateLimiter.has(userId)) {
    rateLimiter.set(userId, []);
  }

  const requests = rateLimiter.get(userId);
  const recentRequests = requests.filter(time => now - time < windowMs);

  if (recentRequests.length >= maxRequests) {
    return next(new Error('Rate limit exceeded'));
  }

  recentRequests.push(now);
  rateLimiter.set(userId, recentRequests);

  next();
});

// Connection handler
io.on('connection', (socket) => {
  console.log(`User ${socket.user.id} connected: ${socket.id}`);

  // Join user-specific room
  socket.join(`user:${socket.user.id}`);

  // Join rooms based on user subscriptions
  getUserRooms(socket.user.id).then(rooms => {
    rooms.forEach(room => socket.join(room));
  });

  // Handle messages
  socket.on('message', async (data) => {
    try {
      const { roomId, content } = data;

      // Валидация прав доступа
      if (!await canSendToRoom(socket.user.id, roomId)) {
        socket.emit('error', { message: 'Access denied' });
        return;
      }

      // Сохранение в БД
      const message = await saveMessage({
        userId: socket.user.id,
        roomId,
        content,
        timestamp: Date.now(),
      });

      // Broadcast в комнату (все серверы через Redis)
      io.to(roomId).emit('message', {
        id: message.id,
        userId: socket.user.id,
        username: socket.user.username,
        content,
        timestamp: message.timestamp,
      });

    } catch (err) {
      console.error('Message error:', err);
      socket.emit('error', { message: 'Failed to send message' });
    }
  });

  // Typing indicator
  socket.on('typing', ({ roomId }) => {
    socket.to(roomId).emit('user-typing', {
      userId: socket.user.id,
      username: socket.user.username,
    });
  });

  // Disconnect
  socket.on('disconnect', (reason) => {
    console.log(`User ${socket.user.id} disconnected: ${reason}`);

    // Уведомить комнаты о выходе
    getUserRooms(socket.user.id).then(rooms => {
      rooms.forEach(room => {
        io.to(room).emit('user-left', {
          userId: socket.user.id,
          username: socket.user.username,
        });
      });
    });
  });
});

const PORT = process.env.PORT || 3000;
httpServer.listen(PORT, () => {
  console.log(`WebSocket server running on port ${PORT}`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, closing server...');

  httpServer.close(() => {
    io.close(() => {
      console.log('Server closed');
      process.exit(0);
    });
  });
});
```

### Клиент с reconnection

```javascript
// client.js
import { io } from 'socket.io-client';

class WebSocketClient {
  constructor(url, token) {
    this.url = url;
    this.token = token;
    this.socket = null;
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 10;
    this.messageQueue = [];
  }

  connect() {
    this.socket = io(this.url, {
      auth: { token: this.token },
      transports: ['websocket', 'polling'],
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000,
      reconnectionAttempts: this.maxReconnectAttempts,
      timeout: 20000,
    });

    this.socket.on('connect', () => {
      console.log('Connected to WebSocket');
      this.reconnectAttempts = 0;

      // Отправить накопленные сообщения
      this.flushMessageQueue();

      // Повторная подписка на комнаты
      this.resubscribeToRooms();
    });

    this.socket.on('disconnect', (reason) => {
      console.log('Disconnected:', reason);

      if (reason === 'io server disconnect') {
        // Сервер явно отключил → переподключение вручную
        this.socket.connect();
      }
    });

    this.socket.on('connect_error', (error) => {
      console.error('Connection error:', error.message);
      this.reconnectAttempts++;

      if (this.reconnectAttempts >= this.maxReconnectAttempts) {
        console.error('Max reconnection attempts reached');
        this.onMaxReconnectAttempts();
      }
    });

    this.socket.on('error', (error) => {
      console.error('Socket error:', error);
    });

    // Application events
    this.socket.on('message', (data) => {
      this.handleMessage(data);
    });

    this.socket.on('user-typing', (data) => {
      this.handleTyping(data);
    });

    this.socket.on('user-left', (data) => {
      this.handleUserLeft(data);
    });
  }

  sendMessage(roomId, content) {
    const message = { roomId, content };

    if (this.socket.connected) {
      this.socket.emit('message', message);
    } else {
      // Накопить для отправки после переподключения
      this.messageQueue.push(message);
    }
  }

  flushMessageQueue() {
    while (this.messageQueue.length > 0) {
      const message = this.messageQueue.shift();
      this.socket.emit('message', message);
    }
  }

  joinRoom(roomId) {
    this.socket.emit('join-room', { roomId });
  }

  leaveRoom(roomId) {
    this.socket.emit('leave-room', { roomId });
  }

  resubscribeToRooms() {
    // Восстановить подписки на комнаты после переподключения
    const rooms = this.getRoomsFromLocalState();
    rooms.forEach(roomId => this.joinRoom(roomId));
  }

  handleMessage(data) {
    console.log('New message:', data);
    // Update UI
  }

  handleTyping(data) {
    console.log(`${data.username} is typing...`);
    // Show typing indicator
  }

  handleUserLeft(data) {
    console.log(`${data.username} left`);
    // Update user list
  }

  onMaxReconnectAttempts() {
    // Показать пользователю сообщение о проблемах с соединением
    alert('Unable to connect to server. Please refresh the page.');
  }

  disconnect() {
    if (this.socket) {
      this.socket.disconnect();
    }
  }
}

// Использование
const client = new WebSocketClient('https://ws.example.com', 'user-token-123');
client.connect();

client.sendMessage('room-1', 'Hello, world!');
```

## Load Balancing

### NGINX конфигурация

```nginx
# nginx.conf

upstream websocket_backend {
    # Sticky sessions на базе IP
    ip_hash;

    server ws-server-1:3000;
    server ws-server-2:3000;
    server ws-server-3:3000;

    # Health checks
    # Требует NGINX Plus или open-source модуль
}

server {
    listen 80;
    server_name ws.example.com;

    location / {
        proxy_pass http://websocket_backend;

        # WebSocket headers
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Standard headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 86400s;  # 24 часа для long-lived connections

        # Buffering
        proxy_buffering off;
    }
}
```

### AWS Application Load Balancer

```yaml
# Target Group с sticky sessions
Resources:
  WebSocketTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Port: 3000
      Protocol: HTTP
      VpcId: !Ref VPC
      HealthCheckEnabled: true
      HealthCheckPath: /health
      HealthCheckProtocol: HTTP
      TargetType: ip
      # Sticky sessions
      TargetGroupAttributes:
        - Key: stickiness.enabled
          Value: true
        - Key: stickiness.type
          Value: lb_cookie
        - Key: stickiness.lb_cookie.duration_seconds
          Value: 86400  # 24 часа

  WebSocketLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Type: application
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref LoadBalancerSecurityGroup

  WebSocketListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref WebSocketLoadBalancer
      Port: 443
      Protocol: HTTPS
      Certificates:
        - CertificateArn: !Ref SSLCertificate
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref WebSocketTargetGroup
```

## Persistence Layer

### Redis для хранения соединений

```javascript
const Redis = require('ioredis');
const redis = new Redis();

// Сохранение информации о подключенных пользователях
io.on('connection', async (socket) => {
  const userId = socket.user.id;

  // Сохранить соединение в Redis
  await redis.hset(`user:${userId}:sockets`, socket.id, JSON.stringify({
    serverId: process.env.SERVER_ID,
    connectedAt: Date.now(),
    ip: socket.handshake.address,
  }));

  socket.on('disconnect', async () => {
    // Удалить соединение
    await redis.hdel(`user:${userId}:sockets`, socket.id);

    // Проверить, есть ли ещё соединения у пользователя
    const sockets = await redis.hgetall(`user:${userId}:sockets`);

    if (Object.keys(sockets).length === 0) {
      // Пользователь полностью отключился
      await redis.del(`user:${userId}:sockets`);
      await markUserOffline(userId);
    }
  });
});

// Проверка онлайн-статуса
async function isUserOnline(userId) {
  const exists = await redis.exists(`user:${userId}:sockets`);
  return exists === 1;
}

// Получить все онлайн пользователей
async function getOnlineUsers() {
  const keys = await redis.keys('user:*:sockets');
  return keys.map(key => key.split(':')[1]);
}
```

### Присутствие (Presence)

```javascript
// Отслеживание присутствия в комнатах
class PresenceManager {
  constructor(redis, io) {
    this.redis = redis;
    this.io = io;
  }

  async userJoinedRoom(userId, roomId, socketId) {
    const key = `room:${roomId}:users`;

    await this.redis.hset(key, userId, JSON.stringify({
      socketId,
      joinedAt: Date.now(),
    }));

    // Broadcast в комнату
    const usersCount = await this.redis.hlen(key);

    this.io.to(roomId).emit('room-presence', {
      roomId,
      usersCount,
      userId,
      action: 'joined',
    });
  }

  async userLeftRoom(userId, roomId) {
    const key = `room:${roomId}:users`;

    await this.redis.hdel(key, userId);

    const usersCount = await this.redis.hlen(key);

    this.io.to(roomId).emit('room-presence', {
      roomId,
      usersCount,
      userId,
      action: 'left',
    });
  }

  async getRoomUsers(roomId) {
    const key = `room:${roomId}:users`;
    const users = await this.redis.hgetall(key);

    return Object.entries(users).map(([userId, data]) => ({
      userId,
      ...JSON.parse(data),
    }));
  }

  async getUserRooms(userId) {
    const keys = await this.redis.keys('room:*:users');
    const rooms = [];

    for (const key of keys) {
      const exists = await this.redis.hexists(key, userId);
      if (exists) {
        const roomId = key.split(':')[1];
        rooms.push(roomId);
      }
    }

    return rooms;
  }
}

const presence = new PresenceManager(redis, io);

io.on('connection', (socket) => {
  socket.on('join-room', async ({ roomId }) => {
    socket.join(roomId);
    await presence.userJoinedRoom(socket.user.id, roomId, socket.id);
  });

  socket.on('leave-room', async ({ roomId }) => {
    socket.leave(roomId);
    await presence.userLeftRoom(socket.user.id, roomId);
  });
});
```

## Мониторинг и метрики

### Prometheus метрики

```javascript
const promClient = require('prom-client');

const register = new promClient.Registry();

// Метрики
const connectedClients = new promClient.Gauge({
  name: 'websocket_connected_clients',
  help: 'Number of connected WebSocket clients',
  registers: [register],
});

const messagesTotal = new promClient.Counter({
  name: 'websocket_messages_total',
  help: 'Total WebSocket messages',
  labelNames: ['direction', 'event'],
  registers: [register],
});

const messageDuration = new promClient.Histogram({
  name: 'websocket_message_duration_seconds',
  help: 'WebSocket message processing duration',
  labelNames: ['event'],
  registers: [register],
});

const roomsTotal = new promClient.Gauge({
  name: 'websocket_rooms_total',
  help: 'Total number of rooms',
  registers: [register],
});

// Обновление метрик
io.on('connection', (socket) => {
  connectedClients.inc();
  messagesTotal.inc({ direction: 'in', event: 'connection' });

  socket.onAny((event, ...args) => {
    messagesTotal.inc({ direction: 'in', event });

    const end = messageDuration.startTimer({ event });

    // Обработка события...

    end();
  });

  socket.on('disconnect', () => {
    connectedClients.dec();
    messagesTotal.inc({ direction: 'in', event: 'disconnect' });
  });
});

// Endpoint для Prometheus
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Периодическое обновление метрик комнат
setInterval(async () => {
  const rooms = await redis.keys('room:*:users');
  roomsTotal.set(rooms.length);
}, 10000);
```

### Structured logging

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple(),
  }));
}

io.on('connection', (socket) => {
  logger.info('WebSocket connection', {
    userId: socket.user.id,
    socketId: socket.id,
    ip: socket.handshake.address,
    userAgent: socket.handshake.headers['user-agent'],
  });

  socket.on('message', (data) => {
    logger.info('Message sent', {
      userId: socket.user.id,
      roomId: data.roomId,
      messageLength: data.content.length,
      timestamp: Date.now(),
    });
  });

  socket.on('error', (error) => {
    logger.error('Socket error', {
      userId: socket.user.id,
      socketId: socket.id,
      error: error.message,
      stack: error.stack,
    });
  });

  socket.on('disconnect', (reason) => {
    logger.info('WebSocket disconnect', {
      userId: socket.user.id,
      socketId: socket.id,
      reason,
      duration: Date.now() - socket.handshake.time,
    });
  });
});
```

## Security

### Аутентификация с JWT

```javascript
const jwt = require('jsonwebtoken');

const SECRET = process.env.JWT_SECRET;

function generateToken(user) {
  return jwt.sign(
    { userId: user.id, username: user.username },
    SECRET,
    { expiresIn: '24h' }
  );
}

async function verifyToken(token) {
  try {
    const decoded = jwt.verify(token, SECRET);
    return await getUserById(decoded.userId);
  } catch (err) {
    throw new Error('Invalid token');
  }
}

// Middleware
io.use(async (socket, next) => {
  const token = socket.handshake.auth.token ||
                socket.handshake.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    return next(new Error('Authentication required'));
  }

  try {
    socket.user = await verifyToken(token);
    next();
  } catch (err) {
    next(new Error('Invalid authentication'));
  }
});
```

### Authorization для комнат

```javascript
async function canAccessRoom(userId, roomId) {
  // Проверка прав доступа из БД
  const room = await db.rooms.findOne({ id: roomId });

  if (!room) {
    return false;
  }

  if (room.isPublic) {
    return true;
  }

  // Проверка членства
  const membership = await db.memberships.findOne({
    userId,
    roomId,
  });

  return membership !== null;
}

socket.on('join-room', async ({ roomId }) => {
  if (!await canAccessRoom(socket.user.id, roomId)) {
    socket.emit('error', { message: 'Access denied to this room' });
    return;
  }

  socket.join(roomId);
  socket.emit('room-joined', { roomId });
});
```

### Input validation

```javascript
const Joi = require('joi');

const messageSchema = Joi.object({
  roomId: Joi.string().uuid().required(),
  content: Joi.string().min(1).max(5000).required(),
  type: Joi.string().valid('text', 'image', 'file').default('text'),
});

socket.on('message', async (data) => {
  // Валидация
  const { error, value } = messageSchema.validate(data);

  if (error) {
    socket.emit('error', {
      message: 'Invalid message format',
      details: error.details,
    });
    return;
  }

  // Обработка валидированных данных
  await handleMessage(socket, value);
});
```

### Rate limiting per user

```javascript
const rateLimit = require('express-rate-limit');

class SocketRateLimiter {
  constructor(options = {}) {
    this.windowMs = options.windowMs || 60000;  // 1 минута
    this.max = options.max || 100;  // 100 сообщений
    this.storage = new Map();
  }

  check(userId) {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    if (!this.storage.has(userId)) {
      this.storage.set(userId, []);
    }

    const timestamps = this.storage.get(userId);

    // Очистка старых записей
    const recentTimestamps = timestamps.filter(ts => ts > windowStart);

    if (recentTimestamps.length >= this.max) {
      return {
        allowed: false,
        retryAfter: recentTimestamps[0] + this.windowMs - now,
      };
    }

    recentTimestamps.push(now);
    this.storage.set(userId, recentTimestamps);

    return {
      allowed: true,
      remaining: this.max - recentTimestamps.length,
    };
  }

  cleanup() {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    for (const [userId, timestamps] of this.storage.entries()) {
      const recentTimestamps = timestamps.filter(ts => ts > windowStart);

      if (recentTimestamps.length === 0) {
        this.storage.delete(userId);
      } else {
        this.storage.set(userId, recentTimestamps);
      }
    }
  }
}

const limiter = new SocketRateLimiter({ max: 50, windowMs: 60000 });

// Очистка каждую минуту
setInterval(() => limiter.cleanup(), 60000);

socket.on('message', async (data) => {
  const result = limiter.check(socket.user.id);

  if (!result.allowed) {
    socket.emit('error', {
      message: 'Rate limit exceeded',
      retryAfter: result.retryAfter,
    });
    return;
  }

  // Обработка сообщения
  await handleMessage(socket, data);
});
```

## Testing

### Unit тесты

```javascript
const { createServer } = require('http');
const { Server } = require('socket.io');
const Client = require('socket.io-client');

describe('WebSocket Server', () => {
  let io, serverSocket, clientSocket;

  beforeAll((done) => {
    const httpServer = createServer();
    io = new Server(httpServer);

    httpServer.listen(() => {
      const port = httpServer.address().port;
      clientSocket = new Client(`http://localhost:${port}`);

      io.on('connection', (socket) => {
        serverSocket = socket;
      });

      clientSocket.on('connect', done);
    });
  });

  afterAll(() => {
    io.close();
    clientSocket.close();
  });

  test('should send message to client', (done) => {
    clientSocket.on('hello', (arg) => {
      expect(arg).toBe('world');
      done();
    });

    serverSocket.emit('hello', 'world');
  });

  test('should receive message from client', (done) => {
    serverSocket.on('hello', (arg) => {
      expect(arg).toBe('world');
      done();
    });

    clientSocket.emit('hello', 'world');
  });

  test('should broadcast to room', (done) => {
    const client2 = new Client(`http://localhost:${httpServer.address().port}`);

    client2.on('connect', () => {
      clientSocket.emit('join-room', 'test-room');
      client2.emit('join-room', 'test-room');

      setTimeout(() => {
        let received = 0;

        const checkDone = () => {
          received++;
          if (received === 2) {
            client2.close();
            done();
          }
        };

        clientSocket.on('room-message', checkDone);
        client2.on('room-message', checkDone);

        serverSocket.to('test-room').emit('room-message', 'hello');
      }, 100);
    });
  });
});
```

### Load testing с Artillery

```yaml
# artillery-test.yml
config:
  target: "https://ws.example.com"
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 300
      arrivalRate: 100
      name: "Sustained load"
    - duration: 60
      arrivalRate: 200
      name: "Spike"
  socketio:
    transports: ["websocket"]

scenarios:
  - name: "Connect and send messages"
    engine: socketio
    flow:
      - emit:
          channel: "auth"
          data:
            token: "test-token-{{ $randomString() }}"
      - think: 2
      - emit:
          channel: "join-room"
          data:
            roomId: "room-{{ $randomNumber(1, 100) }}"
      - think: 1
      - loop:
          - emit:
              channel: "message"
              data:
                roomId: "room-{{ $randomNumber(1, 100) }}"
                content: "Test message {{ $randomString() }}"
          - think: 5
        count: 20
```

```bash
artillery run artillery-test.yml --output report.json
artillery report report.json
```

## Best Practices

### 1. Connection Management

```javascript
// Heartbeat для обнаружения мёртвых соединений
io.on('connection', (socket) => {
  socket.isAlive = true;

  socket.on('pong', () => {
    socket.isAlive = true;
  });
});

setInterval(() => {
  io.sockets.sockets.forEach((socket) => {
    if (socket.isAlive === false) {
      console.log(`Terminating dead socket: ${socket.id}`);
      return socket.terminate();
    }

    socket.isAlive = false;
    socket.ping();
  });
}, 30000);
```

### 2. Message Acknowledgment

```javascript
// Сервер: требовать подтверждения
socket.emit('important-message', data, (ack) => {
  if (ack === 'received') {
    console.log('Message acknowledged');
  } else {
    console.log('Message not acknowledged, retrying...');
  }
});

// Клиент: отправлять подтверждение
socket.on('important-message', (data, callback) => {
  processMessage(data);
  callback('received');
});
```

### 3. Binary Data

```javascript
// Отправка binary data (эффективнее JSON)
const buffer = Buffer.from([1, 2, 3, 4, 5]);
socket.emit('binary-data', buffer);

// Получение
socket.on('binary-data', (buffer) => {
  console.log('Received buffer:', buffer);
});
```

### 4. Namespace Separation

```javascript
// Разделение по namespace для разных функций
const chatNamespace = io.of('/chat');
const notificationNamespace = io.of('/notifications');

chatNamespace.on('connection', (socket) => {
  socket.on('message', handleChatMessage);
});

notificationNamespace.on('connection', (socket) => {
  socket.on('subscribe', handleNotificationSubscription);
});
```

## Выводы

Production-ready WebSocket система требует учёта множества факторов:

**Ключевые аспекты:**
- **Scalability**: Redis adapter для multi-server setup
- **Load balancing**: sticky sessions
- **Persistence**: хранение состояния в Redis
- **Security**: аутентификация, авторизация, rate limiting
- **Monitoring**: метрики, логи, алерты
- **Resilience**: reconnection, heartbeats, graceful shutdown
- **Testing**: unit и load тесты

Socket.IO предоставляет готовые решения для большинства проблем, что делает его отличным выбором для production.

## Что читать дальше?

- [Урок 25: Микросервисы: когда и зачем](25-microservices-when-why.md)
- [Урок 9: WebSockets, SSE, Long Polling](09-websockets-sse-long-polling.md)
- [Урок 12: Rate Limiting](12-rate-limiting-algoritmy.md)

## Проверь себя

1. Почему нужны sticky sessions для WebSocket?
2. Как Redis Adapter помогает масштабировать WebSocket-серверы?
3. Что такое presence tracking и как его реализовать?
4. Как обеспечить безопасность WebSocket-соединений?
5. Зачем нужен heartbeat механизм?
6. Как реализовать graceful shutdown?
7. Какие метрики важны для мониторинга WebSocket-системы?
8. Как тестировать WebSocket-приложения?
9. В чём разница между namespace и room в Socket.IO?
10. Как обрабатывать reconnection на клиенте?

---
**Предыдущий урок**: [Урок 23: Async Workers — Celery, BullMQ](23-async-workers.md)
**Следующий урок**: [Урок 25: Микросервисы — когда и зачем](25-microservices-when-why.md)
