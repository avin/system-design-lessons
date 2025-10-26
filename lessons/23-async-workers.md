# Урок 23: Async Workers — Celery, BullMQ

[← Предыдущий урок: Kafka и Event Streaming](22-kafka-event-streaming.md) | [Следующий урок: WebSockets в production →](24-websockets-production.md)

## Введение

Async Workers (асинхронные воркеры) — это паттерн для выполнения фоновых задач вне основного потока обработки запросов. Они позволяют веб-приложению быстро отвечать пользователю, делегируя долгие операции в фон.

**Зачем нужны воркеры?**

Без воркеров:
```python
@app.post("/signup")
def signup(email, password):
    user = create_user(email, password)        # 50ms
    send_welcome_email(email)                  # 2000ms ❌
    generate_thumbnail(user.avatar)            # 500ms ❌
    update_analytics(user)                     # 300ms ❌
    notify_crm(user)                           # 400ms ❌
    return {"user_id": user.id}                # Total: 3250ms
```

С воркерами:
```python
@app.post("/signup")
def signup(email, password):
    user = create_user(email, password)        # 50ms
    send_welcome_email.delay(email)            # 5ms (в очередь)
    generate_thumbnail.delay(user.avatar)      # 5ms
    update_analytics.delay(user)               # 5ms
    notify_crm.delay(user)                     # 5ms
    return {"user_id": user.id}                # Total: 70ms ✅
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

```python
# celery_app.py
from celery import Celery

app = Celery(
    'myapp',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

# Конфигурация
app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    task_track_started=True,
    task_time_limit=300,  # 5 минут max
    worker_prefetch_multiplier=4,
    worker_max_tasks_per_child=1000,  # Рестарт после 1000 задач
)
```

### Базовая задача

```python
# tasks.py
from celery_app import app
import time

@app.task
def send_email(to, subject, body):
    print(f"Sending email to {to}...")
    time.sleep(2)  # Имитация отправки
    print(f"Email sent to {to}")
    return f"Email sent to {to}"

@app.task
def add(x, y):
    return x + y
```

### Запуск воркера

```bash
# Один воркер с 4 процессами
celery -A celery_app worker --loglevel=info --concurrency=4

# С autoscale (min=2, max=10)
celery -A celery_app worker --autoscale=10,2

# С gevent (для I/O-bound задач)
celery -A celery_app worker --pool=gevent --concurrency=100
```

### Вызов задач

```python
# Асинхронный вызов
result = send_email.delay('user@example.com', 'Welcome', 'Hello!')

# Альтернативный синтаксис
result = send_email.apply_async(
    args=['user@example.com', 'Welcome', 'Hello!'],
    countdown=10  # Выполнить через 10 секунд
)

# Получение результата
print(result.id)  # Task ID
print(result.ready())  # False (ещё выполняется)
print(result.get(timeout=10))  # Блокирующее ожидание результата
print(result.status)  # PENDING, STARTED, SUCCESS, FAILURE
```

### Task Options

```python
@app.task(
    bind=True,  # Доступ к self (task instance)
    name='custom_task_name',
    max_retries=3,
    default_retry_delay=60,  # 1 минута
    time_limit=300,  # Hard limit
    soft_time_limit=250,  # Soft limit (exception)
    rate_limit='10/m',  # 10 задач в минуту
    ignore_result=True,  # Не сохранять результат
)
def process_image(self, image_url):
    try:
        # Обработка
        result = download_and_resize(image_url)
        return result
    except Exception as exc:
        # Retry с exponential backoff
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
```

### Retry механизм

```python
from celery.exceptions import Reject

@app.task(bind=True, max_retries=5)
def fetch_data(self, url):
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.json()

    except requests.RequestException as exc:
        # Retry с backoff
        retry_in = 2 ** self.request.retries  # 1, 2, 4, 8, 16 секунд

        raise self.retry(
            exc=exc,
            countdown=retry_in,
            max_retries=5
        )

    except ValueError:
        # Не retryable ошибка → reject
        raise Reject('Invalid JSON response', requeue=False)
