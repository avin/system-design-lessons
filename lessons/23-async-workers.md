# Урок 23: Async Workers

## Введение

Асинхронные воркеры позволяют выносить длительные или ресурсозатратные операции из основного веб-потока. Вместо того чтобы блокировать ответ пользователю, мы отправляем задачу в очередь и обрабатываем её отдельно. Такой подход улучшает латентность, повышает надёжность и упрощает масштабирование.

В этом уроке мы разберём:

- чем фоновые воркеры отличаются от синхронной обработки;
- базовую архитектуру очередей задач;
- инструменты из экосистемы Node.js (BullMQ, Agenda, Bree, Worker Threads);
- надёжность: повторные попытки, идемпотентность, Dead Letter Queue;
- наблюдаемость и планирование ресурсов;
- практический пример обработки заказов в e-commerce.

## Когда нужны воркеры?

### Синхронный сценарий

```javascript
// Синхронная обработка — весь pipeline внутри HTTP-обработчика
app.post('/signup', async (req, res) => {
  const { email, password } = req.body;

  const user = await createUser(email, password);           // 50 ms
  await sendWelcomeEmail(email);                            // 2 000 ms
  await generateAvatarThumbnail(user.avatarUrl);            // 500 ms
  await pushToCrm(user);                                    // 400 ms
  await updateAnalytics(user);                              // 300 ms

  res.json({ id: user.id });                                // ~3.3 секунды
});
```

Главные проблемы:

- клиент ждёт завершения всех операций;
- одна из долгих функций может сорвать весь запрос;
- горизонтальное масштабирование ограничено, потому что каждый веб-инстанс тратит много времени на фоновые действия.

### Асинхронный сценарий

```javascript
const { Queue } = require('bullmq');
const signupQueue = new Queue('signup', { connection: { host: '127.0.0.1', port: 6379 } });

app.post('/signup', async (req, res) => {
  const { email, password } = req.body;

  const user = await createUser(email, password); // 50 ms

  await signupQueue.add('welcome-email', { email });
  await signupQueue.add('generate-thumbnail', { avatarUrl: user.avatarUrl });
  await signupQueue.add('notify-crm', { userId: user.id });
  await signupQueue.add('analytics', { userId: user.id });

  res.json({ id: user.id }); // ~70 ms
});
```

Всё, что не нужно делать моментально, уходит в очередь. Рабочие (workers) обрабатывают задания независимо, можно настраивать количество воркеров, изолировать ошибки, масштабировать по нагрузке.

## Архитектура асинхронной обработки

```
HTTP/API → Очередь задач → Воркеры → Внешние сервисы
            ↑ кеш/база    ↑ DLQ
```

Компоненты:

- **Producer** — создаёт задачу (чаще всего веб-сервис);
- **Broker** — хранит очередь (Redis, RabbitMQ, Kafka, SQS, NATS);
- **Worker** — читает задачи из очереди, выполняет логику, сообщает результат;
- **Result storage** (опционально) — складывает результат выполнения;
- **Dead Letter Queue** (DLQ) — отдельная очередь для задач, которые не удалось обработать.

## Выбор брокера и инструментов

| Broker / инструмент | Плюсы | Минусы | Подходит когда |
|---------------------|-------|--------|----------------|
| **Redis + BullMQ** | Простой запуск, высокая скорость, хорошая экосистема | Телеметрия/кластеризация нужно добавлять вручную | Веб-приложения, микросервисы, когда важна скорость внедрения |
| **RabbitMQ (AMQP)** | Богатая маршрутизация, подтверждения, TTL, DLX | Сложнее поддерживать, нужна эксплуатация | Микросервисы, сложные топологии, on-prem |
| **Kafka** | Масштабирование, репликация, ретензия | Более тяжёлый, нужен кластер | Event sourcing, потоковые события |
| **SQS / Cloud Tasks** | Managed-сервис, нет эксплуатации | Лимиты AWS, доп. стоимость | Серверлесс, API без инфраструктуры брокера |

Node.js-библиотеки:

- **BullMQ** — де-факто стандарт для Redis (поддерживает flows, повторения, приоритеты);
- **Agenda** (MongoDB) — фокус на планировщике задач;
- **Bree** — cron/JS jobs, lightweight;
- **Worker Threads** — для CPU-задач в рамках одного процесса;
- **Custom + AMQP/Kafka** — когда нужен собственный протокол/формат.

В этом уроке сосредоточимся на BullMQ, как наиболее распространённом решении.

