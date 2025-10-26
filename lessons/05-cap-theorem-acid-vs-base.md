# CAP теорема, ACID vs BASE

## CAP теорема

### Что это?

**CAP теорема** (также известная как теорема Брюера) — фундаментальная теорема в distributed systems, сформулированная Eric Brewer в 2000 году. Она утверждает, что распределенная система может гарантировать только два из трех свойств одновременно:

- **C (Consistency)** — Согласованность
- **A (Availability)** — Доступность
- **P (Partition Tolerance)** — Устойчивость к разделению

### Определения

#### Consistency (Согласованность)

Все узлы видят одинаковые данные в одно и то же время. После выполнения операции записи все последующие чтения вернут это значение (или более новое).

**Пример согласованности**:
```
1. Пользователь обновляет свой профиль на сервере A
2. Другой пользователь читает профиль с сервера B
3. Он видит обновленные данные (consistency)
```

**Пример НЕсогласованности**:
```
1. Пользователь обновляет свой профиль на сервере A
2. Другой пользователь читает профиль с сервера B
3. Он видит старые данные (inconsistency)
```

#### Availability (Доступность)

Каждый запрос получает ответ (успешный или ошибку), даже если некоторые узлы недоступны.

**Ключевое**: система всегда отвечает, но ответ может содержать устаревшие данные.

#### Partition Tolerance (Устойчивость к разделению)

Система продолжает работать, даже если сетевое соединение между узлами нарушено (network partition).

**Network partition** — ситуация, когда часть узлов не может связаться с другой частью из-за сетевых проблем.

### Почему можно выбрать только 2 из 3?

**Важно**: В распределенных системах партиции (P) — неизбежность. Сети ненадежны. Поэтому на практике выбор всегда между:
- **CP** (Consistency + Partition Tolerance) — жертвуем Availability
- **AP** (Availability + Partition Tolerance) — жертвуем Consistency

**CA (Consistency + Availability)** без Partition Tolerance — это не распределенная система, а single-node система.

### Визуализация CAP

```
Нормальная работа (нет партиции):
┌──────────┐         ┌──────────┐
│  Node A  │◄───────►│  Node B  │
└──────────┘         └──────────┘
Можем иметь C + A + P

Network Partition:
┌──────────┐    ✗    ┌──────────┐
│  Node A  │         │  Node B  │
└──────────┘         └──────────┘

Теперь выбор:
CP: Отказаться отвечать (sacrifice Availability)
AP: Отвечать старыми данными (sacrifice Consistency)
```

### CP системы (Consistency + Partition Tolerance)

При network partition система предпочитает consistency и возвращает ошибку, если не может гарантировать свежие данные.

**Характеристики**:
- Strong consistency гарантирована
- Могут быть unavailable во время партиций
- Все узлы видят одинаковые данные
- Операции могут блокироваться или отклоняться

**Примеры технологий**:
- **MongoDB** (в строгом режиме с write concern "majority")
- **HBase**
- **Redis** (в single-master mode)
- **ZooKeeper**
- **etcd**
- **Consul**

**Use cases**:
- Финансовые транзакции (банковские переводы)
- Inventory management (остатки товаров)
- Бронирования (билеты, отели)
- Distributed locking
- Configuration management

**Пример**:

Банковский перевод $100 от Alice к Bob:
```
1. Начальное состояние: Alice $500, Bob $300
2. Network partition между узлами
3. CP система откажется выполнить транзакцию (HTTP 503)
4. Пользователь видит ошибку, но данные остаются consistent
```

### AP системы (Availability + Partition Tolerance)

При network partition система предпочитает availability и возвращает данные (возможно устаревшие), чтобы оставаться доступной.

**Характеристики**:
- Всегда доступны для чтения/записи
- Eventual consistency (данные синхронизируются позже)
- Могут возвращать stale data
- Операции никогда не блокируются

**Примеры технологий**:
- **Cassandra**
- **DynamoDB**
- **Riak**
- **CouchDB**
- **Voldemort**
- **SimpleDB**

**Use cases**:
- Социальные сети (лайки, комментарии)
- E-commerce (просмотр каталога)
- Мониторинг и логи
- Сессии пользователей
- Shopping cart

