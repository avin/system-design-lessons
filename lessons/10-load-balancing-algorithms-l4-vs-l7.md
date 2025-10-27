# Load Balancing: алгоритмы, L4 vs L7, sticky sessions

## Что такое Load Balancing?

**Load Balancer (LB)** — компонент, который распределяет входящий трафик между несколькими серверами для:
- Увеличения capacity (больше requests/sec)
- Повышения availability (если сервер упал, LB направит на другой)
- Улучшения latency (направление на ближайший сервер)

```
         [Users]
            ↓
     [Load Balancer]
       ↙    ↓    ↘
   [S1]  [S2]  [S3]
```

Без LB: один сервер обрабатывает 1K RPS
С LB (3 сервера): система обрабатывает 3K RPS

## Зачем нужен Load Balancer?

### 1. Scalability (Horizontal Scaling)

Добавляйте серверы по мере роста нагрузки:
```
1 server → 1K RPS
Add LB + 2 more servers → 3K RPS
Add 2 more servers → 5K RPS
```

### 2. High Availability

Если сервер упадет, LB перестанет направлять на него трафик:
```
S1: ✅ Healthy
S2: ❌ Down
S3: ✅ Healthy

LB направляет только на S1 и S3
```

### 3. Maintenance без Downtime

```
1. Отключаем S1 от LB
2. Обновляем S1
3. Возвращаем S1 в LB
4. Повторяем для S2, S3...

Система остается доступной!
```

### 4. Geographic Distribution

```
User в US → LB направляет на US datacenter
User в EU → LB направляет на EU datacenter
```

## Layer 4 vs Layer 7 Load Balancing

### OSI Model (краткое напоминание)

| Layer | Название | Протоколы | Данные |
|-------|----------|-----------|--------|
| 7 | Application | HTTP, FTP, SMTP | Сообщения |
| 6 | Presentation | SSL/TLS | Данные |
| 5 | Session | NetBIOS | Сессии |
| 4 | Transport | TCP, UDP | Сегменты |
| 3 | Network | IP | Пакеты |
| 2 | Data Link | Ethernet | Фреймы |
| 1 | Physical | - | Биты |

### Layer 4 Load Balancing (Transport Layer)

**Принимает решения на основе**:
- IP адреса источника
- IP адреса назначения
- TCP/UDP порта

**Не видит**:
- HTTP headers
- Cookies
- URL paths
- Application data

**Как работает**:
```
1. Client открывает TCP connection к LB (IP: 10.0.0.1:80)
2. LB выбирает backend server (например, 192.168.1.5)
3. LB устанавливает connection с backend
4. Весь трафик проксируется между client и backend
```

**Пример**:
```
Client: 203.0.113.5:54321
   ↓
LB: 10.0.0.1:80
   ↓ (выбирает backend на основе IP:Port)
Backend: 192.168.1.5:8080
```

**Technologies**:
- **HAProxy** (TCP mode)
- **NGINX** (stream module)
- **AWS NLB** (Network Load Balancer)
- **Linux IPVS**

### Layer 7 Load Balancing (Application Layer)

**Принимает решения на основе**:
- HTTP headers
- Cookies
- URL path
- Query parameters
- HTTP method
- Request body

**Видит весь application context**!

**Как работает**:
```
1. Client отправляет HTTP request к LB
2. LB читает HTTP headers, URL, etc.
3. На основе правил выбирает backend
4. LB создает новый HTTP request к backend
5. LB получает response и отправляет client
```

**Пример routing rules**:
```
/api/*     → API servers (S1, S2)
/static/*  → Static file servers (S3, S4)
/images/*  → Image servers (S5, S6)
```

**Technologies**:
- **HAProxy** (HTTP mode)
- **NGINX**
- **AWS ALB** (Application Load Balancer)
- **Traefik**
- **Envoy**

### Сравнение L4 vs L7

