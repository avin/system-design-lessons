# Денормализация и когда её применять

## Что такое денормализация?

**Денормализация** — намеренное добавление избыточности (дублирование данных) в БД для улучшения производительности чтения за счет производительности записи и усложнения поддержки.

**Противоположность нормализации**.

## Нормализация vs Денормализация

### Нормализация

**Цель**: Устранить дублирование данных.

**Нормализованная схема**:
```sql
-- Users table
users
id | name          | email
1  | John Doe      | john@example.com
2  | Jane Smith    | jane@example.com

-- Posts table
posts
id | user_id | title           | content
1  | 1       | First Post      | ...
2  | 1       | Second Post     | ...
3  | 2       | Jane's Post     | ...

-- Comments table
comments
id | post_id | user_id | text
1  | 1       | 2       | Great post!
2  | 1       | 1       | Thanks!
```

**Получение данных**: Требует JOINs
```sql
-- Get post with author and comments
SELECT
    posts.*,
    users.name as author_name,
    COUNT(comments.id) as comment_count
FROM posts
JOIN users ON posts.user_id = users.id
LEFT JOIN comments ON posts.id = comments.post_id
WHERE posts.id = 1
GROUP BY posts.id, users.name;
```

**Преимущества нормализации**:
✅ Нет дублирования данных
✅ Проще обновления (один источник правды)
✅ Меньше storage
✅ Data integrity (constraints)

**Недостатки нормализации**:
❌ JOINs медленные на больших таблицах
❌ Сложные queries
❌ Больше round-trips к БД

### Денормализация

**Денормализованная схема**:
```sql
posts
id | title       | content | author_id | author_name  | comment_count
1  | First Post  | ...     | 1         | John Doe     | 2
2  | Second Post | ...     | 1         | John Doe     | 0
3  | Jane's Post | ...     | 2         | Jane Smith   | 5
```

**Получение данных**: Простой SELECT
```sql
SELECT * FROM posts WHERE id = 1;
-- Все данные в одной таблице, нет JOINs!
```

**Преимущества денормализации**:
✅ Быстрые reads (нет JOINs)
✅ Простые queries
✅ Меньше latency

**Недостатки денормализации**:
❌ Дублирование данных (больше storage)
❌ Сложнее updates (нужно обновлять в нескольких местах)
❌ Риск inconsistency

## Когда применять денормализацию

### 1. Read-heavy workload

```
Reads: 10,000 QPS
Writes: 100 QPS

Read performance важнее write performance
→ Денормализация
```

### 2. Joins too expensive

```sql
-- Query с 5 JOINs на таблицах по 100M строк
SELECT ...
FROM orders
JOIN users ON ...
JOIN products ON ...
JOIN categories ON ...
JOIN reviews ON ...
WHERE ...

Execution time: 5 seconds
→ Денормализируйте часто используемые данные
```

### 3. Distributed systems

```
Users: Shard 1
Posts: Shard 2
Comments: Shard 3

Cross-shard JOIN невозможен
→ Денормализация необходима
```

### 4. NoSQL databases

MongoDB, Cassandra не поддерживают JOINs.
```javascript
// MongoDB: embedded denormalization
{
  _id: 1,
  title: "My Post",
  author: {  // Denormalized
    id: 123,
    name: "John Doe",
    avatar: "url"
  },
  comments: [  // Denormalized
    { user: "Jane", text: "Great!" },
    { user: "Bob", text: "Thanks!" }
  ]
}
```

### 5. Analytics и reporting

```sql
-- Reporting query каждый день
SELECT
    DATE(created_at) as date,
    COUNT(*) as orders,
    SUM(total) as revenue
FROM orders
GROUP BY DATE(created_at);

-- Вместо этого: materialized view или отдельная таблица
daily_stats
date       | orders | revenue
2024-01-01 | 1000   | 50000
2024-01-02 | 1200   | 60000
```

### 6. Кэширование часто читаемых данных

```
User profile читается 1000 раз/сек
User profile обновляется 1 раз/день

→ Денормализуйте в Redis cache
```

## Паттерны денормализации

### 1. Duplication (Дублирование)

**Пример**: Хранить author_name в posts таблице.

**Нормализованная**:
```sql
posts: (id, user_id, title)
users: (id, name)

Query: JOIN для получения author_name
```

**Денормализованная**:
```sql
posts: (id, user_id, author_name, title)

Query: SELECT без JOIN
```

**Trade-off**:
- Быстрее reads
- При изменении username нужно обновить все posts этого user

