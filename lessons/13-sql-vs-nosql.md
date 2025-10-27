# SQL vs NoSQL

## Введение

Выбор между SQL и NoSQL — одно из фундаментальных архитектурных решений. Нет универсального ответа: каждый подход имеет свои преимущества и недостатки.

**SQL (Relational)**: PostgreSQL, MySQL, Oracle, SQL Server
**NoSQL**: MongoDB, Cassandra, Redis, DynamoDB, Neo4j

## SQL (Relational Databases)

### Что это?

**SQL (Structured Query Language)** базы данных используют реляционную модель с таблицами, строками и столбцами. Данные структурированы и связаны через отношения (relationships).

### Основные концепции

#### 1. Schema (Схема)

Строгая структура данных, определенная заранее:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    title VARCHAR(200) NOT NULL,
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Характеристики**:
- Фиксированные колонки
- Типы данных строго определены
- Изменение схемы требует migration

#### 2. ACID Transactions

**Atomicity**: Все или ничего
**Consistency**: Данные всегда валидны
**Isolation**: Транзакции изолированы
**Durability**: Committed данные сохранены навсегда

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- Если любой UPDATE fail → весь transaction откатывается
```

#### 3. Normalization (Нормализация)

Организация данных для минимизации дублирования:

**Денормализованные данные**:
```
orders
id | customer_name | customer_email | product_name | price
1  | John Doe      | john@email.com | Laptop       | 1000
2  | John Doe      | john@email.com | Mouse        | 20
```
❌ Дублирование customer данных

**Нормализованные данные**:
```
customers
id | name     | email
1  | John Doe | john@email.com

orders
id | customer_id | product_id
1  | 1           | 1
2  | 1           | 2

products
id | name   | price
1  | Laptop | 1000
2  | Mouse  | 20
```
✅ Нет дублирования

#### 4. Relationships (Связи)

**One-to-Many**:
```sql
users (1) ← (M) posts
```

**Many-to-Many**:
```sql
students (M) ← (junction table) → (M) courses
```

**One-to-One**:
```sql
users (1) ← (1) user_profiles
```

#### 5. JOIN Operations

```sql
-- Посты с авторами
SELECT posts.*, users.username
FROM posts
JOIN users ON posts.user_id = users.id;

-- Комментарии с авторами и постами
SELECT comments.*, users.username, posts.title
FROM comments
JOIN users ON comments.user_id = users.id
JOIN posts ON comments.post_id = posts.id;
```

### Преимущества SQL

✅ **ACID гарантии**: Надежность для критичных данных
✅ **Мощный query язык**: Сложные JOIN, aggregations, subqueries
✅ **Data integrity**: Foreign keys, constraints, triggers
✅ **Стандартизация**: SQL — универсальный язык
✅ **Зрелая экосистема**: Decades of development, tools, expertise
✅ **Vertical scaling**: Можно использовать мощные серверы

### Недостатки SQL

❌ **Rigid schema**: Сложно менять структуру
❌ **Сложное horizontal scaling**: Sharding сложен
❌ **Performance на huge datasets**: JOIN на миллиардах строк медленный
❌ **Impedance mismatch**: ORM сложности с object-relational mapping
❌ **Fixed schema не подходит для быстрых изменений**

### Когда использовать SQL

✅ Данные структурированы и связаны (relationships важны)
✅ Нужны ACID транзакции (финансы, e-commerce)
✅ Сложные queries с JOINs
✅ Data integrity критична
✅ Схема стабильна
✅ Vertical scaling достаточно (до нескольких TB)

**Use cases**:
- Banking systems
- E-commerce (orders, inventory)
- ERP, CRM systems
- Traditional web applications
- Analytics и reporting

## NoSQL Databases

### Что это?

**NoSQL (Not Only SQL)** — широкий класс БД, которые не используют реляционную модель. Оптимизированы для specific use cases.

### Типы NoSQL баз

#### 1. Document Stores (MongoDB, CouchDB)

**Модель**: Хранят документы (обычно JSON/BSON)

```javascript
// MongoDB
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "username": "john_doe",
  "email": "john@example.com",
  "profile": {
    "age": 30,
    "city": "New York"
  },
  "posts": [
    {
      "title": "My first post",
      "content": "Hello world",
      "tags": ["intro", "hello"]
    }
  ],
  "created_at": ISODate("2024-01-15T10:30:00Z")
}
```

**Характеристики**:
- Flexible schema (каждый документ может иметь разные поля)
- Nested data (embedding)
- Rich queries

**Операции**:
```javascript
// Insert
db.users.insertOne({
  username: "john_doe",
  email: "john@example.com"
});

