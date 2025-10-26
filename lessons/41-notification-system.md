# Урок 41: Notification System

## Введение

Notification System — критически важная часть современных приложений. Уведомления держат пользователей в курсе событий, повышают engagement и являются ключевым каналом коммуникации.

В этом уроке мы спроектируем масштабируемую систему уведомлений, поддерживающую multiple channels (push, SMS, email, in-app) и способную обрабатывать миллионы уведомлений в день.

**Примеры:**
- **Social Media**: новые лайки, комментарии, подписчики (Facebook, Instagram)
- **E-commerce**: статус заказа, доставка (Amazon)
- **Banking**: транзакции, security alerts
- **Productivity**: напоминания, deadline (Google Calendar, Slack)

## Требования к системе

### Функциональные требования

1. **Multi-channel поддержка**:
   - Push notifications (iOS, Android, Web)
   - SMS
   - Email
   - In-app notifications

2. **Отправка уведомлений**:
   - Real-time delivery (< 1 sec для критичных)
   - Batch notifications (для маркетинга)
   - Scheduled notifications (отложенная отправка)

3. **Персонализация**:
   - Шаблоны с переменными
   - Мультиязычность (i18n)
   - User preferences (какие каналы включены)

4. **Tracking**:
   - Статус доставки
   - Открытие/клик (для push и email)
   - Analytics

### Нефункциональные требования

1. **Масштабируемость**: 10M notifications/day (≈ 115 notif/sec)
2. **Надёжность**:
   - At-least-once delivery
   - No loss (retry при сбоях)
3. **Latency**:
   - Critical notifications: < 1 sec
   - Non-critical: < 30 sec
4. **Availability**: 99.9%

### Back-of-the-envelope

```
10M notifications/day
10M / 86400 sec ≈ 115 notif/sec (average)

Пиковая нагрузка (5x average): 575 notif/sec

Каналы (распределение):
- Push: 60% = 6M/day
- Email: 30% = 3M/day
- SMS: 5% = 500K/day
- In-app: 5% = 500K/day

Стоимость:
- SMS: $0.01/message = $5K/day
- Push: бесплатно (FCM, APNS)
- Email: $0.0001/email = $300/day (SendGrid)

Хранение (history, 90 days):
10M × 90 days = 900M records
Average size: 500 bytes
Total: 450 GB
```

## Архитектура высокого уровня

```
┌──────────────┐
│  App Servers │
│              │
└──────┬───────┘
       │ Send Notification API
       ↓
┌──────────────────────┐
│ Notification Service │
│   (Orchestrator)     │
└──────┬───────────────┘
       │
       ↓
┌──────────────────────┐      ┌─────────────┐
│   Message Queue      │◄─────┤  Scheduler  │
│  (Kafka / RabbitMQ)  │      │  (Cron jobs)│
└──────┬───────────────┘      └─────────────┘
       │
       ├────────┬─────────┬──────────┐
       │        │         │          │
       ↓        ↓         ↓          ↓
  ┌────────┐┌──────┐┌────────┐┌──────────┐
  │  Push  ││ SMS  ││ Email  ││  In-app  │
  │ Worker ││Worker││ Worker ││  Worker  │
  └───┬────┘└──┬───┘└───┬────┘└────┬─────┘
      │        │        │          │
      ↓        ↓        ↓          ↓
  ┌────────┐┌──────┐┌────────┐┌──────────┐
  │  FCM/  ││Twilio││SendGrid││ Database │
  │  APNS  ││      ││        ││          │
  └────────┘└──────┘└────────┘└──────────┘
```

## Компоненты системы

### 1. Notification Service API