**Update strategy**:
```python
def update_username(user_id, new_name):
    # Update users table
    db.execute("UPDATE users SET name = ? WHERE id = ?", (new_name, user_id))

    # Update denormalized data
    db.execute("UPDATE posts SET author_name = ? WHERE user_id = ?", (new_name, user_id))
```

### 2. Aggregation (Агрегация)

**Пример**: Хранить count комментариев в posts.

**Нормализованная**:
```sql
SELECT COUNT(*) FROM comments WHERE post_id = 1;
-- На каждое отображение поста!
```

**Денормализованная**:
```sql
posts: (id, title, comment_count)

SELECT comment_count FROM posts WHERE id = 1;
```

**Update strategy**:
```python
def add_comment(post_id, text):
    # Insert comment
    db.execute("INSERT INTO comments (post_id, text) VALUES (?, ?)", (post_id, text))

    # Increment counter
    db.execute("UPDATE posts SET comment_count = comment_count + 1 WHERE id = ?", (post_id,))
```

**Alternative**: Trigger
```sql
CREATE TRIGGER increment_comment_count
AFTER INSERT ON comments
FOR EACH ROW
BEGIN
    UPDATE posts
    SET comment_count = comment_count + 1
    WHERE id = NEW.post_id;
END;
```

### 3. Embedding (Встраивание)

**NoSQL pattern**: Embed связанные документы.

**Нормализованная** (SQL-style):
```javascript
// users collection
{ _id: 1, name: "John" }

// posts collection
{ _id: 1, user_id: 1, title: "Post" }
```

**Денормализованная** (embedded):
```javascript
// posts collection
{
  _id: 1,
  title: "Post",
  author: {
    id: 1,
    name: "John",
    avatar: "url"
  }
}
```

**Когда embedding**:
- One-to-Few relationships
- Данные читаются вместе
- Embedded данные редко меняются

**Когда НЕ embedding**:
- One-to-Many с большим Many (unbounded growth)
- Embedded данные часто меняются
- Нужно query embedded данные отдельно

### 4. Materialized Views

**Пример**: Дорогой query → pre-compute результаты.

**Expensive query**:
```sql
-- Daily revenue report (heavy aggregation)
SELECT
    DATE(created_at) as date,
    SUM(total) as revenue,
    COUNT(*) as order_count,
    AVG(total) as avg_order
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY DATE(created_at);
```

**Materialized view**:
```sql
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT
    DATE(created_at) as date,
    SUM(total) as revenue,
    COUNT(*) as order_count,
    AVG(total) as avg_order
FROM orders
GROUP BY DATE(created_at);

-- Query из materialized view (fast)
SELECT * FROM daily_revenue WHERE date = '2024-01-15';
```

**Refresh strategy**:
```sql
-- Manual refresh
REFRESH MATERIALIZED VIEW daily_revenue;

-- Scheduled (cron)
-- Daily at midnight
```

**PostgreSQL**:
```sql
CREATE MATERIALIZED VIEW view_name AS SELECT ...;
REFRESH MATERIALIZED VIEW view_name;
```

**MySQL**: Нет native materialized views, используйте trigger-based tables.

### 5. Denormalized Summary Tables

**Пример**: User statistics table.

```sql
-- Expensive query
SELECT
    user_id,
    COUNT(*) as post_count,
    SUM(views) as total_views,
    MAX(created_at) as last_post_date
FROM posts
GROUP BY user_id;

-- Denormalized summary table
user_stats
user_id | post_count | total_views | last_post_date
1       | 42         | 10000       | 2024-01-15
2       | 15         | 5000        | 2024-01-10

-- Update on post creation
INSERT INTO user_stats (user_id, post_count, total_views)
VALUES (1, 1, 0)
ON CONFLICT (user_id)
DO UPDATE SET
    post_count = user_stats.post_count + 1,
    last_post_date = NOW();
```

### 6. Caching Layer

**Денормализация через cache** (Redis, Memcached).

```python
def get_user_with_stats(user_id):
    # Check cache
    cached = redis.get(f"user:{user_id}:full")
    if cached:
        return json.loads(cached)

    # Cache miss: query БД (с JOINs)
    user = db.query("""
        SELECT users.*,
               COUNT(posts.id) as post_count,
               COUNT(followers.id) as follower_count
        FROM users
        LEFT JOIN posts ON users.id = posts.user_id
        LEFT JOIN followers ON users.id = followers.following_id
        WHERE users.id = ?
        GROUP BY users.id
    """, (user_id,))

    # Store в cache
    redis.setex(f"user:{user_id}:full", 3600, json.dumps(user))
    return user
```

**Invalidation**:
```python
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = ?", (user_id,))

    # Invalidate cache
    redis.delete(f"user:{user_id}:full")
```

## Управление consistency

