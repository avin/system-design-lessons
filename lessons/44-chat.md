# Урок 44: Chat System

## Введение

Chat System (система мгновенных сообщений) — это real-time приложение, позволяющее пользователям обмениваться текстовыми сообщениями, медиа-файлами и voice/video звонками. Это одна из самых требовательных систем из-за необходимости ultra-low latency и high availability.

В этом уроке мы спроектируем масштабируемую систему чата как WhatsApp, Telegram или Slack, способную обслуживать миллиарды сообщений в день.

**Примеры:**
- **WhatsApp**: 100B messages/day
- **Telegram**: groups, channels, bots
- **Slack**: team communication
- **Discord**: communities + voice chat

## Требования к системе

### Функциональные требования

1. **1-on-1 Chat**:
   - Отправка/получение текстовых сообщений
   - Delivery receipts (доставлено, прочитано)
   - Typing indicators

2. **Group Chat**:
   - До 500 участников в группе
   - Админ-функции (kick, mute)

3. **Media Sharing**:
   - Фото, видео, файлы
   - Voice messages

4. **Presence**:
   - Online/offline status
   - Last seen timestamp

5. **History**:
   - Сохранение истории сообщений
   - Поиск по сообщениям

### Нефункциональные требования

1. **Latency**: < 100ms для доставки сообщения
2. **Availability**: 99.99%
3. **Scalability**: 1B users, 100B messages/day
4. **Consistency**: Strong ordering (сообщения в правильном порядке)
5. **Security**: End-to-end encryption (опционально)

### Back-of-the-envelope

```
Users: 1B total, 500M DAU
Messages per user per day: 50
Total messages: 500M × 50 = 25B messages/day ≈ 290K msg/sec

Message size: 100 bytes (text)
Storage: 25B × 100 bytes = 2.5 TB/day
1 year: 2.5 TB × 365 = 912 TB ≈ 1 PB

With media (20% have media, avg 1MB):
25B × 0.2 × 1MB = 5 PB/day

Connections:
500M DAU, average session 2 hours
Concurrent connections: 500M × (2/24) ≈ 42M concurrent WebSocket connections
```

## Архитектура высокого уровня

```
┌──────────┐          ┌──────────┐
│ Client A │          │ Client B │
└────┬─────┘          └─────┬────┘
     │ WebSocket            │ WebSocket
     ↓                      ↓
┌─────────────────────────────────┐
│      Load Balancer              │
│   (sticky sessions / stateful)  │
└──────────────┬──────────────────┘
               │
     ┌─────────┼─────────┐
     │         │         │
┌────▼────┐┌───▼────┐┌──▼──────┐
│  Chat   ││  Chat  ││  Chat   │
│ Server 1││Server 2││ Server 3│
└────┬────┘└───┬────┘└──┬──────┘
     │         │         │
     └─────────┼─────────┘
               ↓
       ┌───────────────┐
       │ Message Queue │
       │  (Kafka)      │
       └───────┬───────┘
               │
     ┌─────────┼─────────┐
     ↓         ↓         ↓
┌─────────┐┌──────┐┌──────────┐
│ Message ││Notif.││ Analytics│
│ Handler ││Worker││  Worker  │
└────┬────┘└──────┘└──────────┘
     │
     ↓
┌─────────────────────┐
│   Message Store     │
│ (Cassandra / Scylla)│
└─────────────────────┘
```

## Ключевые компоненты

### 1. WebSocket Connection Manager