```javascript
const express = require('express');
const app = express();

class NotificationService {
  constructor() {
    this.queue = require('./queue'); // Kafka/RabbitMQ
    this.db = require('./database');
    this.templateEngine = new TemplateEngine();
    this.userPreferences = new UserPreferencesService();

    this.setupRoutes();
  }

  setupRoutes() {
    app.use(express.json());

    // Отправка одного уведомления
    app.post('/notifications/send', async (req, res) => {
      try {
        const { userId, type, channels, data, priority } = req.body;

        const notificationId = await this.sendNotification({
          userId,
          type,
          channels,
          data,
          priority: priority || 'normal'
        });

        res.json({
          success: true,
          notificationId
        });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Batch отправка (для маркетинга)
    app.post('/notifications/send-batch', async (req, res) => {
      try {
        const { userIds, type, channels, data } = req.body;

        const jobId = await this.sendBatch({
          userIds,
          type,
          channels,
          data
        });

        res.json({
          success: true,
          jobId,
          totalUsers: userIds.length
        });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Scheduled notification
    app.post('/notifications/schedule', async (req, res) => {
      try {
        const { userId, type, channels, data, scheduledAt } = req.body;

        const scheduleId = await this.scheduleNotification({
          userId,
          type,
          channels,
          data,
          scheduledAt: new Date(scheduledAt)
        });

        res.json({
          success: true,
          scheduleId
        });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Получение истории уведомлений пользователя
    app.get('/notifications/user/:userId', async (req, res) => {
      try {
        const { userId } = req.params;
        const { limit = 20, offset = 0 } = req.query;

        const notifications = await this.db.query(
          `SELECT id, type, title, body, channel, status, created_at, read_at
           FROM notifications
           WHERE user_id = $1
           ORDER BY created_at DESC
           LIMIT $2 OFFSET $3`,
          [userId, limit, offset]
        );

        res.json({
          notifications: notifications.rows,
          total: notifications.rows.length
        });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Обновление user preferences
    app.put('/notifications/preferences/:userId', async (req, res) => {
      try {
        const { userId } = req.params;
        const preferences = req.body;

        await this.userPreferences.update(userId, preferences);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
  }

  async sendNotification({ userId, type, channels, data, priority }) {
    // Генерируем уникальный ID
    const notificationId = this.generateId();

    // Получаем user preferences
    const userPrefs = await this.userPreferences.get(userId);

    // Фильтруем каналы по preferences
    const enabledChannels = channels.filter(channel =>
      userPrefs[channel] !== false
    );

    if (enabledChannels.length === 0) {
      console.log(`User ${userId} has all channels disabled`);
      return notificationId;
    }

    // Получаем контактную информацию
    const userContact = await this.getUserContact(userId);

    // Рендерим контент из шаблона
    const content = await this.templateEngine.render(type, data, userPrefs.language);

    // Создаём запись в БД
    await this.db.query(
      `INSERT INTO notifications
       (id, user_id, type, title, body, channels, status, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, 'pending', NOW())`,
      [
        notificationId,
        userId,
        type,
        content.title,
        content.body,
        JSON.stringify(enabledChannels)
      ]
    );

    // Отправляем в очередь для каждого канала
    for (const channel of enabledChannels) {
      await this.queue.publish(`notifications.${channel}`, {
        notificationId,
        userId,
        channel,
        contact: userContact[channel],
        content,
        priority,
        timestamp: Date.now()
      });
    }

    return notificationId;
  }

  async getUserContact(userId) {
    const result = await this.db.query(
      `SELECT email, phone, push_tokens
       FROM users
       WHERE id = $1`,
      [userId]
    );

    if (result.rows.length === 0) {
      throw new Error(`User ${userId} not found`);
    }

    const user = result.rows[0];

    return {
      email: user.email,
      sms: user.phone,
      push: user.push_tokens ? JSON.parse(user.push_tokens) : [],
      inapp: userId // для in-app просто нужен user ID
    };
  }

  async sendBatch({ userIds, type, channels, data }) {
    const jobId = this.generateId();

    // Создаём batch job
    await this.db.query(
      `INSERT INTO batch_jobs
       (id, type, total_users, status, created_at)
       VALUES ($1, $2, $3, 'processing', NOW())`,
      [jobId, type, userIds.length]
    );

    // Отправляем по частям (chunks) чтобы не перегрузить систему
    const CHUNK_SIZE = 1000;

    for (let i = 0; i < userIds.length; i += CHUNK_SIZE) {
      const chunk = userIds.slice(i, i + CHUNK_SIZE);

      await this.queue.publish('notifications.batch', {
        jobId,
        userIds: chunk,
        type,
        channels,
        data
      });
    }

    return jobId;
  }

  async scheduleNotification({ userId, type, channels, data, scheduledAt }) {
    const scheduleId = this.generateId();

    await this.db.query(
      `INSERT INTO scheduled_notifications
       (id, user_id, type, channels, data, scheduled_at, status)
       VALUES ($1, $2, $3, $4, $5, $6, 'pending')`,
      [scheduleId, userId, type, JSON.stringify(channels), JSON.stringify(data), scheduledAt]
    );

    return scheduleId;
  }

  generateId() {
    return require('crypto').randomUUID();
  }
}

const service = new NotificationService();
app.listen(3000, () => {
  console.log('Notification Service listening on port 3000');
});
```

