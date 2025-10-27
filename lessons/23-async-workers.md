# Урок 23: Async Workers — Celery, BullMQ

[← Предыдущий урок: Kafka и Event Streaming](22-kafka-event-streaming.md) | [Следующий урок: WebSockets в production →](24-websockets-production.md)

## Введение

Async Workers (асинхронные воркеры) — это паттерн для выполнения фоновых задач вне основного потока обработки запросов. Они позволяют веб-приложению быстро отвечать пользователю, делегируя долгие операции в фон.

**Зачем нужны воркеры?**

Без воркеров:
```javascript
app.post('/signup', async (req, res) => {
  const { email, password } = req.body;
  const user = await createUser(email, password); // 50ms
  await sendWelcomeEmail(email); // 2000ms ❌
  await generateThumbnail(user.avatar); // 500ms ❌
  await updateAnalytics(user); // 300ms ❌
  await notifyCrm(user); // 400ms ❌
  res.json({ userId: user.id }); // Total: 3250ms
});
```

С воркерами:
```javascript
app.post('/signup', async (req, res) => {
  const { email, password } = req.body;
  const user = await createUser(email, password); // 50ms
  await emailQueue.add('send-welcome', { email }); // 5ms (в очередь)
  await imageQueue.add('generate-thumbnail', { avatar: user.avatar }); // 5ms
  await analyticsQueue.add('update-analytics', { user }); // 5ms
  await crmQueue.add('notify-crm', { user }); // 5ms
  res.json({ userId: user.id }); // Total: 70ms ✅
});
```

**Основные use cases:**
- Email рассылка
- Обработка изображений/видео
- Генерация отчётов
- Парсинг данных
- Интеграции с внешними API
- Scheduled tasks (cron jobs)

## Celery (Python)

Celery — популярный фреймворк для распределённой обработки задач в Python.

### Архитектура Celery

```
Client (Web App) → Broker (Redis/RabbitMQ) → Worker → Result Backend
                         ↓
                     [Task Queue]
                         ↓
                   Worker Pool (processes/threads)
```

**Компоненты:**
- **Client**: отправляет задачи
- **Broker**: очередь задач (Redis, RabbitMQ, SQS)
- **Worker**: выполняет задачи
- **Result Backend**: хранит результаты (Redis, Database)

### Установка и настройка

```bash
pip install celery redis
```

```javascript
// queues.js
const { Queue } = require('bullmq');

const connection = {
  host: '127.0.0.1',
  port: 6379,
};

const emailQueue = new Queue('emails', {
  connection,
  defaultJobOptions: {
    removeOnComplete: true,
    attempts: 5,
    backoff: { type: 'exponential', delay: 2000 },
    timeout: 300_000,
  },
});

const imageQueue = new Queue('image-processing', {
  connection,
  defaultJobOptions: {
    removeOnComplete: true,
    timeout: 600_000,
  },
});

const analyticsQueue = new Queue('analytics', { connection });
const crmQueue = new Queue('crm', { connection });

module.exports = {
  connection,
  emailQueue,
  imageQueue,
  analyticsQueue,
  crmQueue,
};
```

### Базовая задача

```javascript
// workers/email-worker.js
const { Worker } = require('bullmq');
const { emailQueue, connection } = require('../queues');

const emailWorker = new Worker(
  emailQueue.name,
  async (job) => {
    const { to, subject, body } = job.data;
    console.log(`Sending email to ${to}...`);
    await sendEmail(to, subject, body);
    console.log(`Email sent to ${to}`);
    return { to };
  },
  { connection },
);

module.exports = { emailWorker };
```

### Запуск воркера

```bash
# Один воркер с concurrency = 4
CONCURRENCY=4 node workers/email-worker.js

# Autoscale можно реализовать через кластер/оркестратор (PM2, Kubernetes HPA)
# Пример: запуск 2–10 воркеров в Kubernetes Deployment

# Высокая I/O-нагрузка — увеличиваем concurrency
CONCURRENCY=100 node workers/email-worker.js
```