```

### Chains и Workflows

```python
from celery import chain, group, chord

# Chain: последовательное выполнение
workflow = chain(
    download_image.s('http://example.com/image.jpg'),
    resize_image.s(800, 600),
    upload_to_s3.s('bucket-name')
)
result = workflow.apply_async()

# Group: параллельное выполнение
job = group(
    send_email.s('user1@example.com'),
    send_email.s('user2@example.com'),
    send_email.s('user3@example.com')
)
result = job.apply_async()

# Chord: group + callback
callback = group(
    process_item.s(item) for item in items
) | aggregate_results.s()

result = callback.apply_async()
```

### Пример: Batch Processing

```python
@app.task
def process_item(item_id):
    item = db.get(item_id)
    result = expensive_computation(item)
    db.save(result)
    return result

@app.task
def aggregate_results(results):
    total = sum(results)
    send_notification(f"Batch completed. Total: {total}")
    return total

# Использование
items = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

workflow = chord(
    group(process_item.s(item_id) for item_id in items),
    aggregate_results.s()
)

result = workflow.apply_async()
```

### Periodic Tasks (Celery Beat)

```python
# celery_app.py
from celery.schedules import crontab

app.conf.beat_schedule = {
    'send-daily-report': {
        'task': 'tasks.send_daily_report',
        'schedule': crontab(hour=9, minute=0),  # Каждый день в 9:00
    },
    'cleanup-old-sessions': {
        'task': 'tasks.cleanup_sessions',
        'schedule': crontab(hour=2, minute=0),  # Каждый день в 2:00
    },
    'update-cache-every-5-minutes': {
        'task': 'tasks.update_cache',
        'schedule': 300.0,  # Каждые 5 минут (в секундах)
    },
    'send-reminder-monday-friday': {
        'task': 'tasks.send_reminder',
        'schedule': crontab(hour=10, minute=0, day_of_week='1-5'),  # Пн-Пт в 10:00
    },
}
```

```bash
# Запуск scheduler
celery -A celery_app beat --loglevel=info
```

### Мониторинг: Flower

```bash
pip install flower

# Запуск веб-интерфейса
celery -A celery_app flower
```

Откройте `http://localhost:5555`:
- Активные воркеры
- Статистика задач
- История выполнения
- Графики производительности

### Пример: Image Processing Service

```python
# tasks.py
from PIL import Image
import boto3
import requests
from io import BytesIO

@app.task(bind=True, max_retries=3)
def process_image_pipeline(self, image_url, user_id):
    try:
        # 1. Download
        response = requests.get(image_url, timeout=30)
        img = Image.open(BytesIO(response.content))

        # 2. Resize
        sizes = [(800, 600), (400, 300), (200, 150)]
        results = []

        for width, height in sizes:
            resized = img.resize((width, height), Image.LANCZOS)

            # 3. Upload to S3
            buffer = BytesIO()
            resized.save(buffer, format='JPEG')
            buffer.seek(0)

            s3 = boto3.client('s3')
            key = f'users/{user_id}/images/{width}x{height}.jpg'

            s3.upload_fileobj(buffer, 'my-bucket', key)

            results.append({
                'size': f'{width}x{height}',
                'url': f'https://my-bucket.s3.amazonaws.com/{key}'
            })

        # 4. Update database
        db.update_user_images(user_id, results)

        return {'user_id': user_id, 'images': results}

    except Exception as exc:
        raise self.retry(exc=exc, countdown=60)

# Web API
@app.post("/upload")
def upload_image(image_url: str, user_id: int):
    task = process_image_pipeline.delay(image_url, user_id)
    return {'task_id': task.id, 'status': 'processing'}

@app.get("/status/{task_id}")
def get_status(task_id: str):
    result = AsyncResult(task_id, app=celery_app)
    return {
        'task_id': task_id,
        'status': result.status,
        'result': result.result if result.ready() else None
    }
```

### Priority Queues