## BullMQ: базовые элементы

### Установка

```bash
npm install bullmq ioredis
```

### Очередь задач

```javascript
const { Queue } = require('bullmq');

const connection = { host: '127.0.0.1', port: 6379 };
const emailQueue = new Queue('emails', {
  connection,
  defaultJobOptions: {
    removeOnComplete: { age: 3600, count: 1000 },
    attempts: 5,
    backoff: { type: 'exponential', delay: 2000 },
    timeout: 30_000,
  },
});
```

### Воркеры

```javascript
const { Worker } = require('bullmq');

const emailWorker = new Worker(
  'emails',
  async (job) => {
    const { to, subject, body } = job.data;
    await sendEmail({ to, subject, body });
    return { deliveredAt: Date.now() };
  },
  { connection, concurrency: 10 },
);

emailWorker.on('completed', (job, result) => {
  console.log(`Email to ${job.data.to} delivered`, result);
});

emailWorker.on('failed', (job, err) => {
  console.error(`Failed to deliver email to ${job.data.to}`, err);
});
```

### Пакетная загрузка работ

```javascript
// Массовая постановка задач
await emailQueue.addBulk(
  users.map((user) => ({
    name: 'newsletter',
    data: { to: user.email, subject: 'Weekly digest', body: renderTemplate(user) },
    opts: { priority: user.isVip ? 1 : 5 },
  })),
);
```

### Повторяющиеся задания

```javascript
await emailQueue.add(
  'daily-report',
  {},
  {
    repeat: { cron: '0 8 * * *', tz: 'Europe/Moscow' }, // каждый день в 08:00
    jobId: 'daily-report',
  },
);
```

### Flows (цепочки/граф)

```javascript
const { FlowProducer } = require('bullmq');

const flowProducer = new FlowProducer({ connection });

await flowProducer.add({
  name: 'process-order',
  queueName: 'orders',
  data: { orderId: 123 },
  children: [
    {
      name: 'charge-payment',
      queueName: 'payments',
      data: { orderId: 123 },
      children: [
        {
          name: 'notify-services',
          queueName: 'notifications',
          data: { orderId: 123 },
          children: [
            { name: 'send-email', queueName: 'emails', data: { type: 'order-confirmation', orderId: 123 } },
            { name: 'notify-crm', queueName: 'crm', data: { orderId: 123 } },
          ],
        },
      ],
    },
  ],
});
```

### Ограничение скорости и токены

```javascript
const rateLimitedQueue = new Queue('sms', {
  connection,
  limiter: {
    max: 30,          // не больше 30 задач
    duration: 1000,   // за 1 секунду
  },
});
```

### Dead Letter Queue (DLQ)

```javascript
const dlqQueue = new Queue('emails-dlq', { connection });

const worker = new Worker(
  'emails',
  async (job) => { /* ... */ },
  {
    connection,
    attempts: 5,
    backoff: { type: 'fixed', delay: 15_000 },
  },
);

worker.on('failed', async (job, error) => {
  if (job.attemptsMade >= job.opts.attempts) {
    await dlqQueue.add('email-failed', {
      jobId: job.id,
      payload: job.data,
      reason: error.message,
    });
  }
});
```

## Другие инструменты из мира Node.js

### Agenda (MongoDB)

- хранит задачи в коллекции MongoDB;
- удобна для cron/periodic jobs;
- подходит, если уже используете MongoDB и нет высоких требований к скорости.

```javascript
const Agenda = require('agenda');
const agenda = new Agenda({ db: { address: process.env.MONGO_URL } });

agenda.define('send notification', async (job) => {
  await sendNotification(job.attrs.data);
});

await agenda.start();
await agenda.every('10 minutes', 'send notification', { type: 'reminder' });
```

### Bree

Bree — простой cron-планировщик (создаёт worker threads). Удобно для сценариев, где задачи — отдельные файлы.

```javascript
const Bree = require('bree');
const bree = new Bree({
  jobs: [
    { name: 'backup', interval: 'at 03:00' },
    { name: 'cache-warmup', timeout: '5m', interval: '1h' },
  ],
});

bree.start();
```

### Worker Threads для CPU-bound

Если задача забивает одно ядро, даже вынесенная в очередь, она всё равно блокирует event loop. Worker Threads — альтернатива: создаём pool и направляем тяжёлые операции туда.

