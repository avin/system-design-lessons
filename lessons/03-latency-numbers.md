# Latency Numbers Every Programmer Should Know

## Введение

**Latency (задержка)** — время, необходимое для выполнения операции. Понимание типичных задержек различных операций критически важно для принятия архитектурных решений. Знание этих чисел помогает:
- Выбрать правильную технологию для задачи
- Идентифицировать bottlenecks
- Оценить performance impact различных решений
- Обосновать архитектурные выборы на интервью

Эти числа впервые опубликовал Jeff Dean (легендарный инженер Google) в 2010 году, и они стали классикой.

## Актуализированные latency numbers (2025)

| Операция | Latency | Примечание |
|----------|---------|------------|
| L1 cache reference | 0.5 ns | |
| Branch mispredict | 5 ns | |
| L2 cache reference | 7 ns | 14x L1 cache |
| Mutex lock/unlock | 25 ns | |
| Main memory reference | 100 ns | 20x L2 cache, 200x L1 cache |
| Compress 1KB с Snappy | 3,000 ns (3 µs) | |
| Отправка 1KB по сети 1 Gbps | 10,000 ns (10 µs) | |
| Чтение 4KB с SSD | 150,000 ns (150 µs) | |
| Чтение 1MB последовательно из памяти | 250,000 ns (250 µs) | |
| Roundtrip в пределах datacenter | 500,000 ns (500 µs) | |
| Чтение 1MB последовательно с SSD | 1,000,000 ns (1 ms) | 4x memory |
| Disk seek | 10,000,000 ns (10 ms) | 20x datacenter roundtrip |
| Чтение 1MB последовательно с HDD | 20,000,000 ns (20 ms) | 80x memory, 20x SSD |
| Отправка пакета US -> EU -> US | 150,000,000 ns (150 ms) | |

## Визуализация масштаба

Чтобы лучше понять разницу, представим, что 1 секунда = 1 секунда:

| Операция | Если 1 сек = 1 сек | Реальное время |
|----------|---------------------|----------------|
| L1 cache | 0.5 сек | 0.5 ns |
| L2 cache | 7 сек | 7 ns |
| Main memory | 100 сек (~1.5 мин) | 100 ns |
| SSD read 4KB | 1.7 дня | 150 µs |
| Network within DC | 5.8 дней | 500 µs |
| SSD read 1MB | 11.5 дней | 1 ms |
| Disk seek | 16.5 недель | 10 ms |
| HDD read 1MB | 7.8 месяцев | 20 ms |
| Packet US-EU-US | 4.8 лет | 150 ms |

**Разница огромна!** Memory доступ в 1.5 минуты vs диск в 7.8 месяцев.

## Единицы измерения latency

Важно свободно ориентироваться в порядках величин:

| Единица | Аббревиатура | В секундах | Пример |
|---------|--------------|------------|--------|
| Nanosecond | ns | 10^-9 | L1 cache access |
| Microsecond | µs | 10^-6 (1,000 ns) | SSD random read |
| Millisecond | ms | 10^-3 (1,000 µs) | HDD read, network call |
| Second | s | 1 (1,000 ms) | Heavy computation |

## Практические выводы

### 1. Memory vs Disk: пропасть в производительности

**Memory**: 100 ns
**SSD**: 150 µs (1,500x медленнее)
**HDD**: 20 ms (200,000x медленнее)

**Вывод**: Кэшируйте данные в памяти везде, где возможно.

**Пример**:
```
Чтение 100 записей из БД:
- Из памяти (cache hit): 100 × 100ns = 10µs
- С SSD: 100 × 150µs = 15ms (1,500x медленнее!)
- С HDD: 100 × 10ms (seek) = 1 секунда (100,000x медленнее!)
```

### 2. Последовательное vs Случайное чтение

**SSD последовательное чтение 1MB**: 1 ms
**SSD случайное чтение 4KB × 250**: 150µs × 250 = 37.5 ms

**Вывод**: Последовательное чтение в 37x быстрее! Проектируйте структуры данных для последовательного доступа.

