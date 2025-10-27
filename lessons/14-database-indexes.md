# Индексы в базах данных

## Что такое индекс?

**Database Index** — структура данных, которая улучшает скорость поиска в таблице за счет дополнительной памяти и медленных операций записи.

**Аналогия**: Индекс в книге позволяет быстро найти страницу с нужной темой вместо просмотра всех страниц подряд.

**Без индекса** (Full Table Scan):
```
Найти пользователя с email = "john@example.com"
→ Сканируем все 10,000,000 строк
→ Время: 10 секунд
```

**С индексом** на email:
```
Найти пользователя с email = "john@example.com"
→ Index lookup
→ Время: 10 миллисекунд (1000x быстрее!)
```

## Как работают индексы

### Full Table Scan vs Index Scan

**Full Table Scan**:
```sql
SELECT * FROM users WHERE email = 'john@example.com';

-- Database читает каждую строку:
Row 1: alice@example.com ❌
Row 2: bob@example.com ❌
Row 3: charlie@example.com ❌
...
Row 5,431,289: john@example.com ✅

Cost: O(N) — линейное сканирование
```

**Index Scan**:
```sql
CREATE INDEX idx_users_email ON users(email);

SELECT * FROM users WHERE email = 'john@example.com';

-- Database:
1. Lookup в индексе (B-tree): O(log N)
2. Переход напрямую к строке

Cost: O(log N) — логарифмический поиск
```

### Пример производительности

```
Table: 10,000,000 строк

Без индекса:
- Scan 10M rows
- Time: ~10 seconds

С индексом (B-tree):
- Tree depth: log₂(10,000,000) ≈ 24 levels
- Disk seeks: ~24
- Time: ~10 milliseconds

Speedup: 1000x!
```

## Типы индексов

### 1. B-Tree Index (Default)

**Самый распространенный тип**. Сбалансированное дерево.

```
              [50]
           /        \
       [25]          [75]
      /    \        /    \
   [10]  [35]   [60]   [90]
```

**Характеристики**:
- Balanced tree (все leaf nodes на одном уровне)
- Sorted order (позволяет range queries)
- Операции: O(log N)

**Когда использовать**:
```sql
-- Equality
WHERE email = 'john@example.com'

-- Range queries
WHERE age BETWEEN 18 AND 65
WHERE created_at > '2024-01-01'

-- Sorting
ORDER BY created_at DESC

-- LIKE с prefix
WHERE name LIKE 'John%'
```

**Создание**:
```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_posts_created_at ON posts(created_at);
```

**Не работает** для:
```sql
-- LIKE без prefix
WHERE name LIKE '%John%'  -- Full scan!

-- Функции на колонке
WHERE LOWER(email) = 'john@example.com'  -- Full scan!
```

### 2. Hash Index

**Key-value хэш-таблица**.

```
hash("john@example.com") = 12345 → Row pointer
hash("jane@example.com") = 67890 → Row pointer
```

**Характеристики**:
- O(1) для equality lookups
- Не поддерживает range queries
- Не поддерживает sorting

**Когда использовать**:
```sql
-- Только equality
WHERE email = 'john@example.com'
```

**Создание** (PostgreSQL):
```sql
CREATE INDEX idx_users_email_hash ON users USING HASH (email);
```

**Не работает** для:
```sql
WHERE email > 'john@example.com'  -- Range не поддерживается
ORDER BY email  -- Sorting не поддерживается
```

**Use case**: Очень редко. B-tree почти всегда лучше.

### 3. Composite (Multi-Column) Index

Индекс на несколько колонок.

```sql
CREATE INDEX idx_users_last_first ON users(last_name, first_name);
```

**Как работает** (Left-to-Right):
```
Sorted by (last_name, first_name):
('Doe', 'Alice')
('Doe', 'Bob')
('Doe', 'John')
('Smith', 'Alice')
('Smith', 'John')
```