### Проблема: Inconsistent data

```sql
-- Денормализация: author_name в posts
posts
id | user_id | author_name | title
1  | 123     | John Doe    | Post 1
2  | 123     | John Doe    | Post 2

-- User меняет имя
UPDATE users SET name = 'Jonathan Doe' WHERE id = 123;

-- Но забывают обновить posts!
-- Теперь:
users.name = 'Jonathan Doe'
posts.author_name = 'John Doe'  (stale!)
```

### Решения

#### 1. Transactional updates

```python
def update_username(user_id, new_name):
    with db.transaction():
        db.execute("UPDATE users SET name = ? WHERE id = ?", (new_name, user_id))
        db.execute("UPDATE posts SET author_name = ? WHERE user_id = ?", (new_name, user_id))
```

**Проблема**: Может быть медленным если много posts.

#### 2. Background jobs

```python
def update_username(user_id, new_name):
    # Update users table (fast)
    db.execute("UPDATE users SET name = ? WHERE id = ?", (new_name, user_id))

    # Queue background job
    queue.enqueue('update_denormalized_username', user_id, new_name)

# Worker
def update_denormalized_username(user_id, new_name):
    db.execute("UPDATE posts SET author_name = ? WHERE user_id = ?", (new_name, user_id))
```

**Trade-off**: Eventual consistency (короткий период inconsistency).

#### 3. Triggers (database-level)

```sql
CREATE TRIGGER update_posts_author_name
AFTER UPDATE ON users
FOR EACH ROW
WHEN OLD.name != NEW.name
BEGIN
    UPDATE posts
    SET author_name = NEW.name
    WHERE user_id = NEW.id;
END;
```

**Преимущество**: Автоматическое, невозможно забыть.
**Недостаток**: Performance overhead, сложнее debugging.

#### 4. Accept inconsistency

Для некритичных данных accept short-term inconsistency.

```python
# Author name обновится eventually (background job)
# В течение нескольких минут может быть stale
# Для большинства приложений — приемлемо
```

#### 5. Read-time resolution

Вместо хранения денормализованных данных, compute при чтении.

```python
# Не храним author_name в posts
# При чтении JOIN или lookup
def get_post(post_id):
    post = db.query("SELECT * FROM posts WHERE id = ?", (post_id,))
    author = db.query("SELECT name FROM users WHERE id = ?", (post.user_id,))
    post['author_name'] = author['name']
    return post

# Кэшируем результат
cache.set(f"post:{post_id}", post, ttl=300)
```

## Примеры из практики

### Пример 1: Twitter timeline

**Проблема**: Генерация timeline — дорогой query.

**Нормализованная**:
```sql
-- Get timeline: posts от followed users
SELECT posts.*
FROM posts
JOIN follows ON posts.user_id = follows.following_id
WHERE follows.follower_id = ?
ORDER BY posts.created_at DESC
LIMIT 20;

-- Для celebrity с 10M followers — очень медленно!
```

**Денормализованная** (fan-out on write):
```python
# При создании поста:
def create_post(user_id, content):
    post = db.insert_post(user_id, content)

    # Fan-out: insert в timeline каждого follower
    followers = db.get_followers(user_id)
    for follower_id in followers:
        db.insert_timeline(follower_id, post.id)

# Чтение timeline (fast):
def get_timeline(user_id):
    return db.query("SELECT * FROM timeline WHERE user_id = ? ORDER BY created_at DESC LIMIT 20", (user_id,))
```

**Hybrid approach** (Twitter):
- Regular users: fan-out on write (денормализация)
- Celebrities: fan-out on read (нормализация, compute при чтении)

### Пример 2: E-commerce product catalog

**Денормализация**: Product info embedded в order.

```sql
-- Если храним только product_id
orders
id | product_id | quantity | price_at_purchase
1  | 123        | 2        | 49.99

-- Проблема: если product удален или изменен, теряем историю

-- Денормализация: embed product info
orders
id | product_id | product_name | product_price | quantity | total
1  | 123        | Widget X     | 49.99         | 2        | 99.98

-- Теперь есть snapshot продукта на момент покупки
```

### Пример 3: MongoDB blog

**Денормализация**: Comments embedded в post.

```javascript
// Нормализованная (SQL-style)
posts: { _id: 1, title: "Post", user_id: 123 }
comments: [
  { _id: 1, post_id: 1, user_id: 456, text: "Great!" },
  { _id: 2, post_id: 1, user_id: 789, text: "Thanks!" }
]

// Денормализованная (embedded)
{
  _id: 1,
  title: "Post",
  author: { id: 123, name: "John" },
  comments: [
    { user: "Jane", text: "Great!" },
    { user: "Bob", text: "Thanks!" }
  ],
  comment_count: 2
}

// Single query для всех данных!
```