### Вызов задач

```javascript
// Асинхронный вызов
const job = await emailQueue.add('send-welcome', {
  to: 'user@example.com',
  subject: 'Welcome',
  body: 'Hello!',
});

// Альтернативный синтаксис с отложенным стартом
const delayedJob = await emailQueue.add(
  'send-welcome',
  { to: 'user@example.com', subject: 'Welcome', body: 'Hello!' },
  { delay: 10_000 }, // Выполнить через 10 секунд
);

// Получение статуса
console.log(delayedJob.id); // Task ID
console.log(await delayedJob.getState()); // waiting, active, completed, failed
console.log(await delayedJob.isCompleted());
console.log(await delayedJob.isFailed());
```

### Task Options

```javascript
const imageWorker = new Worker(
  imageQueue.name,
  async (job) => {
    const { imageUrl } = job.data;
    return processImage(imageUrl);
  },
  {
    connection,
    concurrency: 5,
    limiter: {
      max: 10, // 10 задач в минуту
      duration: 60_000,
    },
  },
);

await imageQueue.add(
  'process-image',
  { imageUrl },
  {
    attempts: 3,
    backoff: { type: 'exponential', delay: 60_000 },
    timeout: 300_000, // Hard limit
    jobId: `process-image:${imageUrl}`,
    removeOnComplete: true,
  },
);
```

### Retry механизм

```javascript
class NonRetryableError extends Error {}

const fetchWorker = new Worker(
  'fetch-data',
  async (job) => {
    const { url } = job.data;

    try {
      const response = await fetch(url, { timeout: 10_000 });
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

      return response.json();
    } catch (error) {
      if (error instanceof SyntaxError) {
        // Не retryable ошибка → не переочередь
        await job.discard();
        throw new NonRetryableError('Invalid JSON response');
      }

      throw error; // BullMQ выполнит повтор с backoff
    }
  },
  {
    connection,
    settings: {
      backoffStrategies: {
        exponential: (attempts) => Math.min(2 ** attempts * 1000, 16_000),
      },
    },
  },
);

await fetchQueue.add(
  'fetch-data',
  { url: 'https://api.example.com/data' },
  {
    attempts: 5,
    backoff: { type: 'exponential' },
  },
);
```

### Chains и Workflows

```javascript
const { FlowProducer } = require('bullmq');
const flow = new FlowProducer({ connection });

// Chain: последовательное выполнение
await flow.add({
  name: 'download-image',
  queueName: imageQueue.name,
  data: { url: 'http://example.com/image.jpg' },
  children: [
    {
      name: 'resize-image',
      queueName: imageQueue.name,
      data: { width: 800, height: 600 },
      children: [
        {
          name: 'upload-to-s3',
          queueName: imageQueue.name,
          data: { bucket: 'bucket-name' },
        },
      ],
    },
  ],
});

// Group: параллельное выполнение
await Promise.all([
  emailQueue.add('send-email', { to: 'user1@example.com' }),
  emailQueue.add('send-email', { to: 'user2@example.com' }),
  emailQueue.add('send-email', { to: 'user3@example.com' }),
]);

// Chord: group + callback (используем FlowProducer)
const items = [1, 2, 3, 4, 5];
await flow.add({
  name: 'aggregate-results',
  queueName: analyticsQueue.name,
  data: {},
  children: items.map((itemId) => ({
    name: 'process-item',
    queueName: analyticsQueue.name,
    data: { itemId },
  })),
});
```

### Пример: Batch Processing