**Пример**:

Лайк поста в Instagram:
```
1. Пост имеет 100 лайков
2. Alice ставит лайк, пишет на узел A (теперь 101)
3. Bob читает с узла B, видит 100 (stale data)
4. Через несколько секунд узлы синхронизируются
5. Теперь все видят 101
```

### Таблица сравнения

| Аспект | CP Systems | AP Systems |
|--------|------------|------------|
| Consistency | Strong | Eventual |
| Availability | Может быть недоступна | Всегда доступна |
| Latency | Выше (ждем подтверждения) | Ниже (быстрый ответ) |
| Use case | Critical data (money) | Non-critical data (social) |
| Conflict resolution | Избегаем конфликтов | Разрешаем конфликты |
| Error handling | Возвращаем ошибку | Возвращаем stale data |

### Практический выбор

**Вопросы для принятия решения**:

1. **Что хуже для бизнеса?**
   - Показать неактуальные данные (выбирай CP)
   - Быть недоступным (выбирай AP)

2. **Критичность данных**:
   - Деньги, остатки товаров → CP
   - Лайки, комментарии → AP

3. **Требования пользователей**:
   - Банковское приложение → CP (пользователи подождут)
   - Социальная сеть → AP (пользователи не подождут)

## ACID vs BASE

Две философии проектирования систем управления данными.

### ACID

**ACID** — набор свойств транзакций в традиционных реляционных БД.

#### A — Atomicity (Атомарность)

Транзакция выполняется полностью или не выполняется вообще. "Все или ничего".

**Пример**:
```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- Alice
  UPDATE accounts SET balance = balance + 100 WHERE id = 2; -- Bob
COMMIT;
```

Если произойдет ошибка между двумя UPDATE:
- ❌ Деньги НЕ исчезнут с счета Alice и не появятся на счете Bob частично
- ✅ Вся транзакция откатится (ROLLBACK)

#### C — Consistency (Согласованность)

Транзакция переводит БД из одного валидного состояния в другое, соблюдая все правила (constraints, triggers, cascades).

**Пример**:
```sql
-- Constraint: balance >= 0

UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
-- Если balance = 500, транзакция откатится (нарушение constraint)
```

#### I — Isolation (Изоляция)

Параллельные транзакции не влияют друг на друга. Результат как будто они выполнялись последовательно.

**Уровни изоляции** (от слабой к строгой):
1. **Read Uncommitted** — видны uncommitted изменения (dirty reads)
2. **Read Committed** — видны только committed изменения
3. **Repeatable Read** — повторное чтение возвращает те же данные
4. **Serializable** — полная изоляция (как последовательное выполнение)

**Пример проблемы без изоляции** (dirty read):
```
T1: UPDATE accounts SET balance = 500 WHERE id = 1;
T2: SELECT balance FROM accounts WHERE id = 1;  -- видит 500
T1: ROLLBACK;  -- откатываем T1
T2 прочитала данные, которых "никогда не было"!
```

#### D — Durability (Долговечность)

После COMMIT данные сохранены навсегда, даже при сбое системы.

**Реализация**:
- Write-Ahead Logging (WAL)
- Репликация
- Backup'ы

**Пример**:
```
1. COMMIT транзакции
2. Сервер получает подтверждение
3. Сервер падает через 1 секунду
4. После перезагрузки данные на месте (durability)
```

### BASE

**BASE** — альтернативная модель для distributed NoSQL систем.

- **B**asically **A**vailable — система в основном доступна
- **S**oft state — состояние может меняться даже без input (из-за eventual consistency)
- **E**ventually consistent — система станет consistent в конечном счете

#### Basically Available

Система отвечает на запросы, даже если некоторые узлы недоступны. Могут быть частичные failures.

**Пример**:
- 3 из 5 узлов доступны → система работает
- Один datacenter недоступен → остальные продолжают работу

#### Soft State

Состояние системы может меняться со временем даже без новых входных данных, из-за eventual consistency.

**Пример**:
```
1. Пользователь пишет пост на узел A
2. Состояние узла A: пост существует
3. Состояние узла B: пост пока не существует
4. Через несколько секунд узел B получает репликацию
5. Теперь оба узла имеют пост
```