// Find
db.users.find({ age: { $gte: 18 } });

// Update
db.users.updateOne(
  { username: "john_doe" },
  { $set: { age: 31 } }
);

// Aggregation
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$user_id", total: { $sum: "$amount" } } }
]);
```

**Когда использовать**:
✅ Flexible/evolving schema
✅ Nested/hierarchical data
✅ Rapid development
✅ Content management systems

**Use cases**:
- Content management (blogs, catalogs)
- User profiles
- Mobile apps
- Real-time analytics

#### 2. Key-Value Stores (Redis, DynamoDB, Riak)

**Модель**: Простейшая — ключ → значение

```javascript
// Redis
const Redis = require('ioredis');
const redis = new Redis();

await redis.set('user:123:name', 'John Doe');
await redis.get('user:123:name'); // "John Doe"

await redis.hset('user:123', {
  name: 'John Doe',
  email: 'john@example.com',
});
await redis.hgetall('user:123');
```

**Характеристики**:
- Очень быстрые (in-memory)
- Простые операции (GET, SET, DELETE)
- Нет сложных queries
- Горизонтально масштабируемы

**Когда использовать**:
✅ Caching
✅ Session storage
✅ Rate limiting
✅ Real-time analytics (counters, leaderboards)

**Use cases**:
- Cache layer
- Session store
- Shopping cart
- Pub/Sub messaging
- Leaderboards

#### 3. Column-Family Stores (Cassandra, HBase, ScyllaDB)

**Модель**: Данные хранятся по колонкам, оптимизировано для write-heavy workloads

```
RowKey | Column1:Value | Column2:Value | Column3:Value
user1  | name:John     | email:j@e.com | age:30
user2  | name:Jane     | email:jane@   | city:NYC
```

**Cassandra CQL**:
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username TEXT,
    email TEXT,
    created_at TIMESTAMP
);

INSERT INTO users (user_id, username, email, created_at)
VALUES (uuid(), 'john_doe', 'john@example.com', toTimestamp(now()));

SELECT * FROM users WHERE user_id = 123e4567-e89b-12d3-a456-426614174000;
```

**Характеристики**:
- Massive scale (petabytes)
- High write throughput
- Tunable consistency
- No JOINs

**Когда использовать**:
✅ Time-series data
✅ Massive write throughput
✅ Распределенная система на несколько DC
✅ Eventual consistency допустима

**Use cases**:
- Time-series data (IoT, metrics)
- Event logging
- Social media feeds
- Recommendation engines

#### 4. Graph Databases (Neo4j, ArangoDB, Amazon Neptune)

**Модель**: Nodes (сущности) и Edges (связи)

```
(User:John)-[:FOLLOWS]->(User:Jane)
(User:John)-[:LIKES]->(Post:123)
(Post:123)-[:TAGGED_WITH]->(Tag:tech)
```

**Neo4j Cypher**:
```cypher
// Создание
CREATE (john:User {name: 'John Doe', email: 'john@example.com'})
CREATE (jane:User {name: 'Jane Smith'})
CREATE (john)-[:FOLLOWS]->(jane)

// Запрос: друзья друзей
MATCH (user:User {name: 'John Doe'})-[:FOLLOWS]->()-[:FOLLOWS]->(fof)
RETURN fof.name

// Кратчайший путь
MATCH path = shortestPath(
  (john:User {name: 'John'})-[:FOLLOWS*]-(jane:User {name: 'Jane'})
)
RETURN path
```