**Применение**:
- Логи пишите append-only (последовательно)
- Используйте LSM-trees (RocksDB, Cassandra) вместо B-trees для write-heavy workloads
- Batch-читайте данные вместо множества мелких запросов

### 3. Network latency доминирует

**Memory access**: 100 ns
**Roundtrip within datacenter**: 500 µs (5,000x медленнее)
**Roundtrip across continent**: 150 ms (1,500,000x медленнее)

**Вывод**: Минимизируйте сетевые вызовы любой ценой.

**Антипаттерн** (N+1 queries):
```
// Получить пользователя
user = db.query("SELECT * FROM users WHERE id = 1")  // 500µs

// Получить каждый пост отдельно
for post_id in user.post_ids:  // 100 постов
    post = db.query("SELECT * FROM posts WHERE id = ?", post_id)  // 500µs × 100

Total: 500µs + 50ms = 50.5ms
```

**Правильно** (batch query):
```
user = db.query("SELECT * FROM users WHERE id = 1")  // 500µs
posts = db.query("SELECT * FROM posts WHERE id IN (?)", user.post_ids)  // 500µs

Total: 1ms (50x быстрее!)
```

### 4. Компрессия может ускорить

**Отправка 1MB некомпрессированных по сети 1Gbps**: 10 ms
**Compress 1MB (ratio 5:1)**: 3 ms
**Отправка 200KB**: 2 ms
**Decompress на другой стороне**: 3 ms

**Total**: 3 + 2 + 3 = 8 ms < 10 ms

**Вывод**: Компрессия может уменьшить latency, если bandwidth ограничен.

**Когда применять**:
- Медленные сети (mobile, географически распределенные)
- Большие payload (JSON/XML API responses)
- Batch data transfers

**Когда не применять**:
- Данные уже сжаты (изображения, видео)
- Очень быстрая сеть внутри datacenter
- CPU bottleneck

### 5. Пропускная способность vs Latency

**Latency**: время для первого байта
**Bandwidth**: сколько байт в секунду

**Аналогия**: Latency — время в пути до магазина, Bandwidth — размер корзины.

**Пример**:
- Datacenter network: latency 0.5ms, bandwidth 10Gbps
- Чтение 1KB: 0.5ms (latency доминирует)
- Чтение 1GB: 0.5ms + 800ms = 800.5ms (bandwidth доминирует)

**Вывод**: Для маленьких данных важна latency, для больших — bandwidth.

## Архитектурные решения на основе latency

### Выбор БД: Local vs Remote

**Встроенная БД** (SQLite, RocksDB):
- Access latency: ~100 µs (SSD)
- Throughput: ограничен одной машиной

**Удаленная БД** (PostgreSQL, MySQL):
- Access latency: ~500 µs + query time
- Throughput: высокий (можно scale горизонтально)

**Применение**:
- Встроенная: мобильные приложения, edge computing, read-only replicas
- Удаленная: веб-приложения, микросервисы

### Кэширование: Where to Cache?

**1. Application-level cache (в памяти процесса)**
- Latency: 100 ns
- Scope: один процесс
- Пример: HashMap в Java

**2. Distributed cache (Redis, Memcached)**
- Latency: 500 µs - 1 ms
- Scope: все серверы
- Пример: session store, API cache

**3. CDN**
- Latency: 10-50 ms (географически близко)
- Scope: глобальный
- Пример: статические файлы, изображения

**Стратегия**: кэшируйте на всех уровнях с учетом latency budget.

### API Design: Chatty vs Chunky

**Chatty** (много мелких вызовов):
```
getUserProfile()      // 1ms
getUserPosts()        // 1ms
getUserFriends()      // 1ms
getUserPhotos()       // 1ms
Total: 4ms
```

**Chunky** (один большой вызов):
```
getUserData(include: ['profile', 'posts', 'friends', 'photos'])
Total: 1ms
```

**Trade-off**:
- Chatty: проще в реализации, лучше для кэширования отдельных частей
- Chunky: меньше latency, но больше данных передается

**Решение**: используйте GraphQL или специальные endpoint'ы для мобильных приложений.

### Database Indexing

**Без индекса** (full table scan):
```
Scan 1M rows × 100ns (memory) = 100ms
```

