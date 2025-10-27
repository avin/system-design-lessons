# Урок 53: Monitoring и Observability

## Введение

Monitoring и Observability (мониторинг и наблюдаемость) — это критически важные практики для поддержания работоспособности production систем. Без правильного мониторинга вы узнаете о проблемах от пользователей, а не от своих систем.

### Разница между Monitoring и Observability

- **Monitoring** — сбор предопределённых метрик и отслеживание известных проблем
- **Observability** — способность понять внутреннее состояние системы по её внешним выходным данным

**Три столпа Observability:**
1. **Metrics** — числовые метрики (CPU, memory, request rate, latency)
2. **Logs** — структурированные события (error logs, access logs)
3. **Traces** — распределённая трассировка запросов через микросервисы

### Зачем это нужно?

1. **Обнаружение проблем** — узнать о проблеме до того, как она повлияет на пользователей
2. **Диагностика** — быстро найти root cause проблемы
3. **Capacity Planning** — понимание, когда нужно масштабироваться
4. **SLA/SLO tracking** — отслеживание соблюдения Service Level Objectives
5. **Business metrics** — понимание бизнес-показателей в реальном времени

---

## Требования

### Функциональные требования

1. **Сбор метрик** — CPU, memory, disk, network, application metrics
2. **Логирование** — централизованный сбор и хранение логов
3. **Alerting** — уведомления при превышении пороговых значений
4. **Dashboards** — визуализация метрик в реальном времени
5. **Distributed Tracing** — трассировка запросов через микросервисы
6. **Health Checks** — проверка работоспособности сервисов

### Нефункциональные требования

1. **Низкий overhead** — мониторинг не должен замедлять систему (< 5% CPU)
2. **Высокая доступность** — мониторинг должен работать даже когда приложение падает
3. **Долгосрочное хранение** — retention метрик и логов (30-90 дней)
4. **Масштабируемость** — поддержка тысяч сервисов и миллионов метрик

### Расчёты (Back-of-the-envelope)

**Дано:**
- 1000 сервисов
- Каждый сервис отправляет 100 метрик каждые 15 секунд
- Логи: 100 запросов/сек на сервис, 1KB на лог-запись

**Метрики:**
- 1000 сервисов × 100 метрик × 4 samples/min = 400,000 samples/min
- В день: 400,000 × 60 × 24 = 576M samples/day
- Хранение (30 дней): 576M × 30 = 17.3B samples
- Storage (8 bytes per sample): 17.3B × 8 = 138GB

**Логи:**
- 1000 сервисов × 100 req/sec × 1KB = 100MB/sec
- В день: 100MB/sec × 86400 = 8.64TB/day
- 30 дней: 8.64TB × 30 = 259TB

**Вывод:** Нужна агрессивная компрессия и sampling для логов, retention policy для метрик.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Applications                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │Service A│  │Service B│  │Service C│  │Service D│    │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘    │
└───────┼────────────┼────────────┼────────────┼──────────┘
        │            │            │            │
        │ metrics    │ logs       │ traces     │
        ▼            ▼            ▼            ▼
┌─────────────────────────────────────────────────────────┐
│              Collection & Aggregation                    │
│  ┌────────────┐  ┌─────────┐  ┌──────────┐             │
│  │ Prometheus │  │ Fluentd │  │  Jaeger  │             │
│  │  (pull)    │  │ (push)  │  │ Collector│             │
│  └─────┬──────┘  └────┬────┘  └────┬─────┘             │
└────────┼──────────────┼────────────┼───────────────────┘
         │              │            │
         ▼              ▼            ▼
┌─────────────────────────────────────────────────────────┐
│                    Storage Layer                         │
│  ┌────────────┐  ┌──────────┐  ┌──────────┐            │
│  │ Prometheus │  │Elasticsearch│ │ Jaeger  │            │
│  │   TSDB     │  │ (ELK Stack)│ │ Storage │            │
│  └─────┬──────┘  └────┬───────┘  └────┬────┘            │
└────────┼──────────────┼───────────────┼─────────────────┘
         │              │               │
         ▼              ▼               ▼