```python
# Конфигурация приоритетных очередей
app.conf.task_routes = {
    'tasks.critical_task': {'queue': 'critical'},
    'tasks.normal_task': {'queue': 'default'},
    'tasks.low_priority': {'queue': 'low'},
}

# Отправка в конкретную очередь
critical_task.apply_async(args=[...], queue='critical')

# Запуск воркеров для разных очередей
# celery -A celery_app worker -Q critical --concurrency=10
# celery -A celery_app worker -Q default --concurrency=5
# celery -A celery_app worker -Q low --concurrency=2
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

```python
# Celery
import redis

redis_client = redis.Redis()

@app.task(bind=True)
def process_payment(self, payment_id):
    # Проверка уже обработан ли платёж
    lock_key = f'payment_lock:{payment_id}'

    if not redis_client.set(lock_key, 'locked', nx=True, ex=3600):
        return {'status': 'already_processed', 'payment_id': payment_id}

    try:
        # Обработка
        result = charge_card(payment_id)
        return result
    finally:
        redis_client.delete(lock_key)
```

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

```python
# Celery
@app.task(bind=True, max_retries=3)
def risky_task(self, data):
    try:
        return perform_operation(data)
    except RetryableError as exc:
        raise self.retry(exc=exc, countdown=60)
    except FatalError:
        # Отправить в DLQ
        dead_letter_queue.send(data, error=str(exc))
        raise Reject('Fatal error', requeue=False)
```

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

```python
# Celery
import signal
import sys

def graceful_shutdown(signum, frame):
    print("Shutting down gracefully...")
    # Celery worker завершит текущие задачи
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
signal.signal(signal.SIGINT, graceful_shutdown)

# Запуск с graceful timeout
# celery -A celery_app worker --time-limit=300 --soft-time-limit=270
```

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

```python
@app.task(bind=True)
def process_file(self, file_path):
    temp_file = None

    try:
        temp_file = download_file(file_path)
        result = process(temp_file)
        upload_result(result)
        return result

    finally:
        # Cleanup в любом случае
        if temp_file and os.path.exists(temp_file):
            os.remove(temp_file)
```

### 5. Task Result Expiration

```python
# Celery
app.conf.result_expires = 3600  # Результаты хранятся 1 час

@app.task(ignore_result=True)  # Не сохранять результат вообще
def log_event(event):
    db.insert(event)
```

```javascript
// BullMQ
await queue.add('task', data, {
  removeOnComplete: 100,   // Хранить только 100 последних completed
  removeOnFail: 500,       // Хранить только 500 последних failed
});
```

### 6. Metrics и Monitoring

```python
# Celery + Prometheus
from prometheus_client import Counter, Histogram

task_counter = Counter('celery_tasks_total', 'Total tasks', ['task_name', 'status'])
task_duration = Histogram('celery_task_duration_seconds', 'Task duration', ['task_name'])

@app.task(bind=True)
def monitored_task(self):
    with task_duration.labels(task_name=self.name).time():
        try:
            result = perform_task()
            task_counter.labels(task_name=self.name, status='success').inc()
            return result
        except Exception:
            task_counter.labels(task_name=self.name, status='failure').inc()
            raise
```

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

```python
from celery import chain, group

@app.task
def validate_order(order_id):
    order = db.get_order(order_id)
    if order.total > 0 and order.items:
        return order_id
    raise ValueError("Invalid order")

@app.task
def charge_payment(order_id):
    order = db.get_order(order_id)
    payment_id = stripe.charge(order.total, order.payment_method)
    db.update_order(order_id, payment_id=payment_id)
    return order_id

@app.task
def update_inventory(order_id):
    order = db.get_order(order_id)
    for item in order.items:
        inventory.decrease(item.product_id, item.quantity)
    return order_id

@app.task
def send_notifications(order_id):
    results = group(
        send_confirmation_email.s(order_id),
        send_sms.s(order_id),
        notify_warehouse.s(order_id),
    ).apply_async()
    return order_id

# Pipeline
workflow = chain(
    validate_order.s(12345),
    charge_payment.s(),
    update_inventory.s(),
    send_notifications.s(),
)

result = workflow.apply_async()
```

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