**Характеристики**:
- Optimized для traversals (обход графа)
- Relationship queries очень быстрые
- Flexible schema

**Когда использовать**:
✅ Highly connected data
✅ Relationship queries критичны
✅ Social networks
✅ Recommendation engines

**Use cases**:
- Social networks (friends, followers)
- Fraud detection
- Knowledge graphs
- Recommendation systems
- Network/IT operations

### Преимущества NoSQL

✅ **Horizontal scalability**: Легко добавлять узлы
✅ **Flexible schema**: Быстрые итерации
✅ **High performance на specific workloads**
✅ **Handle massive data volumes** (petabytes)
✅ **Optimized для modern apps** (mobile, real-time)

### Недостатки NoSQL

❌ **Eventual consistency** (часто)
❌ **Нет стандартизированного query language**
❌ **Ограниченные transactional guarantees**
❌ **Сложнее моделировать связанные данные**
❌ **Less mature ecosystem** (чем SQL)

## Сравнение SQL vs NoSQL

| Характеристика | SQL | NoSQL |
|----------------|-----|-------|
| **Schema** | Fixed, predefined | Flexible, dynamic |
| **Scalability** | Vertical (primarily) | Horizontal |
| **Transactions** | ACID | Eventual consistency (обычно) |
| **Queries** | Powerful (JOIN, subqueries) | Limited (depends on type) |
| **Data model** | Relational (tables) | Document/KV/Column/Graph |
| **Consistency** | Strong | Tunable (often eventual) |
| **Матurity** | Decades | Newer |
| **Use case** | Complex queries, transactions | Massive scale, flexibility |
| **Examples** | PostgreSQL, MySQL | MongoDB, Cassandra, Redis |

## Детальное сравнение по критериям

### 1. Scalability

**SQL (Vertical Scaling)**:
```
Single powerful server
- 64 CPUs
- 256 GB RAM
- Fast SSD

Limit: ~10TB, hardware ceiling
```

**SQL Sharding** (сложно):
```
Shard 1: Users 0-1M
Shard 2: Users 1M-2M
Shard 3: Users 2M-3M

Проблемы:
- Распределенные JOIN сложны
- Cross-shard transactions сложны
- Rebalancing при росте
```

**NoSQL (Horizontal Scaling)**:
```
Add more nodes:
Node 1, Node 2, Node 3 → Node 10, Node 20...

Cassandra/DynamoDB автоматически распределяют данные
```

### 2. Schema Flexibility

**SQL Migration**:
```sql
-- Добавление колонки
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Problem: все существующие строки получают NULL
-- На больших таблицах — долго и блокирует таблицу
```

**NoSQL** (MongoDB):
```javascript
// Просто добавляем поле в новые документы
db.users.insertOne({
  username: "john",
  phone: "+1234567890"  // новое поле
});

// Старые документы без phone продолжают работать
```

### 3. Consistency

**SQL** (Strong Consistency):
```sql
-- Читаем баланс
SELECT balance FROM accounts WHERE id = 1;  -- 100

-- Другая транзакция обновляет
UPDATE accounts SET balance = 200 WHERE id = 1;

-- Следующий read гарантированно видит 200
SELECT balance FROM accounts WHERE id = 1;  -- 200
```

**NoSQL** (Eventual Consistency):
```
Write → Node 1 (balance = 200)
Replicate → Node 2 (still 100) ← Read может вернуть старое значение
Replicate → Node 3 (still 100)

Через несколько мс все nodes будут 200
```

**Cassandra** (Tunable Consistency):
```sql
-- Write с QUORUM (большинство узлов)
INSERT INTO users ... USING CONSISTENCY QUORUM;

-- Read с QUORUM
SELECT * FROM users WHERE id = 123 USING CONSISTENCY QUORUM;

-- Trade-off: latency vs consistency
```

### 4. Transactions

