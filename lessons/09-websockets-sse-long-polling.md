# WebSockets, Server-Sent Events, Long Polling

## Проблема: Real-time Communication

Традиционный HTTP — это request-response протокол. Клиент запрашивает, сервер отвечает. Но что если нужно:
- Чат в реальном времени
- Live обновления цен акций
- Уведомления
- Collaborative editing (Google Docs)
- Онлайн игры

**Проблема**: Сервер не может инициировать коммуникацию с клиентом.

**Решения**:
1. **Polling** (устаревший)
2. **Long Polling**
3. **Server-Sent Events (SSE)**
4. **WebSockets**

## Polling (Short Polling)

### Как работает

Клиент периодически спрашивает сервер: "Есть новые данные?"

```
Client: GET /api/messages?since=timestamp
Server: [] (нет новых)

[через 5 секунд]
Client: GET /api/messages?since=timestamp
Server: [] (нет новых)

[через 5 секунд]
Client: GET /api/messages?since=timestamp
Server: [{"text": "Hello!"}] (есть новые!)
```

### Пример кода

```javascript
// Client
function poll() {
  fetch('/api/messages?since=' + lastTimestamp)
    .then(res => res.json())
    .then(messages => {
      if (messages.length > 0) {
        updateUI(messages);
        lastTimestamp = messages[messages.length - 1].timestamp;
      }
    });
}

setInterval(poll, 5000); // каждые 5 секунд
```

### Преимущества

✅ Простота реализации
✅ Работает везде (обычный HTTP)
✅ Firewall-friendly

### Недостатки

❌ Высокая latency (до N секунд, где N — интервал)
❌ Много пустых запросов (waste bandwidth)
❌ Нагрузка на сервер (постоянные запросы)
❌ Не масштабируется

### Когда использовать

⚠️ Почти никогда в production. Используйте только если:
- Нет других опций
- Обновления очень редкие (раз в минуты/часы)
- Tiny scale

## Long Polling

### Как работает

Клиент делает запрос, сервер держит соединение открытым до появления данных.

```
Client: GET /api/messages (соединение остается открытым)
[сервер ждет...]
[через 30 секунд приходит сообщение]
Server: [{"text": "Hello!"}]

Client: GET /api/messages (сразу новый запрос)
[сервер снова ждет...]
```

### Пример кода

**Server (Node.js)**:
```javascript
app.get('/api/messages', async (req, res) => {
  const timeout = 30000; // 30 секунд
  const startTime = Date.now();

  while (Date.now() - startTime < timeout) {
    const messages = await db.getNewMessages(req.query.since);

    if (messages.length > 0) {
      return res.json(messages);
    }

    // Ждем 100ms перед следующей проверкой
    await sleep(100);
  }

  // Timeout — возвращаем пустой массив
  res.json([]);
});
```

**Client**:
```javascript
function longPoll() {
  fetch('/api/messages?since=' + lastTimestamp)
    .then(res => res.json())
    .then(messages => {
      if (messages.length > 0) {
        updateUI(messages);
        lastTimestamp = messages[messages.length - 1].timestamp;
      }

      // Сразу следующий long poll
      longPoll();
    })
    .catch(err => {
      console.error(err);
      setTimeout(longPoll, 5000); // retry через 5 сек
    });
}

longPoll();
```

### Оптимизация: Event-driven

Вместо busy-waiting используем события:

```javascript
const EventEmitter = require('events');
const messageEmitter = new EventEmitter();

app.get('/api/messages', (req, res) => {
  const timeout = setTimeout(() => {
    res.json([]);
  }, 30000);

  const handler = (message) => {
    clearTimeout(timeout);
    res.json([message]);
  };

  messageEmitter.once('new-message', handler);

  req.on('close', () => {
    clearTimeout(timeout);
    messageEmitter.off('new-message', handler);
  });
});

// Когда приходит новое сообщение
function onNewMessage(message) {
  messageEmitter.emit('new-message', message);
}
```

### Преимущества

✅ Меньше latency, чем short polling
✅ Меньше пустых запросов
✅ Работает через firewalls и proxies
✅ Fallback для WebSockets