**Работает для**:
```sql
-- Использует индекс
WHERE last_name = 'Doe'
WHERE last_name = 'Doe' AND first_name = 'John'

-- НЕ использует индекс (first_name не первая колонка)
WHERE first_name = 'John'
```

**Правило**: Индекс используется **слева направо**. Если пропускаете первую колонку — индекс не работает.

**Пример**:
```sql
INDEX (a, b, c)

✅ WHERE a = 1
✅ WHERE a = 1 AND b = 2
✅ WHERE a = 1 AND b = 2 AND c = 3
✅ WHERE a = 1 AND c = 3  (partial use: только 'a')

❌ WHERE b = 2
❌ WHERE c = 3
❌ WHERE b = 2 AND c = 3
```

### 4. Unique Index

Гарантирует уникальность значений.

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Теперь:
INSERT INTO users (email) VALUES ('john@example.com');  -- OK
INSERT INTO users (email) VALUES ('john@example.com');  -- ERROR: duplicate
```

**Автоматически создается** для:
```sql
-- PRIMARY KEY
CREATE TABLE users (
  id SERIAL PRIMARY KEY  -- Автоматически создает unique index
);

-- UNIQUE constraint
CREATE TABLE users (
  email VARCHAR(100) UNIQUE  -- Автоматически создает unique index
);
```

### 5. Partial (Filtered) Index

Индекс только для части строк.

```sql
-- Индекс только для активных пользователей
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
```

**Преимущества**:
- Меньше размер индекса
- Быстрее updates (не нужно обновлять индекс для неактивных)

**Use case**:
```sql
-- Query на активных пользователей (использует индекс)
SELECT * FROM users WHERE email = 'john@example.com' AND status = 'active';

-- Query на неактивных (НЕ использует этот индекс)
SELECT * FROM users WHERE email = 'john@example.com' AND status = 'inactive';
```

### 6. Full-Text Index

Для текстового поиска.

**PostgreSQL**:
```sql
CREATE INDEX idx_posts_content_fulltext ON posts USING GIN(to_tsvector('english', content));

SELECT * FROM posts
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'search & term');
```

**MySQL**:
```sql
CREATE FULLTEXT INDEX idx_posts_content ON posts(content);

SELECT * FROM posts WHERE MATCH(content) AGAINST('search term' IN NATURAL LANGUAGE MODE);
```

**Возможности**:
- Stemming (search finds "searching", "searched")
- Stop words (игнорирует "the", "a", "is")
- Ranking по relevance

### 7. Geospatial Index

Для географических данных.

**PostgreSQL (PostGIS)**:
```sql
CREATE INDEX idx_locations_geom ON locations USING GIST(geom);

-- Найти точки в радиусе 1км
SELECT * FROM locations
WHERE ST_DWithin(geom, ST_MakePoint(lon, lat)::geography, 1000);
```

**Use case**: Location-based services, maps

### 8. Covering Index (Index-Only Scan)

Индекс содержит все нужные колонки — query не нужно обращаться к таблице.

```sql
CREATE INDEX idx_users_email_name ON users(email, name);

-- Index-only scan (все данные в индексе)
SELECT email, name FROM users WHERE email = 'john@example.com';

-- Требует table access (age не в индексе)
SELECT email, name, age FROM users WHERE email = 'john@example.com';
```

**PostgreSQL**:
```sql
CREATE INDEX idx_users_email_name ON users(email) INCLUDE (name);
```

## Индексы и Performance

### EXPLAIN / EXPLAIN ANALYZE

Как узнать, используется ли индекс?

**PostgreSQL**:
```sql
EXPLAIN SELECT * FROM users WHERE email = 'john@example.com';

-- Без индекса:
Seq Scan on users  (cost=0.00..180.00 rows=1 width=100)
  Filter: (email = 'john@example.com'::text)

-- С индексом:
Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=100)
  Index Cond: (email = 'john@example.com'::text)
