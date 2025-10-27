# Урок 26: Service Mesh — Istio, Linkerd


## Введение

Service Mesh — это выделенная инфраструктурная прослойка для управления коммуникацией между микросервисами. Вместо того чтобы каждый сервис сам реализовывал retry logic, circuit breaking, mTLS и т.д., эти функции выносятся в отдельный слой.

**Без Service Mesh:**
```
Service A → [код retry, circuit breaker, mTLS] → Service B
```

**С Service Mesh:**
```
Service A → Sidecar Proxy → Sidecar Proxy → Service B
              ↓                    ↓
         Control Plane управляет политиками
```

**Проблемы, которые решает Service Mesh:**
- Retry и timeout логика дублируется в каждом сервисе
- Distributed tracing требует инструментации кода
- mTLS между сервисами сложно настроить
- Нет единого способа управления traffic
- Сложно A/B тестировать и canary deployments

## Архитектура Service Mesh

### Data Plane vs Control Plane

```
┌─────────────────────────────────────────┐
│         Control Plane                   │
│  (Istio Pilot, Linkerd Controller)      │
│  - Configuration                         │
│  - Service Discovery                     │
│  - Certificate Management                │
└────────────┬────────────────────────────┘
             │ Push config
    ┌────────┼────────┬────────┐
    ↓        ↓        ↓        ↓
┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
│Sidecar│ │Sidecar│ │Sidecar│ │Sidecar│  ← Data Plane
│ Proxy │ │ Proxy │ │ Proxy │ │ Proxy │     (Envoy)
└───┬───┘ └───┬───┘ └───┬───┘ └───┬───┘
    │         │         │         │
┌───┴───┐ ┌───┴───┐ ┌───┴───┐ ┌───┴───┐
│Service│ │Service│ │Service│ │Service│
│   A   │ │   B   │ │   C   │ │   D   │
└───────┘ └───────┘ └───────┘ └───────┘
```

**Data Plane (Envoy Proxy):**
- Перехватывает весь network traffic
- Применяет политики (retry, timeout, circuit breaking)
- Собирает метрики и trace
- Шифрует трафик (mTLS)

**Control Plane:**
- Управляет конфигурацией sidecar proxy
- Service discovery
- Certificate authority для mTLS
- Телеметрия и мониторинг

### Sidecar Pattern

```
Pod (Kubernetes):
┌─────────────────────────┐
│  ┌──────────────────┐   │
│  │  Application     │   │
│  │  Container       │   │
│  └────────┬─────────┘   │
│           │ localhost   │
│  ┌────────┴─────────┐   │
│  │  Envoy Proxy     │   │
│  │  (Sidecar)       │   │
│  └──────────────────┘   │
└─────────────────────────┘
```

Каждый pod получает sidecar container с proxy (обычно Envoy).

## Istio

Istio — самый популярный Service Mesh.

### Установка Istio (Kubernetes)

```bash
# Скачать Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.20.0

# Установить istioctl
export PATH=$PWD/bin:$PATH

# Установить Istio в кластер
istioctl install --set profile=demo -y

# Включить автоматический sidecar injection
kubectl label namespace default istio-injection=enabled
```

### Пример приложения

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders
      version: v1
  template:
    metadata:
      labels:
        app: orders
        version: v1
    spec:
      containers:
        - name: orders
          image: myregistry/orders-service:v1
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: orders-service
spec:
  selector:
    app: orders
  ports:
    - port: 80
      targetPort: 8080
```

После apply, Istio автоматически добавит sidecar:

```bash
kubectl apply -f deployment.yaml

# Проверка sidecar injection
kubectl get pods
# NAME                              READY   STATUS
# orders-service-xxx                2/2     Running  ← 2 контейнера!
```

### Traffic Management

#### Virtual Service: Routing

```yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-routing
spec:
  hosts:
    - orders-service
  http:
    # 90% трафика на v1
    - match:
        - headers:
            end-user:
              exact: debug
      route:
        - destination:
            host: orders-service
            subset: v2
    # Canary deployment: 10% на v2
    - route:
        - destination:
            host: orders-service
            subset: v1
          weight: 90
        - destination:
            host: orders-service
            subset: v2
          weight: 10
```

#### Destination Rule: Subsets и Load Balancing

```yaml
# destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: orders-destination
spec:
  host: orders-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST  # или ROUND_ROBIN, RANDOM
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        connectionPool:
          tcp:
            maxConnections: 50
    - name: v2
      labels:
        version: v2
```

### Resilience Patterns

#### Retry

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-retry
spec:
  hosts:
    - orders-service
  http:
    - route:
        - destination:
            host: orders-service
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure,refused-stream
```