┌─────────────────────────────────────────────────────────┐
│              Visualization & Alerting                    │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Grafana │  │  Kibana  │  │  Jaeger  │               │
│  │         │  │          │  │    UI    │               │
│  └────┬────┘  └──────────┘  └──────────┘               │
└───────┼──────────────────────────────────────────────────┘
        │
        ▼
┌─────────────┐
│ AlertManager│──► PagerDuty / Slack / Email
└─────────────┘
```

---

## Metrics с Prometheus

### Типы метрик

1. **Counter** — монотонно возрастающее значение (requests_total, errors_total)
2. **Gauge** — значение, которое может увеличиваться и уменьшаться (memory_usage, active_connections)
3. **Histogram** — распределение значений (request_duration_seconds)
4. **Summary** — похож на Histogram, но с квантилями

### Инструментирование приложения (Node.js)

```javascript
const express = require('express');
const promClient = require('prom-client');

const app = express();

// Enable default metrics (CPU, memory, etc.)
promClient.collectDefaultMetrics({ timeout: 5000 });

// Custom metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5]
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

const activeConnections = new promClient.Gauge({
  name: 'active_connections',
  help: 'Number of active connections'
});

const dbQueryDuration = new promClient.Histogram({
  name: 'db_query_duration_seconds',
  help: 'Duration of database queries',
  labelNames: ['operation', 'table'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1]
});

// Middleware для автоматического трекинга HTTP метрик
app.use((req, res, next) => {
  const start = Date.now();
  activeConnections.inc();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route ? req.route.path : req.path;

    httpRequestDuration.observe(
      { method: req.method, route, status_code: res.statusCode },
      duration
    );

    httpRequestTotal.inc({
      method: req.method,
      route,
      status_code: res.statusCode
    });

    activeConnections.dec();
  });

  next();
});

// Business metrics
const ordersTotal = new promClient.Counter({
  name: 'orders_total',
  help: 'Total number of orders',
  labelNames: ['status']
});

const orderValue = new promClient.Histogram({
  name: 'order_value_dollars',
  help: 'Order value in dollars',
  buckets: [10, 50, 100, 500, 1000, 5000]
});

// Example endpoint
app.post('/api/orders', async (req, res) => {
  try {
    const order = req.body;

    // Track DB query time
    const dbStart = Date.now();
    await db.query('INSERT INTO orders VALUES ($1, $2)', [order.id, order.amount]);
    dbQueryDuration.observe(
      { operation: 'insert', table: 'orders' },
      (Date.now() - dbStart) / 1000
    );

    // Business metrics
    ordersTotal.inc({ status: 'created' });
    orderValue.observe(order.amount);

    res.json({ success: true });
  } catch (error) {
    ordersTotal.inc({ status: 'failed' });
    res.status(500).json({ error: error.message });
  }
});

// Metrics endpoint для Prometheus
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Scraping node exporter (system metrics)
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  # Scraping application metrics
  - job_name: 'api-service'
    static_configs:
      - targets:
        - 'api-1:3000'
        - 'api-2:3000'
        - 'api-3:3000'
    metrics_path: '/metrics'
    scrape_interval: 10s

  # Service discovery в Kubernetes
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

# Alerting rules
rule_files:
  - 'alerts.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

### Alerting Rules

```yaml
# alerts.yml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value }} (threshold: 0.05)"

      # High latency (p95)
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.instance }}"
          description: "P95 latency is {{ $value }}s (threshold: 1s)"

      # Low availability
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "Instance {{ $labels.instance }} is unreachable"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
          / node_memory_MemTotal_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanizePercentage }}"

      # Disk space
      - alert: DiskSpaceLow
        expr: |
          (node_filesystem_avail_bytes{mountpoint="/"}
          / node_filesystem_size_bytes{mountpoint="/"}) < 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Only {{ $value | humanizePercentage }} disk space left"

  - name: business_alerts
    interval: 60s
    rules:
      # Drop in order rate
      - alert: OrderRateDrop
        expr: |
          rate(orders_total[5m]) < 0.5 * rate(orders_total[1h] offset 1h)
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Order rate dropped significantly"
          description: "Current rate: {{ $value }}"

      # High order failure rate
      - alert: HighOrderFailureRate
        expr: |
          rate(orders_total{status="failed"}[5m])
          / rate(orders_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High order failure rate"
          description: "Failure rate: {{ $value | humanizePercentage }}"
```