```javascript
const batchWorker = new Worker(
  analyticsQueue.name,
  async (job) => {
    if (job.name === 'process-item') {
      const item = await db.get(job.data.itemId);
      const result = await expensiveComputation(item);
      await db.save(result);
      return result;
    }

    if (job.name === 'aggregate-results') {
      const { childrenValues } = job;
      const total = childrenValues.reduce((sum, value) => sum + value, 0);
      await sendNotification(`Batch completed. Total: ${total}`);
      return total;
    }

    throw new Error(`Unknown job ${job.name}`);
  },
  { connection },
);

const items = Array.from({ length: 10 }, (_, i) => i + 1);
await flow.add({
  name: 'aggregate-results',
  queueName: analyticsQueue.name,
  data: {},
  children: items.map((itemId) => ({
    name: 'process-item',
    queueName: analyticsQueue.name,
    data: { itemId },
  })),
});
```

### Periodic Tasks (Celery Beat)

```javascript
await emailQueue.add(
  'send-daily-report',
  {},
  {
    repeat: { cron: '0 9 * * *' }, // Каждый день в 9:00
  },
);

await analyticsQueue.add(
  'cleanup-old-sessions',
  {},
  {
    repeat: { cron: '0 2 * * *' }, // Каждый день в 2:00
  },
);

await analyticsQueue.add(
  'update-cache',
  {},
  {
    repeat: { every: 300_000 }, // Каждые 5 минут
  },
);

await emailQueue.add(
  'send-reminder',
  {},
  {
    repeat: { cron: '0 10 * * 1-5' }, // Пн-Пт в 10:00
  },
);
```

```bash
# Повторяющиеся задачи требуют QueueScheduler
node schedulers/queue-scheduler.js
```

### Мониторинг: Flower

```bash
npm install -g bull-board

# Запуск веб-интерфейса
npx bull-board --queue "redis://127.0.0.1:6379#emails"
```

Откройте `http://localhost:3000`:
- Активные воркеры
- Статистика задач
- История выполнения
- Графики производительности

### Пример: Image Processing Service

```javascript
// workers/image-pipeline.js
const { Worker } = require('bullmq');
const fetch = require('node-fetch');
const sharp = require('sharp');
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
const { imageQueue, connection } = require('../queues');

const s3 = new S3Client({ region: 'us-east-1' });

const imagePipelineWorker = new Worker(
  imageQueue.name,
  async (job) => {
    const { imageUrl, userId } = job.data;

    // 1. Download
    const response = await fetch(imageUrl, { timeout: 30_000 });
    const buffer = Buffer.from(await response.arrayBuffer());

    const sizes = [
      { width: 800, height: 600 },
      { width: 400, height: 300 },
      { width: 200, height: 150 },
    ];

    const results = [];

    for (const { width, height } of sizes) {
      // 2. Resize
      const resized = await sharp(buffer).resize(width, height).jpeg().toBuffer();

      const key = `users/${userId}/images/${width}x${height}.jpg`;

      // 3. Upload to S3
      await s3.send(
        new PutObjectCommand({
          Bucket: 'my-bucket',
          Key: key,
          Body: resized,
          ContentType: 'image/jpeg',
        }),
      );

      results.push({ size: `${width}x${height}`, url: `https://my-bucket.s3.amazonaws.com/${key}` });
    }

    // 4. Update database
    await db.updateUserImages(userId, results);

    return { userId, images: results };
  },
  {
    connection,
    attempts: 3,
    backoff: { type: 'fixed', delay: 60_000 },
  },
);

// Web API
app.post('/upload', async (req, res) => {
  const { imageUrl, userId } = req.body;
  const job = await imageQueue.add('image-pipeline', { imageUrl, userId });
  res.json({ jobId: job.id, status: 'processing' });
});

app.get('/status/:jobId', async (req, res) => {
  const job = await imageQueue.getJob(req.params.jobId);
  if (!job) {
    return res.status(404).json({ error: 'Job not found' });
  }

  const state = await job.getState();
  const result = state === 'completed' ? await job.returnvalue : null;
  res.json({ jobId: job.id, status: state, result });
});
```

### Priority Queues

```javascript
await emailQueue.add('critical-task', { payload }, { priority: 1 }); // highest priority
await emailQueue.add('normal-task', { payload }, { priority: 5 });
await emailQueue.add('low-task', { payload }, { priority: 10 });