```

**MySQL**:
```sql
EXPLAIN SELECT * FROM users WHERE email = 'john@example.com';

+----+-------------+-------+------+---------------+---------+---------+-------+------+-------+
| id | select_type | table | type | possible_keys | key     | key_len | ref   | rows | Extra |
+----+-------------+-------+------+---------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | users | ref  | idx_email     | idx_email | 303   | const |    1 |       |
+----+-------------+-------+------+---------------+---------+---------+-------+------+-------+
```

**Ключевые метрики**:
- **Seq Scan** (Sequential Scan) — плохо, full table scan
- **Index Scan** — хорошо, использует индекс
- **Index Only Scan** — отлично, все данные из индекса
- **rows** — сколько строк сканируется (меньше = лучше)
- **cost** — приблизительная стоимость (меньше = лучше)

### Когда индекс НЕ используется

#### 1. Функции на колонке

```sql
-- НЕ использует индекс
WHERE LOWER(email) = 'john@example.com'
WHERE YEAR(created_at) = 2024

-- Решение: Function-based index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
CREATE INDEX idx_posts_year ON posts((YEAR(created_at)));
```

#### 2. OR между разными колонками

```sql
-- Может не использовать индекс
WHERE email = 'john@example.com' OR phone = '1234567890'

-- Решение: UNION
SELECT * FROM users WHERE email = 'john@example.com'
UNION
SELECT * FROM users WHERE phone = '1234567890';
```

#### 3. Неравенство (!=, <>)

```sql
-- Часто не использует индекс
WHERE status != 'inactive'

-- Лучше:
WHERE status IN ('active', 'pending', 'suspended')
```

#### 4. LIKE с wildcard в начале

```sql
-- НЕ использует индекс
WHERE name LIKE '%John%'

-- Использует индекс
WHERE name LIKE 'John%'

-- Решение для %...%: Full-text index
```

#### 5. Малая селективность

Если query возвращает большой процент строк, индекс может не использоваться.

```sql
-- 90% пользователей активны
WHERE status = 'active'  -- Может использовать full scan вместо индекса
```

**Причина**: Дешевле прочитать таблицу последовательно, чем делать миллионы index lookups.

## Trade-offs индексов

### Преимущества

✅ **Быстрее reads**: O(log N) вместо O(N)
✅ **Быстрее sorting**: Данные уже отсортированы
✅ **Enforce constraints**: UNIQUE, PRIMARY KEY

### Недостатки

❌ **Медленнее writes**: Нужно обновлять индекс
❌ **Больше storage**: Индекс занимает место
❌ **Overhead на maintenance**: Vacuum, reindex

### Пример: Insert Performance

```
Table без индексов:
INSERT: 1ms

Table с 5 индексами:
INSERT: 1ms (table) + 5×0.5ms (indexes) = 3.5ms

3.5x slower writes!
```

**Trade-off**: Быстрые reads vs медленные writes.

### Сколько индексов создавать?

**Правило**: Создавайте индексы только для **часто используемых queries**.

**Слишком мало**:
- Slow reads
- Full table scans

**Слишком много**:
- Slow writes
- Wasted storage
- Confusing для query optimizer

**Рекомендация**:
- Индексы на foreign keys
- Индексы на WHERE/JOIN колонки
- Не создавайте "на всякий случай"

## Стратегии индексирования

### 1. Индексируйте Foreign Keys

```sql
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),  -- FK
  ...
);

-- Обязательно создайте индекс на user_id!
CREATE INDEX idx_posts_user_id ON posts(user_id);
```

**Почему**: JOIN по FK — очень частая операция.

### 2. Composite Index для частых пар

```sql
-- Часто делаете:
WHERE user_id = 123 AND status = 'published'