### AlertManager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h

  routes:
    # Critical alerts -> PagerDuty
    - match:
        severity: critical
      receiver: pagerduty
      continue: true

    # Warnings -> Slack
    - match:
        severity: warning
      receiver: slack

receivers:
  - name: 'default'
    email_configs:
      - to: 'ops-team@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'

  - name: 'slack'
    slack_configs:
      - channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'
        description: '{{ .GroupLabels.alertname }}'

inhibit_rules:
  # Если сервис down, не алертить про latency
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

---

## Structured Logging

### Best Practices

```javascript
const winston = require('winston');

// Structured logging с Winston
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'api-service',
    version: process.env.APP_VERSION,
    hostname: require('os').hostname()
  },
  transports: [
    // Console для локальной разработки
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    // File для production
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 10485760, // 10MB
      maxFiles: 5
    }),
    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 10485760,
      maxFiles: 5
    })
  ]
});

// Request logging middleware
const requestLogger = (req, res, next) => {
  const start = Date.now();
  const requestId = req.headers['x-request-id'] || uuid.v4();

  // Add request ID to request object
  req.requestId = requestId;

  // Add request ID to all logs in this request
  req.logger = logger.child({ requestId });

  res.on('finish', () => {
    const duration = Date.now() - start;

    req.logger.info('HTTP Request', {
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration,
      userAgent: req.headers['user-agent'],
      ip: req.ip,
      userId: req.user?.id
    });
  });

  next();
};

app.use(requestLogger);

// Example usage
app.post('/api/orders', async (req, res) => {
  const { userId, items } = req.body;

  req.logger.info('Creating order', { userId, itemCount: items.length });

  try {
    const order = await createOrder(userId, items);

    req.logger.info('Order created successfully', {
      orderId: order.id,
      userId,
      amount: order.totalAmount
    });

    res.json(order);
  } catch (error) {
    req.logger.error('Failed to create order', {
      error: error.message,
      stack: error.stack,
      userId,
      items
    });

    res.status(500).json({ error: 'Failed to create order' });
  }
});

// Unhandled errors
process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection', {
    reason,
    promise
  });
});

process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception', {
    error: error.message,
    stack: error.stack
  });
  process.exit(1);
});
```

### Centralized Logging с ELK Stack

**Fluentd configuration для отправки логов в Elasticsearch:**

```yaml
# fluentd.conf
<source>
  @type tail
  path /var/log/app/*.log
  pos_file /var/log/fluentd/app.log.pos
  tag app.logs
  <parse>
    @type json
    time_key timestamp
    time_format %Y-%m-%dT%H:%M:%S.%L%z
  </parse>
</source>

# Add Kubernetes metadata
<filter app.logs>
  @type kubernetes_metadata
  @id filter_kube_metadata
</filter>

# Parse and enrich
<filter app.logs>
  @type record_transformer
  <record>
    cluster "production"
    environment "prod"
  </record>
</filter>

# Output to Elasticsearch
<match app.logs>
  @type elasticsearch
  host elasticsearch.example.com
  port 9200
  logstash_format true
  logstash_prefix app-logs
  include_tag_key true
  type_name _doc

  <buffer>
    @type file
    path /var/log/fluentd/buffer
    flush_interval 5s
    retry_type exponential_backoff
    retry_max_interval 30
  </buffer>
</match>
```

---

## Distributed Tracing с Jaeger

### Инструментирование приложения