| Характеристика | Layer 4 | Layer 7 |
|----------------|---------|---------|
| Скорость | ✅ Быстрее | Медленнее |
| Latency | ✅ Низкая (< 1ms) | Выше (~5ms) |
| Сложность | ✅ Проще | Сложнее |
| Routing rules | IP:Port only | URL, headers, cookies |
| SSL Termination | ❌ Нет (pass-through) | ✅ Да |
| Caching | ❌ Нет | ✅ Возможно |
| Compression | ❌ Нет | ✅ Возможно |
| WAF | ❌ Нет | ✅ Возможно |
| WebSocket | ✅ Да (transparent) | ✅ Да (с поддержкой) |
| Протоколы | Любые TCP/UDP | HTTP/HTTPS/HTTP2/gRPC |
| Use case | High performance, simple | Content-based routing |

### Когда использовать L4

✅ Нужна максимальная производительность
✅ Простое распределение по IP:Port
✅ Non-HTTP протоколы (SMTP, MySQL, Redis)
✅ SSL pass-through (без termination)
✅ Очень высокий throughput (millions RPS)

**Примеры**:
- Database load balancing (PostgreSQL, MySQL)
- Cache layer (Redis, Memcached)
- WebSocket connections
- Gaming servers

### Когда использовать L7

✅ Routing на основе URL paths
✅ SSL termination нужен
✅ Нужна модификация headers
✅ A/B testing, canary deployments
✅ Authentication/authorization на LB
✅ Rate limiting per user/endpoint

**Примеры**:
- Web applications
- Microservices с API gateway
- Multi-tenant applications
- Content-based routing

## Алгоритмы Load Balancing

### 1. Round Robin

**Принцип**: Запросы распределяются по очереди.

```
Request 1 → S1
Request 2 → S2
Request 3 → S3
Request 4 → S1
Request 5 → S2
...
```

**Преимущества**:
✅ Простота
✅ Равномерное распределение

**Недостатки**:
❌ Не учитывает нагрузку серверов
❌ Не учитывает мощность серверов

**Когда использовать**:
- Серверы одинаковой мощности
- Запросы примерно одинаковой сложности

### 2. Weighted Round Robin

**Принцип**: Как Round Robin, но с весами для каждого сервера.

```
S1: weight 3 (мощный сервер)
S2: weight 2
S3: weight 1

Request 1 → S1
Request 2 → S1
Request 3 → S1
Request 4 → S2
Request 5 → S2
Request 6 → S3
Request 7 → S1 (цикл повторяется)
```

**Конфигурация (NGINX)**:
```nginx
upstream backend {
  server s1.example.com weight=3;
  server s2.example.com weight=2;
  server s3.example.com weight=1;
}
```

**Когда использовать**:
- Серверы разной мощности
- Хотите дать больше трафика мощным серверам

### 3. Least Connections

**Принцип**: Направлять на сервер с наименьшим количеством активных соединений.

```
S1: 10 connections
S2: 5 connections   ← выбран
S3: 15 connections

New request → S2
```

**NGINX**:
```nginx
upstream backend {
  least_conn;
  server s1.example.com;
  server s2.example.com;
  server s3.example.com;
}
```

**Преимущества**:
✅ Учитывает текущую нагрузку
✅ Хорошо для long-lived connections

**Когда использовать**:
- Запросы разной длительности
- WebSocket connections
- Database connections

### 4. Weighted Least Connections

Комбинация Least Connections + веса серверов.

```
S1: 10 conn, weight 3 → ratio = 10/3 = 3.33
S2: 5 conn, weight 2  → ratio = 5/2 = 2.5  ← выбран
S3: 15 conn, weight 1 → ratio = 15/1 = 15

New request → S2 (наименьший ratio)
```

### 5. IP Hash

**Принцип**: Hash IP адреса клиента определяет сервер.

```
hash(203.0.113.5) % 3 = 2 → S3
hash(198.51.100.20) % 3 = 0 → S1

Тот же IP всегда попадает на тот же сервер!
```