```javascript
const WebSocket = require('ws');
const Redis = require('ioredis');
const redis = new Redis();

class ChatServer {
  constructor(port) {
    this.port = port;
    this.wss = new WebSocket.Server({ port });
    this.connections = new Map(); // userId → WebSocket
    this.redis = redis;

    this.setupWebSocket();
    this.setupPresence();
  }

  setupWebSocket() {
    this.wss.on('connection', (ws, req) => {
      console.log('New WebSocket connection');

      // Аутентификация
      ws.on('message', async (data) => {
        try {
          const message = JSON.parse(data);

          if (message.type === 'auth') {
            await this.handleAuth(ws, message);
          } else if (ws.userId) {
            await this.handleMessage(ws, message);
          } else {
            ws.send(JSON.stringify({ error: 'Not authenticated' }));
          }
        } catch (error) {
          console.error('Message handling error:', error);
          ws.send(JSON.stringify({ error: error.message }));
        }
      });

      ws.on('close', () => {
        if (ws.userId) {
          console.log(`User ${ws.userId} disconnected`);
          this.handleDisconnect(ws.userId);
        }
      });

      ws.on('error', (error) => {
        console.error('WebSocket error:', error);
      });
    });

    console.log(`Chat server listening on port ${this.port}`);
  }

  async handleAuth(ws, message) {
    const { token } = message;

    // Проверяем JWT token
    const userId = await this.verifyToken(token);

    if (!userId) {
      ws.send(JSON.stringify({ type: 'auth', success: false }));
      ws.close();
      return;
    }

    // Сохраняем соединение
    ws.userId = userId;
    this.connections.set(userId, ws);

    // Регистрируем в Redis (для distributed setup)
    await this.redis.setex(`presence:${userId}`, 300, this.port.toString());

    // Подтверждение
    ws.send(JSON.stringify({ type: 'auth', success: true, userId }));

    // Отправляем pending messages
    await this.deliverPendingMessages(userId);

    // Обновляем presence
    await this.updatePresence(userId, 'online');
  }

  async handleMessage(ws, message) {
    const { type, ...data } = message;

    switch (type) {
      case 'chat_message':
        await this.handleChatMessage(ws.userId, data);
        break;

      case 'read_receipt':
        await this.handleReadReceipt(ws.userId, data);
        break;

      case 'typing':
        await this.handleTypingIndicator(ws.userId, data);
        break;

      default:
        ws.send(JSON.stringify({ error: 'Unknown message type' }));
    }
  }

  async handleChatMessage(senderId, data) {
    const { recipientId, content, mediaUrl, replyTo } = data;

    const messageId = this.generateMessageId();
    const timestamp = Date.now();

    // Создаём message object
    const message = {
      id: messageId,
      senderId,
      recipientId,
      content,
      mediaUrl,
      replyTo,
      timestamp,
      status: 'sent'
    };

    // Сохраняем в БД (асинхронно через Kafka)
    await this.publishToKafka('messages', message);

    // Пытаемся доставить сразу (если получатель online)
    const delivered = await this.deliverMessage(recipientId, message);

    if (delivered) {
      message.status = 'delivered';

      // Уведомляем отправителя о доставке
      this.sendToUser(senderId, {
        type: 'delivery_receipt',
        messageId,
        status: 'delivered',
        timestamp: Date.now()
      });
    } else {
      // Получатель offline — сохраняем для доставки позже
      await this.redis.lpush(`pending:${recipientId}`, JSON.stringify(message));
    }

    // Отправляем ACK отправителю
    this.sendToUser(senderId, {
      type: 'message_ack',
      messageId,
      timestamp
    });
  }

  async deliverMessage(userId, message) {
    // Проверяем: есть ли активное соединение
    const ws = this.connections.get(userId);

    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({
        type: 'chat_message',
        ...message
      }));

      return true;
    }

    // Проверяем в Redis (может на другом сервере)
    const serverPort = await this.redis.get(`presence:${userId}`);

    if (serverPort && serverPort !== this.port.toString()) {
      // Пользователь на другом сервере — отправляем через Redis Pub/Sub
      await this.redis.publish(`chat:${userId}`, JSON.stringify(message));

      return true;
    }

    return false; // offline
  }

  async deliverPendingMessages(userId) {
    // Получаем pending messages из Redis
    const pending = await this.redis.lrange(`pending:${userId}`, 0, -1);

    if (pending.length === 0) return;

    console.log(`Delivering ${pending.length} pending messages to ${userId}`);

    for (const msgJson of pending) {
      const message = JSON.parse(msgJson);
      await this.deliverMessage(userId, message);
    }

    // Очищаем pending queue
    await this.redis.del(`pending:${userId}`);
  }

  async handleReadReceipt(userId, data) {
    const { messageId, senderId } = data;

    // Обновляем статус в БД
    await this.publishToKafka('receipts', {
      messageId,
      userId,
      status: 'read',
      timestamp: Date.now()
    });

    // Уведомляем отправителя
    this.sendToUser(senderId, {
      type: 'read_receipt',
      messageId,
      readBy: userId,
      timestamp: Date.now()
    });
  }

  async handleTypingIndicator(userId, data) {
    const { recipientId, isTyping } = data;

    // Отправляем индикатор получателю
    this.sendToUser(recipientId, {
      type: 'typing',
      userId,
      isTyping
    });
  }

  sendToUser(userId, data) {
    const ws = this.connections.get(userId);

    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(data));
    }
  }

  async handleDisconnect(userId) {
    this.connections.delete(userId);
    await this.redis.del(`presence:${userId}`);
    await this.updatePresence(userId, 'offline');
  }

  async updatePresence(userId, status) {
    await this.redis.setex(`user_presence:${userId}`, 3600, status);

    // Уведомляем друзей о изменении статуса
    const friends = await this.getFriends(userId);

    for (const friendId of friends) {
      this.sendToUser(friendId, {
        type: 'presence',
        userId,
        status,
        lastSeen: status === 'offline' ? Date.now() : null
      });
    }
  }

  async verifyToken(token) {
    // JWT verification
    const jwt = require('jsonwebtoken');

    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      return decoded.userId;
    } catch (error) {
      return null;
    }
  }

  async publishToKafka(topic, message) {
    const kafka = require('./kafka');
    await kafka.send({
      topic,
      messages: [{ value: JSON.stringify(message) }]
    });
  }

  async getFriends(userId) {
    // Получаем список друзей из БД
    const result = await db.query(
      'SELECT friend_id FROM friendships WHERE user_id = $1',
      [userId]
    );

    return result.rows.map(r => r.friend_id);
  }

  setupPresence() {
    // Heartbeat: каждые 30 секунд обновляем presence в Redis
    setInterval(async () => {
      const pipeline = this.redis.pipeline();

      for (const [userId, ws] of this.connections) {
        if (ws.readyState === WebSocket.OPEN) {
          pipeline.setex(`presence:${userId}`, 300, this.port.toString());
        } else {
          this.connections.delete(userId);
        }
      }

      await pipeline.exec();
    }, 30000);
  }

  generateMessageId() {
    // Используем Snowflake ID generator
    const Snowflake = require('./snowflake');
    return new Snowflake(1, 1).generate().toString();
  }
}

// Запуск
const server = new ChatServer(8080);
```

