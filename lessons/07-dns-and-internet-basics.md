# DNS и основы работы интернета

## Введение

Когда вы набираете `www.google.com` в браузере, как компьютер знает, куда отправить запрос? За этой простотой скрывается сложная инфраструктура Domain Name System (DNS) — одна из фундаментальных технологий интернета.

Понимание DNS критично для system design, потому что:
- DNS влияет на latency (может добавить 20-100ms)
- DNS используется для load balancing и failover
- DNS — часть многих архитектурных решений (geo-routing, CDN)
- Проблемы с DNS могут сделать весь сервис недоступным

## Что такое DNS?

**DNS (Domain Name System)** — распределенная иерархическая система, которая переводит human-readable доменные имена (например, `google.com`) в IP адреса (например, `142.250.185.46`).

**Аналогия**: DNS — это телефонная книга интернета.

### Зачем нужен DNS?

**Проблема**: Компьютеры общаются по IP адресам, люди предпочитают имена.

**Без DNS**:
```
Вы: Хочу зайти на Google
Браузер: Окей, подключаюсь к 142.250.185.46
Вы: Что за цифры? Я хочу google.com!
```

**С DNS**:
```
Вы: google.com
DNS: 142.250.185.46
Браузер: Подключаюсь...
```

### IP адреса: краткое введение

#### IPv4
- Формат: 4 октета (4 × 8 бит)
- Пример: `192.168.1.1`
- Диапазон: 0.0.0.0 до 255.255.255.255
- Всего: ~4.3 миллиарда адресов (2^32)
- Проблема: адреса почти закончились

#### IPv6
- Формат: 8 групп по 4 hex цифры (128 бит)
- Пример: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`
- Всего: ~340 undecillion адресов (2^128)
- Решает проблему нехватки адресов

## Как работает DNS

### DNS Lookup процесс

Когда вы запрашиваете `www.example.com`:

```
1. Browser Cache
   ↓ (cache miss)
2. OS Cache
   ↓ (cache miss)
3. Recursive Resolver (ISP DNS)
   ↓
4. Root Nameserver (.)
   → "Спроси .com nameserver"
   ↓
5. TLD Nameserver (.com)
   → "Спроси example.com nameserver"
   ↓
6. Authoritative Nameserver (example.com)
   → "IP address: 93.184.216.34"
   ↓
7. Response возвращается через всю цепочку
   ↓
8. Browser получает IP и подключается
```

### Детальный разбор

#### 1. Browser Cache

Браузер запоминает DNS записи на время TTL (Time To Live).

```
$ chrome://net-internals/#dns
www.example.com → 93.184.216.34 (expires in 300s)
```

**TTL**: Время, в течение которого запись считается валидной.
- Короткий TTL (60s): быстрое обновление, но больше нагрузка на DNS
- Длинный TTL (86400s = 24h): меньше нагрузка, но медленное обновление

#### 2. OS Cache

Операционная система также кэширует DNS записи.

```bash
# Linux
$ cat /etc/resolv.conf
nameserver 8.8.8.8  # Google DNS
nameserver 1.1.1.1  # Cloudflare DNS

# Очистить кэш
$ sudo systemd-resolve --flush-caches  # Linux
$ sudo dscacheutil -flushcache  # macOS
$ ipconfig /flushdns  # Windows
```

#### 3. Recursive Resolver

**Recursive Resolver** (также Recursive DNS Server) — сервер, который делает всю работу по поиску IP адреса.

**Примеры**:
- ISP DNS: предоставляется вашим интернет-провайдером
- Google Public DNS: 8.8.8.8, 8.8.4.4
- Cloudflare DNS: 1.1.1.1, 1.0.0.1
- OpenDNS: 208.67.222.222

**Функции**:
- Кэширует ответы для будущих запросов
- Рекурсивно запрашивает DNS иерархию
- Может фильтровать вредоносные домены

#### 4. Root Nameservers

**13 логических Root Nameservers** (физически их сотни через anycast).

```
a.root-servers.net → 198.41.0.4
b.root-servers.net → 199.9.14.201
...
m.root-servers.net → 202.12.27.33
```

**Функция**: Знают адреса TLD nameservers (.com, .org, .ru и т.д.).

**Ответ Root NS**:
```
Query: www.example.com
Answer: Я не знаю, но спроси у .com nameserver по адресу X.X.X.X
```

#### 5. TLD (Top-Level Domain) Nameservers

**TLD** — последняя часть домена (.com, .org, .ru, .io и т.д.).

**Типы TLD**:
- **gTLD** (generic TLD): .com, .org, .net
- **ccTLD** (country code TLD): .ru, .uk, .de
- **New gTLD**: .app, .dev, .blog

**Функция**: Знают адреса Authoritative nameservers для доменов в их зоне.

**Ответ TLD NS**:
```
Query: www.example.com
Answer: Authoritative nameserver для example.com — ns1.example.com (IP: Y.Y.Y.Y)
```

#### 6. Authoritative Nameserver

**Authoritative Nameserver** — финальный источник правды о домене. Содержит DNS записи.

**Ответ Authoritative NS**:
```
Query: www.example.com
Answer: IP address 93.184.216.34, TTL 3600
```

### Типы DNS запросов

#### Recursive Query

Client просит resolver полностью разрешить имя.

```
Client → Resolver: "Дай мне IP для www.example.com"
Resolver: *делает всю работу*
Resolver → Client: "93.184.216.34"
```

#### Iterative Query

Resolver просит каждый сервер дать лучший ответ, который он знает.

```
Resolver → Root NS: "www.example.com?"
Root NS: "Я не знаю, но вот адрес .com nameserver"