```javascript
const { Worker } = require('node:worker_threads');

function runInWorker(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./workers/heavy-task.js', { workerData: data });
    worker.once('message', resolve);
    worker.once('error', reject);
    worker.once('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker stopped with exit code ${code}`));
    });
  });
}
```

## Надёжность

### Идемпотентность

- задавайте `jobId`, чтобы предотвращать дубли;
- сохраняйте факт обработки в базе/Redis;
- используйте «dedupe-ключи» (message key).

```javascript
await paymentsQueue.add(
  'charge-card',
  { paymentId: 'pay_123' },
  { jobId: 'payment:pay_123' },
);
```

### Повторные попытки

- настраивайте `attempts` и `backoff` (fixed/exponential/custom);
- учитывайте ошибки, которые не стоит ретраить (валидация, «пользователь не найден»).

```javascript
await queue.add('fetch-report', { reportId }, {
  attempts: 6,
  backoff: {
    type: 'custom',
    delay: (attemptsMade) => Math.min(2 ** attemptsMade * 1000, 60_000),
  },
});
```

### Circuit breaker / Drop

```javascript
const { RateLimiterMemory } = require('rate-limiter-flexible');

const breaker = new RateLimiterMemory({ points: 10, duration: 60 });

async function guardedJob(job) {
  try {
    await breaker.consume('external-service');
    return await callExternalService(job.data);
  } catch (error) {
    if (error instanceof RateLimiterMemory.RateLimiterRes) {
      throw new Error('Circuit open, retry later');
    }
    throw error;
  }
}
```

### DLQ и алерты

Раз в N минут опрашивайте DLQ, отправляйте уведомления в Slack/Email, сохраняйте статистику.

```javascript
const stuckJobs = await dlqQueue.getJobs(['waiting', 'delayed']);
if (stuckJobs.length > 0) {
  await notifyOnCall({ queue: 'emails-dlq', count: stuckJobs.length });
}
```

## Наблюдаемость (Observability)

### Метрики

```javascript
const promClient = require('prom-client');

const jobsProcessed = new promClient.Counter({
  name: 'queue_jobs_total',
  help: 'Total jobs processed',
  labelNames: ['queue', 'status'],
});

const jobDuration = new promClient.Histogram({
  name: 'queue_job_duration_seconds',
  help: 'Job duration',
  labelNames: ['queue'],
  buckets: [0.05, 0.1, 0.25, 0.5, 1, 2, 5, 10, 30],
});

worker.on('completed', (job) => {
  jobsProcessed.labels(job.queueName, 'completed').inc();
  jobDuration.labels(job.queueName).observe((job.finishedOn - job.processedOn) / 1000);
});

worker.on('failed', (job) => {
  jobsProcessed.labels(job.queueName, 'failed').inc();
});
```

### Логи

- логируйте id задачи, payload, попытки;
- используйте correlation-id/trace-id для связи с HTTP-запросом;
- храните логи в централизованном хранилище (ELK, Loki, Datadog).

### Bull Board / Arena

```javascript
const express = require('express');
const { createBullBoard } = require('@bull-board/api');
const { BullMQAdapter } = require('@bull-board/api/bullMQAdapter');
const { ExpressAdapter } = require('@bull-board/express');

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/queues');

createBullBoard({
  queues: [
    new BullMQAdapter(emailQueue),
    new BullMQAdapter(signupQueue),
  ],
  serverAdapter,
});

const app = express();
app.use('/queues', serverAdapter.getRouter());
app.listen(3000, () => console.log('Bull Board: http://localhost:3000/queues'));
```

## Масштабирование

### Вертикальное

- увеличиваем concurrency в рамках одного воркера (`concurrency: 50`);
- используем worker threads для CPU-задач.

### Горизонтальное

- поднимаем больше экземпляров воркеров (в Docker/Kubernetes/PM2);
- следим за лимитами Redis (подключения и память);
- используем Redis Cluster либо sharding по типам задач.

### Авто-масштабирование

- метрики (Jobs waiting, обработка > X секунд);
- в Kubernetes — HPA + кастомные metrics;
- в AWS — Lambda + SQS (serverless) или ECS Fargate.

### Тонкости

- не забывайте про «произвольный порядок» — одна задача может выполняться несколько раз, поэтому все обработчики должны быть идемпотентными;
- Heavy users: разделяйте очереди по типам задач (email / CRM / analytics), чтобы медленные работы не забивали быстрые;
- максимальный размер payload — не храните большие файлы; лучше складывайте их в S3 и кладите ссылку в job.data.

## Практический пример: обработка заказа

### Очереди

```javascript
const queues = {
  orders: new Queue('orders', { connection }),
  payments: new Queue('payments', { connection }),
  inventory: new Queue('inventory', { connection }),
  notifications: new Queue('notifications', { connection }),
};
```

### Постановка pipeline

```javascript
const flow = new FlowProducer({ connection });