// Worker будет обрабатывать задачи в порядке приоритета
const priorityWorker = new Worker(
  emailQueue.name,
  async (job) => handleTask(job),
  { connection, concurrency: 10 },
);
```

## BullMQ (Node.js)

BullMQ — мощная библиотека для работы с очередями задач в Node.js (на базе Redis).

### Установка

```bash
npm install bullmq ioredis
```

### Базовая очередь

```javascript
// queue.js
const { Queue } = require('bullmq');

const emailQueue = new Queue('emails', {
  connection: {
    host: 'localhost',
    port: 6379,
  }
});

module.exports = { emailQueue };
```

### Добавление задач

```javascript
// producer.js
const { emailQueue } = require('./queue');

async function sendWelcomeEmail(userId, email) {
  await emailQueue.add('welcome-email', {
    userId,
    email,
    timestamp: Date.now()
  }, {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000,
    },
    removeOnComplete: 100,  // Хранить только 100 завершённых
    removeOnFail: 1000,
  });

  console.log(`Email job added for ${email}`);
}

// Отложенная отправка
await emailQueue.add('reminder', { userId: 123 }, {
  delay: 3600000,  // Через 1 час
});

// Повторяющаяся задача
await emailQueue.add('daily-digest', {}, {
  repeat: {
    pattern: '0 9 * * *',  // Каждый день в 9:00 (cron)
  },
});
```

### Worker

```javascript
// worker.js
const { Worker } = require('bullmq');
const nodemailer = require('nodemailer');

const emailWorker = new Worker('emails', async (job) => {
  const { userId, email, timestamp } = job.data;

  console.log(`Processing job ${job.id} for ${email}`);

  // Имитация отправки email
  await sendEmail(email, 'Welcome!', 'Thanks for signing up');

  // Progress reporting
  await job.updateProgress(50);

  await new Promise(resolve => setTimeout(resolve, 1000));

  await job.updateProgress(100);

  return { sent: true, email, sentAt: new Date() };
}, {
  connection: {
    host: 'localhost',
    port: 6379,
  },
  concurrency: 5,  // Обрабатывать 5 задач параллельно
  limiter: {
    max: 10,  // Максимум 10 задач
    duration: 1000,  // за 1 секунду
  },
});

emailWorker.on('completed', (job, result) => {
  console.log(`Job ${job.id} completed:`, result);
});

emailWorker.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed:`, err.message);
});

emailWorker.on('progress', (job, progress) => {
  console.log(`Job ${job.id} progress: ${progress}%`);
});

async function sendEmail(to, subject, text) {
  // Реальная отправка через SMTP
  const transporter = nodemailer.createTransport({
    host: 'smtp.example.com',
    port: 587,
    auth: { user: 'user', pass: 'pass' }
  });

  await transporter.sendMail({ from: 'noreply@example.com', to, subject, text });
}
```

### Приоритеты

```javascript
// Высокий приоритет (меньшее число = выше приоритет)
await emailQueue.add('urgent-email', { email: 'ceo@example.com' }, {
  priority: 1,
});

// Обычный приоритет
await emailQueue.add('normal-email', { email: 'user@example.com' }, {
  priority: 10,
});

// Низкий приоритет
await emailQueue.add('newsletter', { email: 'subscriber@example.com' }, {
  priority: 100,
});
```

### Job Events и прогресс