### 2. Push Notification Worker

```javascript
const admin = require('firebase-admin');
const apn = require('apn');

class PushNotificationWorker {
  constructor() {
    // Firebase Cloud Messaging (Android + iOS)
    admin.initializeApp({
      credential: admin.credential.cert(require('./firebase-key.json'))
    });

    // Apple Push Notification Service (iOS альтернатива)
    this.apnProvider = new apn.Provider({
      token: {
        key: './apns-key.p8',
        keyId: process.env.APNS_KEY_ID,
        teamId: process.env.APNS_TEAM_ID
      },
      production: process.env.NODE_ENV === 'production'
    });

    this.queue = require('./queue');
    this.db = require('./database');
  }

  async start() {
    console.log('Push notification worker started');

    await this.queue.subscribe('notifications.push', async (message) => {
      try {
        await this.processPush(message);
      } catch (error) {
        console.error('Push processing error:', error);
        // Retry logic (если не fatal error)
        if (this.isRetryable(error)) {
          await this.queue.publish('notifications.push.retry', message);
        }
      }
    });
  }

  async processPush(message) {
    const { notificationId, userId, contact, content, priority } = message;

    console.log(`Sending push to user ${userId}`);

    // contact.push = массив токенов устройств
    const tokens = contact.push;

    if (!tokens || tokens.length === 0) {
      console.log(`No push tokens for user ${userId}`);
      await this.updateStatus(notificationId, 'failed', 'No push tokens');
      return;
    }

    // Отправляем через FCM
    const fcmMessage = {
      notification: {
        title: content.title,
        body: content.body,
        imageUrl: content.imageUrl
      },
      data: {
        notificationId,
        type: content.type,
        ...content.data
      },
      android: {
        priority: priority === 'high' ? 'high' : 'normal',
        notification: {
          sound: 'default',
          clickAction: content.clickAction
        }
      },
      apns: {
        payload: {
          aps: {
            sound: 'default',
            badge: 1
          }
        }
      },
      tokens
    };

    try {
      const response = await admin.messaging().sendMulticast(fcmMessage);

      console.log(`Push sent: ${response.successCount} success, ${response.failureCount} failures`);

      // Обрабатываем результаты
      const failedTokens = [];
      response.responses.forEach((resp, idx) => {
        if (!resp.success) {
          failedTokens.push(tokens[idx]);
          console.error(`Failed to send to token ${tokens[idx]}:`, resp.error);

          // Удаляем невалидные токены
          if (resp.error.code === 'messaging/invalid-registration-token' ||
              resp.error.code === 'messaging/registration-token-not-registered') {
            this.removeInvalidToken(userId, tokens[idx]);
          }
        }
      });

      if (response.successCount > 0) {
        await this.updateStatus(notificationId, 'sent');
      } else {
        await this.updateStatus(notificationId, 'failed', 'All tokens failed');
      }
    } catch (error) {
      console.error('FCM error:', error);
      await this.updateStatus(notificationId, 'failed', error.message);
      throw error;
    }
  }

  async updateStatus(notificationId, status, errorMessage = null) {
    await this.db.query(
      `UPDATE notifications
       SET status = $1, error_message = $2, sent_at = NOW()
       WHERE id = $3`,
      [status, errorMessage, notificationId]
    );
  }

  async removeInvalidToken(userId, token) {
    // Удаляем невалидный токен из БД
    await this.db.query(
      `UPDATE users
       SET push_tokens = push_tokens - $1
       WHERE id = $2`,
      [token, userId]
    );
  }

  isRetryable(error) {
    // Retry на временных ошибках (network, 5xx)
    return (
      error.code === 'ETIMEDOUT' ||
      error.code === 'ECONNRESET' ||
      (error.response && error.response.status >= 500)
    );
  }
}

// Запуск
const worker = new PushNotificationWorker();
worker.start();
```