-- Создайте composite index:
CREATE INDEX idx_posts_user_status ON posts(user_id, status);
```

**Порядок колонок важен**:
- Более селективная (уникальная) колонка первой
- Колонка из WHERE в equality первой, затем range

```sql
-- Good
CREATE INDEX idx ON posts(user_id, created_at);  -- user_id equality, created_at range
WHERE user_id = 123 AND created_at > '2024-01-01'

-- Bad
CREATE INDEX idx ON posts(created_at, user_id);  -- created_at range первым
```

### 3. Covering Index для Hot Queries

```sql
-- Hot query:
SELECT user_id, title, created_at FROM posts WHERE user_id = 123;

-- Covering index (все колонки в индексе):
CREATE INDEX idx_posts_covering ON posts(user_id) INCLUDE (title, created_at);
```

### 4. Partial Index для подмножества данных

```sql
-- Только 10% orders завершены
CREATE INDEX idx_completed_orders ON orders(created_at) WHERE status = 'completed';
```

### 5. Избегайте дублирования

```sql
-- Дублирование:
CREATE INDEX idx1 ON users(email);
CREATE INDEX idx2 ON users(email, name);  -- idx2 покрывает idx1!

-- idx1 избыточен, удалите его
DROP INDEX idx1;
```

**Исключение**: Unique index + non-unique covering index могут сосуществовать.

## Monitoring индексов

### Неиспользуемые индексы (PostgreSQL)

```sql
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan,  -- Сколько раз использовался
  idx_tup_read,
  idx_tup_fetch,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Никогда не использовался
  AND indexrelname NOT LIKE '%_pkey'  -- Исключить primary keys
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Действие**: Удалите неиспользуемые индексы!

### Missing Indexes (PostgreSQL)

```sql
-- Queries с Seq Scan на больших таблицах
SELECT
  schemaname,
  tablename,
  seq_scan,
  seq_tup_read,
  idx_scan,
  seq_tup_read / seq_scan AS avg_seq_tup,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_stat_user_tables
WHERE seq_scan > 0
  AND pg_total_relation_size(schemaname||'.'||tablename) > 10000000  -- >10MB
ORDER BY seq_tup_read DESC
LIMIT 20;
```

**Действие**: Добавьте индексы на часто сканируемые таблицы.

### Index Bloat

Индексы могут "раздуваться" со временем из-за updates/deletes.

**PostgreSQL**:
```sql
-- Reindex для уменьшения bloat
REINDEX INDEX idx_users_email;
REINDEX TABLE users;  -- All indexes
```

**Maintenance**:
```sql
-- Vacuum для очистки мертвых tuples
VACUUM ANALYZE users;

-- Autovacuum (обычно включен по умолчанию)
```

## Практические примеры

### Пример 1: E-commerce Orders

```sql
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER,
  status VARCHAR(20),
  total DECIMAL(10,2),
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

-- Индексы:

-- 1. FK index (для JOIN с users)
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 2. Status queries
CREATE INDEX idx_orders_status ON orders(status);

-- 3. Composite для user's orders
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);

-- 4. Partial index для pending orders (hot queries)
CREATE INDEX idx_pending_orders ON orders(created_at) WHERE status = 'pending';

-- 5. Covering index для dashboard
CREATE INDEX idx_orders_dashboard ON orders(user_id, created_at DESC)
  INCLUDE (status, total);
```

### Пример 2: Social Media Posts

```sql
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER,
  content TEXT,
  likes_count INTEGER DEFAULT 0,
  created_at TIMESTAMP,
  is_deleted BOOLEAN DEFAULT FALSE
);

-- Индексы:

-- 1. FK
CREATE INDEX idx_posts_user_id ON posts(user_id);

-- 2. Newsfeed query (только не удаленные)
CREATE INDEX idx_posts_feed ON posts(created_at DESC) WHERE is_deleted = FALSE;

-- 3. Popular posts
CREATE INDEX idx_posts_popular ON posts(likes_count DESC, created_at DESC)
  WHERE is_deleted = FALSE;

-- 4. Full-text search
CREATE INDEX idx_posts_content_fts ON posts USING GIN(to_tsvector('english', content));
```