### 2. Message Handler Worker

```javascript
const { Kafka } = require('kafkajs');

class MessageHandler {
  constructor() {
    this.kafka = new Kafka({
      clientId: 'message-handler',
      brokers: ['kafka:9092']
    });

    this.consumer = this.kafka.consumer({ groupId: 'message-handlers' });
    this.db = require('./cassandra'); // Cassandra client
  }

  async start() {
    await this.consumer.connect();
    await this.consumer.subscribe({ topic: 'messages' });

    console.log('Message handler started');

    await this.consumer.run({
      eachMessage: async ({ message }) => {
        try {
          const msg = JSON.parse(message.value.toString());
          await this.handleMessage(msg);
        } catch (error) {
          console.error('Message handling error:', error);
        }
      }
    });
  }

  async handleMessage(message) {
    const { id, senderId, recipientId, content, mediaUrl, timestamp } = message;

    // Сохраняем в Cassandra
    await this.db.execute(
      `INSERT INTO messages
       (id, sender_id, recipient_id, content, media_url, timestamp, status)
       VALUES (?, ?, ?, ?, ?, ?, ?)`,
      [id, senderId, recipientId, content, mediaUrl, timestamp, 'delivered'],
      { prepare: true }
    );

    // Для группового чата: сохраняем также в chat_messages таблицу
    // (denormalized для быстрого получения истории)

    console.log(`Message ${id} saved to database`);
  }
}

const handler = new MessageHandler();
handler.start();
```