Состояние B изменилось не из-за действий пользователя, а из-за внутренней репликации.

#### Eventually Consistent

Все узлы рано или поздно увидят одинаковые данные, но не мгновенно.

**Временное окно inconsistency**: от миллисекунд до секунд.

**Пример**:
```
T=0s:  Alice ставит лайк, запись на узел A
T=0s:  Bob читает с узла B, видит 100 лайков (old)
T=0.5s: Репликация A → B
T=1s:  Bob читает с узла B, видит 101 лайк (new)
```

### ACID vs BASE: Сравнение

| Характеристика | ACID | BASE |
|----------------|------|------|
| Consistency | Strong (немедленная) | Eventual (отложенная) |
| Availability | Может жертвовать | Высокая |
| Complexity | Проще для разработчика | Сложнее (нужно обрабатывать stale data) |
| Performance | Ниже (coordination overhead) | Выше (меньше coordination) |
| Scalability | Сложнее | Проще |
| Use cases | Транзакции, критичные данные | High-scale, social apps |
| Примеры | PostgreSQL, MySQL | Cassandra, DynamoDB |

### Когда использовать ACID

**Подходит для**:
- Финансовые транзакции
- E-commerce orders
- Inventory management
- Бронирования
- Медицинские записи
- Любые данные, где consistency критична

**Характеристики приложения**:
- Небольшой/средний scale (до нескольких TB)
- Сложные query с JOIN'ами
- Транзакции с несколькими таблицами
- Нужны ACID гарантии

**Примеры систем**:
- Банковское приложение
- Платежная система
- ERP система
- Билетная касса

### Когда использовать BASE

**Подходит для**:
- Социальные сети (likes, comments, follows)
- Мониторинг и аналитика
- Кэширование
- Логи и events
- Shopping cart
- User sessions

**Характеристики приложения**:
- Огромный scale (PB данных)
- Простые query (key-value lookups)
- Высокий write throughput
- Eventual consistency приемлема

**Примеры систем**:
- Twitter timeline
- Instagram likes
- IoT telemetry
- Real-time analytics

### Гибридные подходы

Современные приложения часто используют оба подхода:

#### Polyglot Persistence

Использование разных БД для разных частей системы.

**Пример e-commerce**:
```
Orders & Payments → PostgreSQL (ACID)
  ↓
Product Catalog → Elasticsearch (BASE)
  ↓
User Sessions → Redis (BASE)
  ↓
Recommendations → Cassandra (BASE)
```

#### Саги (Sagas)

Реализация распределенных транзакций через последовательность локальных транзакций.

**Пример**:
```
Order Service: создать заказ (local ACID)
  ↓
Payment Service: списать деньги (local ACID)
  ↓
Inventory Service: зарезервировать товар (local ACID)
  ↓
Shipping Service: создать shipment (local ACID)
```

Если один шаг fails → компенсирующие транзакции откатывают предыдущие.

## Eventual Consistency на практике

### Типы Eventual Consistency

#### Read-your-writes Consistency

Пользователь всегда видит свои собственные изменения.

**Реализация**:
```
1. Пользователь пишет данные на узел A
2. Запоминаем session ID → узел A
3. Последующие чтения этого пользователя идут на узел A
4. Пользователь видит свои изменения немедленно
```

#### Session Consistency

В рамках одной сессии consistency гарантирована.

#### Monotonic Reads

Если пользователь прочитал значение X, он не увидит более старое значение в будущем.

**Проблема без monotonic reads**:
```
T1: Читаем с узла A, видим version 5
T2: Читаем с узла B, видим version 3 (!)
```

**Решение**: sticky sessions — пользователь всегда читает с одного узла.

### Conflict Resolution

В AP системах могут возникать конфликты.

#### Last Write Wins (LWW)

Побеждает последняя запись (по timestamp).

**Проблема**: синхронизация времени между узлами сложна.

**Пример**:
```
Узел A: Alice меняет статус на "busy" в 10:00:00.001
Узел B: Alice меняет статус на "away" в 10:00:00.002
Результат: "away" (last write wins)
```

#### Vector Clocks

Отслеживание причинно-следственных связей между версиями.

**Используется в**: Riak, Dynamo