Resolver → TLD NS: "www.example.com?"
TLD NS: "Я не знаю, но вот адрес example.com nameserver"

Resolver → Authoritative NS: "www.example.com?"
Authoritative NS: "93.184.216.34"
```

## DNS Record Types

### A Record (Address Record)

Связывает доменное имя с IPv4 адресом.

```
example.com.     IN  A  93.184.216.34
www.example.com. IN  A  93.184.216.34
```

### AAAA Record (IPv6 Address Record)

Связывает доменное имя с IPv6 адресом.

```
example.com. IN AAAA 2606:2800:220:1:248:1893:25c8:1946
```

### CNAME Record (Canonical Name)

Алиас для другого доменного имени.

```
www.example.com.  IN  CNAME  example.com.
blog.example.com. IN  CNAME  example.com.
```

**Use case**: Несколько поддоменов указывают на один сервер.

**Ограничение**: CNAME не может сосуществовать с другими записями для того же имени.

### MX Record (Mail Exchange)

Указывает почтовые серверы для домена.

```
example.com. IN MX 10 mail1.example.com.
example.com. IN MX 20 mail2.example.com.
```

**Число (priority)**: Меньше = выше приоритет.

### NS Record (Name Server)

Указывает authoritative nameservers для домена.

```
example.com. IN NS ns1.example.com.
example.com. IN NS ns2.example.com.
```

### TXT Record

Произвольный текст, часто для верификации.

```
example.com. IN TXT "v=spf1 include:_spf.google.com ~all"
example.com. IN TXT "google-site-verification=abc123..."
```

**Use cases**:
- SPF (Sender Policy Framework) для email
- Domain verification для сервисов
- DKIM (DomainKeys Identified Mail)

### SOA Record (Start of Authority)

Информация о DNS зоне.

```
example.com. IN SOA ns1.example.com. admin.example.com. (
    2024010101  ; Serial
    3600        ; Refresh (1 hour)
    900         ; Retry (15 min)
    1209600     ; Expire (2 weeks)
    86400       ; Minimum TTL (1 day)
)
```

## DNS в System Design

### 1. Load Balancing через DNS

**DNS Round Robin**: Возвращать несколько IP адресов и ротировать их.

```
Query: www.example.com
Answer:
  93.184.216.34  (Server 1)
  93.184.216.35  (Server 2)
  93.184.216.36  (Server 3)
```

Client случайным образом выбирает один из IP.

**Проблемы**:
- Не учитывает health серверов (может направить на упавший)
- Не учитывает нагрузку серверов
- Кэширование может нарушить распределение

**Улучшение**: GeoDNS (см. ниже).

### 2. GeoDNS (Geographic DNS)

Возвращать разные IP в зависимости от местоположения клиента.

```
Client из US → 192.0.2.1 (US datacenter)
Client из EU → 192.0.2.2 (EU datacenter)
Client из Asia → 192.0.2.3 (Asia datacenter)
```

**Провайдеры**:
- AWS Route 53
- Cloudflare DNS
- Google Cloud DNS
- Dyn

**Use cases**:
- Снижение latency (близкий datacenter)
- Compliance (данные остаются в регионе)
- Load distribution

### 3. Failover

Автоматическое переключение на backup при падении primary.

```
Health Check: Primary server (93.184.216.34) - DOWN
Action: DNS отвечает Backup server (93.184.216.35)
```

**TTL**: Короткий TTL (60s) для быстрого failover.

**Trade-off**: Короткий TTL = больше нагрузка на DNS.

### 4. CDN Integration

CDN использует DNS для направления пользователей на ближайший edge сервер.

```
Query: cdn.example.com
CloudFlare DNS → 104.16.132.229 (ближайший edge)
```

**Как работает**:
1. CNAME на CDN: `static.example.com CNAME example.cdn.com`
2. CDN DNS анализирует geo-location клиента
3. Возвращает IP ближайшего edge сервера

## DNS Security

### 1. DNS Spoofing/Cache Poisoning

**Атака**: Злоумышленник подделывает DNS ответ.

```
Legitimate:
example.com → 93.184.216.34