### 3. Group Chat

```javascript
class GroupChatService {
  async sendGroupMessage(senderId, groupId, content) {
    const messageId = this.generateMessageId();
    const timestamp = Date.now();

    // Получаем участников группы
    const members = await this.getGroupMembers(groupId);

    // Создаём message
    const message = {
      id: messageId,
      groupId,
      senderId,
      content,
      timestamp
    };

    // Сохраняем в БД (одна запись для группы)
    await db.execute(
      `INSERT INTO group_messages
       (id, group_id, sender_id, content, timestamp)
       VALUES (?, ?, ?, ?, ?)`,
      [messageId, groupId, senderId, content, timestamp]
    );

    // Рассылаем всем участникам (кроме отправителя)
    const deliveryPromises = members
      .filter(memberId => memberId !== senderId)
      .map(memberId => this.deliverMessage(memberId, {
        type: 'group_message',
        ...message
      }));

    await Promise.allSettled(deliveryPromises);

    return messageId;
  }

  async getGroupMembers(groupId) {
    const result = await db.query(
      'SELECT user_id FROM group_members WHERE group_id = $1',
      [groupId]
    );

    return result.rows.map(r => r.user_id);
  }

  async createGroup(creatorId, name, memberIds) {
    const groupId = this.generateGroupId();

    // Создаём группу
    await db.query(
      `INSERT INTO groups (id, name, creator_id, created_at)
       VALUES ($1, $2, $3, NOW())`,
      [groupId, name, creatorId]
    );

    // Добавляем участников (включая создателя)
    const allMembers = [creatorId, ...memberIds];

    const pipeline = db.pipeline();

    for (const memberId of allMembers) {
      pipeline.query(
        `INSERT INTO group_members (group_id, user_id, role, joined_at)
         VALUES ($1, $2, $3, NOW())`,
        [groupId, memberId, memberId === creatorId ? 'admin' : 'member']
      );
    }

    await pipeline.exec();

    return groupId;
  }
}
```

### 4. Message Search

```javascript
const { Client } = require('@elastic/elasticsearch');
const esClient = new Client({ node: 'http://elasticsearch:9200' });

class MessageSearchService {
  async indexMessage(message) {
    // Индексируем сообщение в Elasticsearch
    await esClient.index({
      index: 'messages',
      id: message.id,
      document: {
        senderId: message.senderId,
        recipientId: message.recipientId,
        content: message.content,
        timestamp: message.timestamp
      }
    });
  }

  async search(userId, query, limit = 20) {
    const result = await esClient.search({
      index: 'messages',
      body: {
        query: {
          bool: {
            must: [
              {
                multi_match: {
                  query,
                  fields: ['content'],
                  fuzziness: 'AUTO'
                }
              },
              {
                bool: {
                  should: [
                    { term: { senderId: userId } },
                    { term: { recipientId: userId } }
                  ]
                }
              }
            ]
          }
        },
        sort: [{ timestamp: 'desc' }],
        size: limit
      }
    });

    return result.hits.hits.map(hit => hit._source);
  }
}
```

## Schema Design

### Cassandra Schema

```sql
-- Messages table (1-on-1)
CREATE TABLE messages (
  id TEXT,
  sender_id TEXT,
  recipient_id TEXT,
  content TEXT,
  media_url TEXT,
  timestamp BIGINT,
  status TEXT,
  PRIMARY KEY ((sender_id, recipient_id), timestamp, id)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- Reverse index для получения сообщений как получателю
CREATE TABLE messages_by_recipient (
  id TEXT,
  sender_id TEXT,
  recipient_id TEXT,
  content TEXT,
  media_url TEXT,
  timestamp BIGINT,
  status TEXT,
  PRIMARY KEY ((recipient_id, sender_id), timestamp, id)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- Group messages
CREATE TABLE group_messages (
  id TEXT,
  group_id TEXT,
  sender_id TEXT,
  content TEXT,
  timestamp BIGINT,
  PRIMARY KEY (group_id, timestamp, id)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- Group members
CREATE TABLE group_members (
  group_id TEXT,
  user_id TEXT,
  role TEXT,
  joined_at TIMESTAMP,
  PRIMARY KEY (group_id, user_id)
);

-- Read receipts
CREATE TABLE read_receipts (
  message_id TEXT,
  user_id TEXT,
  read_at BIGINT,
  PRIMARY KEY (message_id, user_id)
);
```