```javascript
const { Queue } = require('bullmq');

const queue = new Queue('video-processing');

async function processVideo(videoId) {
  const job = await queue.add('process', { videoId });

  // Слушаем события конкретной задачи
  job.waitUntilFinished().then((result) => {
    console.log('Video processed:', result);
  }).catch((err) => {
    console.error('Processing failed:', err);
  });

  return job.id;
}

// Worker с reporting прогресса
const worker = new Worker('video-processing', async (job) => {
  const { videoId } = job.data;

  await job.updateProgress(0);
  await job.log('Starting video download');

  const video = await downloadVideo(videoId);
  await job.updateProgress(30);

  await job.log('Transcoding video');
  const transcoded = await transcodeVideo(video);
  await job.updateProgress(70);

  await job.log('Uploading to CDN');
  const cdnUrl = await uploadToCDN(transcoded);
  await job.updateProgress(100);

  return { cdnUrl, videoId };
});
```

### Batch Jobs

```javascript
const { Queue } = require('bullmq');

const queue = new Queue('batch-processing');

// Добавление батча задач
async function processBatch(items) {
  const jobs = items.map((item, index) => ({
    name: 'process-item',
    data: { item },
    opts: {
      jobId: `batch-1-item-${index}`,  // Уникальный ID
    },
  }));

  await queue.addBulk(jobs);
}

// Обработка
const worker = new Worker('batch-processing', async (job) => {
  const { item } = job.data;
  return await processItem(item);
});

// Мониторинг завершения батча
async function waitForBatchCompletion(batchId, itemCount) {
  let completed = 0;

  const listener = new QueueEvents('batch-processing');

  listener.on('completed', ({ jobId }) => {
    if (jobId.startsWith(`batch-${batchId}`)) {
      completed++;

      if (completed === itemCount) {
        console.log(`Batch ${batchId} completed!`);
        listener.close();
      }
    }
  });
}
```

### Rate Limiting

```javascript
const worker = new Worker('api-calls', async (job) => {
  // Вызов внешнего API
  const result = await fetch(`https://api.example.com/data/${job.data.id}`);
  return await result.json();
}, {
  limiter: {
    max: 100,      // Максимум 100 запросов
    duration: 60000,  // за 60 секунд
    groupKey: 'api-calls',
  },
});
```

### Scheduled/Recurring Jobs

```javascript
const { Queue } = require('bullmq');

const scheduledQueue = new Queue('scheduled-tasks');

// Cron-like scheduling
await scheduledQueue.add('backup-database', {}, {
  repeat: {
    pattern: '0 2 * * *',  // Каждый день в 2:00 AM
    tz: 'America/New_York',
  },
});

await scheduledQueue.add('send-weekly-report', {}, {
  repeat: {
    pattern: '0 9 * * 1',  // Каждый понедельник в 9:00
  },
});

// Every N minutes
await scheduledQueue.add('health-check', {}, {
  repeat: {
    every: 300000,  // Каждые 5 минут
  },
});
```

### Flow: Параллельные и последовательные задачи

```javascript
const { FlowProducer } = require('bullmq');

const flowProducer = new FlowProducer({
  connection: { host: 'localhost', port: 6379 }
});

// Сложный workflow
const flow = await flowProducer.add({
  name: 'process-order',
  queueName: 'orders',
  data: { orderId: 12345 },
  children: [
    {
      name: 'charge-payment',
      queueName: 'payments',
      data: { orderId: 12345, amount: 99.99 },
    },
    {
      name: 'update-inventory',
      queueName: 'inventory',
      data: { orderId: 12345 },
      children: [
        {
          name: 'notify-warehouse',
          queueName: 'notifications',
          data: { type: 'inventory-updated' },
        },
      ],
    },
    {
      name: 'send-confirmation',
      queueName: 'emails',
      data: { orderId: 12345 },
    },
  ],
});
```

### Мониторинг с Bull Board

```bash
npm install @bull-board/express @bull-board/api
```

```javascript
const { createBullBoard } = require('@bull-board/api');
const { BullMQAdapter } = require('@bull-board/api/bullMQAdapter');
const { ExpressAdapter } = require('@bull-board/express');
const express = require('express');

const app = express();

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/admin/queues');