#### Timeout

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-timeout
spec:
  hosts:
    - orders-service
  http:
    - route:
        - destination:
            host: orders-service
      timeout: 5s
```

#### Circuit Breaker

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: orders-circuit-breaker
spec:
  host: orders-service
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 2
```

**Как работает:**
- 5 последовательных ошибок → instance выкидывается из pool
- На 30 секунд (baseEjectionTime)
- Максимум 50% instances могут быть выброшены
- Минимум 50% должны быть здоровы

### Security: mTLS

#### Автоматический mTLS

```yaml
# peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT  # STRICT, PERMISSIVE, DISABLE
```

**STRICT**: только mTLS трафик
**PERMISSIVE**: mTLS и plaintext (для миграции)
**DISABLE**: отключить mTLS

Istio автоматически:
- Генерирует сертификаты для каждого сервиса
- Ротирует сертификаты
- Шифрует трафик между sidecar

#### Authorization Policy

```yaml
# authorization-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: orders-authz
  namespace: default
spec:
  selector:
    matchLabels:
      app: orders
  action: ALLOW
  rules:
    # Только authenticated users
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/payment-service"]
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/orders"]
    # Public endpoints
    - to:
        - operation:
            methods: ["GET"]
            paths: ["/api/orders/*/status"]
```

### Observability

#### Distributed Tracing (Jaeger)

Istio автоматически добавляет trace headers и отправляет spans в Jaeger:

```bash
# Установить Jaeger addon
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# Открыть UI
istioctl dashboard jaeger
```

Trace показывает путь запроса через все сервисы:

```
API Gateway (10ms)
  → Auth Service (5ms)
  → Orders Service (50ms)
      → Payment Service (30ms)
      → Inventory Service (15ms)
  → Notification Service (8ms)

Total: 118ms
```

#### Metrics (Prometheus & Grafana)

```bash
# Установить Prometheus и Grafana
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml

# Открыть Grafana
istioctl dashboard grafana
```

Istio собирает метрики:
- Request rate, latency, error rate (RED metrics)
- Connection pool stats
- Circuit breaker events
- mTLS status

#### Kiali: Service Graph

```bash
# Установить Kiali
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml

# Открыть UI
istioctl dashboard kiali
```

Kiali визуализирует:
- Service topology (граф зависимостей)
- Traffic flow
- Error rates
- Latency

### Traffic Splitting: Canary Deployment

```yaml
# Canary: 95% v1, 5% v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-canary
spec:
  hosts:
    - orders-service
  http:
    - route:
        - destination:
            host: orders-service
            subset: v1
          weight: 95
        - destination:
            host: orders-service
            subset: v2
          weight: 5
```

Постепенное увеличение трафика:

```bash
# 5% → 25% → 50% → 100%

# Шаг 1: 5% (выше)
kubectl apply -f canary-5.yaml

# Мониторинг метрик v2

# Шаг 2: 25%
kubectl apply -f canary-25.yaml

# Шаг 3: 50%
kubectl apply -f canary-50.yaml

# Шаг 4: 100%
kubectl apply -f canary-100.yaml
```

### A/B Testing

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-ab-test
spec:
  hosts:
    - orders-service
  http:
    # Mobile users → v2
    - match:
        - headers:
            user-agent:
              regex: ".*Mobile.*"
      route:
        - destination:
            host: orders-service
            subset: v2
    # Desktop users → v1
    - route:
        - destination:
            host: orders-service
            subset: v1
```

### Fault Injection (Testing)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-fault-injection
spec:
  hosts:
    - orders-service
  http:
    - fault:
        delay:
          percentage:
            value: 10.0  # 10% запросов
          fixedDelay: 5s  # Задержка 5 секунд
        abort:
          percentage:
            value: 5.0  # 5% запросов
          httpStatus: 500  # Вернуть 500 ошибку
      route:
        - destination:
            host: orders-service
```

Полезно для chaos testing: проверка resilience приложения.

## Linkerd

Linkerd — более лёгкий и простой Service Mesh.

### Установка Linkerd

```bash
# Установить CLI
curl -sL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin

# Pre-flight check
linkerd check --pre

# Установить в кластер
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# Проверка
linkerd check

# Визуализация
linkerd viz install | kubectl apply -f -
```

### Добавление Linkerd к сервису

```bash
# Автоматический inject для namespace
kubectl annotate namespace default linkerd.io/inject=enabled

# Или вручную для конкретного deployment
kubectl get deploy orders-service -o yaml | linkerd inject - | kubectl apply -f -
```