async function processOrder(orderId) {
  await flow.add({
    name: 'validate-order',
    queueName: 'orders',
    data: { orderId },
    children: [
      {
        name: 'charge-payment',
        queueName: 'payments',
        data: { orderId },
        opts: { jobId: `payment:${orderId}` },
        children: [
          {
            name: 'reserve-inventory',
            queueName: 'inventory',
            data: { orderId },
            children: [
              { name: 'send-confirmation-email', queueName: 'notifications', data: { orderId } },
              { name: 'notify-warehouse', queueName: 'notifications', data: { orderId, channel: 'webhook' } },
            ],
          },
        ],
      },
    ],
  });
}
```

### Воркеры

```javascript
new Worker('orders', async (job) => {
  const order = await OrdersRepository.get(job.data.orderId);
  if (!order.items.length || order.total <= 0) {
    throw new Error('Invalid order');
  }
  return { valid: true };
}, { connection });

new Worker('payments', async (job) => {
  const { orderId } = job.data;
  const order = await OrdersRepository.get(orderId);
  const payment = await PaymentGateway.charge(order);
  await OrdersRepository.setPayment(orderId, payment.id);
  return { paymentId: payment.id };
}, { connection, attempts: 3, backoff: { type: 'exponential', delay: 5_000 } });

new Worker('inventory', async (job) => {
  const order = await OrdersRepository.get(job.data.orderId);
  await InventoryService.reserve(order.items);
}, { connection });

new Worker('notifications', async (job) => {
  if (job.name === 'send-confirmation-email') {
    await EmailService.sendConfirmation(job.data.orderId);
  } else if (job.name === 'notify-warehouse') {
    await WarehouseClient.notify(job.data.orderId);
  }
}, { connection });
```

### Мониторинг DLQ

```javascript
const failed = await queues.notifications.getJobs(['failed'], 0, 50);
for (const job of failed) {
  await alertOpsTeam({
    queue: job.queueName,
    jobId: job.id,
    error: job.failedReason,
    payload: job.data,
  });
}
```

## Практические советы

1. **Храните минимум данных в задачах** — лучше передавать ID и заново извлекать state (из БД, кеша).
2. **Используйте версии схемы** в payload (`{ version: 2, ... }`), чтобы воркеры могли работать с несколькими версиями.
3. **Прозрачные ретраи** — логируйте сведения о попытках (`job.attemptsMade`), сохраняйте их в метриках.
4. **Graceful shutdown** — закрывайте воркеры корректно.

```javascript
async function shutdown() {
  await emailWorker.close();
  await queues.emails.close();
  process.exit(0);
}

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

5. **Разделяйте очереди**: быстрые и медленные задачи, разные SLA, приоритеты.
6. **Дублируйте критичные события** — если часть pipeline критична, публикуйте событие в Kafka/SQS как вторичную защиту.
7. **Тестируйте failure-сценарии** — имитация недоступности Redis, внешних API, ошибочных payload.

## Чек-лист перед продом

- [ ] Очереди отделены по доменам и SLA.
- [ ] Все обработчики идемпотентны (или используют `jobId`/locks).
- [ ] Настроены повторные попытки, backoff, DLQ.
- [ ] Есть мониторинг (Bull Board, метрики, алерты).
- [ ] Воркеры умеют graceful shutdown, перезапуск без потери данных.
- [ ] Есть стратегия масштабирования (HPA, auto-scaling, вручную).
- [ ] Документация описывает формат сообщений и схемы.

## Выводы

- Асинхронные воркеры разгружают основное приложение и улучшают пользовательский опыт.
- Node.js предлагает богатый набор инструментов: BullMQ — основной выбор, но есть и другие опции.
- Надёжность — ретраевые стратегии, DLQ, идемпотентность — обязательны в продакшене.
- Переход на асинхронную обработку требует наблюдаемости и планирования ресурсов, но окупается повышением устойчивости и масштабируемости.

---
**Предыдущий урок**: [Урок 22: Kafka и Event Streaming](22-kafka-event-streaming.md)
**Следующий урок**: [Урок 24: WebSockets в production](24-websockets-production.md)