**С индексом** (B-tree, depth 3):
```
3 seeks × 150µs (SSD) = 450µs (220x быстрее!)
```

**Вывод**: Индексы критичны для performance, но добавляют overhead на запись.

## Latency Budget для реальных систем

### Web Application

**Цель**: 100ms для 99 percentile (P99)

**Breakdown**:
- DNS lookup: 10ms
- TCP handshake: 10ms
- TLS handshake: 20ms
- HTTP request/response: 10ms
- Server processing: **50ms** ← ваш budget
- Остается для операций: 50ms

**Что можно сделать за 50ms?**:
- Cache lookup (Redis): 1ms
- Database query (indexed): 5ms
- Business logic: 10ms
- Rendering: 10ms
- Запас: 24ms

**Если превышаете**: используйте async processing, background jobs.

### Mobile Application

**Цель**: 300ms для P99 (пользователи более терпеливы на мобильных)

**Дополнительные факторы**:
- Mobile network latency: 50-200ms
- Lower bandwidth
- Less CPU power

**Оптимизации**:
- Aggressive caching
- Prefetching
- Progressive loading
- Optimistic UI updates

### Real-time Systems (Gaming, Trading)

**Цель**: <10ms для P99

**Требует**:
- In-memory все данные
- Co-location серверов
- Оптимизация на уровне CPU cache
- Low-latency networking (kernel bypass)

## Мониторинг latency

### Метрики

Не смотрите только на average — он обманчив!

**Percentiles**:
- **P50 (median)**: 50% запросов быстрее
- **P95**: 95% запросов быстрее
- **P99**: 99% запросов быстрее
- **P999**: 99.9% запросов быстрее

**Пример**:
- P50: 10ms
- P95: 50ms
- P99: 200ms
- P999: 5s (!)

**Проблема**: если P999 = 5s, значит 0.1% пользователей получают ужасный experience. При 1M requests/day это 1,000 недовольных пользователей!

### Tail Latency

**Tail latency** — высокие значения latency на хвосте распределения (P99, P999).

**Почему важно**:
- Пользователи запоминают плохой experience
- В микросервисной архитектуре один медленный сервис замедляет всю цепочку
- SLA обычно на P99, не на average

**Причины tail latency**:
- Garbage collection паузы
- Disk I/O (если cache miss)
- Lock contention
- Packet loss и TCP retransmissions
- Noisy neighbors (в cloud)

**Решения**:
- Hedged requests (отправить дубликат запроса если первый медленный)
- Circuit breakers
- Async processing для медленных операций

## Практические упражнения

### Задача 1: Cache or Not?

У вас есть API endpoint, который:
- Делает 5 запросов к БД: 5 × 5ms = 25ms
- Делает computation: 10ms
- Total: 35ms
- Вызывается 1000 раз в секунду

Стоит ли кэшировать результат в Redis (1ms latency)?

<details>
<summary>Решение</summary>

**Без cache**:
- Total latency: 35ms
- Load на БД: 5,000 queries/sec

**С cache (90% hit rate)**:
- Cache hit: 1ms (900 requests)
- Cache miss: 35ms + 1ms (set cache) = 36ms (100 requests)
- Weighted average: 0.9 × 1 + 0.1 × 36 = 4.5ms
- Load на БД: 500 queries/sec

**Вывод**: Да, кэшировать! Latency снижена в 7.7x, load на БД в 10x.
</details>

### Задача 2: Optimize Network Calls

У вас микросервис, который для обработки запроса:
- Вызывает Auth Service: 2ms
- Вызывает User Service: 3ms
- Вызывает Payment Service: 5ms
- Делает computation: 5ms
- Total: 15ms

Как оптимизировать?

<details>
<summary>Решение</summary>

**Последовательно**:
```
Auth (2ms) → User (3ms) → Payment (5ms) → Compute (5ms) = 15ms
```

**Параллельно** (если нет зависимостей):
```
     ┌─ Auth (2ms)
     ├─ User (3ms)
     └─ Payment (5ms)
         └─ Compute (5ms)

Total: max(2, 3, 5) + 5 = 10ms
```

