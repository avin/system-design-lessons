# Урок 48: Ticketing System (Система продажи билетов)

## Введение

Ticketing System — это высоконагруженная система для продажи билетов на концерты, спортивные события, кино. Ключевая сложность — обработка огромных всплесков трафика при открытии продаж популярных событий и предотвращение overbooking (продажа большего количества билетов, чем есть мест).

В этом уроке мы спроектируем систему уровня Ticketmaster, способную обрабатывать миллионы одновременных запросов с гарантией консистентности.

**Примеры:**
- **Ticketmaster**: концерты, спорт
- **Fandango**: кинотеатры
- **Eventbrite**: события
- **StubHub**: перепродажа билетов

## Требования к системе

### Функциональные требования

1. **Event Management**:
   - Создание событий (название, дата, venue)
   - Управление местами (seat map, pricing tiers)

2. **Ticket Booking**:
   - Поиск событий
   - Выбор мест
   - Бронирование (hold) на N минут
   - Оплата и подтверждение

3. **Ticket Management**:
   - Просмотр купленных билетов
   - QR-код для входа
   - Отмена/возврат (refund)

4. **Предотвращение Fraud**:
   - Защита от bots
   - Rate limiting
   - CAPTCHA

### Нефункциональные требования

1. **High Availability**: 99.99%
2. **Consistency**:
   - **Strong consistency** для бронирования (no overbooking)
   - Eventual consistency для поиска OK
3. **Scalability**:
   - Peak load: 1M concurrent users
   - 100K bookings/sec
4. **Latency**:
   - Search: < 500ms
   - Booking: < 2 sec
5. **Fairness**: First-come-first-served (с некоторыми исключениями)

### Back-of-the-envelope

```
Events: 100K active events
Seats per event: average 10K
Total seats: 1B seats

Popular event (stadium 50K seats):
- Sale opens at 10:00 AM
- Sold out in 5 minutes
- Peak traffic: 1M users trying to buy

Bookings/sec during peak: 50K / 300 = 167 bookings/sec
Requests/sec: 1M users × 10 req/user / 300 sec = 33K req/sec

Storage:
- Event metadata: 1KB × 100K = 100 MB
- Seat inventory: 50 bytes × 1B = 50 GB
- Bookings: 500 bytes × 100M bookings/year = 50 GB/year
```

## Ключевые вызовы

1. **Race Condition**: 2 пользователя пытаются купить последний билет
2. **Thundering Herd**: 1M пользователей одновременно при открытии продаж
3. **Bots**: автоматические скупщики билетов
4. **Overbooking**: продажа больше билетов, чем есть мест

## Архитектура высокого уровня

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     ↓
┌─────────────────┐
│  Load Balancer  │
│   + WAF         │
└────┬────────────┘
     │
     ↓
┌─────────────────┐
│  API Gateway    │
│ (Rate Limiting) │
└────┬────────────┘
     │
     ├─────────┬─────────┬─────────┐
     ↓         ↓         ↓         ↓
┌────────┐┌────────┐┌────────┐┌────────┐
│ Event  ││ Booking││Payment ││ Queue  │
│Service ││Service ││Service ││Service │
└───┬────┘└───┬────┘└───┬────┘└───┬────┘
    │         │         │         │
    └─────────┼─────────┴─────────┘
              │
    ┌─────────┼─────────┐
    ↓         ↓         ↓
┌────────┐┌────────┐┌────────┐
│ Events ││ Seats  ││Bookings│
│   DB   ││  DB    ││   DB   │
└────────┘└────────┘└────────┘
              │
              ↓
       ┌─────────────┐
       │    Redis    │
       │ (Inventory  │
       │   Cache)    │
       └─────────────┘
```

## Ключевые компоненты

### 1. Event Service

```javascript
const express = require('express');