#### CRDTs (Conflict-free Replicated Data Types)

Структуры данных, которые автоматически разрешают конфликты.

**Примеры**:
- G-Counter (Grow-only Counter) — можно только увеличивать
- PN-Counter (Positive-Negative Counter) — счетчик с инкрементом/декрементом
- LWW-Element-Set — множество с last-write-wins

## Практические примеры

### Пример 1: Банковский перевод (ACID, CP)

**Требования**:
- Деньги не должны пропадать или дублироваться
- Баланс всегда корректен
- Consistency критична

**Решение**: PostgreSQL с ACID транзакциями
```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

  -- Проверяем баланс
  SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
  -- balance = 500

  IF balance >= 100 THEN
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
    COMMIT;
  ELSE
    ROLLBACK;
  END IF;
```

**Гарантии**:
- Atomicity: оба UPDATE или ни одного
- Consistency: balance constraint соблюден
- Isolation: другие транзакции не видят промежуточное состояние
- Durability: после COMMIT данные сохранены

### Пример 2: Twitter Likes (BASE, AP)

**Требования**:
- Очень высокая нагрузка (миллионы лайков/сек)
- Eventual consistency приемлема
- Availability критична

**Решение**: Cassandra с eventual consistency
```
1. Пользователь ставит лайк
2. Запись идет на ближайший узел Cassandra
3. Узел отвечает "OK" немедленно (availability)
4. Асинхронная репликация на другие узлы
5. Через 100-500ms все узлы имеют обновление
```

**Trade-offs**:
- ✅ Availability: всегда можно поставить лайк
- ✅ Latency: мгновенный ответ
- ❌ Consistency: count может быть неточным пару секунд
- ✅ Scalability: легко добавлять узлы

### Пример 3: E-commerce Order (Hybrid)

**Разные части используют разные модели**:

```
Shopping Cart → Redis (BASE, AP)
  - Можно потерять cart (приемлемо)
  - Высокая доступность важна

Order Creation → PostgreSQL (ACID, CP)
  - Нельзя потерять order
  - Consistency критична

Inventory Check → PostgreSQL (ACID, CP)
  - Нельзя oversell
  - Strong consistency нужна

Recommendation Engine → Cassandra (BASE, AP)
  - Eventual consistency OK
  - Высокая нагрузка
```

## Что почитать дальше

### Статьи
- "CAP Twelve Years Later: How the Rules Have Changed" by Eric Brewer
- "Eventually Consistent" by Werner Vogels (CTO Amazon)
- "Consistency Models" by Jepsen (Kyle Kingsbury)

### Книги
- "Designing Data-Intensive Applications" by Martin Kleppmann — Chapter 5, 7, 9
- "Database Internals" by Alex Petrov

## Проверьте себя

1. Что означает каждая буква в CAP?
2. Почему нельзя иметь CA без P в распределенной системе?
3. Приведите пример use case для CP системы и для AP системы.
4. Что означает каждая буква в ACID?
5. В чем основная разница между ACID и BASE?
6. Что такое eventual consistency и как долго может длиться inconsistency?
7. Когда стоит выбрать PostgreSQL, а когда Cassandra?
8. Что такое dirty read и на каком isolation level это возможно?

## Ключевые выводы

- CAP теорема: можно выбрать только 2 из 3 (Consistency, Availability, Partition Tolerance)
- В реальности партиции неизбежны → выбор между CP и AP
- CP системы жертвуют доступностью ради consistency (банки, inventory)
- AP системы жертвуют consistency ради доступности (соцсети, логи)
- ACID — сильные гарантии для транзакций (PostgreSQL, MySQL)
- BASE — eventual consistency для высокого scale (Cassandra, DynamoDB)
- Современные приложения часто используют polyglot persistence
- Нет универсального решения — выбор зависит от requirements
- Eventual consistency требует обработки stale data в приложении
- Измеряйте latency inconsistency — она должна быть приемлемой для бизнеса

---

**Предыдущий урок**: [Функциональные и нефункциональные требования](04-funkcionalnye-i-nefunkcionalnye-trebovaniya.md)
**Следующий урок**: [Consistent Hashing](06-consistent-hashing.md)