**NGINX**:
```nginx
upstream backend {
  ip_hash;
  server s1.example.com;
  server s2.example.com;
  server s3.example.com;
}
```

**Преимущества**:
✅ Sticky sessions без cookies
✅ Кэш locality (тот же сервер → cache hit)

**Недостатки**:
❌ Неравномерное распределение (если клиенты за NAT)
❌ При добавлении/удалении сервера много sessions переместятся

**Когда использовать**:
- Нужны sticky sessions
- Stateful applications
- Хотите кэшировать данные локально

### 6. Least Response Time

**Принцип**: Направлять на сервер с наименьшим response time.

```
S1: avg response time 50ms
S2: avg response time 30ms  ← выбран
S3: avg response time 70ms

New request → S2
```

**Преимущества**:
✅ Оптимизирует latency
✅ Учитывает производительность сервера

**Когда использовать**:
- Latency критична
- Серверы географически распределены

### 7. Random

**Принцип**: Случайный выбор сервера.

```
Request 1 → S2 (random)
Request 2 → S1 (random)
Request 3 → S2 (random)
```

**Когда использовать**:
- Простота важнее оптимальности
- Все серверы одинаковые
- Stateless приложения

### 8. Consistent Hashing

**Принцип**: Используется для кэш-серверов (см. урок 6).

```
hash(cache_key) → определяет сервер
```

**Когда использовать**:
- Distributed caching (Memcached, Redis)
- Нужно минимизировать cache invalidation при изменении количества серверов

## Sticky Sessions (Session Affinity)

### Проблема

Многие приложения хранят session state на сервере:

```
1. User логинится → session сохраняется на S1
2. Следующий request → LB направляет на S2
3. S2 не знает о session → User не авторизован!
```

### Решения

#### 1. Cookie-based Sticky Sessions

LB добавляет cookie, указывающий на сервер:

```http
Request 1 → S1
Response:
Set-Cookie: SERVERID=s1; path=/

Request 2 (с cookie SERVERID=s1) → S1
Request 3 (с cookie SERVERID=s1) → S1
```

**NGINX**:
```nginx
upstream backend {
  server s1.example.com;
  server s2.example.com;
  server s3.example.com;
  sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```

#### 2. IP Hash

Тот же IP → тот же сервер (как описано выше).

#### 3. Session ID в URL

```
https://example.com/app;jsessionid=ABC123
```

LB парсит session ID и направляет на соответствующий сервер.

#### 4. Shared Session Storage (рекомендуется!)

**Лучшее решение**: Не использовать sticky sessions вообще!

```
[Client] → [LB] → [S1, S2, S3]
                        ↓
                  [Redis/Memcached]
                  (shared sessions)
```

**Преимущества**:
✅ Любой сервер может обработать запрос
✅ Нет проблем при падении сервера
✅ Лучше для масштабирования

**Session store options**:
- Redis
- Memcached
- Database
- DynamoDB

### Проблемы Sticky Sessions

❌ Неравномерное распределение нагрузки
❌ Если сервер упал, пользователи теряют sessions
❌ Сложнее масштабировать
❌ Не работает с autoscaling

## Health Checks

LB должен знать, какие серверы живы.

### Active Health Checks

LB периодически проверяет серверы:

```
Каждые 5 секунд:
  GET /health → S1
  Response: 200 OK ✅

  GET /health → S2
  Timeout ❌

LB исключает S2 из rotation
```

**NGINX**:
```nginx
upstream backend {
  server s1.example.com;
  server s2.example.com;
  server s3.example.com;

  # Health check (NGINX Plus)
  health_check interval=5s fails=3 passes=2 uri=/health;
}
```

**HAProxy**:
```
backend servers
  option httpchk GET /health
  http-check expect status 200
  server s1 192.168.1.1:8080 check inter 5s fall 3 rise 2
  server s2 192.168.1.2:8080 check inter 5s fall 3 rise 2
```