### Недостатки

❌ Держит много соединений открытыми (memory/threads)
❌ Не настоящий bidirectional communication
❌ Overhead создания новых HTTP connections
❌ Сложность обработки timeouts и reconnections

### Когда использовать

✅ Fallback когда WebSockets недоступны
✅ Обновления не очень частые (каждые несколько секунд)
✅ Нужна совместимость со старыми системами

## Server-Sent Events (SSE)

### Как работает

**SSE** — стандарт для server-to-client streaming через обычный HTTP.

```
Client: GET /api/events
        Accept: text/event-stream

Server: (держит соединение открытым)
        Content-Type: text/event-stream

        data: {"message": "Hello"}

        data: {"message": "World"}

        data: {"message": "!"}
```

### EventSource API

**Client**:
```javascript
const eventSource = new EventSource('/api/events');

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
};

eventSource.onerror = (error) => {
  console.error('SSE error:', error);
};

// Именованные события
eventSource.addEventListener('user-joined', (event) => {
  const user = JSON.parse(event.data);
  console.log('User joined:', user.name);
});
```

### Server Implementation

**Node.js**:
```javascript
app.get('/api/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Отправка события
  const sendEvent = (data) => {
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  // Heartbeat каждые 30 секунд
  const heartbeat = setInterval(() => {
    res.write(': heartbeat\n\n');
  }, 30000);

  // Слушаем новые сообщения
  const handler = (message) => {
    sendEvent(message);
  };

  messageEmitter.on('new-message', handler);

  // Cleanup при закрытии соединения
  req.on('close', () => {
    clearInterval(heartbeat);
    messageEmitter.off('new-message', handler);
  });
});
```

**Именованные события**:
```javascript
// Server
res.write('event: user-joined\n');
res.write(`data: ${JSON.stringify({name: 'John'})}\n\n`);

res.write('event: user-left\n');
res.write(`data: ${JSON.stringify({name: 'Jane'})}\n\n`);
```

**Event ID (для восстановления)**:
```javascript
// Server
res.write(`id: ${eventId}\n`);
res.write(`data: ${JSON.stringify(message)}\n\n`);

// Client автоматически отправит Last-Event-ID при reconnect
// Server
app.get('/api/events', (req, res) => {
  const lastEventId = req.headers['last-event-id'];
  // Отправить пропущенные события
});
```

### Преимущества SSE

✅ Простой API (EventSource)
✅ Автоматический reconnect
✅ Event IDs для восстановления
✅ Работает через HTTP (firewall-friendly)
✅ Меньше overhead, чем WebSockets
✅ Можно использовать HTTP/2 multiplexing

### Недостатки SSE

❌ Только server→client (unidirectional)
❌ Только текстовые данные (нет binary)
❌ Ограничение на количество открытых соединений в браузере (6 per domain)
❌ Не поддерживается IE (только Edge+, Chrome, Firefox, Safari)

### Когда использовать SSE

✅ Нужны только server→client updates
✅ Текстовые данные (JSON)
✅ Хотите простоту реализации
✅ Live updates: новости, котировки, уведомления
✅ Нужен автоматический reconnect

**Примеры use cases**:
- Live dashboard (метрики, графики)
- Новостная лента
- Stock ticker
- Push уведомления
- Progress updates (загрузка файла)

## WebSockets

### Как работает

**WebSocket** — full-duplex bidirectional protocol поверх TCP.

**Handshake (HTTP Upgrade)**:
```http
Client → Server:
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server → Client:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

[После этого — WebSocket protocol, не HTTP]
```

### WebSocket API

**Client**:
```javascript
const ws = new WebSocket('ws://localhost:8080/chat');

ws.onopen = () => {
  console.log('Connected');
  ws.send(JSON.stringify({ type: 'join', room: 'general' }));
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log('Received:', message);
};

ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};

ws.onclose = (event) => {
  console.log('Disconnected:', event.code, event.reason);
};

// Отправка сообщения
ws.send(JSON.stringify({ type: 'message', text: 'Hello!' }));

// Закрытие соединения
ws.close();
```