### Traffic Splitting (Linkerd SMI)

```yaml
# traffic-split.yaml (SMI standard)
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: orders-split
spec:
  service: orders-service
  backends:
    - service: orders-service-v1
      weight: 90
    - service: orders-service-v2
      weight: 10
```

### Retry и Timeout

```yaml
# Service Profile
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: orders-service.default.svc.cluster.local
spec:
  routes:
    - name: POST /api/orders
      condition:
        method: POST
        pathRegex: /api/orders
      timeout: 5s
      retries:
        limit: 3
        timeout: 2s
    - name: GET /api/orders
      condition:
        method: GET
        pathRegex: /api/orders/.*
      timeout: 10s
```

### mTLS

Linkerd автоматически включает mTLS для всего mesh-трафика без дополнительной конфигурации:

```bash
# Проверка mTLS
linkerd viz -n default tap deploy/orders-service | grep tls

# Вывод:
# tls=true
```

### Observability

```bash
# Dashboard
linkerd viz dashboard

# Top (как htop для сервисов)
linkerd viz top deploy/orders-service

# Tap: live traffic
linkerd viz tap deploy/orders-service

# Routes (метрики по эндпоинтам)
linkerd viz routes deploy/orders-service
```

## Istio vs Linkerd

| Критерий | Istio | Linkerd |
|----------|-------|---------|
| **Сложность** | Высокая | Низкая |
| **Performance** | Хорошая | Отличная (меньше overhead) |
| **Features** | Очень много | Основные (достаточно) |
| **Proxy** | Envoy (C++) | linkerd2-proxy (Rust) |
| **Memory** | ~100-200 MB/pod | ~20-50 MB/pod |
| **Поддержка не-K8s** | Да (VM) | Только Kubernetes |
| **Extensibility** | WebAssembly, Lua | Ограничена |
| **Maturity** | Зрелый | Зрелый |
| **Learning curve** | Крутая | Пологая |
| **Best for** | Сложные use cases, enterprise | Простота, производительность |

## Практические паттерны

### 1. Progressive Delivery

```yaml
# Шаг 1: Deploy v2 без трафика
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-progressive
spec:
  hosts:
    - orders-service
  http:
    - route:
        - destination:
            host: orders-service
            subset: v1
          weight: 100
        - destination:
            host: orders-service
            subset: v2
          weight: 0

---
# Шаг 2: Smoke testing (internal traffic only)
# Используем header для routing internal requests на v2

# Шаг 3: Canary (5%)
# weight: v1=95, v2=5

# Шаг 4: Мониторинг error rate, latency
# Если OK → увеличить до 25%, 50%, 100%
# Если NOT OK → rollback (weight v2=0)
```

### 2. Multi-cluster Service Mesh

```yaml
# Istio Multi-Primary: два кластера видят друг друга
# cluster-1:
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: cluster-1-gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        number: 15443
        name: tls
        protocol: TLS
      tls:
        mode: AUTO_PASSTHROUGH
      hosts:
        - "*.cluster-2.global"

# Сервисы из cluster-2 доступны в cluster-1 как:
# orders-service.default.svc.cluster-2.global
```

### 3. External Service Entry

```yaml
# Управление трафиком к внешним сервисам
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
    - api.stripe.com
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS

---
# Применить retry и timeout к внешнему API
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: stripe-retry
spec:
  hosts:
    - api.stripe.com
  http:
    - route:
        - destination:
            host: api.stripe.com
      retries:
        attempts: 3
        perTryTimeout: 5s
      timeout: 15s
```

### 4. Rate Limiting

```yaml
# Envoy Filter для rate limiting
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: rate-limit-filter
spec:
  workloadSelector:
    labels:
      app: orders
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            stat_prefix: http_local_rate_limiter
            token_bucket:
              max_tokens: 100
              tokens_per_fill: 100
              fill_interval: 60s
            filter_enabled:
              runtime_key: local_rate_limit_enabled
              default_value:
                numerator: 100
                denominator: HUNDRED
            filter_enforced:
              runtime_key: local_rate_limit_enforced
              default_value:
                numerator: 100
                denominator: HUNDRED
```

## Мониторинг и Troubleshooting

### Проверка конфигурации

```bash
# Istio
istioctl analyze

# Вывод:
# ✔ No validation issues found when analyzing namespace: default.

# Проверка конкретного сервиса
istioctl proxy-status

# Получить конфигурацию Envoy
istioctl proxy-config routes deploy/orders-service
istioctl proxy-config clusters deploy/orders-service
```

### Debugging

