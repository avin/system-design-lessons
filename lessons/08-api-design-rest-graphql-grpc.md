# API Design: REST, GraphQL, gRPC

## Введение

**API (Application Programming Interface)** — контракт между клиентом и сервером, определяющий, как они взаимодействуют. Правильный выбор API стиля критически важен для:
- Performance системы
- Developer experience
- Масштабируемости
- Поддерживаемости

В современных системах используются три основных подхода:
- **REST** — самый распространенный, основан на HTTP
- **GraphQL** — гибкий query language от Facebook
- **gRPC** — высокопроизводительный RPC от Google

## REST (Representational State Transfer)

### Что это?

**REST** — архитектурный стиль для создания web services, использующий HTTP методы для CRUD операций.

**Ключевые принципы REST**:
1. **Stateless** — сервер не хранит состояние клиента
2. **Client-Server** — разделение ответственности
3. **Cacheable** — ответы могут кэшироваться
4. **Uniform Interface** — стандартизированные методы
5. **Layered System** — клиент не знает о промежуточных слоях

### HTTP Методы

| Метод | Операция | Идемпотентность | Safe |
|-------|----------|-----------------|------|
| GET | Получить ресурс | ✅ Да | ✅ Да |
| POST | Создать ресурс | ❌ Нет | ❌ Нет |
| PUT | Обновить/создать ресурс полностью | ✅ Да | ❌ Нет |
| PATCH | Частично обновить ресурс | ❌ Нет | ❌ Нет |
| DELETE | Удалить ресурс | ✅ Да | ❌ Нет |

**Идемпотентность**: повторные идентичные запросы дают одинаковый результат.
**Safe**: операция не меняет состояние сервера.

### REST API Design

#### Ресурсы и URI

**Хороший дизайн**:
```
GET    /users              # Список пользователей
GET    /users/123          # Конкретный пользователь
POST   /users              # Создать пользователя
PUT    /users/123          # Обновить пользователя полностью
PATCH  /users/123          # Частично обновить
DELETE /users/123          # Удалить пользователя

GET    /users/123/posts    # Посты пользователя
GET    /posts/456/comments # Комментарии к посту
```

**Плохой дизайн**:
```
❌ GET  /getUser?id=123
❌ POST /createUser
❌ POST /users/delete
❌ GET  /user_posts_list
```

#### Принципы именования

1. **Используйте существительные, не глаголы**
   - ✅ `/users`
   - ❌ `/getUsers`

2. **Множественное число для коллекций**
   - ✅ `/users`
   - ❌ `/user`

3. **Lowercase с дефисами**
   - ✅ `/user-profiles`
   - ❌ `/userProfiles`, `/user_profiles`

4. **Иерархия ресурсов**
   - ✅ `/users/123/posts/456/comments`
   - ❌ `/user123post456comments`

#### HTTP Status Codes

**2xx Success**:
- `200 OK` — успешный запрос (GET, PUT, PATCH)
- `201 Created` — ресурс создан (POST)
- `204 No Content` — успешно, но нет тела ответа (DELETE)

**3xx Redirection**:
- `301 Moved Permanently` — ресурс перемещен навсегда
- `304 Not Modified` — кэш актуален

**4xx Client Errors**:
- `400 Bad Request` — невалидные данные
- `401 Unauthorized` — не аутентифицирован
- `403 Forbidden` — аутентифицирован, но нет прав
- `404 Not Found` — ресурс не найден
- `409 Conflict` — конфликт (например, email уже существует)
- `422 Unprocessable Entity` — валидация не прошла
- `429 Too Many Requests` — rate limit exceeded

**5xx Server Errors**:
- `500 Internal Server Error` — ошибка сервера
- `502 Bad Gateway` — ошибка upstream сервера
- `503 Service Unavailable` — сервис временно недоступен
- `504 Gateway Timeout` — timeout от upstream

#### Пример REST API

**Создание пользователя**:
```http
POST /api/v1/users
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30
}

Response: 201 Created
Location: /api/v1/users/123
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30,
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Получение списка с пагинацией**:
```http
GET /api/v1/users?page=2&limit=20&sort=-created_at