**SQL**:
```sql
BEGIN;
  INSERT INTO orders (user_id, total) VALUES (123, 100);
  UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 456;
  UPDATE accounts SET balance = balance - 100 WHERE user_id = 123;
COMMIT;
```

**NoSQL** (обычно нет multi-document transactions):
```javascript
// MongoDB (до версии 4.0 не было transactions)
// Нужно моделировать по-другому:

// Option 1: Embedding (одна операция)
db.orders.insertOne({
  user_id: 123,
  total: 100,
  items: [{ product_id: 456, quantity: 1 }],
  payment: { method: 'card', status: 'completed' }
});

// Option 2: Two-Phase Commit (manual)
// Option 3: Compensating transactions (Saga pattern)
```

**MongoDB 4.0+** (multi-document transactions):
```javascript
const session = client.startSession();
session.startTransaction();

try {
  await orders.insertOne({ user_id: 123, total: 100 }, { session });
  await accounts.updateOne({ user_id: 123 }, { $inc: { balance: -100 } }, { session });
  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
}
```

### 5. Query Capabilities

**SQL**:
```sql
-- Complex JOIN
SELECT u.username, COUNT(p.id) as post_count, AVG(c.rating) as avg_rating
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
LEFT JOIN comments c ON p.id = c.post_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id
HAVING COUNT(p.id) > 5
ORDER BY avg_rating DESC
LIMIT 10;
```

**NoSQL** (MongoDB):
```javascript
// Aggregation pipeline (более verbose)
db.users.aggregate([
  { $match: { created_at: { $gt: new Date('2024-01-01') } } },
  { $lookup: {
      from: 'posts',
      localField: '_id',
      foreignField: 'user_id',
      as: 'posts'
  }},
  { $unwind: '$posts' },
  { $lookup: {
      from: 'comments',
      localField: 'posts._id',
      foreignField: 'post_id',
      as: 'comments'
  }},
  { $group: {
      _id: '$_id',
      username: { $first: '$username' },
      post_count: { $sum: 1 },
      avg_rating: { $avg: '$comments.rating' }
  }},
  { $match: { post_count: { $gt: 5 } } },
  { $sort: { avg_rating: -1 } },
  { $limit: 10 }
]);
```

## Polyglot Persistence

**Современный подход**: Используйте разные БД для разных задач!

```
[Application]
    ↓
PostgreSQL     → Orders, Payments (ACID критично)
    ↓
MongoDB        → Product catalog (flexible schema)
    ↓
Redis          → Caching, sessions
    ↓
Elasticsearch  → Search
    ↓
Neo4j          → Recommendations (graph)
    ↓
Cassandra      → Time-series metrics
```

**Пример: E-commerce**

```
Users, Orders → PostgreSQL (transactional)
Product Catalog → MongoDB (flexible attributes)
Session Store → Redis
Full-text Search → Elasticsearch
Recommendations → Neo4j
Analytics Events → Cassandra
```

## Migration: SQL → NoSQL (или наоборот)

### SQL → NoSQL Migration Challenges

**1. Нет JOINs → Денормализация**

SQL:
```sql
SELECT posts.*, users.username
FROM posts JOIN users ON posts.user_id = users.id;
```

MongoDB (embedding):
```javascript
{
  _id: ObjectId("..."),
  title: "My Post",
  content: "...",
  author: {  // Embedded!
    id: 123,
    username: "john_doe"
  }
}
```

**Trade-off**: Дублирование данных vs performance

**2. Транзакции → Application-level logic**

SQL transaction → Saga pattern или 2PC в NoSQL

**3. Schema changes → Versioning**

```javascript
// v1 documents
{ username: "john", email: "..." }

// v2 documents (после миграции)
{ username: "john", email: "...", phone: "..." }

// Handle both versions в application
function getUser(doc) {
  return {
    username: doc.username,
    email: doc.email,
    phone: doc.phone || null  // Handle old docs
  };
}
```

## Практические рекомендации

### Выбор БД: Decision Tree