**Ограничение**: Если комментариев слишком много (1000+), embedding не подходит (document size limit 16MB в MongoDB).

**Hybrid**:
```javascript
// Post document
{
  _id: 1,
  title: "Post",
  comment_count: 1523,  // Денормализация
  recent_comments: [    // Только последние 10
    { user: "Jane", text: "Great!" },
    ...
  ]
}

// Отдельная коллекция для всех комментариев
comments: [
  { post_id: 1, user: "...", text: "..." },
  ...
]
```

## Best Practices

### 1. Денормализуйте выборочно

Не всё подряд, только:
- Часто читаемые данные
- Данные, которые редко меняются
- Queries с expensive JOINs

### 2. Документируйте денормализацию

```python
# models.py
class Post(models.Model):
    user_id = models.ForeignKey(User)
    author_name = models.CharField()  # DENORMALIZED from users.name
    comment_count = models.IntegerField()  # DENORMALIZED count from comments

# Документируйте update logic
# When updating User.name:
# 1. Update users table
# 2. Update posts.author_name (background job)
```

### 3. Автоматизируйте updates

```python
# Используйте ORM signals, triggers, или CDC (Change Data Capture)
@receiver(post_save, sender=User)
def update_denormalized_username(sender, instance, **kwargs):
    if 'name' in instance.changed_fields:
        update_posts_author_name.delay(instance.id, instance.name)
```

### 4. Мониторьте consistency

```python
# Периодически проверяйте consistency
def check_author_name_consistency():
    inconsistent = db.query("""
        SELECT posts.id, posts.author_name, users.name
        FROM posts
        JOIN users ON posts.user_id = users.id
        WHERE posts.author_name != users.name
    """)

    if inconsistent:
        alert("Denormalized data inconsistency detected", inconsistent)
```

### 5. Используйте версионирование

```javascript
// Version field для tracking staleness
{
  _id: 1,
  title: "Post",
  author: { id: 123, name: "John", version: 5 },  // Version of user data
}

// При чтении check version
user_version = get_user_version(123)  // Current version: 7
if post.author.version < user_version:
    refresh_author_data()
```

## Альтернативы денормализации

### 1. Better indexing

Иногда правильные индексы достаточны.

```sql
-- Вместо денормализации
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);

-- Теперь query быстрый даже с JOIN
```

### 2. Caching

```python
# Cache результаты queries вместо денормализации
@cache(ttl=300)
def get_post_with_author(post_id):
    return db.query("SELECT posts.*, users.name FROM posts JOIN users ...")
```

### 3. Materialized views

```sql
-- View auto-maintained БД
CREATE MATERIALIZED VIEW user_post_stats AS
SELECT user_id, COUNT(*) as post_count
FROM posts
GROUP BY user_id;
```

### 4. Read replicas

```
Writes → Master
Complex reads → Read replica (не влияет на master)
```

### 5. Separate OLAP database

```
OLTP (transactional) → Normalized PostgreSQL
OLAP (analytics) → Denormalized data warehouse (Redshift, BigQuery)
```

## Что почитать дальше

- "Designing Data-Intensive Applications" by Martin Kleppmann
- MongoDB Schema Design Patterns
- "High Performance MySQL" — Denormalization chapter

## Проверьте себя

1. В чем разница между нормализацией и денормализацией?
2. Когда стоит применять денормализацию?
3. Какие основные паттерны денормализации существуют?
4. Как управлять consistency денормализованных данных?
5. Что такое fan-out on write vs fan-out on read?
6. Когда embedding подходит в NoSQL?
7. Какие альтернативы денормализации?
8. Как мониторить consistency денормализованных данных?

## Ключевые выводы

- Денормализация — trade-off: быстрые reads за счет медленных writes и сложности
- Применяйте для read-heavy workloads и expensive JOINs
- Основные паттерны: duplication, aggregation, embedding, materialized views
- Управление consistency — ключевая сложность (transactions, triggers, background jobs)
- NoSQL требует денормализации (нет JOINs)
- Distributed systems требуют денормализации (cross-shard JOINs)
- Документируйте денормализацию и автоматизируйте updates
- Мониторьте consistency регулярно
- Рассмотрите альтернативы: indexing, caching, read replicas
- Hybrid approaches (Twitter: fan-out для обычных, compute для celebrities)

---

**Предыдущий урок**: [Шардирование и стратегии партиционирования](16-shardirovanie-strategii.md)
**Следующий урок**: [Blob Storage и CDN](18-blob-storage-i-cdn.md)