### Пример 3: Time-Series Events

```sql
CREATE TABLE events (
  id BIGSERIAL PRIMARY KEY,
  user_id INTEGER,
  event_type VARCHAR(50),
  properties JSONB,
  timestamp TIMESTAMP
);

-- Индексы:

-- 1. Time-range queries (BRIN для больших таблиц)
CREATE INDEX idx_events_timestamp ON events USING BRIN(timestamp);

-- 2. User events
CREATE INDEX idx_events_user_timestamp ON events(user_id, timestamp DESC);

-- 3. Event type filtering
CREATE INDEX idx_events_type_timestamp ON events(event_type, timestamp DESC);

-- 4. JSONB properties (GIN)
CREATE INDEX idx_events_properties ON events USING GIN(properties);
```

## Advanced Topics

### 1. Index-Only Scans

**PostgreSQL**:
```sql
CREATE INDEX idx ON users(email) INCLUDE (name, created_at);

-- Index-only scan (все данные из индекса)
EXPLAIN SELECT email, name, created_at FROM users WHERE email = 'john@example.com';
```

**Visibility Map**: PostgreSQL использует visibility map для определения, нужна ли проверка tuple visibility в таблице.

### 2. Clustered vs Non-Clustered Index

**Clustered Index** (SQL Server, InnoDB):
- Таблица физически упорядочена по индексу
- Только один clustered index на таблицу (обычно PRIMARY KEY)
- Leaf nodes содержат сами строки

**Non-Clustered Index**:
- Separate структура
- Leaf nodes содержат pointers к строкам
- Может быть много на одной таблице

**PostgreSQL**: Все индексы non-clustered (но можно использовать CLUSTER command для физической сортировки).

### 3. Expression Indexes

```sql
-- Index на вычисляемое значение
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Теперь работает:
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';
```

### 4. Concurrent Index Creation

**Проблема**: CREATE INDEX блокирует writes.

**Решение** (PostgreSQL):
```sql
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

Не блокирует writes, но:
- Занимает больше времени
- Требует больше ресурсов
- Может fail (нужно проверить и удалить invalid index)

## Что почитать дальше

- "Use The Index, Luke!" by Markus Winand — отличный ресурс
- PostgreSQL Documentation: Indexes
- MySQL Documentation: Optimization and Indexes
- "High Performance MySQL" by Baron Schwartz

## Проверьте себя

1. Какая сложность поиска без индекса и с B-tree индексом?
2. В чем разница между B-tree и Hash индексом?
3. Как работает composite index (a, b, c)? Какие queries его используют?
4. Почему `WHERE LOWER(email) = '...'` не использует обычный индекс на email?
5. Что такое covering index и зачем он нужен?
6. Какие недостатки у индексов?
7. Как найти неиспользуемые индексы?
8. Что такое index bloat и как с ним бороться?

## Ключевые выводы

- Индексы ускоряют reads (O(log N)) за счет медленных writes и storage
- B-tree — default и наиболее универсальный тип индекса
- Composite index работает слева направо: (a,b,c) → можно использовать для a, a+b, a+b+c
- Индексируйте foreign keys, WHERE clauses, JOIN columns
- Covering index избегает table access — fastest possible query
- Функции на колонках не используют обычный индекс → нужен expression index
- Слишком много индексов замедляют writes — баланс важен
- Мониторьте unused indexes и удаляйте их
- EXPLAIN/EXPLAIN ANALYZE — ваш лучший друг для optimization
- Partial indexes экономят storage для filtered queries
- CREATE INDEX CONCURRENTLY для production (PostgreSQL)

---
**Предыдущий урок**: [SQL vs NoSQL](13-sql-vs-nosql.md)
**Следующий урок**: [Репликация: Master-Slave и Master-Master](15-master-slave-and-multi-master-replication.md)