Poisoned:
example.com → 6.6.6.6 (attacker's server)
```

**Защита**: DNSSEC (см. ниже).

### 2. DDoS на DNS

**Атака**: Overwhelm DNS сервера запросами.

```
Millions of requests/sec to DNS resolver
→ DNS becomes unavailable
→ Nobody can resolve domains
→ Internet "breaks"
```

**Защита**:
- Anycast (распределение по множеству серверов)
- Rate limiting
- Caching
- DDoS mitigation services

### 3. DNSSEC (DNS Security Extensions)

**Проблема**: DNS ответы не аутентифицированы.

**Решение**: Цифровые подписи для DNS записей.

```
1. DNS response включает цифровую подпись
2. Client проверяет подпись с public key
3. Если подпись валидна → ответ подлинный
```

**Цепочка доверия**:
```
Root (.) → .com → example.com
Каждый уровень подписан вышестоящим
```

**Проблемы DNSSEC**:
- Сложность внедрения
- Увеличивает размер DNS ответов
- Не шифрует данные (только подписывает)

### 4. DNS over HTTPS (DoH) и DNS over TLS (DoT)

**Проблема**: DNS запросы в plaintext, ISP может видеть все сайты, которые вы посещаете.

**DoH (DNS over HTTPS)**:
- DNS запросы через HTTPS (port 443)
- Выглядят как обычный HTTPS трафик
- Поддерживается: Firefox, Chrome, Cloudflare (1.1.1.1)

**DoT (DNS over TLS)**:
- DNS запросы через TLS (port 853)
- Dedicated port для DNS
- Легче фильтровать

**Пример (Cloudflare DoH)**:
```bash
$ curl -H 'accept: application/dns-json' \
  'https://cloudflare-dns.com/dns-query?name=example.com&type=A'
```

## DNS Performance

### Latency Components

```
Total DNS Latency = Cache Lookup + Network RTT + Query Processing

Если в кэше: ~1-5ms
Если cache miss: ~20-100ms
  - Recursive resolver: 10-50ms
  - Root/TLD/Auth lookup: 10-50ms
```

### Оптимизации

#### 1. Prefetching

Браузер заранее резолвит домены, которые могут понадобиться.

```html
<!-- Подсказка браузеру -->
<link rel="dns-prefetch" href="//cdn.example.com">
<link rel="dns-prefetch" href="//api.example.com">
```

#### 2. Aggressive Caching

```
Короткий TTL (60s):
- Быстрое обновление
- Больше нагрузка на DNS
- Лучше для failover

Длинный TTL (86400s):
- Меньше latency (больше cache hits)
- Меньше нагрузка на DNS
- Медленное обновление
```

#### 3. Multiple Nameservers

```
example.com. IN NS ns1.example.com.  ; Primary
example.com. IN NS ns2.example.com.  ; Secondary (другой datacenter)
example.com. IN NS ns3.example.com.  ; Tertiary (третий datacenter)
```

**Redundancy**: Если один упадет, остальные работают.

#### 4. Anycast

Один IP адрес, множество серверов по всему миру. Запрос идет на ближайший.

```
8.8.8.8 (Google DNS):
- LA datacenter
- Tokyo datacenter
- Frankfurt datacenter
- ...

Пользователь в EU → автоматически Frankfurt datacenter
```

## DNS в больших системах

### AWS Route 53

**Features**:
- Managed DNS service
- Health checks и failover
- GeoDNS (Geolocation и Geoproximity routing)
- Traffic flow (visual editor для routing)
- Integration с AWS services (ELB, S3, CloudFront)

**Routing Policies**:
1. **Simple**: Один IP
2. **Weighted**: Распределение по весу (80% traffic на A, 20% на B)
3. **Latency-based**: Направление на сервер с минимальной latency
4. **Failover**: Primary/Secondary
5. **Geolocation**: По стране/континенту
6. **Geoproximity**: По расстоянию с bias
7. **Multi-value**: Несколько IP с health checks

### Cloudflare DNS

**Features**:
- Fastest DNS resolver (1.1.1.1)
- Global anycast network
- DDoS protection
- DNSSEC
- DNS Analytics
- Load balancing

**Performance**: Среднее разрешение ~10-15ms (быстрейший в индустрии).

### Google Cloud DNS

**Features**:
- Managed authoritative DNS
- 100% uptime SLA
- Global anycast
- DNSSEC
- Integration с GCP

## Практический пример: DNS для multi-region приложения

### Требования
- Приложение в 3 регионах: US, EU, Asia
- Пользователи должны попадать на ближайший регион
- Failover при падении региона
- Минимальная latency

### DNS Setup

```
1. Создаем A records для каждого региона:
   us.example.com   → 192.0.2.1
   eu.example.com   → 192.0.2.2
   asia.example.com → 192.0.2.3

2. Настраиваем GeoDNS для www.example.com:
   - Users в Americas → us.example.com
   - Users в Europe → eu.example.com
   - Users в Asia/Pacific → asia.example.com

3. Настраиваем health checks:
   - Каждые 30s проверяем /health endpoint
   - Если US упал → направляем на EU (failover)

4. Устанавливаем TTL:
   - A records: TTL 300s (5 min)
   - www CNAME: TTL 60s (быстрый failover)
```

### AWS Route 53 Config

```json
{
  "ResourceRecordSets": [
    {
      "Name": "www.example.com",
      "Type": "A",
      "SetIdentifier": "US-East",
      "GeoLocation": {
        "ContinentCode": "NA"
      },
      "TTL": 60,
      "ResourceRecords": [{"Value": "192.0.2.1"}],
      "HealthCheckId": "health-us-east"
    },
    {
      "Name": "www.example.com",
      "Type": "A",
      "SetIdentifier": "EU-West",
      "GeoLocation": {
        "ContinentCode": "EU"
      },
      "TTL": 60,
      "ResourceRecords": [{"Value": "192.0.2.2"}],
      "HealthCheckId": "health-eu-west"
    }
  ]
}
```

## Troubleshooting DNS

### Инструменты

#### 1. dig (Domain Information Groper)

```bash
# Простой запрос
$ dig example.com

# Конкретный record type
$ dig example.com A
$ dig example.com MX
$ dig example.com NS

# Указать DNS сервер
$ dig @8.8.8.8 example.com

# Trace полного пути
$ dig +trace example.com
```

#### 2. nslookup

```bash
$ nslookup example.com
Server:  8.8.8.8
Address: 8.8.8.8#53

Non-authoritative answer:
Name: example.com
Address: 93.184.216.34
```

#### 3. host

```bash
$ host example.com
example.com has address 93.184.216.34
example.com has IPv6 address 2606:2800:220:1:248:1893:25c8:1946
```

### Типичные проблемы

#### 1. DNS Propagation Delay

**Проблема**: Изменили A record, но старый IP еще кэшируется.

**Решение**:
- Подождать TTL времени
- Проверить разные DNS resolvers
- Уменьшить TTL заранее (за 24-48h до изменения)

#### 2. CNAME Flattening

**Проблема**: CNAME на root domain (example.com) запрещен RFC.

```
❌ example.com CNAME cdn.provider.com  # Недопустимо
✅ www.example.com CNAME cdn.provider.com  # OK
```

**Решение**: CNAME Flattening (Cloudflare, Route 53 Alias records).

#### 3. DNS Timeout

**Проблема**: DNS resolver не отвечает.

**Причины**:
- DDoS атака
- Firewall блокирует UDP port 53
- DNS сервер недоступен

**Решение**:
- Использовать multiple nameservers
- Fallback на альтернативный resolver (8.8.8.8)

## Что почитать дальше

### RFCs
- RFC 1035 — Domain Names - Implementation and Specification
- RFC 4034 — DNSSEC Resource Records
- RFC 8484 — DNS Queries over HTTPS (DoH)

### Ресурсы
- [How DNS Works (Comic)](https://howdns.works/)
- [DNS Explained](https://www.cloudflare.com/learning/dns/what-is-dns/)
- [DNS Performance Comparison](https://www.dnsperf.com/)

## Проверьте себя

1. Объясните полный путь DNS lookup от браузера до IP адреса.
2. В чем разница между recursive и iterative DNS query?
3. Какие 5 основных DNS record types вы знаете?
4. Как DNS используется для load balancing?
5. Что такое TTL и как он влияет на failover?
6. В чем разница между DoH и DoT?
7. Как GeoDNS помогает снизить latency?
8. Почему DNSSEC важен для безопасности?

## Ключевые выводы

- DNS переводит доменные имена в IP адреса через иерархическую систему
- Полный DNS lookup: Browser Cache → OS Cache → Recursive Resolver → Root NS → TLD NS → Authoritative NS
- Основные record types: A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), NS (nameserver), TXT (text)
- DNS используется для load balancing, failover, и geo-routing
- TTL определяет время кэширования — trade-off между latency и актуальностью
- DNS security: DNSSEC (подписи), DoH/DoT (шифрование), DDoS protection
- GeoDNS направляет пользователей на ближайший datacenter
- DNS может добавить 20-100ms latency — оптимизируйте через caching и prefetching
- Managed DNS services (Route 53, Cloudflare) упрощают сложные routing scenarios
- Всегда имейте multiple nameservers для redundancy

---
**Предыдущий урок**: [Consistent Hashing](06-consistent-hashing.md)
**Следующий урок**: [API Design: REST, GraphQL, gRPC](08-api-design-rest-graphql-grpc.md)