class EventService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Create event
    app.post('/events', async (req, res) => {
      try {
        const { name, date, venueId, description, seatMap } = req.body;

        const eventId = await this.createEvent({
          name,
          date,
          venueId,
          description,
          seatMap
        });

        res.json({ success: true, eventId });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Search events
    app.get('/events/search', async (req, res) => {
      try {
        const { query, city, date, category, page = 1, limit = 20 } = req.query;

        const events = await this.searchEvents({
          query,
          city,
          date,
          category,
          page: parseInt(page),
          limit: parseInt(limit)
        });

        res.json(events);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Get event details
    app.get('/events/:eventId', async (req, res) => {
      try {
        const { eventId } = req.params;

        const event = await this.getEvent(eventId);

        res.json(event);
      } catch (error) {
        res.status(404).json({ error: 'Event not found' });
      }
    });

    // Get available seats
    app.get('/events/:eventId/seats', async (req, res) => {
      try {
        const { eventId } = req.params;

        const seats = await this.getAvailableSeats(eventId);

        res.json({ seats });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3001, () => {
      console.log('Event Service listening on port 3001');
    });
  }

  async createEvent({ name, date, venueId, description, seatMap }) {
    const eventId = this.generateEventId();

    await this.db.query('BEGIN');

    try {
      // Создаём event
      await this.db.query(
        `INSERT INTO events
         (id, name, date, venue_id, description, status, created_at)
         VALUES ($1, $2, $3, $4, $5, 'upcoming', NOW())`,
        [eventId, name, date, venueId, description]
      );

      // Создаём seats
      await this.createSeats(eventId, seatMap);

      await this.db.query('COMMIT');

      return eventId;
    } catch (error) {
      await this.db.query('ROLLBACK');
      throw error;
    }
  }

  async createSeats(eventId, seatMap) {
    // seatMap = { sections: [{ name, rows: [{ number, seats: [{number, price}] }] }] }
    const seatInserts = [];

    for (const section of seatMap.sections) {
      for (const row of section.rows) {
        for (const seat of row.seats) {
          seatInserts.push([
            this.generateSeatId(),
            eventId,
            section.name,
            row.number,
            seat.number,
            seat.price,
            'available'
          ]);
        }
      }
    }

    // Batch insert
    const values = seatInserts.map((_, i) =>
      `($${i * 7 + 1}, $${i * 7 + 2}, $${i * 7 + 3}, $${i * 7 + 4}, $${i * 7 + 5}, $${i * 7 + 6}, $${i * 7 + 7})`
    ).join(',');

    await this.db.query(
      `INSERT INTO seats
       (id, event_id, section, row, number, price, status)
       VALUES ${values}`,
      seatInserts.flat()
    );

    // Кэшируем inventory в Redis
    await this.cacheInventory(eventId, seatInserts.length);
  }

  async cacheInventory(eventId, totalSeats) {
    await this.redis.hset(`event:${eventId}`, {
      totalSeats,
      availableSeats: totalSeats
    });
  }

  async searchEvents({ query, city, date, category, page, limit }) {
    let sql = `
      SELECT e.id, e.name, e.date, v.name as venue_name, v.city,
             (SELECT COUNT(*) FROM seats WHERE event_id = e.id AND status = 'available') as available_seats
      FROM events e
      JOIN venues v ON e.venue_id = v.id
      WHERE e.status = 'upcoming'
    `;

    const params = [];
    let paramIndex = 1;

    if (query) {
      sql += ` AND e.name ILIKE $${paramIndex}`;
      params.push(`%${query}%`);
      paramIndex++;
    }

    if (city) {
      sql += ` AND v.city = $${paramIndex}`;
      params.push(city);
      paramIndex++;
    }

    if (date) {
      sql += ` AND DATE(e.date) = $${paramIndex}`;
      params.push(date);
      paramIndex++;
    }

    if (category) {
      sql += ` AND e.category = $${paramIndex}`;
      params.push(category);
      paramIndex++;
    }

    sql += ` ORDER BY e.date ASC LIMIT $${paramIndex} OFFSET $${paramIndex + 1}`;
    params.push(limit, (page - 1) * limit);

    const result = await this.db.query(sql, params);

    return {
      events: result.rows,
      page,
      limit,
      total: result.rowCount
    };
  }

  async getEvent(eventId) {
    const result = await this.db.query(
      `SELECT e.*, v.name as venue_name, v.address, v.city
       FROM events e
       JOIN venues v ON e.venue_id = v.id
       WHERE e.id = $1`,
      [eventId]
    );

    if (result.rows.length === 0) {
      throw new Error('Event not found');
    }

    return result.rows[0];
  }

  async getAvailableSeats(eventId) {
    // Проверяем cache
    const cached = await this.redis.get(`seats:${eventId}`);

    if (cached) {
      return JSON.parse(cached);
    }

    // Загружаем из БД
    const result = await this.db.query(
      `SELECT id, section, row, number, price, status
       FROM seats
       WHERE event_id = $1
       ORDER BY section, row, number`,
      [eventId]
    );

    const seats = result.rows;

    // Кэшируем (TTL 30 sec)
    await this.redis.setex(`seats:${eventId}`, 30, JSON.stringify(seats));

    return seats;
  }

  generateEventId() {
    const Snowflake = require('./snowflake');
    return new Snowflake(1, 1).generate().toString();
  }

  generateSeatId() {
    return require('crypto').randomUUID();
  }
}

const service = new EventService();
```

### 2. Booking Service (Критический компонент)

```javascript
class BookingService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Hold seats (temporary reservation)
    app.post('/bookings/hold', async (req, res) => {
      try {
        const { userId, eventId, seatIds } = req.body;

        const holdId = await this.holdSeats(userId, eventId, seatIds);

        res.json({
          success: true,
          holdId,
          expiresAt: Date.now() + 10 * 60 * 1000 // 10 minutes
        });
      } catch (error) {
        res.status(400).json({ error: error.message });
      }
    });

    // Confirm booking (after payment)
    app.post('/bookings/confirm', async (req, res) => {
      try {
        const { holdId, paymentId } = req.body;

        const bookingId = await this.confirmBooking(holdId, paymentId);

        res.json({ success: true, bookingId });
      } catch (error) {
        res.status(400).json({ error: error.message });
      }
    });

    // Cancel hold (release seats)
    app.post('/bookings/cancel-hold', async (req, res) => {
      try {
        const { holdId } = req.body;

        await this.cancelHold(holdId);

        res.json({ success: true });
      } catch (error) {
        res.status(400).json({ error: error.message });
      }
    });

    // Get user bookings
    app.get('/bookings/user/:userId', async (req, res) => {
      try {
        const { userId } = req.params;

        const bookings = await this.getUserBookings(userId);

        res.json({ bookings });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3002, () => {
      console.log('Booking Service listening on port 3002');
    });
  }

  async holdSeats(userId, eventId, seatIds) {
    // Критичная секция: нужна атомарность
    const holdId = this.generateHoldId();

    // Используем Redis для атомарного захвата мест
    const success = await this.atomicHold(eventId, seatIds, holdId);

    if (!success) {
      throw new Error('Some seats are no longer available');
    }

    // Сохраняем hold в БД
    await this.db.query(
      `INSERT INTO seat_holds
       (id, user_id, event_id, seat_ids, expires_at, created_at)
       VALUES ($1, $2, $3, $4, NOW() + INTERVAL '10 minutes', NOW())`,
      [holdId, userId, eventId, JSON.stringify(seatIds)]
    );

    // Устанавливаем TTL в Redis (автоматическое освобождение)
    await this.redis.setex(`hold:${holdId}`, 600, JSON.stringify({
      userId,
      eventId,
      seatIds
    }));

    return holdId;
  }

  async atomicHold(eventId, seatIds, holdId) {
    // Lua script для атомарности
    const luaScript = `
      local eventId = ARGV[1]
      local holdId = ARGV[2]
      local seats = {}

      for i = 3, #ARGV do
        table.insert(seats, ARGV[i])
      end

      -- Проверяем доступность всех мест
      for _, seatId in ipairs(seats) do
        local key = "seat:" .. eventId .. ":" .. seatId
        local status = redis.call("GET", key)

        if status and status ~= "available" then
          return 0  -- seat not available
        end
      end

      -- Захватываем все места
      for _, seatId in ipairs(seats) do
        local key = "seat:" .. eventId .. ":" .. seatId
        redis.call("SET", key, "held:" .. holdId, "EX", 600)
      end

      return 1  -- success
    `;

    const result = await this.redis.eval(
      luaScript,
      0,
      eventId,
      holdId,
      ...seatIds
    );

    return result === 1;
  }

  async confirmBooking(holdId, paymentId) {
    // Получаем hold
    const hold = await this.getHold(holdId);

    if (!hold) {
      throw new Error('Hold not found or expired');
    }

    const bookingId = this.generateBookingId();

    await this.db.query('BEGIN');

    try {
      // Создаём booking
      await this.db.query(
        `INSERT INTO bookings
         (id, user_id, event_id, seat_ids, payment_id, status, created_at)
         VALUES ($1, $2, $3, $4, $5, 'confirmed', NOW())`,
        [bookingId, hold.userId, hold.eventId, JSON.stringify(hold.seatIds), paymentId]
      );

      // Обновляем seats в БД
      await this.db.query(
        `UPDATE seats
         SET status = 'sold', booking_id = $1
         WHERE id = ANY($2)`,
        [bookingId, hold.seatIds]
      );

      // Обновляем inventory count
      await this.redis.hincrby(`event:${hold.eventId}`, 'availableSeats', -hold.seatIds.length);

      // Удаляем hold
      await this.db.query('DELETE FROM seat_holds WHERE id = $1', [holdId]);
      await this.redis.del(`hold:${holdId}`);

      // Обновляем Redis seat status
      const pipeline = this.redis.pipeline();

      for (const seatId of hold.seatIds) {
        pipeline.set(`seat:${hold.eventId}:${seatId}`, 'sold');
      }

      await pipeline.exec();

      await this.db.query('COMMIT');

      // Генерируем QR-код
      await this.generateTicketQR(bookingId);

      return bookingId;
    } catch (error) {
      await this.db.query('ROLLBACK');
      throw error;
    }
  }

  async cancelHold(holdId) {
    const hold = await this.getHold(holdId);

    if (!hold) return;

    // Освобождаем места в Redis
    const pipeline = this.redis.pipeline();

    for (const seatId of hold.seatIds) {
      pipeline.set(`seat:${hold.eventId}:${seatId}`, 'available');
    }

    await pipeline.exec();

    // Удаляем hold
    await this.db.query('DELETE FROM seat_holds WHERE id = $1', [holdId]);
    await this.redis.del(`hold:${holdId}`);
  }

  async getHold(holdId) {
    const cached = await this.redis.get(`hold:${holdId}`);

    if (cached) {
      return JSON.parse(cached);
    }

    const result = await this.db.query(
      `SELECT * FROM seat_holds
       WHERE id = $1 AND expires_at > NOW()`,
      [holdId]
    );

    if (result.rows.length === 0) return null;

    return {
      userId: result.rows[0].user_id,
      eventId: result.rows[0].event_id,
      seatIds: result.rows[0].seat_ids
    };
  }

  async getUserBookings(userId) {
    const result = await this.db.query(
      `SELECT b.id, b.event_id, e.name as event_name, e.date,
              b.seat_ids, b.status, b.qr_code, b.created_at
       FROM bookings b
       JOIN events e ON b.event_id = e.id
       WHERE b.user_id = $1
       ORDER BY e.date DESC`,
      [userId]
    );

    return result.rows;
  }

  async generateTicketQR(bookingId) {
    const QRCode = require('qrcode');

    // Генерируем QR-код с зашифрованными данными
    const qrData = this.encryptBookingData(bookingId);
    const qrCode = await QRCode.toDataURL(qrData);

    // Сохраняем в БД
    await this.db.query(
      'UPDATE bookings SET qr_code = $1 WHERE id = $2',
      [qrCode, bookingId]
    );

    return qrCode;
  }

  encryptBookingData(bookingId) {
    const crypto = require('crypto');
    const cipher = crypto.createCipher('aes-256-cbc', process.env.QR_SECRET);

    let encrypted = cipher.update(bookingId, 'utf8', 'hex');
    encrypted += cipher.final('hex');

    return encrypted;
  }

  generateHoldId() {
    return require('crypto').randomUUID();
  }

  generateBookingId() {
    const Snowflake = require('./snowflake');
    return new Snowflake(1, 1).generate().toString();
  }
}

const service = new BookingService();
```

### 3. Queue Management (Для популярных событий)

```javascript
class QueueService {
  constructor() {
    this.redis = require('./redis');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Join queue
    app.post('/queue/join', async (req, res) => {
      try {
        const { userId, eventId } = req.body;

        const position = await this.joinQueue(userId, eventId);

        res.json({
          success: true,
          position,
          estimatedWaitTime: this.estimateWaitTime(position)
        });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Check queue status
    app.get('/queue/status/:userId/:eventId', async (req, res) => {
      try {
        const { userId, eventId } = req.params;

        const status = await this.getQueueStatus(userId, eventId);

        res.json(status);
      } catch (error) {
        res.status(404).json({ error: 'Not in queue' });
      }
    });

    app.listen(3003, () => {
      console.log('Queue Service listening on port 3003');
    });
  }

  async joinQueue(userId, eventId) {
    const queueKey = `queue:${eventId}`;
    const timestamp = Date.now();

    // Добавляем в sorted set (по времени прихода)
    await this.redis.zadd(queueKey, timestamp, userId);

    // Получаем позицию
    const position = await this.redis.zrank(queueKey, userId);

    return position + 1;
  }

  async getQueueStatus(userId, eventId) {
    const queueKey = `queue:${eventId}`;

    const position = await this.redis.zrank(queueKey, userId);

    if (position === null) {
      throw new Error('Not in queue');
    }

    const totalInQueue = await this.redis.zcard(queueKey);

    return {
      position: position + 1,
      totalInQueue,
      estimatedWaitTime: this.estimateWaitTime(position + 1),
      canBuy: position < 1000 // первые 1000 могут покупать
    };
  }

  estimateWaitTime(position) {
    // Примерно 30 секунд на 100 человек
    return Math.ceil(position / 100) * 30;
  }

  async processQueue(eventId) {
    // Background worker: периодически обрабатывает очередь
    const queueKey = `queue:${eventId}`;

    // Берём первых 100 человек
    const users = await this.redis.zrange(queueKey, 0, 99);

    for (const userId of users) {
      // Отправляем notification о доступе к покупке
      await this.notifyUserCanBuy(userId, eventId);
    }

    // Удаляем обработанных из очереди (через 5 минут)
    setTimeout(async () => {
      await this.redis.zremrangebyrank(queueKey, 0, 99);
    }, 300000);
  }

  async notifyUserCanBuy(userId, eventId) {
    // Publish to notification service
    const kafka = require('./kafka');

    await kafka.send({
      topic: 'notifications',
      messages: [{
        value: JSON.stringify({
          type: 'queue_ready',
          userId,
          eventId,
          message: 'You can now purchase tickets!',
          timestamp: Date.now()
        })
      }]
    });
  }
}

const service = new QueueService();
```

### 4. Anti-Bot Protection

```javascript
class BotProtectionMiddleware {
  constructor() {
    this.redis = require('./redis');
  }

  async middleware(req, res, next) {
    const userId = req.user?.id;
    const ip = req.ip;

    // Rate limiting per user
    if (userId) {
      const userKey = `rate_limit:user:${userId}`;
      const userRequests = await this.redis.incr(userKey);

      if (userRequests === 1) {
        await this.redis.expire(userKey, 60);
      }

      if (userRequests > 100) {
        return res.status(429).json({
          error: 'Too many requests',
          message: 'Please slow down'
        });
      }
    }

    // Rate limiting per IP
    const ipKey = `rate_limit:ip:${ip}`;
    const ipRequests = await this.redis.incr(ipKey);

    if (ipRequests === 1) {
      await this.redis.expire(ipKey, 60);
    }

    if (ipRequests > 200) {
      return res.status(429).json({
        error: 'Too many requests from this IP'
      });
    }

    // CAPTCHA verification (для критичных endpoints)
    if (req.path.includes('/bookings/hold')) {
      const captchaToken = req.headers['x-captcha-token'];

      if (!captchaToken) {
        return res.status(400).json({
          error: 'CAPTCHA required'
        });
      }

      const valid = await this.verifyCaptcha(captchaToken);

      if (!valid) {
        return res.status(400).json({
          error: 'Invalid CAPTCHA'
        });
      }
    }

    next();
  }

  async verifyCaptcha(token) {
    // Verify with Google reCAPTCHA или hCaptcha
    const axios = require('axios');

    const response = await axios.post(
      'https://www.google.com/recaptcha/api/siteverify',
      null,
      {
        params: {
          secret: process.env.RECAPTCHA_SECRET,
          response: token
        }
      }
    );

    return response.data.success && response.data.score > 0.5;
  }
}

module.exports = new BotProtectionMiddleware();
```

## Оптимизации

### 1. Database Indexing

```sql
-- Critical indexes for performance
CREATE INDEX idx_seats_event_status ON seats(event_id, status);
CREATE INDEX idx_seats_booking ON seats(booking_id);
CREATE INDEX idx_bookings_user ON bookings(user_id, created_at DESC);
CREATE INDEX idx_events_date_status ON events(date, status);
CREATE INDEX idx_seat_holds_expires ON seat_holds(expires_at);
```

### 2. Connection Pooling

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 100, // максимум connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});
```

### 3. Caching Strategy

```javascript
class CachingStrategy {
  async getEventWithCache(eventId) {
    // L1: In-memory cache
    if (this.memoryCache.has(eventId)) {
      return this.memoryCache.get(eventId);
    }

    // L2: Redis cache
    const cached = await this.redis.get(`event:${eventId}`);

    if (cached) {
      const event = JSON.parse(cached);
      this.memoryCache.set(eventId, event);
      return event;
    }

    // L3: Database
    const event = await this.db.query(
      'SELECT * FROM events WHERE id = $1',
      [eventId]
    );

    // Cache in Redis (1 hour)
    await this.redis.setex(`event:${eventId}`, 3600, JSON.stringify(event));

    return event;
  }
}
```

## Обработка edge cases

### 1. Hold Expiration Worker

```javascript
class HoldExpirationWorker {
  async start() {
    setInterval(async () => {
      await this.cleanupExpiredHolds();
    }, 10000); // каждые 10 секунд
  }

  async cleanupExpiredHolds() {
    const result = await db.query(
      `SELECT id, event_id, seat_ids
       FROM seat_holds
       WHERE expires_at < NOW()`
    );

    for (const hold of result.rows) {
      // Освобождаем места
      const pipeline = redis.pipeline();

      for (const seatId of hold.seat_ids) {
        pipeline.set(`seat:${hold.event_id}:${seatId}`, 'available');
      }

      await pipeline.exec();

      // Удаляем hold
      await db.query('DELETE FROM seat_holds WHERE id = $1', [hold.id]);

      console.log(`Expired hold ${hold.id} cleaned up`);
    }
  }
}

const worker = new HoldExpirationWorker();
worker.start();
```

### 2. Oversold Prevention

```javascript
async function verifyNoOversold(eventId) {
  // Периодическая проверка консистентности
  const [redisCount, dbCount] = await Promise.all([
    redis.hget(`event:${eventId}`, 'availableSeats'),
    db.query(
      'SELECT COUNT(*) FROM seats WHERE event_id = $1 AND status = \'available\'',
      [eventId]
    )
  ]);

  const redisAvailable = parseInt(redisCount);
  const dbAvailable = parseInt(dbCount.rows[0].count);

  if (redisAvailable !== dbAvailable) {
    console.error(`Inconsistency detected for event ${eventId}: Redis=${redisAvailable}, DB=${dbAvailable}`);

    // Reconcile: trust database
    await redis.hset(`event:${eventId}`, 'availableSeats', dbAvailable);
  }
}
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const bookingsTotal = new prometheus.Counter({
  name: 'ticketing_bookings_total',
  help: 'Total bookings',
  labelNames: ['status'] // confirmed, cancelled
});

const seatsAvailable = new prometheus.Gauge({
  name: 'ticketing_seats_available',
  help: 'Available seats per event',
  labelNames: ['event_id']
});

const holdDuration = new prometheus.Histogram({
  name: 'ticketing_hold_duration_seconds',
  help: 'Time from hold to confirmation',
  buckets: [30, 60, 120, 300, 600]
});

const queueSize = new prometheus.Gauge({
  name: 'ticketing_queue_size',
  help: 'Number of users in queue',
  labelNames: ['event_id']
});
```

## Schema Design

```sql
-- Events
CREATE TABLE events (
  id BIGINT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  date TIMESTAMP NOT NULL,
  venue_id BIGINT NOT NULL,
  category VARCHAR(50),
  status VARCHAR(20) DEFAULT 'upcoming',
  created_at TIMESTAMP DEFAULT NOW(),
  INDEX idx_date_status (date, status),
  INDEX idx_venue (venue_id)
);

-- Venues
CREATE TABLE venues (
  id BIGINT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  address TEXT,
  city VARCHAR(100),
  capacity INT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Seats
CREATE TABLE seats (
  id UUID PRIMARY KEY,
  event_id BIGINT NOT NULL,
  section VARCHAR(50),
  row VARCHAR(10),
  number VARCHAR(10),
  price DECIMAL(10, 2),
  status VARCHAR(20) DEFAULT 'available',
  booking_id BIGINT,
  INDEX idx_event_status (event_id, status),
  INDEX idx_booking (booking_id)
);

-- Seat Holds
CREATE TABLE seat_holds (
  id UUID PRIMARY KEY,
  user_id BIGINT NOT NULL,
  event_id BIGINT NOT NULL,
  seat_ids JSONB NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  INDEX idx_expires (expires_at),
  INDEX idx_user (user_id)
);

-- Bookings
CREATE TABLE bookings (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  event_id BIGINT NOT NULL,
  seat_ids JSONB NOT NULL,
  payment_id VARCHAR(255),
  status VARCHAR(20) DEFAULT 'confirmed',
  qr_code TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  INDEX idx_user_date (user_id, created_at DESC),
  INDEX idx_event (event_id)
);
```

## Что читать дальше

- **Урок 37**: Rate Limiter — защита от перегрузки
- **Урок 41**: Notification System — уведомления о билетах
- **Урок 32**: Distributed Transactions — обеспечение консистентности

## Проверь себя

1. Как предотвратить overbooking при 1M concurrent requests?
2. Спроектируйте систему для 100K seats sold out за 1 минуту.
3. Какую стратегию использовать: optimistic locking vs pessimistic locking?
4. Как обрабатывать refunds (возврат билетов)?
5. Реализуйте dynamic pricing (цена растёт при низком остатке мест).

---

[← Урок 47: Social Network](47-social-network.md) | [Урок 49: Video Streaming →](49-video-streaming.md)
