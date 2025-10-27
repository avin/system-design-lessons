# API Gateway и Reverse Proxy

## Reverse Proxy

### Что это?

**Reverse Proxy** — сервер, который принимает requests от клиентов и перенаправляет их на backend серверы.

```
[Client] → [Reverse Proxy] → [Backend Servers]
```

**Отличие от Forward Proxy**:
- **Forward Proxy**: Client → Proxy → Internet (прокси для клиента)
- **Reverse Proxy**: Client → Proxy → Backend (прокси для сервера)

### Зачем нужен Reverse Proxy?

#### 1. Load Balancing

Распределение запросов между серверами:
```
         [Reverse Proxy]
            ↙    ↓    ↘
         [S1]  [S2]  [S3]
```

#### 2. SSL Termination

Расшифровка HTTPS на proxy, backend получает HTTP:
```
[Client] --HTTPS--> [Proxy] --HTTP--> [Backend]
```

**Преимущества**:
- Централизованное управление сертификатами
- Offload SSL processing от backend
- Backend проще (нет SSL config)

#### 3. Caching

Кэширование статики и API responses:
```
GET /api/products
  ↓
[Reverse Proxy Cache]
  ✅ Cache HIT → return cached response
  ❌ Cache MISS → forward to backend
```

#### 4. Compression

Сжатие responses (gzip, brotli):
```
Backend → 500KB JSON
  ↓
[Reverse Proxy] (gzip compression)
  ↓
Client ← 50KB (10x меньше!)
```

#### 5. Security

- DDoS protection
- Rate limiting
- IP whitelisting/blacklisting
- WAF (Web Application Firewall)
- Скрытие внутренней структуры

#### 6. Static File Serving

Эффективная отдача статики:
```
/static/* → served directly by proxy (NGINX)
/api/*    → proxied to backend
```

### NGINX как Reverse Proxy

**Простейшая конфигурация**:
```nginx
server {
  listen 80;
  server_name example.com;

  location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

**Headers**:
- `Host`: оригинальный hostname
- `X-Real-IP`: IP клиента
- `X-Forwarded-For`: цепочка proxies
- `X-Forwarded-Proto`: http или https

**С кэшированием**:
```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m;

server {
  location /api/ {
    proxy_cache my_cache;
    proxy_cache_valid 200 10m;
    proxy_cache_valid 404 1m;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    add_header X-Cache-Status $upstream_cache_status;

    proxy_pass http://backend;
  }
}
```

**Статика через NGINX**:
```nginx
location /static/ {
  root /var/www;
  expires 30d;
  add_header Cache-Control "public, immutable";
}

location /api/ {
  proxy_pass http://backend;
}
```

## API Gateway

### Что это?

**API Gateway** — reverse proxy с дополнительными функциями для управления API:
- Routing
- Authentication & Authorization
- Rate Limiting
- Request/Response transformation
- Analytics & Monitoring
- Protocol translation (REST → gRPC, HTTP → WebSocket)

```
[Mobile App]
[Web App]      → [API Gateway] → [Microservice 1]
[3rd Party]                    → [Microservice 2]
                               → [Microservice 3]
```

### Паттерн: Backend for Frontend (BFF)

Разные gateways для разных клиентов:

```
[Mobile App] → [Mobile BFF] ──┐
                              ├→ [Microservices]