### 3. Email Worker

```javascript
const sgMail = require('@sendgrid/mail');
const Handlebars = require('handlebars');
const fs = require('fs').promises;

class EmailWorker {
  constructor() {
    sgMail.setApiKey(process.env.SENDGRID_API_KEY);

    this.queue = require('./queue');
    this.db = require('./database');

    // Кэш шаблонов
    this.templates = new Map();
  }

  async start() {
    console.log('Email worker started');

    await this.queue.subscribe('notifications.email', async (message) => {
      try {
        await this.processEmail(message);
      } catch (error) {
        console.error('Email processing error:', error);

        if (this.isRetryable(error)) {
          await this.queue.publish('notifications.email.retry', message);
        }
      }
    });
  }

  async processEmail(message) {
    const { notificationId, userId, contact, content } = message;

    console.log(`Sending email to ${contact.email}`);

    // Загружаем шаблон
    const template = await this.getTemplate(content.type);

    // Рендерим HTML
    const html = template({
      ...content,
      year: new Date().getFullYear(),
      unsubscribeUrl: `https://example.com/unsubscribe?user=${userId}`
    });

    // Отправляем через SendGrid
    const msg = {
      to: contact.email,
      from: {
        email: 'noreply@example.com',
        name: 'Example App'
      },
      subject: content.title,
      html,
      text: content.body, // fallback для plain text
      trackingSettings: {
        clickTracking: { enable: true },
        openTracking: { enable: true }
      },
      customArgs: {
        notificationId,
        userId
      }
    };

    try {
      await sgMail.send(msg);

      console.log(`Email sent to ${contact.email}`);

      await this.updateStatus(notificationId, 'sent');
    } catch (error) {
      console.error('SendGrid error:', error);

      if (error.response) {
        console.error(error.response.body);
      }

      await this.updateStatus(notificationId, 'failed', error.message);
      throw error;
    }
  }

  async getTemplate(type) {
    if (this.templates.has(type)) {
      return this.templates.get(type);
    }

    // Загружаем из файла
    const templatePath = `./templates/${type}.hbs`;
    const templateSource = await fs.readFile(templatePath, 'utf-8');
    const template = Handlebars.compile(templateSource);

    this.templates.set(type, template);

    return template;
  }

  async updateStatus(notificationId, status, errorMessage = null) {
    await this.db.query(
      `UPDATE notifications
       SET status = $1, error_message = $2, sent_at = NOW()
       WHERE id = $3`,
      [status, errorMessage, notificationId]
    );
  }

  isRetryable(error) {
    return error.code >= 500 || error.code === 429; // 5xx или rate limit
  }
}