**Вывод**: Используйте async/await или параллельные вызовы. Latency снижена с 15ms до 10ms (1.5x).
</details>

### Задача 3: N+1 Problem

Нужно отобразить 20 постов, каждый с автором и комментариями.

**Текущая реализация**:
```
posts = db.query("SELECT * FROM posts LIMIT 20")  // 5ms
for post in posts:
    author = db.query("SELECT * FROM users WHERE id = ?", post.author_id)  // 5ms × 20
    comments = db.query("SELECT * FROM comments WHERE post_id = ?", post.id)  // 5ms × 20
```

Total: 5ms + 200ms = 205ms

Как оптимизировать?

<details>
<summary>Решение</summary>

**Batch queries**:
```
posts = db.query("SELECT * FROM posts LIMIT 20")  // 5ms

author_ids = [post.author_id for post in posts]
authors = db.query("SELECT * FROM users WHERE id IN (?)", author_ids)  // 5ms

post_ids = [post.id for post in posts]
comments = db.query("SELECT * FROM comments WHERE post_id IN (?)", post_ids)  // 5ms

Total: 5 + 5 + 5 = 15ms (13.6x быстрее!)
```

**Альтернатива**: JOIN query
```
results = db.query("""
    SELECT posts.*, users.*, comments.*
    FROM posts
    LEFT JOIN users ON posts.author_id = users.id
    LEFT JOIN comments ON posts.id = comments.post_id
    WHERE posts.id IN (SELECT id FROM posts LIMIT 20)
""")  // 10-15ms
```
</details>

## Latency для различных систем

### Storage Systems

| Система | Read Latency | Write Latency |
|---------|--------------|---------------|
| Redis | 0.1-1 ms | 0.1-1 ms |
| Memcached | 0.1-1 ms | 0.1-1 ms |
| PostgreSQL (local) | 1-5 ms | 1-5 ms |
| PostgreSQL (remote) | 5-20 ms | 5-20 ms |
| MongoDB | 1-10 ms | 1-10 ms |
| Cassandra (local DC) | 1-10 ms | 1-10 ms |
| S3 (standard) | 100-200 ms | 100-200 ms |
| DynamoDB | 5-20 ms | 5-20 ms |

### Message Queues

| Система | Latency |
|---------|---------|
| RabbitMQ | 1-5 ms |
| Kafka | 2-10 ms |
| SQS | 10-50 ms |
| Kinesis | 200-500 ms (batching) |

### Network

| Соединение | Latency |
|------------|---------|
| Localhost | 0.1-0.5 ms |
| Same datacenter | 0.5-2 ms |
| Different datacenter (same region) | 5-10 ms |
| Cross-continent | 50-200 ms |
| Satellite | 500-700 ms |

## Что почитать дальше

- [Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832) — оригинальный список
- [Interactive Latency Chart](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
- "Designing Data-Intensive Applications" Chapter 1 — детальный разбор latency
- [brendangregg.com](http://www.brendangregg.com/linuxperf.html) — performance analysis

## Проверьте себя

1. Во сколько раз SSD медленнее памяти?
2. Что быстрее: 100 random reads по 4KB с SSD или 1 sequential read 400KB?
3. Почему сетевые вызовы — главный источник latency в распределенных системах?
4. Что такое tail latency и почему он важен?
5. В чем разница между latency и bandwidth?
6. Почему percentiles важнее average при измерении latency?

## Ключевые выводы

- Memory в 1,000x быстрее SSD, в 100,000x быстрее HDD — кэшируйте в памяти
- Последовательное чтение в 10-100x быстрее случайного — проектируйте структуры данных соответственно
- Network calls доминируют latency — минимизируйте их количество через batching
- Смотрите на percentiles (P99, P999), не на average
- Latency budget — планируйте, сколько времени можете потратить на каждую операцию
- Компрессия может снизить latency если bandwidth ограничен
- Всегда измеряйте latency в production — цифры в документации могут отличаться

---

**Предыдущий урок**: [Back-of-the-envelope calculations](02-back-of-the-envelope-calculations.md)
**Следующий урок**: [Функциональные и нефункциональные требования](04-funkcionalnye-i-nefunkcionalnye-trebovaniya.md)