```javascript
const { initTracer } = require('jaeger-client');

// Initialize tracer
const tracer = initTracer({
  serviceName: 'api-service',
  sampler: {
    type: 'probabilistic',
    param: 0.1 // Sample 10% of requests
  },
  reporter: {
    logSpans: true,
    agentHost: 'jaeger-agent',
    agentPort: 6831
  }
}, {
  logger: console
});

// Middleware для создания root span
const tracingMiddleware = (req, res, next) => {
  const parentSpan = tracer.extract('http_headers', req.headers);

  const span = tracer.startSpan('http_request', {
    childOf: parentSpan,
    tags: {
      'span.kind': 'server',
      'http.method': req.method,
      'http.url': req.url
    }
  });

  req.span = span;

  res.on('finish', () => {
    span.setTag('http.status_code', res.statusCode);

    if (res.statusCode >= 500) {
      span.setTag('error', true);
    }

    span.finish();
  });

  next();
};

app.use(tracingMiddleware);

// Example: трассировка через несколько сервисов
app.get('/api/user/:userId/recommendations', async (req, res) => {
  const { userId } = req.params;

  // Child span для database query
  const dbSpan = tracer.startSpan('db_query', {
    childOf: req.span,
    tags: { 'db.type': 'postgresql' }
  });

  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  dbSpan.finish();

  // Child span для вызова другого сервиса
  const recoSpan = tracer.startSpan('recommendation_service_call', {
    childOf: req.span,
    tags: { 'peer.service': 'recommendation-service' }
  });

  const headers = {};
  tracer.inject(recoSpan, 'http_headers', headers);

  const recommendations = await axios.get(
    `http://recommendation-service/api/recommendations/${userId}`,
    { headers }
  );

  recoSpan.setTag('recommendation.count', recommendations.data.length);
  recoSpan.finish();

  res.json({
    user: user.rows[0],
    recommendations: recommendations.data
  });
});

// Wrapper для автоматической трассировки async функций
function traced(name, fn) {
  return async function (...args) {
    const span = tracer.startSpan(name);

    try {
      const result = await fn.apply(this, args);
      return result;
    } catch (error) {
      span.setTag('error', true);
      span.log({ event: 'error', message: error.message });
      throw error;
    } finally {
      span.finish();
    }
  };
}

// Usage
const processOrder = traced('process_order', async (orderId) => {
  const order = await db.query('SELECT * FROM orders WHERE id = $1', [orderId]);
  await sendEmail(order.userId, 'Order confirmation');
  await updateInventory(order.items);
  return order;
});
```

### OpenTelemetry (современный стандарт)

```javascript
const { NodeTracerProvider } = require('@opentelemetry/node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

// Setup tracer
const provider = new NodeTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'api-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0'
  })
});

const exporter = new JaegerExporter({
  endpoint: 'http://jaeger:14268/api/traces'
});

provider.addSpanProcessor(new BatchSpanProcessor(exporter));
provider.register();

// Auto-instrumentation для популярных библиотек
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');
const { PgInstrumentation } = require('@opentelemetry/instrumentation-pg');

registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
    new PgInstrumentation()
  ]
});
```

---

## Health Checks

```javascript
class HealthCheckService {
  constructor(db, redis, kafka) {
    this.db = db;
    this.redis = redis;
    this.kafka = kafka;
  }

  // Liveness probe (Is the app running?)
  async liveness() {
    // Простая проверка, что процесс жив
    return {
      status: 'UP',
      timestamp: new Date().toISOString()
    };
  }

  // Readiness probe (Is the app ready to serve traffic?)
  async readiness() {
    const checks = await Promise.allSettled([
      this.checkDatabase(),
      this.checkRedis(),
      this.checkKafka()
    ]);

    const results = {
      database: checks[0].status === 'fulfilled' ? checks[0].value : { status: 'DOWN', error: checks[0].reason?.message },
      redis: checks[1].status === 'fulfilled' ? checks[1].value : { status: 'DOWN', error: checks[1].reason?.message },
      kafka: checks[2].status === 'fulfilled' ? checks[2].value : { status: 'DOWN', error: checks[2].reason?.message }
    };

    const allHealthy = Object.values(results).every(r => r.status === 'UP');

    return {
      status: allHealthy ? 'UP' : 'DOWN',
      checks: results,
      timestamp: new Date().toISOString()
    };
  }

  async checkDatabase() {
    const start = Date.now();

    try {
      await this.db.query('SELECT 1');
      return {
        status: 'UP',
        responseTime: Date.now() - start
      };
    } catch (error) {
      return {
        status: 'DOWN',
        error: error.message,
        responseTime: Date.now() - start
      };
    }
  }