### Server Implementation

**Node.js (ws library)**:
```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws, req) => {
  console.log('Client connected');

  ws.on('message', (data) => {
    const message = JSON.parse(data);
    console.log('Received:', message);

    // Отправка ответа
    ws.send(JSON.stringify({
      type: 'ack',
      timestamp: Date.now()
    }));

    // Broadcast всем клиентам
    wss.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify(message));
      }
    });
  });

  ws.on('close', () => {
    console.log('Client disconnected');
  });

  ws.on('error', (error) => {
    console.error('WebSocket error:', error);
  });

  // Heartbeat (ping/pong)
  ws.isAlive = true;
  ws.on('pong', () => {
    ws.isAlive = true;
  });
});

// Heartbeat interval
const heartbeat = setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) {
      return ws.terminate();
    }
    ws.isAlive = false;
    ws.ping();
  });
}, 30000);
```

**Express.js integration**:
```javascript
const express = require('express');
const http = require('http');
const WebSocket = require('ws');

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

wss.on('connection', (ws) => {
  // WebSocket logic
});

server.listen(8080);
```

### Socket.IO (библиотека поверх WebSockets)

**Features**:
- Автоматический fallback (WebSocket → long polling)
- Rooms и namespaces
- Acknowledgements
- Binary support
- Reconnection logic

**Server**:
```javascript
const io = require('socket.io')(server);

io.on('connection', (socket) => {
  console.log('User connected:', socket.id);

  // Слушать события
  socket.on('chat-message', (msg) => {
    console.log('Message:', msg);

    // Отправить всем в комнате
    io.to('room1').emit('chat-message', msg);
  });

  // Join room
  socket.on('join-room', (room) => {
    socket.join(room);
  });

  // Отправить конкретному client
  socket.emit('welcome', 'Hello!');

  // Broadcast всем кроме отправителя
  socket.broadcast.emit('user-joined', socket.id);

  socket.on('disconnect', () => {
    console.log('User disconnected:', socket.id);
  });
});
```

**Client**:
```javascript
const socket = io('http://localhost:8080');

socket.on('connect', () => {
  console.log('Connected:', socket.id);
  socket.emit('join-room', 'room1');
});

socket.on('chat-message', (msg) => {
  displayMessage(msg);
});

// Отправка с acknowledgement
socket.emit('chat-message', 'Hello!', (ack) => {
  console.log('Message delivered:', ack);
});
```

### Binary Data

WebSocket поддерживает binary frames:

```javascript
// Client
const blob = new Blob(['binary data']);
ws.send(blob);

const arrayBuffer = new ArrayBuffer(8);
ws.send(arrayBuffer);

// Server
ws.on('message', (data, isBinary) => {
  if (isBinary) {
    console.log('Binary data:', data);
  } else {
    console.log('Text data:', data.toString());
  }
});
```

### Scaling WebSockets

**Проблема**: WebSocket connections stateful и sticky к серверу.

#### 1. Sticky Sessions (Load Balancer)

```nginx
upstream websocket {
  ip_hash;  # Sticky sessions
  server server1:8080;
  server server2:8080;
  server server3:8080;
}

server {
  location /ws {
    proxy_pass http://websocket;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

#### 2. Redis Pub/Sub для broadcast

```javascript
const redis = require('redis');
const publisher = redis.createClient();
const subscriber = redis.createClient();

// Subscribe к каналу
subscriber.subscribe('chat-messages');

subscriber.on('message', (channel, message) => {
  // Broadcast всем локальным WebSocket clients
  wss.clients.forEach((client) => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(message);
    }
  });
});

// Когда получаем сообщение от client
ws.on('message', (data) => {
  // Публикуем в Redis
  publisher.publish('chat-messages', data);
});
```

#### 3. Dedicated WebSocket servers

```
[Users] → [Load Balancer]
            ↓
    [WS Server 1] ←→ [Redis Pub/Sub] ←→ [WS Server 2]
         ↓                                    ↓
    [API Server 1]                      [API Server 2]