**Parameters**:
- **interval**: как часто проверять
- **timeout**: сколько ждать ответа
- **fails**: сколько failures для marking unhealthy
- **passes**: сколько successes для marking healthy

### Passive Health Checks

LB мониторит реальные requests:

```
Request 1 to S1 → 500 Error
Request 2 to S1 → Timeout
Request 3 to S1 → 500 Error

После N failures → S1 marked unhealthy
```

**NGINX**:
```nginx
upstream backend {
  server s1.example.com max_fails=3 fail_timeout=30s;
  server s2.example.com max_fails=3 fail_timeout=30s;
}
```

### Health Check Endpoints

**Простой**:
```javascript
app.get('/health', (req, res) => {
  res.status(200).send('OK');
});
```

**Детальный**:
```javascript
app.get('/health', async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    diskSpace: checkDiskSpace(),
  };

  const isHealthy = Object.values(checks).every(Boolean);
  res.status(isHealthy ? 200 : 503).json(checks);
});
```

## Load Balancer Архитектуры

### 1. Single Load Balancer

```
[Internet]
    ↓
   [LB]  ← SPOF!
 ↙  ↓  ↘
[S1][S2][S3]
```

**Проблема**: LB — single point of failure.

### 2. Active-Passive LB (High Availability)

```
[Internet]
 ↓      ↓
[LB1]  [LB2]
active passive

Virtual IP (VIP) указывает на active LB
Если LB1 упал → VIP переключается на LB2
```

**VRRP (Virtual Router Redundancy Protocol)** или **Keepalived**.

### 3. Active-Active LB (DNS Round Robin)

```
DNS:
example.com → 10.0.0.1 (LB1)
example.com → 10.0.0.2 (LB2)

Клиенты распределяются между LB через DNS
```

### 4. Multi-tier Load Balancing

```
      [Internet]
          ↓
    [Global LB] (DNS-based, GeoDNS)
       ↙        ↘
   [US LB]    [EU LB]
     ↓            ↓
[US Servers] [EU Servers]
```

### 5. Load Balancing для Microservices

```
[API Gateway / L7 LB]
   ↓       ↓       ↓
[Auth]  [Users]  [Orders]
         ↓
    [L4 LB] (для DB)
      ↙    ↘
   [DB1]  [DB2]
```

## Популярные Load Balancers

### 1. NGINX

**Тип**: L4 и L7
**Язык**: C
**Performance**: Отличная

**Конфигурация**:
```nginx
http {
  upstream backend {
    least_conn;
    server s1.example.com:8080 weight=3;
    server s2.example.com:8080 weight=2;
    server s3.example.com:8080;
  }

  server {
    listen 80;
    location / {
      proxy_pass http://backend;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }
}
```

### 2. HAProxy

**Тип**: L4 и L7
**Язык**: C
**Performance**: Отличная (рекорд: 2M+ RPS)

**Конфигурация**:
```
frontend http_front
  bind *:80
  default_backend http_back

backend http_back
  balance leastconn
  option httpchk GET /health
  server s1 192.168.1.1:8080 check
  server s2 192.168.1.2:8080 check
  server s3 192.168.1.3:8080 check
```

### 3. AWS Elastic Load Balancer (ELB)

**Типы**:
- **ALB** (Application LB) — L7, HTTP/HTTPS
- **NLB** (Network LB) — L4, TCP/UDP, ultra-high performance
- **CLB** (Classic LB) — legacy

**ALB Features**:
- Path-based routing
- Host-based routing
- WebSocket support
- HTTP/2 support
- Integrated с AWS WAF

### 4. Google Cloud Load Balancer

**Типы**:
- HTTP(S) Load Balancer — L7
- TCP/UDP Network Load Balancer — L4
- Internal Load Balancer

**Features**:
- Global load balancing
- Cross-region failover
- Anycast IPs

### 5. Traefik

**Тип**: L7
**Особенность**: Cloud-native, автоматическое обнаружение сервисов