  async checkRedis() {
    const start = Date.now();

    try {
      await this.redis.ping();
      return {
        status: 'UP',
        responseTime: Date.now() - start
      };
    } catch (error) {
      return {
        status: 'DOWN',
        error: error.message
      };
    }
  }

  async checkKafka() {
    // Kafka health check сложнее
    try {
      const admin = this.kafka.admin();
      await admin.connect();
      await admin.listTopics();
      await admin.disconnect();

      return { status: 'UP' };
    } catch (error) {
      return { status: 'DOWN', error: error.message };
    }
  }
}

// Endpoints
const healthCheck = new HealthCheckService(db, redis, kafka);

app.get('/health/liveness', async (req, res) => {
  const result = await healthCheck.liveness();
  res.json(result);
});

app.get('/health/readiness', async (req, res) => {
  const result = await healthCheck.readiness();
  const statusCode = result.status === 'UP' ? 200 : 503;
  res.status(statusCode).json(result);
});
```

### Kubernetes Health Check Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: api
          image: api-service:1.0.0
          ports:
            - containerPort: 3000

          # Liveness probe (restart if fails)
          livenessProbe:
            httpGet:
              path: /health/liveness
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          # Readiness probe (remove from load balancer if fails)
          readinessProbe:
            httpGet:
              path: /health/readiness
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2

          # Startup probe (для медленного старта)
          startupProbe:
            httpGet:
              path: /health/liveness
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 30  # 5s * 30 = 150s max startup time
```

---

## Grafana Dashboards

### Пример Dashboard Definition (JSON)

```json
{
  "dashboard": {
    "title": "API Service Overview",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{route}}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total{status_code=~\"5..\"}[5m])",
            "legendFormat": "5xx errors"
          }
        ],
        "type": "graph",
        "alert": {
          "conditions": [
            {
              "evaluator": {
                "params": [0.05],
                "type": "gt"
              },
              "operator": {
                "type": "and"
              },
              "query": {
                "params": ["A", "5m", "now"]
              },
              "reducer": {
                "type": "avg"
              },
              "type": "query"
            }
          ],
          "frequency": "60s",
          "handler": 1,
          "name": "High Error Rate Alert"
        }
      },
      {
        "title": "Latency (p50, p95, p99)",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "p50"
          },
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "p99"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Active Connections",
        "targets": [
          {
            "expr": "active_connections",
            "legendFormat": "{{instance}}"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

---

## SLI/SLO/SLA

### Определения

- **SLI (Service Level Indicator)** — метрика, которая измеряет уровень сервиса (latency, error rate, availability)
- **SLO (Service Level Objective)** — целевое значение SLI (например, "99.9% availability")
- **SLA (Service Level Agreement)** — контракт с клиентом с последствиями при нарушении

### Примеры SLI/SLO

```yaml
# SLO Definition
slos:
  - name: api_availability
    description: API должно быть доступно 99.9% времени
    sli:
      metric: up
      threshold: 1
    objective: 0.999
    window: 30d

  - name: api_latency
    description: 95% запросов должны выполняться быстрее 500ms
    sli:
      metric: http_request_duration_seconds
      percentile: 0.95
    objective: 0.5  # 500ms
    window: 30d

  - name: error_budget
    description: Допустимо не более 0.1% ошибок
    sli:
      metric: http_requests_total
      error_query: status_code >= 500
    objective: 0.999
    window: 30d
```

### Error Budget Calculation

```javascript
class SLOTracker {
  constructor(prometheus) {
    this.prometheus = prometheus;
  }