createBullBoard({
  queues: [
    new BullMQAdapter(emailQueue),
    new BullMQAdapter(videoQueue),
    new BullMQAdapter(batchQueue),
  ],
  serverAdapter,
});

app.use('/admin/queues', serverAdapter.getRouter());

app.listen(3000, () => {
  console.log('Bull Board available at http://localhost:3000/admin/queues');
});
```

## Паттерны и Best Practices

### 1. Idempotency

```javascript
// BullMQ
const worker = new Worker('payments', async (job) => {
  const { paymentId } = job.data;

  // BullMQ автоматически предотвращает дублирование с jobId
  const existing = await paymentQueue.getJob(paymentId);
  if (existing && existing.finishedOn) {
    return { status: 'already_processed' };
  }

  return await processPayment(paymentId);
});

// При добавлении задачи используем paymentId как jobId
await paymentQueue.add('process', { paymentId: '123' }, {
  jobId: paymentId,  // Уникальный ID предотвратит дубликаты
});
```

### 2. Error Handling & Dead Letter Queue

```javascript
// BullMQ
const worker = new Worker('tasks', async (job) => {
  try {
    return await performOperation(job.data);
  } catch (error) {
    if (error.retriable) {
      throw error;  // BullMQ сделает retry автоматически
    } else {
      // Отправить в DLQ
      await dlqQueue.add('failed-task', {
        originalJob: job.data,
        error: error.message,
        attemptsMade: job.attemptsMade,
      });

      await job.moveToFailed(error, true);  // true = не retry
    }
  }
}, {
  attempts: 3,
  backoff: {
    type: 'exponential',
    delay: 2000,
  },
});
```

### 3. Graceful Shutdown

```javascript
// BullMQ
async function gracefulShutdown() {
  console.log('Shutting down gracefully...');

  await worker.close();  // Дождаться завершения текущих задач
  await queue.close();

  process.exit(0);
}

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);
```

### 4. Resource Cleanup

```javascript
const fs = require('fs/promises');

const fileWorker = new Worker(
  'files',
  async (job) => {
    let tempFilePath;
    try {
      tempFilePath = await downloadFile(job.data.filePath);
      const result = await processFile(tempFilePath);
      await uploadResult(result);
      return result;
    } finally {
      if (tempFilePath) {
        await fs.rm(tempFilePath, { force: true });
      }
    }
  },
  { connection },
);
```

### 5. Task Result Expiration

```javascript
// BullMQ
await queue.add('task', data, {
  removeOnComplete: 100,   // Хранить только 100 последних completed
  removeOnFail: 500,       // Хранить только 500 последних failed
});
```

### 6. Metrics и Monitoring

```javascript
// BullMQ + Prometheus
const client = require('prom-client');

const taskCounter = new client.Counter({
  name: 'bullmq_tasks_total',
  help: 'Total tasks',
  labelNames: ['queue', 'status'],
});

const taskDuration = new client.Histogram({
  name: 'bullmq_task_duration_seconds',
  help: 'Task duration',
  labelNames: ['queue'],
});

worker.on('completed', (job) => {
  taskCounter.labels(job.queueName, 'completed').inc();
  taskDuration.labels(job.queueName).observe(job.finishedOn - job.processedOn);
});

worker.on('failed', (job) => {
  taskCounter.labels(job.queueName, 'failed').inc();
});
```

## Celery vs BullMQ

| Критерий | Celery | BullMQ |
|----------|--------|--------|
| **Язык** | Python | Node.js |
| **Broker** | Redis, RabbitMQ, SQS | Redis only |
| **Features** | Rich (canvas, chains, chords) | Modern, fast |
| **Scheduling** | Celery Beat | Built-in (repeat) |
| **UI** | Flower | Bull Board |
| **Performance** | Good | Excellent |
| **Complexity** | Medium | Lower |
| **Ecosystem** | Mature | Growing |
| **Best for** | Python apps, complex workflows | Node.js apps, simplicity |

## Практический пример: E-commerce Order Processing

### Celery версия

### BullMQ версия

```javascript
const { Queue, Worker, FlowProducer } = require('bullmq');