```

### Преимущества WebSockets

✅ Full-duplex bidirectional communication
✅ Низкая latency (single TCP connection)
✅ Меньше overhead, чем HTTP polling
✅ Binary support
✅ Отлично для real-time

### Недостатки WebSockets

❌ Сложнее реализация и debugging
❌ Stateful (сложнее масштабировать)
❌ Некоторые proxies/firewalls блокируют
❌ Нет автоматического reconnect (нужна библиотека)
❌ Нет HTTP caching
❌ Требует отдельной инфраструктуры

### Когда использовать WebSockets

✅ Real-time chat
✅ Multiplayer games
✅ Collaborative editing
✅ Live trading platforms
✅ IoT dashboards
✅ Нужен bidirectional communication с низкой latency

## Сравнительная таблица

| Характеристика | Polling | Long Polling | SSE | WebSockets |
|----------------|---------|--------------|-----|------------|
| Latency | Высокая (секунды) | Средняя | Низкая | Очень низкая |
| Overhead | Очень высокий | Высокий | Средний | Низкий |
| Bidirectional | ❌ Нет | ❌ Нет | ❌ Нет | ✅ Да |
| Binary Support | ✅ Да (HTTP) | ✅ Да | ❌ Нет | ✅ Да |
| Browser Support | ✅ Все | ✅ Все | ⚠️ Не IE | ✅ Все modern |
| Автореконнект | Manual | Manual | ✅ Да | Manual (Socket.IO да) |
| Сложность | Низкая | Средняя | Низкая | Высокая |
| Scalability | ❌ Плохая | ⚠️ Средняя | ✅ Хорошая | ⚠️ Средняя |
| Use Case | Legacy | Fallback | Live updates | Real-time apps |

## Выбор технологии

### Decision Tree

```
Нужен bidirectional communication?
├─ Да → WebSockets (или Socket.IO)
└─ Нет
    ├─ Только server→client updates?
    │   ├─ Да → SSE
    │   └─ Нет → HTTP API
    └─ Обновления очень редкие?
        ├─ Да → Polling
        └─ Нет → Long Polling или SSE
```

### По Use Case

**Chat Application**:
- ✅ WebSockets (Socket.IO)
- ❌ SSE (только server→client)

**Live Dashboard** (графики, метрики):
- ✅ SSE (простота + автореконнект)
- ✅ WebSockets (если нужен control)

**Notifications**:
- ✅ SSE (perfect fit)
- ✅ Long Polling (fallback)

**Multiplayer Game**:
- ✅ WebSockets (bidirectional, low latency)
- Можно дополнить UDP для критичных данных

**Stock Ticker**:
- ✅ SSE (server→client streaming)

**Collaborative Editing** (Google Docs):
- ✅ WebSockets (bidirectional + OT/CRDT)

## Практические рекомендации

### Authentication

**WebSockets**:
```javascript
// Option 1: Token в query string
const ws = new WebSocket('ws://example.com?token=abc123');

// Option 2: Отправить auth после connect
ws.onopen = () => {
  ws.send(JSON.stringify({ type: 'auth', token: 'abc123' }));
};

// Server
wss.on('connection', (ws, req) => {
  const token = new URLSearchParams(req.url.split('?')[1]).get('token');
  if (!validateToken(token)) {
    ws.close(4001, 'Unauthorized');
  }
});
```

**SSE**:
```javascript
// Cookie автоматически отправляется
const eventSource = new EventSource('/api/events', {
  withCredentials: true
});

// Или custom header через fetch + SSE polyfill
```

### Heartbeat / Keep-Alive

Определить мертвые соединения:

```javascript
// Server
const HEARTBEAT_INTERVAL = 30000;
const HEARTBEAT_TIMEOUT = 35000;

wss.on('connection', (ws) => {
  ws.isAlive = true;

  ws.on('pong', () => {
    ws.isAlive = true;
  });
});