```bash
# Логи sidecar proxy
kubectl logs orders-service-xxx -c istio-proxy

# Envoy admin interface
kubectl port-forward orders-service-xxx 15000:15000
# Открыть http://localhost:15000

# Linkerd
linkerd viz tap deploy/orders-service --to deploy/payment-service
```

### Ключевые метрики

```promql
# Request rate
sum(rate(istio_requests_total{destination_service="orders-service"}[5m]))

# Error rate
sum(rate(istio_requests_total{destination_service="orders-service",response_code=~"5.."}[5m]))
/
sum(rate(istio_requests_total{destination_service="orders-service"}[5m]))

# Latency (P99)
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket{destination_service="orders-service"}[5m])) by (le)
)

# Circuit breaker ejections
sum(rate(envoy_cluster_outlier_detection_ejections_active[5m]))
```

## Best Practices

### 1. Постепенное внедрение

```bash
# Начните с одного namespace
kubectl label namespace staging istio-injection=enabled

# Затем production
kubectl label namespace production istio-injection=enabled

# Не включайте для kube-system
```

### 2. Resource Limits

```yaml
# Sidecar resource limits
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-sidecar-injector
  namespace: istio-system
data:
  values: |
    global:
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

### 3. Disable для некритичных сервисов

```yaml
# Отключить injection для конкретного pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debug-tool
spec:
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "false"
    spec:
      containers:
        - name: debug
          image: nicolaka/netshoot
```

### 4. Мониторинг overhead

```bash
# Сравнить latency с и без mesh
# Обычно overhead: 1-3ms per hop
```

### 5. Security Policies

```yaml
# Default deny all
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  {}

# Затем явно разрешайте нужные connections
```

## Когда использовать Service Mesh?

### ✅ Используйте когда:

- Много микросервисов (10+)
- Polyglot environment (разные языки)
- Нужен zero-trust security (mTLS everywhere)
- Сложные traffic patterns (canary, A/B)
- Требуется distributed tracing
- Compliance требования (audit, encryption)

### ❌ Не используйте когда:

- Монолит или мало сервисов (< 5)
- Команда не готова к operational complexity
- Ограниченные ресурсы кластера
- Latency критична (каждая миллисекунда важна)

## Альтернативы Service Mesh

### 1. Library-based (Netflix OSS)

```javascript
// Hystrix для circuit breaking
const hystrixCommand = commandFactory.getOrCreate("PaymentService")
  .circuitBreakerErrorThresholdPercentage(50)
  .timeout(5000)
  .run(async () => {
    return await paymentService.charge(amount);
  })
  .fallbackTo(() => {
    return { status: 'queued' };
  });

const result = await hystrixCommand.execute();
```

**Минусы:** код дублируется, зависимость от языка.

### 2. API Gateway (Kong, Ambassador)

Централизованное управление, но только для north-south traffic (client → service), не east-west (service → service).

### 3. Consul Connect

Более лёгкая альтернатива, интегрируется с Consul для service discovery.

## Выводы

Service Mesh автоматизирует сложные аспекты микросервисной коммуникации:

**Ключевые возможности:**
- **Traffic management**: routing, load balancing, canary
- **Security**: mTLS, authorization policies
- **Observability**: metrics, tracing, logs
- **Resilience**: retry, timeout, circuit breaking

**Istio** — feature-rich, но сложнее.
**Linkerd** — проще и быстрее, но меньше функций.

Service Mesh — не обязательный компонент, но при росте числа микросервисов значительно упрощает операционную работу.

## Что читать дальше?

- [Урок 27: Event-Driven Architecture](27-event-driven-architecture.md)
- [Урок 25: Микросервисы: когда и зачем](25-microservices-when-why.md)
- [Урок 10: Load Balancing](10-load-balancing-algoritmy-l4-vs-l7.md)

## Проверь себя

1. Что такое Service Mesh и какие проблемы он решает?
2. В чём разница между Data Plane и Control Plane?
3. Что такое sidecar pattern?
4. Как Istio реализует canary deployment?
5. В чём разница между Istio и Linkerd?
6. Как работает автоматический mTLS в Service Mesh?
7. Что такое Circuit Breaker и как его настроить в Istio?
8. Какие метрики важны для мониторинга Service Mesh?
9. Когда НЕ стоит использовать Service Mesh?
10. Как Service Mesh помогает с distributed tracing?

---
**Предыдущий урок**: [Урок 25: Микросервисы — когда и зачем](25-microservices-when-why.md)
**Следующий урок**: [Урок 27: Event-Driven Architecture](27-event-driven-architecture.md)