**Docker integration**:
```yaml
services:
  app:
    image: myapp
    labels:
      - "traefik.http.routers.app.rule=Host(`example.com`)"
      - "traefik.http.services.app.loadbalancer.server.port=8080"
```

### 6. Envoy

**Тип**: L7
**Особенность**: Modern, используется в service mesh (Istio)

**Features**:
- Advanced load balancing
- Circuit breaking
- Retries
- Timeouts
- Observability (metrics, tracing)

## Практические рекомендации

### 1. Выбор алгоритма

```
Stateless app + одинаковые серверы
  → Round Robin

Stateless app + разные серверы
  → Weighted Round Robin

Stateful app или WebSockets
  → Least Connections или IP Hash

Caching layer
  → Consistent Hashing

Latency-sensitive
  → Least Response Time
```

### 2. Health Checks

- Используйте dedicated `/health` endpoint
- Проверяйте критичные dependencies (DB, Redis)
- Не делайте health check слишком тяжелым
- Используйте разумные timeouts (2-5 секунд)

### 3. Timeouts

```nginx
proxy_connect_timeout 5s;   # Подключение к backend
proxy_send_timeout 10s;     # Отправка request
proxy_read_timeout 30s;     # Чтение response
```

### 4. Connection Pooling

LB переиспользует connections к backend:

```nginx
upstream backend {
  server s1.example.com;
  keepalive 32;  # Pool size
}
```

**Преимущества**:
- Меньше overhead (TCP handshakes)
- Меньше latency

### 5. SSL Termination

LB расшифровывает SSL, backend получает plain HTTP:

```
[Client] --HTTPS--> [LB] --HTTP--> [Backend]
```

**Преимущества**:
✅ Offload SSL от backend (экономия CPU)
✅ Централизованное управление сертификатами
✅ Можно инспектировать трафик на LB

**Недостатки**:
❌ Трафик LB→Backend незашифрован (внутри datacenter обычно OK)

### 6. Monitoring

Метрики для мониторинга:
- Requests per second
- Response time (P50, P95, P99)
- Error rate (5xx errors)
- Active connections
- Backend health status
- Queue length (если есть)

## Что почитать дальше

- [HAProxy Documentation](http://www.haproxy.org/)
- [NGINX Load Balancing Guide](https://docs.nginx.com/nginx/admin-guide/load-balancer/)
- "The Art of Capacity Planning" by John Allspaw
- [AWS ELB Best Practices](https://docs.aws.amazon.com/elasticloadbalancing/)

## Проверьте себя

1. В чем разница между L4 и L7 load balancing?
2. Когда стоит использовать Least Connections вместо Round Robin?
3. Что такое sticky sessions и почему их лучше избегать?
4. Как работает IP Hash и какие у него недостатки?
5. Зачем нужны health checks и какие бывают типы?
6. Что такое SSL termination и каковы его преимущества?
7. Как сделать Load Balancer высокодоступным (избежать SPOF)?
8. В каких случаях L4 предпочтительнее L7?

## Ключевые выводы

- Load Balancer критичен для scalability и high availability
- L4 (TCP) быстрее, L7 (HTTP) умнее — выбор зависит от use case
- Round Robin — простой, Least Connections — учитывает нагрузку, IP Hash — для sticky sessions
- Избегайте sticky sessions — используйте shared session storage (Redis)
- Обязательно реализуйте health checks (active и passive)
- LB сам может быть SPOF — используйте Active-Passive или Active-Active setup
- SSL termination на LB снижает нагрузку на backend
- NGINX и HAProxy — популярные open-source решения
- Cloud LB (AWS ALB/NLB, GCP LB) упрощают операции и интегрируются с инфраструктурой
- Мониторьте latency, error rate, и backend health

---
**Предыдущий урок**: [WebSockets, Server-Sent Events, Long Polling](09-websockets-sse-long-polling.md)
**Следующий урок**: [API Gateway и Reverse Proxy](11-api-gateway-and-reverse-proxy.md)