// Запуск
const worker = new EmailWorker();
worker.start();
```

### 4. SMS Worker

```javascript
const twilio = require('twilio');

class SMSWorker {
  constructor() {
    this.client = twilio(
      process.env.TWILIO_ACCOUNT_SID,
      process.env.TWILIO_AUTH_TOKEN
    );

    this.queue = require('./queue');
    this.db = require('./database');
  }

  async start() {
    console.log('SMS worker started');

    await this.queue.subscribe('notifications.sms', async (message) => {
      try {
        await this.processSMS(message);
      } catch (error) {
        console.error('SMS processing error:', error);

        if (this.isRetryable(error)) {
          await this.queue.publish('notifications.sms.retry', message);
        }
      }
    });
  }

  async processSMS(message) {
    const { notificationId, userId, contact, content } = message;

    console.log(`Sending SMS to ${contact.sms}`);

    // Twilio ограничение: 160 символов для одного SMS
    const body = this.truncate(content.body, 160);

    try {
      const result = await this.client.messages.create({
        body,
        from: process.env.TWILIO_PHONE_NUMBER,
        to: contact.sms
      });

      console.log(`SMS sent: ${result.sid}`);

      await this.updateStatus(notificationId, 'sent', result.sid);
    } catch (error) {
      console.error('Twilio error:', error);

      await this.updateStatus(notificationId, 'failed', error.message);
      throw error;
    }
  }

  truncate(text, maxLength) {
    if (text.length <= maxLength) {
      return text;
    }

    return text.substring(0, maxLength - 3) + '...';
  }

  async updateStatus(notificationId, status, twilioSid = null) {
    await this.db.query(
      `UPDATE notifications
       SET status = $1, external_id = $2, sent_at = NOW()
       WHERE id = $3`,
      [status, twilioSid, notificationId]
    );
  }

  isRetryable(error) {
    // Retry на временных ошибках
    return error.code >= 500 || error.code === 429;
  }
}

// Запуск
const worker = new SMSWorker();
worker.start();
```

### 5. Template Engine

```javascript
const Handlebars = require('handlebars');
const i18n = require('i18n');

class TemplateEngine {
  constructor() {
    // Настройка i18n
    i18n.configure({
      locales: ['en', 'es', 'fr', 'de', 'ru'],
      directory: './locales',
      defaultLocale: 'en'
    });

    // Регистрируем helpers
    Handlebars.registerHelper('t', (key, options) => {
      return i18n.__(key);
    });
  }

  async render(type, data, language = 'en') {
    i18n.setLocale(language);

    // Шаблоны хранятся в БД или файлах
    const template = await this.getTemplate(type, language);

    const compiledTitle = Handlebars.compile(template.title);
    const compiledBody = Handlebars.compile(template.body);

    return {
      type,
      title: compiledTitle(data),
      body: compiledBody(data),
      imageUrl: template.imageUrl,
      clickAction: template.clickAction,
      data
    };
  }

  async getTemplate(type, language) {
    // Загружаем из БД
    const result = await db.query(
      `SELECT title, body, image_url, click_action
       FROM notification_templates
       WHERE type = $1 AND language = $2`,
      [type, language]
    );

    if (result.rows.length === 0) {
      // Fallback на английский
      if (language !== 'en') {
        return this.getTemplate(type, 'en');
      }

      throw new Error(`Template not found: ${type}`);
    }

    return {
      title: result.rows[0].title,
      body: result.rows[0].body,
      imageUrl: result.rows[0].image_url,
      clickAction: result.rows[0].click_action
    };
  }
}