const interval = setInterval(() => {
  wss.clients.forEach((ws) => {
    if (ws.isAlive === false) {
      return ws.terminate();
    }

    ws.isAlive = false;
    ws.ping();
  });
}, HEARTBEAT_INTERVAL);
```

### Reconnection Logic

```javascript
class ReconnectingWebSocket {
  constructor(url, options = {}) {
    this.url = url;
    this.reconnectInterval = options.reconnectInterval || 1000;
    this.maxReconnectInterval = options.maxReconnectInterval || 30000;
    this.reconnectDecay = options.reconnectDecay || 1.5;
    this.currentReconnectInterval = this.reconnectInterval;

    this.connect();
  }

  connect() {
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log('Connected');
      this.currentReconnectInterval = this.reconnectInterval;
      if (this.onopen) this.onopen();
    };

    this.ws.onmessage = (event) => {
      if (this.onmessage) this.onmessage(event);
    };

    this.ws.onclose = () => {
      console.log('Disconnected, reconnecting...');
      setTimeout(() => {
        this.currentReconnectInterval = Math.min(
          this.currentReconnectInterval * this.reconnectDecay,
          this.maxReconnectInterval
        );
        this.connect();
      }, this.currentReconnectInterval);
    };
  }

  send(data) {
    if (this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(data);
    } else {
      console.error('WebSocket not connected');
    }
  }
}

// Usage
const ws = new ReconnectingWebSocket('ws://localhost:8080');
ws.onmessage = (event) => console.log(event.data);
ws.send('Hello!');
```

### Rate Limiting

```javascript
// Server-side
const rateLimit = new Map(); // userId -> {count, resetTime}

ws.on('message', (data) => {
  const userId = ws.userId;
  const now = Date.now();

  if (!rateLimit.has(userId)) {
    rateLimit.set(userId, { count: 0, resetTime: now + 60000 });
  }

  const limit = rateLimit.get(userId);

  if (now > limit.resetTime) {
    limit.count = 0;
    limit.resetTime = now + 60000;
  }

  if (limit.count >= 100) { // 100 messages per minute
    ws.send(JSON.stringify({ error: 'Rate limit exceeded' }));
    return;
  }

  limit.count++;

  // Process message
});
```

## Что почитать дальше

### Спецификации
- [RFC 6455 - WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
- [Server-Sent Events Specification](https://html.spec.whatwg.org/multipage/server-sent-events.html)

### Библиотеки
- [Socket.IO](https://socket.io/) — WebSockets с fallbacks
- [ws](https://github.com/websockets/ws) — WebSocket library для Node.js
- [SockJS](https://github.com/sockjs/sockjs-client) — WebSocket emulation

### Статьи
- "WebSockets vs Server-Sent Events" by Ably
- "Scaling WebSocket in Go and beyond" by Centrifugo

## Проверьте себя

1. В чем разница между polling и long polling?
2. Когда стоит использовать SSE вместо WebSockets?
3. Что такое WebSocket handshake и как он работает?
4. Как масштабировать WebSocket connections на несколько серверов?
5. Почему SSE поддерживает автоматический reconnect, а WebSocket нет?
6. Как реализовать heartbeat для определения мертвых соединений?
7. В чем преимущество Socket.IO перед нативными WebSockets?
8. Какие проблемы могут возникнуть с WebSockets и proxies?

## Ключевые выводы

- Polling устарел — используйте только если нет других опций
- Long Polling — хороший fallback для WebSockets
- SSE идеален для server→client updates: простота + автореконнект
- WebSockets — для full-duplex real-time communication
- Socket.IO упрощает WebSockets: fallbacks + rooms + reconnection
- WebSocket connections stateful — scaling требует sticky sessions или pub/sub
- Всегда реализуйте heartbeat для определения dead connections
- Добавляйте reconnection logic с exponential backoff
- Rate limiting критичен для WebSockets
- Выбор зависит от use case: bidirectional? binary? browser support?

---

**Предыдущий урок**: [API Design: REST, GraphQL, gRPC](08-api-design-rest-graphql-grpc.md)
**Следующий урок**: [Load Balancing: алгоритмы, L4 vs L7, sticky sessions](10-load-balancing-algoritmy-l4-vs-l7.md)