  async calculateErrorBudget(slo, windowDays = 30) {
    const windowSeconds = windowDays * 24 * 60 * 60;

    // Total requests
    const totalRequestsQuery = `sum(increase(http_requests_total[${windowDays}d]))`;
    const totalRequests = await this.prometheus.query(totalRequestsQuery);

    // Failed requests
    const failedRequestsQuery = `sum(increase(http_requests_total{status_code=~"5.."}[${windowDays}d]))`;
    const failedRequests = await this.prometheus.query(failedRequestsQuery);

    // Calculate error rate
    const errorRate = failedRequests / totalRequests;

    // Error budget
    const allowedErrorRate = 1 - slo.objective; // e.g., 1 - 0.999 = 0.001
    const errorBudget = allowedErrorRate - errorRate;
    const errorBudgetPercent = (errorBudget / allowedErrorRate) * 100;

    return {
      totalRequests,
      failedRequests,
      errorRate,
      objective: slo.objective,
      errorBudget,
      errorBudgetPercent,
      status: errorBudget > 0 ? 'OK' : 'EXHAUSTED'
    };
  }

  async calculateAvailability(windowDays = 30) {
    // Uptime calculation
    const uptimeQuery = `avg_over_time(up[${windowDays}d])`;
    const uptime = await this.prometheus.query(uptimeQuery);

    const availability = uptime * 100;

    return {
      availability,
      downtime: (1 - uptime) * windowDays * 24 * 60, // minutes
      objective: 99.9,
      status: availability >= 99.9 ? 'OK' : 'VIOLATED'
    };
  }
}
```

---

## Best Practices

### 1. Golden Signals

Мониторьте 4 ключевых метрики (Google SRE):
- **Latency** — время отклика
- **Traffic** — количество запросов
- **Errors** — процент ошибок
- **Saturation** — загрузка ресурсов (CPU, memory, disk)

### 2. USE Method (для ресурсов)

- **Utilization** — процент использования
- **Saturation** — очередь ожидающих запросов
- **Errors** — количество ошибок

### 3. RED Method (для сервисов)

- **Rate** — запросов в секунду
- **Errors** — процент ошибок
- **Duration** — время выполнения

### 4. Логирование

- Используйте structured logging (JSON)
- Добавляйте request ID для корреляции
- Логируйте на правильном уровне (DEBUG/INFO/WARN/ERROR)
- Не логируйте sensitive data (пароли, токены, PII)

### 5. Tracing

- Используйте sampling для production (не трассируйте 100% запросов)
- Добавляйте custom tags для бизнес-контекста
- Пропагируйте trace context через все сервисы

### 6. Alerting

- Alert на симптомы, а не на причины (alerting on what users care about)
- Каждый alert должен быть actionable
- Избегайте alert fatigue (слишком много ложных срабатываний)
- Используйте severity levels (critical, warning, info)

---

## Trade-offs

| Аспект | Вариант A | Вариант B |
|--------|-----------|-----------|
| **Sampling** | 100% requests | 1-10% sampling |
| Accuracy | Полная картина | Статистическая выборка |
| Overhead | Высокий | Низкий |
| Storage | Большой объём | Меньший объём |
| **Metrics Retention** | 1 year | 30 days |
| Historical analysis | Возможен | Ограничен |
| Storage cost | Высокая | Низкая |
| Query performance | Медленнее | Быстрее |
| **Push vs Pull** | Push (StatsD) | Pull (Prometheus) |
| Network | Outbound от app | Inbound к app |
| Failure handling | Может терять метрики | Нет доставки = видно |
| Service discovery | Не нужно | Нужно |

---

## Что почитать дальше?

1. **Урок 54: Deployment Strategies** — rolling updates, blue-green, canary
2. **Урок 55: Kubernetes в Production** — production-ready k8s
3. **Урок 56: Disaster Recovery** — backup, restore, failover

---

## Проверь себя

1. В чём разница между Monitoring и Observability?
2. Что такое "три столпа observability"?
3. Какие типы метрик бывают в Prometheus (Counter, Gauge, Histogram, Summary)?
4. Что такое Error Budget и как его рассчитывать?
5. В чём разница между Liveness и Readiness probes?
6. Что такое Golden Signals?
7. Как правильно настроить alerting, чтобы избежать alert fatigue?
8. Зачем нужен distributed tracing?

---
**Предыдущий урок**: [Урок 52: Recommendation System — Система рекомендаций](52-recommendation-system.md)
**Следующий урок**: [Урок 54: Deployment Strategies](54-deployment.md)