// Пример шаблона в БД:
/*
INSERT INTO notification_templates (type, language, title, body)
VALUES (
  'new_follower',
  'en',
  '{{followerName}} started following you',
  'Check out their profile and follow them back!'
);

INSERT INTO notification_templates (type, language, title, body)
VALUES (
  'new_follower',
  'ru',
  '{{followerName}} подписался на вас',
  'Посмотрите его профиль и подпишитесь в ответ!'
);
*/
```

### 6. User Preferences Service

```javascript
class UserPreferencesService {
  async get(userId) {
    const result = await db.query(
      `SELECT preferences
       FROM user_notification_preferences
       WHERE user_id = $1`,
      [userId]
    );

    if (result.rows.length === 0) {
      // Возвращаем дефолтные preferences
      return {
        push: true,
        email: true,
        sms: false,
        inapp: true,
        language: 'en',
        quietHours: { enabled: false, start: '22:00', end: '08:00' }
      };
    }

    return result.rows[0].preferences;
  }

  async update(userId, preferences) {
    await db.query(
      `INSERT INTO user_notification_preferences (user_id, preferences)
       VALUES ($1, $2)
       ON CONFLICT (user_id) DO UPDATE SET preferences = $2`,
      [userId, JSON.stringify(preferences)]
    );
  }

  async isQuietHours(userId) {
    const prefs = await this.get(userId);

    if (!prefs.quietHours || !prefs.quietHours.enabled) {
      return false;
    }

    const now = new Date();
    const currentTime = `${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}`;

    const { start, end } = prefs.quietHours;

    // Простая проверка (не учитывает переход через полночь)
    return currentTime >= start || currentTime < end;
  }
}
```

## Продвинутые фичи

### 1. Rate Limiting (per user)

```javascript
class RateLimiter {
  constructor(redis) {
    this.redis = redis;
  }

  async checkLimit(userId, channel, limit = 10, windowSec = 3600) {
    const key = `rate_limit:${userId}:${channel}`;
    const now = Date.now();
    const windowStart = now - windowSec * 1000;

    // Удаляем старые записи
    await this.redis.zremrangebyscore(key, 0, windowStart);

    // Считаем текущие уведомления
    const count = await this.redis.zcard(key);

    if (count >= limit) {
      return { allowed: false, remaining: 0 };
    }

    // Добавляем текущее уведомление
    await this.redis.zadd(key, now, `${now}-${Math.random()}`);
    await this.redis.expire(key, windowSec);

    return { allowed: true, remaining: limit - count - 1 };
  }
}
```

### 2. Deduplication

```javascript
class NotificationDeduplicator {
  async isDuplicate(userId, type, data, windowSec = 3600) {
    // Генерируем content hash
    const content = JSON.stringify({ type, data });
    const hash = crypto.createHash('sha256').update(content).digest('hex');

    const key = `dedup:${userId}:${hash}`;

    // Проверяем существование
    const exists = await redis.exists(key);

    if (exists) {
      return true; // дубликат
    }

    // Устанавливаем с TTL
    await redis.setex(key, windowSec, '1');

    return false;
  }
}

// Использование в sendNotification():
const isDup = await deduplicator.isDuplicate(userId, type, data);

if (isDup) {
  console.log(`Duplicate notification suppressed for user ${userId}`);
  return null;
}
```

### 3. Analytics & Tracking

```javascript
class NotificationAnalytics {
  async trackDelivery(notificationId, channel, status) {
    await db.query(
      `INSERT INTO notification_events
       (notification_id, event_type, channel, timestamp)
       VALUES ($1, $2, $3, NOW())`,
      [notificationId, status, channel]
    );
  }

  async trackOpen(notificationId) {
    await db.query(
      `UPDATE notifications
       SET read_at = NOW()
       WHERE id = $1 AND read_at IS NULL`,
      [notificationId]
    );

    await this.trackDelivery(notificationId, null, 'opened');
  }

  async trackClick(notificationId, linkUrl) {
    await db.query(
      `INSERT INTO notification_events
       (notification_id, event_type, metadata, timestamp)
       VALUES ($1, 'clicked', $2, NOW())`,
      [notificationId, JSON.stringify({ linkUrl })]
    );
  }