## Оптимизации

### 1. Message Pagination

```javascript
async getMessageHistory(userId, otherUserId, cursor, limit = 50) {
  // cursor = timestamp последнего сообщения
  const query = cursor
    ? `SELECT * FROM messages
       WHERE sender_id = ? AND recipient_id = ?
         AND timestamp < ?
       ORDER BY timestamp DESC
       LIMIT ?`
    : `SELECT * FROM messages
       WHERE sender_id = ? AND recipient_id = ?
       ORDER BY timestamp DESC
       LIMIT ?`;

  const params = cursor
    ? [userId, otherUserId, cursor, limit]
    : [userId, otherUserId, limit];

  const result = await db.execute(query, params, { prepare: true });

  const messages = result.rows;

  const nextCursor = messages.length > 0
    ? messages[messages.length - 1].timestamp
    : null;

  return { messages, nextCursor };
}
```

### 2. Media Upload

```javascript
const AWS = require('aws-sdk');
const s3 = new AWS.S3();

class MediaService {
  async uploadMedia(file, userId) {
    const fileId = this.generateFileId();
    const key = `media/${userId}/${fileId}`;

    // Загружаем в S3
    await s3.putObject({
      Bucket: process.env.S3_BUCKET,
      Key: key,
      Body: file.buffer,
      ContentType: file.mimetype,
      ACL: 'private'
    }).promise();

    // Генерируем signed URL (для доступа)
    const url = s3.getSignedUrl('getObject', {
      Bucket: process.env.S3_BUCKET,
      Key: key,
      Expires: 3600 // 1 hour
    });

    return { fileId, url };
  }

  async getMediaUrl(fileId, userId) {
    const key = `media/${userId}/${fileId}`;

    // Генерируем новый signed URL
    const url = s3.getSignedUrl('getObject', {
      Bucket: process.env.S3_BUCKET,
      Key: key,
      Expires: 3600
    });

    return url;
  }
}
```

### 3. Connection Recovery

```javascript
class ConnectionRecovery {
  constructor(ws) {
    this.ws = ws;
    this.lastMessageId = null;
  }

  onReconnect(lastReceivedMessageId) {
    // При переподключении: отправляем пропущенные сообщения
    this.fetchMissedMessages(lastReceivedMessageId).then(messages => {
      messages.forEach(msg => {
        this.ws.send(JSON.stringify(msg));
      });
    });
  }

  async fetchMissedMessages(sinceMessageId) {
    // Получаем сообщения после sinceMessageId
    const result = await db.query(
      `SELECT * FROM messages
       WHERE recipient_id = ?
         AND id > ?
       ORDER BY timestamp ASC`,
      [this.ws.userId, sinceMessageId]
    );

    return result.rows;
  }
}
```

### 4. Rate Limiting

```javascript
class ChatRateLimiter {
  async checkLimit(userId) {
    const key = `rate_limit:chat:${userId}`;
    const limit = 100; // 100 messages per minute
    const window = 60;

    const current = await redis.incr(key);

    if (current === 1) {
      await redis.expire(key, window);
    }

    if (current > limit) {
      throw new Error('Rate limit exceeded');
    }

    return { allowed: true, remaining: limit - current };
  }
}
```

## End-to-End Encryption (опционально)

