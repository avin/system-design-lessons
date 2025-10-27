# Функциональные и нефункциональные требования

## Введение

Первый и критически важный шаг в system design — определение требований. Без четкого понимания того, что система должна делать, невозможно создать правильную архитектуру. Требования делятся на две категории:

- **Функциональные требования (Functional Requirements)** — ЧТО система должна делать
- **Нефункциональные требования (Non-Functional Requirements)** — КАК система должна работать

## Функциональные требования

Функциональные требования описывают конкретные возможности и поведение системы. Это features, которые видит и использует пользователь.

### Характеристики хороших функциональных требований

1. **Конкретные**: четко описывают функциональность
2. **Измеримые**: можно проверить, выполнено ли требование
3. **Реализуемые**: технически возможно сделать
4. **Приоритизированные**: разделены на must-have и nice-to-have

### Примеры для разных систем

#### URL Shortener
**Must-have**:
- Пользователь может создать короткий URL из длинного
- Пользователь может перейти по короткому URL и будет перенаправлен на оригинальный
- Система генерирует уникальные короткие URL

**Nice-to-have**:
- Пользователь может задать кастомный короткий URL
- Пользователь может видеть аналитику (количество переходов, география)
- Короткие URL могут истекать через определенное время
- Пользователь может удалить созданные URL

#### Instagram-like система
**Must-have**:
- Пользователь может загрузить фото
- Пользователь может просмотреть feed фотографий
- Пользователь может подписаться на других пользователей
- Пользователь может лайкнуть фото
- Пользователь может комментировать фото

**Nice-to-have**:
- Пользователь может редактировать фото (фильтры)
- Пользователь может создавать Stories
- Пользователь может отмечать других пользователей на фото
- Direct messages
- Reels (короткие видео)

#### E-commerce система
**Must-have**:
- Пользователь может просматривать каталог товаров
- Пользователь может добавлять товары в корзину
- Пользователь может оформить заказ
- Система обрабатывает платежи
- Пользователь получает подтверждение заказа

**Nice-to-have**:
- Рекомендации товаров
- Отзывы и рейтинги
- Wishlist
- История просмотров
- Программа лояльности

### Как определять функциональные требования на интервью

**Задавайте вопросы**:

1. "Какие основные use cases?" (top 3-5)
2. "Кто основные пользователи системы?"
3. "Какой критичный functionality, без которого система не может существовать?"
4. "Что можно исключить из scope интервью?"

**Пример диалога**:
```
Интервьюер: "Спроектируйте Twitter"
Вы: "Давайте уточним scope. Нужны ли нам:
     - Создание твитов? (Да)
     - Timeline (feed)? (Да)
     - Follow/Unfollow? (Да)
     - Likes и Retweets? (Да, но упростим)
     - Direct Messages? (Нет, вне scope)
     - Trends? (Нет, вне scope)
     - Notifications? (Упростим)
     - Search? (Нет, вне scope)
     Правильно?"
```

### Приоритизация: MoSCoW метод

- **Must have** — критичный functionality
- **Should have** — важный, но не блокирующий
- **Could have** — nice-to-have
- **Won't have** — вне scope

## Нефункциональные требования

Нефункциональные требования определяют качественные характеристики системы. Они часто важнее функциональных, т.к. определяют architecture.

### Основные категории

#### 1. Performance (Производительность)

**Throughput (пропускная способность)**:
- Сколько операций система обрабатывает в единицу времени
- Измеряется в QPS (Queries Per Second), TPS (Transactions Per Second)

**Latency (задержка)**:
- Время ответа системы на запрос
- Измеряется в миллисекундах (обычно P50, P95, P99)

**Примеры**:
- "Система должна обрабатывать 10,000 QPS"
- "P99 latency для read операций < 100ms"
- "Загрузка feed должна занимать < 500ms"

#### 2. Scalability (Масштабируемость)

Способность системы справляться с ростом нагрузки.

**Вопросы для уточнения**:
- Ожидаемый рост пользователей? (10x за год? 100x?)
- Пиковая нагрузка? (Black Friday, вирусный контент)
- Географическое распределение?

**Примеры**:
- "Система должна масштабироваться от 1M до 100M пользователей"
- "Поддерживать 100x пиковую нагрузку во время событий"
- "Добавление capacity не должно требовать downtime"

#### 3. Availability (Доступность)

Процент времени, когда система доступна.

**Метрики**:
- 99% (two nines) = 3.65 дней downtime/год
- 99.9% (three nines) = 8.76 часов downtime/год
- 99.99% (four nines) = 52.56 минут downtime/год
- 99.999% (five nines) = 5.26 минут downtime/год