  async getMetrics(startDate, endDate) {
    const result = await db.query(
      `SELECT
         channel,
         COUNT(*) as sent,
         SUM(CASE WHEN status = 'sent' THEN 1 ELSE 0 END) as delivered,
         SUM(CASE WHEN read_at IS NOT NULL THEN 1 ELSE 0 END) as opened
       FROM notifications
       WHERE created_at BETWEEN $1 AND $2
       GROUP BY channel`,
      [startDate, endDate]
    );

    return result.rows.map(row => ({
      channel: row.channel,
      sent: parseInt(row.sent),
      delivered: parseInt(row.delivered),
      opened: parseInt(row.opened),
      deliveryRate: (row.delivered / row.sent * 100).toFixed(2) + '%',
      openRate: (row.opened / row.delivered * 100).toFixed(2) + '%'
    }));
  }
}
```

### 4. Webhooks для tracking

```javascript
// Webhook endpoint для SendGrid events
app.post('/webhooks/sendgrid', async (req, res) => {
  const events = req.body;

  for (const event of events) {
    const { notificationId } = event.customArgs || {};

    if (!notificationId) continue;

    switch (event.event) {
      case 'delivered':
        await analytics.trackDelivery(notificationId, 'email', 'delivered');
        break;

      case 'open':
        await analytics.trackOpen(notificationId);
        break;

      case 'click':
        await analytics.trackClick(notificationId, event.url);
        break;

      case 'bounce':
      case 'dropped':
        await analytics.trackDelivery(notificationId, 'email', 'failed');
        break;
    }
  }

  res.status(200).send('OK');
});

// Webhook для Twilio SMS status
app.post('/webhooks/twilio', async (req, res) => {
  const { MessageSid, MessageStatus } = req.body;

  // Находим notification по Twilio SID
  const result = await db.query(
    'SELECT id FROM notifications WHERE external_id = $1',
    [MessageSid]
  );

  if (result.rows.length > 0) {
    const notificationId = result.rows[0].id;

    switch (MessageStatus) {
      case 'delivered':
        await analytics.trackDelivery(notificationId, 'sms', 'delivered');
        break;

      case 'failed':
      case 'undelivered':
        await analytics.trackDelivery(notificationId, 'sms', 'failed');
        break;
    }
  }

  res.status(200).send('OK');
});
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const notificationsSent = new prometheus.Counter({
  name: 'notifications_sent_total',
  help: 'Total notifications sent',
  labelNames: ['channel', 'status']
});

const notificationLatency = new prometheus.Histogram({
  name: 'notification_latency_seconds',
  help: 'Time from creation to delivery',
  labelNames: ['channel'],
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60]
});

const queueSize = new prometheus.Gauge({
  name: 'notification_queue_size',
  help: 'Number of notifications in queue',
  labelNames: ['channel']
});

// В worker
async processPush(message) {
  const start = Date.now();

  try {
    // ... отправка ...

    notificationsSent.inc({ channel: 'push', status: 'success' });

    const latency = (Date.now() - message.timestamp) / 1000;
    notificationLatency.observe({ channel: 'push' }, latency);
  } catch (error) {
    notificationsSent.inc({ channel: 'push', status: 'failed' });
    throw error;
  }
}
```

## Что читать дальше

- **Урок 21**: Message Queues — детали работы с очередями
- **Урок 24**: WebSockets — real-time in-app notifications
- **Урок 42**: Autocomplete — ещё одна система с real-time требованиями

## Проверь себя

1. Почему используется message queue между API и workers?
2. Как обеспечить at-least-once delivery?
3. Спроектируйте систему для 100M notifications/day с приоритизацией.
4. Как обработать ситуацию, когда все каналы пользователя отключены?
5. Реализуйте batching для email (1000 emails за раз через SendGrid).

---

[← Урок 40: Web Crawler](40-web-crawler.md) | [Урок 42: Autocomplete System →](42-autocomplete.md)