const orderQueue = new Queue('orders');
const paymentQueue = new Queue('payments');
const inventoryQueue = new Queue('inventory');
const notificationQueue = new Queue('notifications');

// Flow: последовательная обработка
async function processOrder(orderId) {
  const flow = await flowProducer.add({
    name: 'validate-order',
    queueName: 'orders',
    data: { orderId },
    children: [
      {
        name: 'charge-payment',
        queueName: 'payments',
        data: { orderId },
        children: [
          {
            name: 'update-inventory',
            queueName: 'inventory',
            data: { orderId },
            children: [
              {
                name: 'send-email',
                queueName: 'notifications',
                data: { orderId, type: 'email' },
              },
              {
                name: 'send-sms',
                queueName: 'notifications',
                data: { orderId, type: 'sms' },
              },
              {
                name: 'notify-warehouse',
                queueName: 'notifications',
                data: { orderId, type: 'warehouse' },
              },
            ],
          },
        ],
      },
    ],
  });

  return flow;
}

// Workers
new Worker('orders', async (job) => {
  const { orderId } = job.data;
  const order = await db.getOrder(orderId);

  if (order.total <= 0 || !order.items.length) {
    throw new Error('Invalid order');
  }

  return { orderId, validated: true };
});

new Worker('payments', async (job) => {
  const { orderId } = job.data;
  const order = await db.getOrder(orderId);

  const paymentId = await stripe.charge(order.total, order.paymentMethod);
  await db.updateOrder(orderId, { paymentId });

  return { orderId, paymentId };
});

new Worker('inventory', async (job) => {
  const { orderId } = job.data;
  const order = await db.getOrder(orderId);

  for (const item of order.items) {
    await inventory.decrease(item.productId, item.quantity);
  }

  return { orderId, inventoryUpdated: true };
});

new Worker('notifications', async (job) => {
  const { orderId, type } = job.data;

  switch (type) {
    case 'email':
      await sendConfirmationEmail(orderId);
      break;
    case 'sms':
      await sendSMS(orderId);
      break;
    case 'warehouse':
      await notifyWarehouse(orderId);
      break;
  }

  return { orderId, notificationType: type, sent: true };
});
```

## Выводы

Async Workers — критический компонент современных веб-приложений:

**Ключевые преимущества:**
- **Responsiveness**: быстрый ответ пользователю
- **Scalability**: независимое масштабирование воркеров
- **Reliability**: retry механизмы и DLQ
- **Decoupling**: разделение concerns
- **Resource optimization**: эффективное использование ресурсов

**Celery** — зрелое решение для Python с богатым функционалом.
**BullMQ** — современное и быстрое решение для Node.js.

Выбор зависит от стека технологий и сложности workflows.

## Что читать дальше?

- [Урок 24: WebSockets в production](24-websockets-production.md)
- [Урок 21: Message Queues](21-message-queues.md)
- [Урок 22: Kafka и Event Streaming](22-kafka-event-streaming.md)

## Проверь себя

1. В чём разница между синхронной и асинхронной обработкой задач?
2. Какие компоненты входят в архитектуру Celery?
3. Как реализовать retry с exponential backoff?
4. Что такое Celery Beat и зачем он нужен?
5. Чем отличается chain от group в Celery?
6. Как обеспечить idempotency при обработке задач?
7. Что такое Dead Letter Queue и когда его использовать?
8. Как реализовать graceful shutdown для воркеров?
9. В чём разница между Celery и BullMQ?
10. Когда использовать async workers вместо синхронной обработки?

---

[← Предыдущий урок: Kafka и Event Streaming](22-kafka-event-streaming.md) | [Следующий урок: WebSockets в production →](24-websockets-production.md)