**Примеры**:
- "SLA 99.9% uptime"
- "Система должна быть доступна 24/7"
- "Максимальный downtime 1 час в месяц"

**Trade-off**: Выше availability = выше стоимость (репликация, redundancy).

#### 4. Reliability (Надежность)

Вероятность корректной работы системы в течение времени.

**Включает**:
- Отказоустойчивость (fault tolerance)
- Обработка ошибок (error handling)
- Data integrity

**Примеры**:
- "Никакие данные пользователя не должны теряться"
- "Система должна корректно работать при падении 2 из 5 серверов"
- "Транзакции должны быть ACID-compliant"

#### 5. Consistency (Согласованность)

Гарантии относительно состояния данных.

**Уровни**:
- **Strong Consistency**: все видят одинаковые данные одновременно
- **Eventual Consistency**: данные станут одинаковыми "в конечном счете"
- **Causal Consistency**: причинно-следственная связь сохраняется

**Примеры**:
- "Балансы счетов должны быть strongly consistent"
- "Лайки и комментарии могут быть eventually consistent"
- "После публикации пост должен быть виден автору немедленно"

#### 6. Durability (Долговечность)

Гарантия сохранности данных.

**Примеры**:
- "Данные должны храниться минимум 7 лет (regulatory requirement)"
- "Uploaded фото не должны теряться"
- "Backups каждые 24 часа с retention 30 дней"

#### 7. Maintainability (Поддерживаемость)

Легкость модификации и поддержки системы.

**Включает**:
- Модульность
- Читаемость кода
- Документация
- Observability (логи, метрики, трейсинг)

**Примеры**:
- "Новый инженер должен понять архитектуру за 1 неделю"
- "Deployment новой версии не должен требовать manual steps"
- "Все критичные операции должны иметь метрики и алерты"

#### 8. Security (Безопасность)

Защита от несанкционированного доступа и атак.

**Включает**:
- Authentication (кто вы?)
- Authorization (что вам разрешено?)
- Encryption (данные зашифрованы)
- Audit logging

**Примеры**:
- "Все API должны требовать authentication"
- "Пароли должны храниться в hashed виде (bcrypt, scrypt)"
- "Данные в transit и at rest должны быть encrypted"
- "PCI DSS compliance для платежной информации"

#### 9. Cost (Стоимость)

Бюджетные ограничения.

**Факторы**:
- Infrastructure costs (servers, storage, bandwidth)
- Operational costs (engineers, support)
- Licensing costs

**Примеры**:
- "Бюджет на infrastructure < $10K/месяц"
- "Cost per user < $0.10/месяц"
- "Использовать open-source решения где возможно"

### Типичные trade-offs нефункциональных требований

#### Consistency vs Availability (CAP теорема)
- **Strong consistency**: медленнее, может быть unavailable при network partition
- **Eventual consistency**: быстрее, всегда available, но данные могут быть stale

**Выбор**:
- Банковские транзакции → strong consistency
- Социальные сети → eventual consistency

#### Latency vs Throughput
- **Low latency**: быстрый ответ на каждый запрос
- **High throughput**: больше запросов в секунду

**Выбор**:
- Real-time gaming → latency
- Batch processing → throughput

#### Performance vs Cost
- **High performance**: дорогие servers, много репликации
- **Low cost**: более медленная система

**Выбор**:
- Стартап → cost
- Прибыльная компания → performance

#### Security vs Usability
- **High security**: multi-factor auth, частые re-authentication
- **High usability**: простой вход, запоминание сессии

**Выбор**:
- Банкинг → security
- Social media → usability

## Процесс определения требований на интервью

### 1. Clarification Phase (5-10 минут)

**Шаг 1**: Определите функциональные требования

Пример для "Design Twitter":
```
Interviewer: Design Twitter
You: Let me clarify the functional requirements:
     - Users can post tweets (280 chars)
     - Users can follow others
     - Users see timeline of tweets from people they follow
     - Users can like and retweet

     Out of scope:
     - Direct messages
     - Trends
     - Search

     Is this correct?
```

**Шаг 2**: Определите нефункциональные требования

```
You: Now for non-functional requirements:
     - Scale: how many users?
       (Interviewer: 100M users, 10M DAU)

     - Availability: what SLA?
       (Interviewer: 99.9%)

     - Latency: what's acceptable for timeline load?
       (Interviewer: < 500ms for P99)

     - Consistency: can timeline be slightly stale?
       (Interviewer: yes, eventual consistency is fine)
```

**Шаг 3**: Запишите требования

Во время интервью записывайте на доске/в заметках:

```
Functional Requirements:
✓ Post tweet (280 chars)
✓ Follow/Unfollow users
✓ View timeline
✓ Like/Retweet

Non-Functional Requirements:
✓ 100M users, 10M DAU
✓ 99.9% availability
✓ P99 latency < 500ms
✓ Eventual consistency OK
```

### 2. Используйте requirements для принятия решений

Требования определяют архитектуру:

**Пример**:
- Requirement: "P99 latency < 100ms для timeline"
- Decision: Нужен aggressive caching (Redis)
- Trade-off: Может быть stale data (acceptable для соцсети)

**Пример 2**:
- Requirement: "Never lose user data"
- Decision: Репликация БД, backups, durability guarantees
- Trade-off: Выше стоимость infrastructure

## Примеры для разных систем

### YouTube (Video Streaming)

**Functional**:
- Upload video
- Watch video
- Search videos
- Like/Comment
- Subscribe to channels

**Non-Functional**:
- Scale: 2B users, 500M DAU
- Storage: 500 hours of video uploaded per minute
- Availability: 99.99%
- Latency: Video start < 2 seconds
- Bandwidth: Support HD/4K streaming
- Consistency: View counts can be eventually consistent
- Durability: Videos never lost

**Key decisions based on requirements**:
- CDN для delivery (latency + bandwidth)
- Distributed storage (S3) для durability
- Eventual consistency для view counts (performance)

### Uber (Ride Sharing)

**Functional**:
- Request ride
- Match driver
- Track ride in real-time
- Payment processing
- Rating system

**Non-Functional**:
- Scale: 100M users, 15M DAU
- Availability: 99.99% (critical for safety)
- Latency: Ride matching < 5 seconds
- Consistency: Location updates real-time (strong consistency)
- Reliability: Payment processing must be ACID
- Security: PCI compliance для payments

**Key decisions**:
- WebSockets для real-time location
- Strong consistency для payments (SQL DB)
- Geospatial indexing (QuadTree, Geohash)

### WhatsApp (Messaging)

**Functional**:
- Send text message
- Send media (photos, videos)
- Group chats
- End-to-end encryption
- Message delivery status

**Non-Functional**:
- Scale: 2B users, 1B DAU
- Availability: 99.99%
- Latency: Message delivery < 1 second
- Consistency: Messages must be delivered in order
- Security: End-to-end encryption mandatory
- Reliability: Messages never lost

**Key decisions**:
- Message queue для reliable delivery
- Strong ordering guarantees
- Encryption at application level

## Документирование требований

### Формат для design doc

```markdown
# System Design: [System Name]

## 1. Functional Requirements

### Must Have
- [ ] Requirement 1
- [ ] Requirement 2

### Should Have
- [ ] Requirement 3

### Out of Scope
- Direct messages
- Search

## 2. Non-Functional Requirements

### Performance
- Latency: P99 < 100ms
- Throughput: 10K QPS

### Scale
- Users: 10M DAU
- Storage: 1PB
- Growth: 2x per year

### Availability
- SLA: 99.9%
- Downtime: < 1hr/month

### Consistency
- User data: Strong
- Social interactions: Eventual

### Security
- Authentication: OAuth 2.0
- Encryption: TLS 1.3
- Compliance: GDPR

### Cost
- Budget: $50K/month
- Cost per user: < $0.05
```

## Что почитать дальше

- "Writing Effective Use Cases" by Alistair Cockburn
- "Software Requirements" by Karl Wiegers
- RFC 2119 — ключевые слова для requirements (MUST, SHOULD, MAY)

## Проверьте себя

1. В чем разница между функциональными и нефункциональными требованиями?
2. Назовите 5 категорий нефункциональных требований.
3. Какой SLA соответствует 99.99% availability?
4. Когда можно использовать eventual consistency вместо strong consistency?
5. Как функциональные требования влияют на архитектурные решения?
6. Почему важно определять scope (что вне границ системы) на интервью?

## Ключевые выводы

- Требования — фундамент system design, без них невозможно принимать решения
- Функциональные требования определяют ЧТО, нефункциональные — КАК
- На интервью обязательно уточняйте требования — не стройте систему в вакууме
- Разделяйте must-have и nice-to-have
- Нефункциональные требования часто противоречат друг другу — выбирайте trade-offs
- Документируйте требования и возвращайтесь к ним при принятии решений
- Требования определяют архитектуру: latency → caching, availability → replication, scale → sharding

---
**Предыдущий урок**: [Latency Numbers Every Programmer Should Know](03-latency-numbers.md)
**Следующий урок**: [CAP теорема, ACID vs BASE](05-cap-theorem-acid-vs-base.md)