Response: 200 OK
{
  "data": [
    {"id": 123, "name": "John Doe", ...},
    {"id": 124, "name": "Jane Smith", ...}
  ],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 1000,
    "total_pages": 50
  },
  "links": {
    "next": "/api/v1/users?page=3&limit=20",
    "prev": "/api/v1/users?page=1&limit=20"
  }
}
```

**Фильтрация и поиск**:
```http
GET /api/v1/users?age_min=18&age_max=65&country=US&search=john
```

### Versioning

**URL Versioning** (рекомендуется):
```
/api/v1/users
/api/v2/users
```

**Header Versioning**:
```http
GET /api/users
Accept: application/vnd.myapi.v2+json
```

**Query Parameter**:
```
/api/users?version=2
```

### HATEOAS (Hypermedia as the Engine of Application State)

Включение ссылок для навигации:
```json
{
  "id": 123,
  "name": "John Doe",
  "links": {
    "self": "/users/123",
    "posts": "/users/123/posts",
    "friends": "/users/123/friends"
  }
}
```

### Преимущества REST

✅ Простота и понятность
✅ Широкая поддержка (браузеры, библиотеки)
✅ Кэширование через HTTP
✅ Stateless (легко масштабировать)
✅ Хорошо документируется (OpenAPI/Swagger)

### Недостатки REST

❌ Over-fetching (получаем лишние поля)
❌ Under-fetching (нужны дополнительные запросы)
❌ Множество endpoints для сложных данных
❌ Нет стандарта для real-time updates

## GraphQL

### Что это?

**GraphQL** — query language и runtime для API, разработанный Facebook. Клиент запрашивает точно те данные, которые нужны.

### Основные концепции

#### Schema

Определяет типы данных и операции:
```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
}

type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  post(id: ID!): Post
}

type Mutation {
  createUser(name: String!, email: String!): User!
  updateUser(id: ID!, name: String, email: String): User!
  deleteUser(id: ID!): Boolean!
}
```

#### Queries

**Простой запрос**:
```graphql
query {
  user(id: "123") {
    id
    name
    email
  }
}