```javascript
// Client-side encryption
const crypto = require('crypto');

class E2EEncryption {
  // Генерация ключей (на клиенте)
  generateKeyPair() {
    const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
    });

    return { publicKey, privateKey };
  }

  // Шифрование сообщения (на клиенте отправителя)
  encryptMessage(content, recipientPublicKey) {
    const encrypted = crypto.publicEncrypt(
      recipientPublicKey,
      Buffer.from(content)
    );

    return encrypted.toString('base64');
  }

  // Расшифровка (на клиенте получателя)
  decryptMessage(encryptedContent, privateKey) {
    const buffer = Buffer.from(encryptedContent, 'base64');

    const decrypted = crypto.privateDecrypt(privateKey, buffer);

    return decrypted.toString('utf-8');
  }
}

// Сервер НЕ имеет доступа к приватным ключам
// Хранит только зашифрованные сообщения и публичные ключи
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const activeConnections = new prometheus.Gauge({
  name: 'chat_active_connections',
  help: 'Number of active WebSocket connections'
});

const messagesSent = new prometheus.Counter({
  name: 'chat_messages_sent_total',
  help: 'Total messages sent',
  labelNames: ['type'] // 1on1, group
});

const messageLatency = new prometheus.Histogram({
  name: 'chat_message_delivery_seconds',
  help: 'Time from send to delivery',
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2]
});

// Обновление метрик
setInterval(() => {
  activeConnections.set(this.connections.size);
}, 10000);
```

## Масштабирование

### 1. Horizontal Scaling с Pub/Sub

```javascript
// Chat Server подписывается на Redis channels
class ScalableChatServer extends ChatServer {
  constructor(port) {
    super(port);

    this.subscriber = new Redis();
    this.setupPubSub();
  }

  setupPubSub() {
    // Подписываемся на messages для наших пользователей
    this.subscriber.psubscribe('chat:*');

    this.subscriber.on('pmessage', (pattern, channel, message) => {
      const userId = channel.split(':')[1];
      const msg = JSON.parse(message);

      this.deliverMessage(userId, msg);
    });
  }

  async deliverMessage(userId, message) {
    const ws = this.connections.get(userId);

    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(message));
      return true;
    }

    // Если у нас нет этого пользователя — игнорируем
    // (он на другом сервере и получит через свой Pub/Sub)
    return false;
  }
}
```

### 2. Service Discovery

```javascript
const consul = require('consul')();

class ChatServerRegistry {
  async register(serverId, port) {
    await consul.agent.service.register({
      id: serverId,
      name: 'chat-server',
      address: process.env.HOST,
      port,
      check: {
        http: `http://${process.env.HOST}:${port}/health`,
        interval: '10s'
      }
    });

    console.log(`Registered chat server ${serverId}`);
  }

  async discover() {
    const result = await consul.health.service('chat-server');

    return result.map(service => ({
      id: service.Service.ID,
      host: service.Service.Address,
      port: service.Service.Port
    }));
  }
}
```

## Trade-offs

| Аспект | WebSocket | Long Polling | SSE |
|--------|-----------|--------------|-----|
| Latency | Очень низкая (< 50ms) | Высокая (1-30s) | Средняя (< 1s) |
| Overhead | Низкий | Высокий | Средний |
| Bidirectional | Да | Нет (нужен separate upload) | Нет |
| Browser Support | 99%+ | 100% | 95%+ |
| Рекомендация | **Production** | Fallback | Server → Client only |

## Что читать дальше

- **Урок 24**: WebSockets in Production — детали WebSocket инфраструктуры
- **Урок 45**: Search System — полнотекстовый поиск по сообщениям
- **Урок 49**: Video Streaming — для voice/video calls

## Проверь себя

1. Почему WebSocket лучше чем HTTP polling для чата?
2. Как обеспечить доставку сообщений при offline получателя?
3. Спроектируйте group chat для 10,000 участников.
4. Сколько памяти нужно для хранения 42M concurrent WebSocket connections?
5. Как реализовать message reactions (emoji)?

---

[← Урок 43: Newsfeed System](43-newsfeed.md) | [Урок 45: Search System →](45-search.md)