[Web App]    → [Web BFF] ─────┘
```

**Зачем**:
- Mobile нужны оптимизированные responses (меньше данных)
- Web может хотеть более rich данные
- Разные auth flows

### Функции API Gateway

#### 1. Routing

Направление запросов на соответствующие сервисы:

```
GET /users/*     → User Service
GET /products/*  → Product Service
GET /orders/*    → Order Service
```

**Пример (Kong)**:
```bash
curl -X POST http://localhost:8001/services \
  --data name=user-service \
  --data url=http://user-service:8080

curl -X POST http://localhost:8001/services/user-service/routes \
  --data paths[]=/users
```

#### 2. Authentication & Authorization

Централизованная проверка auth:

```
Client → Gateway
  ↓
Gateway проверяет JWT token
  ✅ Valid → forward to microservice
  ❌ Invalid → return 401
```

**Пример (NGINX + Lua)**:
```nginx
location /api/ {
  access_by_lua_block {
    local jwt = require "resty.jwt"
    local token = ngx.var.http_authorization

    if not token then
      ngx.status = 401
      ngx.say("Unauthorized")
      ngx.exit(401)
    end

    local jwt_obj = jwt:verify(secret, token)
    if not jwt_obj.verified then
      ngx.status = 401
      ngx.say("Invalid token")
      ngx.exit(401)
    end
  }

  proxy_pass http://backend;
}
```

#### 3. Rate Limiting

Ограничение количества requests:

**NGINX**:
```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

location /api/ {
  limit_req zone=api_limit burst=20 nodelay;
  proxy_pass http://backend;
}
```

**Kong**:
```bash
curl -X POST http://localhost:8001/services/user-service/plugins \
  --data name=rate-limiting \
  --data config.minute=100 \
  --data config.policy=local
```

#### 4. Request/Response Transformation

Модификация данных на лету:

**Добавление headers**:
```nginx
location /api/ {
  proxy_set_header X-API-Version "v1";
  proxy_set_header X-Request-ID $request_id;
  proxy_pass http://backend;
}
```

**Response transformation (Kong)**:
```bash
curl -X POST http://localhost:8001/services/user-service/plugins \
  --data name=response-transformer \
  --data config.remove.headers=X-Internal-Header \
  --data config.add.headers=X-Gateway-Version:1.0
```

#### 5. Protocol Translation

Преобразование между протоколами:

```
HTTP/1.1 → HTTP/2
REST → gRPC
HTTP → WebSocket
```

#### 6. Aggregation (Backend for Frontend pattern)

Объединение multiple API calls в один:

```
Mobile Request: GET /dashboard
  ↓
Gateway делает:
  - GET /users/123
  - GET /orders/recent
  - GET /notifications
  ↓
Gateway возвращает объединенный response
```

**Пример (GraphQL Gateway)**:
```graphql
type Query {
  dashboard: Dashboard
}

type Dashboard {
  user: User
  recentOrders: [Order]
  notifications: [Notification]
}

# Resolver делает 3 запроса к разным сервисам
```

#### 7. Circuit Breaking

Защита от cascading failures:

```
If downstream service fails 50% of requests:
  → Gateway opens circuit
  → Returns cached/fallback response
  → Doesn't overload failing service
```

**Envoy**:
```yaml
circuit_breakers:
  thresholds:
    - priority: DEFAULT
      max_connections: 1000
      max_requests: 1000
      max_retries: 3
```

#### 8. Retry Logic

Автоматические повторные попытки:

```nginx
location /api/ {
  proxy_next_upstream error timeout http_500 http_502 http_503;
  proxy_next_upstream_tries 3;
  proxy_next_upstream_timeout 10s;
  proxy_pass http://backend;
}
```

#### 9. Monitoring & Analytics

Сбор метрик:
- Request count
- Latency (P50, P95, P99)
- Error rate
- Top endpoints
- Top clients

**Интеграции**:
- Prometheus
- Grafana
- Datadog
- New Relic

#### 10. API Versioning

Управление разными версиями API:

```
GET /v1/users → User Service v1
GET /v2/users → User Service v2
```

**Header-based**:
```
GET /users
Accept: application/vnd.api+json; version=2
  → routes to v2
```

### Популярные API Gateways

#### 1. Kong

**Тип**: Open-source, на базе NGINX + Lua
**Features**: Plugins, DB or DB-less mode

**Установка**:
```bash
docker run -d --name kong \
  -e "KONG_DATABASE=off" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -p 8000:8000 \
  -p 8443:8443 \
  -p 8001:8001 \
  -p 8444:8444 \
  kong:latest
```

**Plugins**:
- Authentication (JWT, OAuth 2.0, Key Auth)
- Rate Limiting
- CORS
- Request/Response Transformation
- Logging

#### 2. AWS API Gateway

**Тип**: Managed service
**Features**: Lambda integration, Cognito auth

**Pricing**: Pay per request (~$3.50 per million requests)

**Конфигурация**:
```yaml
openapi: 3.0.0
paths:
  /users:
    get:
      x-amazon-apigateway-integration:
        type: http_proxy
        httpMethod: GET
        uri: http://backend.example.com/users
```

#### 3. Envoy

**Тип**: Modern C++ proxy, используется в Istio
**Features**: Advanced load balancing, observability

**Конфигурация**:
```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match: { prefix: "/users" }
                route: { cluster: user_service }
          http_filters:
          - name: envoy.filters.http.router
  clusters:
  - name: user_service
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: user_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: user-service
                port_value: 8080
```

#### 4. Traefik

**Тип**: Cloud-native, автоконфигурация
**Features**: Docker/Kubernetes integration

**Docker Compose**:
```yaml
version: '3'
services:
  traefik:
    image: traefik:v2.9
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  whoami:
    image: traefik/whoami
    labels:
      - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)"
```

#### 5. Apigee (Google Cloud)

**Тип**: Enterprise, managed
**Features**: Full API lifecycle management

**Цена**: $$$, enterprise-level

#### 6. Zuul (Netflix)

**Тип**: Java-based, part of Spring Cloud
**Features**: Dynamic routing, resilience

**Spring Boot**:
```java
@EnableZuulProxy
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

```yaml
zuul:
  routes:
    users:
      path: /users/**
      url: http://user-service:8080
    orders:
      path: /orders/**
      url: http://order-service:8080
```

## Reverse Proxy vs API Gateway

| Характеристика | Reverse Proxy | API Gateway |
|----------------|---------------|-------------|
| Основная цель | Forwarding requests | API management |
| Routing | Простой (path-based) | Сложный (headers, methods, etc.) |
| Authentication | Basic | JWT, OAuth, API Keys |
| Rate Limiting | Простой (per IP) | Per user, per API key |
| Transformation | Минимальная | Богатая (request/response) |
| Analytics | Базовая | Детальная (per endpoint, user) |
| Protocol | HTTP(S) | HTTP, gRPC, WebSocket, etc. |
| Use Case | Web apps, static files | Microservices, API management |
| Примеры | NGINX, HAProxy | Kong, AWS API Gateway, Apigee |

## Архитектурные паттерны

### 1. Single API Gateway

```
[Clients] → [API Gateway] → [Microservices]
```

**Преимущества**:
✅ Простота
✅ Централизованное управление

**Недостатки**:
❌ Single point of failure
❌ Может стать bottleneck
❌ Coupling (все клиенты через один gateway)

### 2. Multiple API Gateways (BFF)

```
[Mobile]  → [Mobile Gateway]  ──┐
[Web]     → [Web Gateway]     ──┼→ [Microservices]
[Partner] → [Partner Gateway] ──┘
```

**Преимущества**:
✅ Оптимизация под каждый клиент
✅ Isolation (проблема в одном не влияет на другие)
✅ Независимое масштабирование

### 3. Gateway Aggregation Pattern

Gateway объединяет responses от нескольких сервисов:

```
Client: GET /user-profile
  ↓
Gateway:
  ├─ GET /user-service/users/123
  ├─ GET /order-service/orders?user=123
  └─ GET /recommendation-service/recommendations?user=123
  ↓
Gateway: Объединяет в один response
```

### 4. Gateway Offloading Pattern

Gateway берет на себя cross-cutting concerns:

```
[API Gateway]
  ├─ SSL Termination
  ├─ Authentication
  ├─ Rate Limiting
  ├─ Caching
  ├─ Compression
  └─ Logging
        ↓
  [Microservices] (упрощенные, без этих concerns)
```

### 5. Service Mesh + Gateway

```
[API Gateway] (north-south traffic)
      ↓
[Service Mesh] (east-west traffic между сервисами)
  ↓   ↓   ↓
[S1][S2][S3]
```

**Service Mesh** (Istio, Linkerd):
- Управление трафиком между сервисами
- mTLS (mutual TLS)
- Observability
- Circuit breaking

**API Gateway**:
- Управление external трафиком
- Authentication для внешних клиентов
- Public API management

## Практический пример: E-commerce API Gateway

### Требования

- Mobile app, Web app, Partner API
- Authentication: JWT
- Rate limiting: 100 req/min per user
- Caching для product catalog
- Routing к микросервисам

### Архитектура

```
[Mobile/Web/Partners]
        ↓
   [Kong Gateway]
        ↓
    ┌───┴───┬────────┬─────────┐
    ↓       ↓        ↓         ↓
 [Auth] [Users] [Products] [Orders]
```

### Kong Конфигурация

**1. Создание сервисов**:
```bash
# User Service
curl -X POST http://localhost:8001/services \
  --data name=user-service \
  --data url=http://user-service:8080

# Product Service
curl -X POST http://localhost:8001/services \
  --data name=product-service \
  --data url=http://product-service:8080

# Order Service
curl -X POST http://localhost:8001/services \
  --data name=order-service \
  --data url=http://order-service:8080
```

**2. Создание routes**:
```bash
# Users
curl -X POST http://localhost:8001/services/user-service/routes \
  --data paths[]=/api/users

# Products
curl -X POST http://localhost:8001/services/product-service/routes \
  --data paths[]=/api/products

# Orders
curl -X POST http://localhost:8001/services/order-service/routes \
  --data paths[]=/api/orders
```

**3. JWT Authentication**:
```bash
# Enable JWT plugin
curl -X POST http://localhost:8001/routes/{route-id}/plugins \
  --data name=jwt \
  --data config.secret_is_base64=false
```

**4. Rate Limiting**:
```bash
curl -X POST http://localhost:8001/routes/{route-id}/plugins \
  --data name=rate-limiting \
  --data config.minute=100 \
  --data config.policy=local
```

**5. Response Caching (для products)**:
```bash
curl -X POST http://localhost:8001/services/product-service/plugins \
  --data name=proxy-cache \
  --data config.response_code=200 \
  --data config.request_method=GET \
  --data config.content_type=application/json \
  --data config.cache_ttl=300 \
  --data config.strategy=memory
```

**6. CORS**:
```bash
curl -X POST http://localhost:8001/plugins \
  --data name=cors \
  --data config.origins=* \
  --data config.methods=GET,POST,PUT,DELETE \
  --data config.headers=Authorization,Content-Type \
  --data config.max_age=3600
```

## Performance Optimization

### 1. Connection Pooling

```nginx
upstream backend {
  server backend1:8080;
  server backend2:8080;
  keepalive 32;  # Connection pool
}

location / {
  proxy_pass http://backend;
  proxy_http_version 1.1;
  proxy_set_header Connection "";
}
```

### 2. Buffering

```nginx
location / {
  proxy_buffering on;
  proxy_buffer_size 4k;
  proxy_buffers 8 4k;
  proxy_busy_buffers_size 8k;
  proxy_pass http://backend;
}
```

### 3. Caching

**Static content**:
```nginx
location ~* \.(jpg|jpeg|png|gif|css|js)$ {
  expires 30d;
  add_header Cache-Control "public, immutable";
}
```

**API responses**:
```nginx
proxy_cache_path /var/cache/nginx keys_zone=api_cache:10m;

location /api/products {
  proxy_cache api_cache;
  proxy_cache_valid 200 5m;
  proxy_cache_key "$request_uri";
  proxy_pass http://backend;
}
```

### 4. Compression

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000;
gzip_comp_level 6;
```

## Security Best Practices

### 1. Rate Limiting

Предотвращение abuse и DDoS:
```nginx
limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
limit_req_zone $http_authorization zone=authenticated:10m rate=100r/s;

location /api/ {
  limit_req zone=authenticated burst=20 nodelay;
  proxy_pass http://backend;
}
```

### 2. IP Whitelisting

```nginx
location /admin/ {
  allow 203.0.113.0/24;  # Office IP range
  deny all;
  proxy_pass http://backend;
}
```

### 3. Request Size Limits

```nginx
client_max_body_size 10M;
client_body_buffer_size 128k;
```

### 4. Timeout Protection

```nginx
proxy_connect_timeout 5s;
proxy_send_timeout 10s;
proxy_read_timeout 30s;
```

### 5. Hide Internal Errors

```nginx
proxy_intercept_errors on;

error_page 500 502 503 504 /50x.html;
location = /50x.html {
  root /usr/share/nginx/html;
}
```

## Monitoring

### Метрики для мониторинга

1. **Request metrics**:
   - Requests per second
   - Request duration (P50, P95, P99)
   - Status code distribution

2. **Upstream metrics**:
   - Upstream response time
   - Upstream failures
   - Upstream availability

3. **System metrics**:
   - CPU usage
   - Memory usage
   - Network I/O
   - Open connections

### Prometheus + NGINX

**NGINX Prometheus Exporter**:
```bash
docker run -p 9113:9113 nginx/nginx-prometheus-exporter:latest \
  -nginx.scrape-uri=http://nginx:8080/stub_status
```

**NGINX stub_status**:
```nginx
server {
  listen 8080;
  location /stub_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
  }
}
```

## Что почитать дальше

- [NGINX Documentation](https://nginx.org/en/docs/)
- [Kong Documentation](https://docs.konghq.com/)
- "Building Microservices" by Sam Newman — API Gateway chapter
- [AWS API Gateway Best Practices](https://docs.aws.amazon.com/apigateway/latest/developerguide/best-practices.html)

## Проверьте себя

1. В чем разница между forward proxy и reverse proxy?
2. Какие преимущества дает SSL termination на gateway?
3. Что такое Backend for Frontend (BFF) pattern?
4. Назовите 5 основных функций API Gateway.
5. В чем разница между Reverse Proxy и API Gateway?
6. Когда стоит использовать multiple gateways вместо single?
7. Что такое Gateway Aggregation pattern?
8. Как реализовать rate limiting на API Gateway?

## Ключевые выводы

- Reverse Proxy — базовое forwarding + SSL + caching + load balancing
- API Gateway — Reverse Proxy + authentication + transformation + analytics
- SSL termination на gateway снижает нагрузку на backend
- BFF pattern — разные gateways для разных типов клиентов
- Gateway Aggregation объединяет multiple API calls в один response
- Gateway Offloading Pattern — cross-cutting concerns на gateway
- Kong, AWS API Gateway, Envoy — популярные решения
- Обязательно мониторьте latency, error rate, throughput
- Rate limiting и circuit breaking защищают от overload
- Connection pooling и caching критичны для performance

---
**Предыдущий урок**: [Load Balancing: алгоритмы, L4 vs L7, sticky sessions](10-load-balancing-algorithms-l4-vs-l7.md)
**Следующий урок**: [Rate Limiting и алгоритмы подсчёта](12-rate-limiting-algorithms.md)