Response:
{
  "data": {
    "user": {
      "id": "123",
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
}
```

**Nested запрос**:
```graphql
query {
  user(id: "123") {
    name
    posts {
      title
      comments {
        text
        author {
          name
        }
      }
    }
  }
}
```

**С REST это было бы**:
```
GET /users/123
GET /users/123/posts
GET /posts/1/comments
GET /posts/2/comments
GET /users/456
GET /users/789
...
```

#### Mutations

```graphql
mutation {
  createUser(name: "John Doe", email: "john@example.com") {
    id
    name
    email
    created_at
  }
}
```

#### Variables

```graphql
query GetUser($userId: ID!) {
  user(id: $userId) {
    name
    email
  }
}

Variables:
{
  "userId": "123"
}
```

#### Fragments

Переиспользуемые части запросов:
```graphql
fragment UserInfo on User {
  id
  name
  email
}

query {
  user1: user(id: "123") {
    ...UserInfo
  }
  user2: user(id: "456") {
    ...UserInfo
  }
}
```

#### Subscriptions (Real-time)

```graphql
subscription {
  newPost {
    id
    title
    author {
      name
    }
  }
}
```

### Resolver Functions

Backend код, который получает данные:
```javascript
const resolvers = {
  Query: {
    user: async (parent, { id }, context) => {
      return await context.db.users.findById(id);
    },
    users: async (parent, { limit, offset }, context) => {
      return await context.db.users.find({ limit, offset });
    }
  },

  User: {
    posts: async (user, args, context) => {
      return await context.db.posts.findByUserId(user.id);
    }
  },

  Mutation: {
    createUser: async (parent, { name, email }, context) => {
      return await context.db.users.create({ name, email });
    }
  }
};
```

### N+1 Problem

**Проблема**:
```graphql
query {
  users {
    name
    posts {  # N queries (один на каждого user)
      title
    }
  }
}
```

**Решение: DataLoader**
```javascript
const DataLoader = require('dataloader');

const postLoader = new DataLoader(async (userIds) => {
  const posts = await db.posts.findByUserIds(userIds);
  return userIds.map(id => posts.filter(p => p.userId === id));
});

// В resolver
User: {
  posts: (user) => postLoader.load(user.id)
}
```

### Преимущества GraphQL

✅ Клиент запрашивает только нужные данные (нет over-fetching)
✅ Один endpoint для всего API
✅ Strongly typed schema
✅ Introspection (self-documenting)
✅ Real-time через subscriptions
✅ Эволюция API без breaking changes

### Недостатки GraphQL

❌ Сложность реализации
❌ Сложнее кэшировать (не HTTP caching)
❌ N+1 problem требует DataLoader
❌ Потенциально дорогие запросы (нужен rate limiting по сложности)
❌ Learning curve для команды
❌ Нет встроенной file upload

### Когда использовать GraphQL

✅ Mobile приложения (экономия трафика)
✅ Сложные связанные данные
✅ Много различных клиентов с разными потребностями
✅ Быстрая итерация и изменение требований
✅ Real-time features

❌ Простые CRUD API
❌ Public API (REST проще для внешних разработчиков)
❌ Нужно максимальное caching

## gRPC

### Что это?

**gRPC (gRPC Remote Procedure Call)** — высокопроизводительный RPC framework от Google, использующий Protocol Buffers и HTTP/2.

### Protocol Buffers (protobuf)

**IDL (Interface Definition Language)** для сериализации данных:

```protobuf
syntax = "proto3";

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  repeated Post posts = 4;
}

message Post {
  int32 id = 1;
  string title = 2;
  string content = 3;
}

message GetUserRequest {
  int32 id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
}
```

### Типы RPC

#### 1. Unary RPC (простой запрос-ответ)
```protobuf
rpc GetUser(GetUserRequest) returns (User);
```

```javascript
// Client
client.getUser({ id: 123 }, (error, user) => {
  if (error) {
    console.error(error);
    return;
  }

  console.log(user);
});
```

#### 2. Server Streaming
```protobuf
rpc ListUsers(ListUsersRequest) returns (stream User);
```

```javascript
// Client
const stream = client.listUsers({});
stream.on('data', (user) => {
  console.log(user.name);
});
stream.on('error', (error) => console.error(error));
stream.on('end', () => console.log('Stream completed'));
```

#### 3. Client Streaming
```protobuf
rpc CreateUsers(stream CreateUserRequest) returns (Summary);
```

```javascript
// Client
const call = client.createUsers((error, summary) => {
  if (error) {
    console.error(error);
    return;
  }

  console.log(summary);
});

for (let i = 0; i < 100; i += 1) {
  call.write({ name: `User ${i}` });
}

call.end();
```

#### 4. Bidirectional Streaming
```protobuf
rpc Chat(stream Message) returns (stream Message);
```

### HTTP/2 Features

- **Multiplexing** — несколько запросов по одному соединению
- **Header compression** (HPACK)
- **Binary protocol** (быстрее текстового)
- **Server push**
- **Prioritization**

### Сравнение размера

**JSON (REST)**:
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com"
}
```
Размер: ~60 bytes

**Protobuf (gRPC)**:
```
Бинарные данные
```
Размер: ~20 bytes (3x меньше!)

### Code Generation

Из `.proto` файла генерируется код для разных языков:

```bash
# Node.js
npx grpc_tools_node_protoc --js_out=import_style=commonjs,binary:. --grpc_out=grpc_js:. user.proto

# Go
protoc --go_out=. --go-grpc_out=. user.proto

# Java
protoc --java_out=. --grpc-java_out=. user.proto
```

### Пример Server (Node.js)

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDefinition = protoLoader.loadSync('user.proto', {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const userProto = grpc.loadPackageDefinition(packageDefinition);

const server = new grpc.Server();

server.addService(userProto.UserService.service, {
  getUser(call, callback) {
    const user = db.getUser(call.request.id);
    callback(null, {
      id: user.id,
      name: user.name,
      email: user.email,
    });
  },
  listUsers(call) {
    db.getAllUsers().forEach((user) => {
      call.write({
        id: user.id,
        name: user.name,
        email: user.email,
      });
    });
    call.end();
  },
});

server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => {
  server.start();
});
```

### Пример Client (Node.js)

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDefinition = protoLoader.loadSync('user.proto', {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const userProto = grpc.loadPackageDefinition(packageDefinition);
const client = new userProto.UserService('localhost:50051', grpc.credentials.createInsecure());

// Unary call
client.getUser({ id: 123 }, (error, user) => {
  if (error) {
    console.error(error);
    return;
  }
  console.log(`User: ${user.name}`);
});

// Server streaming
const usersStream = client.listUsers({});
usersStream.on('data', (user) => console.log(`User: ${user.name}`));
usersStream.on('error', (error) => console.error(error));
usersStream.on('end', () => console.log('Stream finished'));
```

### Преимущества gRPC

✅ Высокая производительность (binary protocol)
✅ Маленький размер сообщений (protobuf)
✅ Strongly typed (code generation)
✅ Streaming support (bidirectional)
✅ HTTP/2 multiplexing
✅ Многоязычность (10+ языков)
✅ Отличное для microservices communication

### Недостатки gRPC

❌ Не поддерживается браузерами напрямую (нужен gRPC-Web)
❌ Binary format (сложнее debugging)
❌ Сложнее, чем REST
❌ Меньшая распространенность
❌ Сложнее для public API

### Когда использовать gRPC

✅ Microservices communication
✅ Нужна высокая производительность
✅ Real-time bidirectional streaming
✅ Polyglot environment (много языков)
✅ Mobile clients (экономия батареи и трафика)

❌ Browser-based приложения (используйте REST или GraphQL)
❌ Public API для внешних разработчиков
❌ Простой CRUD

## Сравнительная таблица

| Характеристика | REST | GraphQL | gRPC |
|----------------|------|---------|------|
| Protocol | HTTP/1.1 | HTTP/1.1 | HTTP/2 |
| Format | JSON (обычно) | JSON | Protocol Buffers |
| Schema | OpenAPI (опционально) | Обязательная | Protobuf (обязательна) |
| Latency | Средняя | Средняя | Низкая |
| Bandwidth | Высокий | Средний | Низкий |
| Browser Support | ✅ Да | ✅ Да | ❌ Нет (нужен gRPC-Web) |
| Streaming | ❌ Нет (SSE отдельно) | ✅ Subscriptions | ✅ Bidirectional |
| Caching | ✅ HTTP caching | ⚠️ Сложно | ❌ Нет |
| Learning Curve | Низкая | Средняя | Высокая |
| Tooling | Отличное | Хорошее | Хорошее |
| Mobile-Friendly | Средне | ✅ Да | ✅ Да |
| Versioning | URL/Header | Нет нужды | Protobuf evolving |

## Гибридные подходы

### REST + GraphQL

```
REST для public API
GraphQL для mobile/web клиентов
```

### REST + gRPC

```
REST для external API
gRPC для internal microservices
```

### BFF (Backend For Frontend)

```
Mobile App → GraphQL BFF → gRPC microservices
Web App → REST BFF → gRPC microservices
```

## Практические рекомендации

### Выбор по типу приложения

**E-commerce**:
- Public API → REST
- Internal services → gRPC
- Mobile app → GraphQL или REST

**Social Network**:
- Web/Mobile → GraphQL (сложные nested данные)
- Real-time feed → GraphQL subscriptions или gRPC streaming
- Internal → gRPC

**Microservices**:
- Service-to-service → gRPC (performance)
- API Gateway → REST или GraphQL для клиентов

### REST Best Practices

1. **Используйте правильные HTTP методы**
2. **Возвращайте правильные status codes**
3. **Версионируйте API**
4. **Документируйте с OpenAPI/Swagger**
5. **Реализуйте pagination, filtering, sorting**
6. **Используйте HATEOAS для discoverable API**
7. **Implement rate limiting**
8. **Support compression (gzip)**

### GraphQL Best Practices

1. **Используйте DataLoader для N+1 problem**
2. **Ограничивайте сложность запросов (query complexity)**
3. **Реализуйте pagination (cursor-based)**
4. **Используйте persisted queries для production**
5. **Enable query whitelisting**
6. **Monitor slow queries**
7. **Implement proper error handling**

### gRPC Best Practices

1. **Используйте deadlines/timeouts**
2. **Implement retry logic с exponential backoff**
3. **Use interceptors для logging, auth, metrics**
4. **Версионируйте protobuf правильно (добавляйте поля, не удаляйте)**
5. **Enable health checking**
6. **Use connection pooling**
7. **Implement circuit breakers**

## Что почитать дальше

### REST
- "RESTful Web APIs" by Leonard Richardson
- [HTTP Status Codes](https://httpstatuses.com/)
- [OpenAPI Specification](https://swagger.io/specification/)

### GraphQL
- [GraphQL.org](https://graphql.org/)
- "Production Ready GraphQL" by Marc-André Giroux
- [Apollo GraphQL Docs](https://www.apollographql.com/docs/)

### gRPC
- [gRPC.io](https://grpc.io/)
- [Protocol Buffers Guide](https://developers.google.com/protocol-buffers)
- "gRPC: Up and Running" by Kasun Indrasiri

## Проверьте себя

1. В чем разница между PUT и PATCH в REST?
2. Что такое идемпотентность и какие HTTP методы идемпотентны?
3. Как GraphQL решает проблему over-fetching?
4. Что такое N+1 problem в GraphQL и как его решить?
5. Почему gRPC быстрее REST?
6. Какие типы streaming поддерживает gRPC?
7. Когда стоит выбрать REST вместо GraphQL?
8. Что такое Protocol Buffers и в чем их преимущество перед JSON?

## Ключевые выводы

- REST — простой и универсальный, лучший выбор для public API
- GraphQL — гибкий, решает over/under-fetching, отлично для сложных данных
- gRPC — высокопроизводительный, идеален для microservices communication
- Выбор зависит от use case: browser support, performance requirements, complexity
- Можно комбинировать: REST для external, gRPC для internal, GraphQL для mobile
- REST хорошо кэшируется через HTTP
- GraphQL требует решения N+1 problem через DataLoader
- gRPC использует HTTP/2 и binary protocol для скорости
- Документируйте API (OpenAPI для REST, introspection для GraphQL, protobuf для gRPC)

---
**Предыдущий урок**: [DNS и основы работы интернета](07-dns-and-internet-basics.md)
**Следующий урок**: [WebSockets, Server-Sent Events, Long Polling](09-websockets-sse-long-polling.md)