```
Нужны ACID транзакции?
├─ Да → SQL
└─ Нет
    ├─ Сложные JOINs и relationships?
    │   ├─ Да → SQL или Graph DB
    │   └─ Нет
    │       ├─ Massive scale (>10TB)?
    │       │   ├─ Да → NoSQL (Cassandra/DynamoDB)
    │       │   └─ Нет → SQL или Document DB
    │       └─ Schema часто меняется?
    │           ├─ Да → Document DB (MongoDB)
    │           └─ Нет → SQL
```

### SQL: Когда точно использовать

✅ Banking, финансы
✅ E-commerce (orders, payments)
✅ ERP, CRM
✅ Inventory management
✅ Любые данные с complex relationships
✅ Compliance требует ACID

### NoSQL: Когда точно использовать

**MongoDB**: CMS, catalogs, user profiles
**Redis**: Caching, sessions, real-time
**Cassandra**: Time-series, IoT, massive writes
**Neo4j**: Social graphs, recommendations
**DynamoDB**: Serverless apps, auto-scaling needs

### Hybrid: Когда использовать оба

Большинство modern applications!

```
SQL для:
- Transactional data (orders, payments)
- Complex queries

NoSQL для:
- Caching (Redis)
- Search (Elasticsearch)
- Analytics (Cassandra)
- Real-time features (MongoDB)
```

## Популярные базы данных

### SQL

**PostgreSQL**:
- ✅ Open-source, мощный
- ✅ JSONB support (гибрид SQL+NoSQL)
- ✅ Full-text search
- ✅ Extensions (PostGIS для geo)
- Use case: Most applications

**MySQL**:
- ✅ Popular, простой
- ✅ Отлично для read-heavy
- ⚠️ Меньше features чем PostgreSQL
- Use case: Web apps, WordPress

### NoSQL

**MongoDB**:
- ✅ Popular, легко начать
- ✅ Rich queries
- ✅ Transactions (с 4.0)
- Use case: Flexible schemas, rapid development

**Redis**:
- ✅ In-memory, очень быстрый
- ✅ Rich data structures
- Use case: Caching, real-time

**Cassandra**:
- ✅ Massive scale
- ✅ High write throughput
- ⚠️ Сложнее моделировать
- Use case: Time-series, IoT

## Что почитать дальше

### Книги
- "Designing Data-Intensive Applications" by Martin Kleppmann — библия
- "Seven Databases in Seven Weeks" — обзор разных БД

### Статьи
- [MongoDB vs PostgreSQL](https://www.mongodb.com/compare/mongodb-postgresql)
- [CAP Theorem explained](https://www.ibm.com/cloud/learn/cap-theorem)

## Проверьте себя

1. В чем главная разница между SQL и NoSQL?
2. Что такое ACID и почему это важно?
3. Какие 4 типа NoSQL баз вы знаете?
4. Когда стоит выбрать MongoDB вместо PostgreSQL?
5. Что такое eventual consistency?
6. Почему horizontal scaling проще в NoSQL?
7. Что такое polyglot persistence?
8. Как обрабатывать транзакции в NoSQL БД без multi-document transactions?

## Ключевые выводы

- SQL — для structured data, complex relationships, ACID transactions
- NoSQL — для massive scale, flexible schemas, specific use cases
- SQL масштабируется вертикально, NoSQL — горизонтально
- Eventual consistency — trade-off за scalability в NoSQL
- JOINs мощные в SQL, но сложны при шардинге
- Document DBs (MongoDB) — хороший компромисс: flexibility + rich queries
- Key-Value (Redis) — для caching и simple access patterns
- Column-family (Cassandra) — для massive writes и time-series
- Graph DBs (Neo4j) — для highly connected data
- Polyglot persistence — современный подход: используйте лучшую БД для каждой задачи
- Нет универсального решения — выбор зависит от requirements

---
**Предыдущий урок**: [Rate Limiting и алгоритмы подсчёта](12-rate-limiting-algorithms.md)
**Следующий урок**: [Индексы в базах данных](14-database-indexes.md)
