# Учебник по System Design

Добро пожаловать в практический курс по системному дизайну. Материалы собраны в виде коротких, но ёмких уроков, которые постепенно проведут вас от фундаментальных понятий до проектирования сложных распределённых систем и эксплуатации их в продакшене.

## Как пользоваться
- Изучайте блоки последовательно: каждый блок опирается на ключевые идеи предыдущего.
- В конце каждого урока ищите разделы «Что почитать дальше» и «Проверьте себя», чтобы закрепить материал.
- Возвращайтесь к файлам как к справочнику перед собеседованиями или при проектировании новых систем.

## Структура курса

### Блок 1: Фундамент
1. [Введение в System Design](01-vvedenie-v-system-design.md)
2. [Back-of-the-envelope calculations](02-back-of-the-envelope-calculations.md)
3. [Latency numbers](03-latency-numbers.md)
4. [Функциональные и нефункциональные требования](04-funkcionalnye-i-nefunkcionalnye-trebovaniya.md)
5. [CAP теорема, ACID vs BASE](05-cap-teorema-acid-vs-base.md)
6. [Consistent Hashing](06-consistent-hashing.md)
7. [DNS и основы работы интернета](07-dns-i-osnovy-raboty-interneta.md)

### Блок 2: Networking & Communication
8. [API Design: REST, GraphQL, gRPC](08-api-design-rest-graphql-grpc.md)
9. [WebSockets, Server-Sent Events, Long Polling](09-websockets-sse-long-polling.md)
10. [Load Balancing: алгоритмы, L4 vs L7, sticky sessions](10-load-balancing-algoritmy-l4-vs-l7.md)
11. [API Gateway и Reverse Proxy](11-api-gateway-i-reverse-proxy.md)
12. [Rate Limiting и алгоритмы подсчёта](12-rate-limiting-algoritmy.md)

### Блок 3: Storage & Data
13. [SQL vs NoSQL](13-sql-vs-nosql.md)
14. [Индексы в базах данных](14-database-indexes.md)
15. [Репликация: Master-Slave и Master-Master](15-replikaciya-master-slave-master-master.md)
16. [Шардирование и стратегии партиционирования](16-shardirovanie-strategii.md)
17. [Денормализация и когда её применять](17-denormalizaciya.md)
18. [Blob Storage и CDN](18-blob-storage-i-cdn.md)
19. [Кэширование: стратегии и политики вытеснения](19-keshirovanie-strategii.md)
20. [Distributed Caching: Redis и Memcached](20-distributed-caching.md)

### Блок 4: Асинхронная обработка
21. [Message Queues: RabbitMQ, SQS](21-message-queues.md)
22. [Event Streaming и паттерн Pub/Sub](22-event-streaming.md)
23. [Batch vs Stream Processing](23-batch-vs-stream-processing.md)
24. [MapReduce и основы Big Data обработки](24-mapreduce-osnovy-big-data.md)

### Блок 5: Микросервисы и паттерны
25. [Монолит vs Микросервисы](25-monolit-vs-mikroservisy.md)
26. [Service Discovery и Service Mesh](26-service-discovery-i-service-mesh.md)
27. [Паттерны отказоустойчивости](27-patterny-otkazoustojchivosti.md)
28. [Distributed Transactions: 2PC и Saga](28-distributed-transactions-2pc-saga.md)
29. [Идемпотентность и exactly-once processing](29-idempotentnost-i-exactly-once.md)
30. [Heartbeats, Health Checks, Failure Detection](30-heartbeats-health-checks.md)

### Блок 6: Distributed Systems
31. [Distributed Consensus: Raft и Paxos](31-distributed-consensus.md)
32. [Leader Election](32-leader-election.md)
33. [Distributed Locking](33-distributed-locking.md)
34. [Gossip Protocol](34-gossip-protocol.md)

### Блок 7: Практические системы — Простые
35. [URL Shortener (bit.ly)](35-url-shortener.md)
36. [Pastebin](36-pastebin.md)
37. [Rate Limiter как система](37-rate-limiter-kak-sistema.md)
38. [Распределённое хранилище ключ-значение](38-distributed-key-value-store.md)
39. [Генератор уникальных идентификаторов](39-unique-id-generator.md)

### Блок 8: Практические системы — Средние
40. [Web Crawler](40-web-crawler.md)
41. [Notification System](41-notification-system.md)
42. [Autocomplete/Typeahead система](42-autocomplete-typeahead.md)
43. [Newsfeed система](43-newsfeed-system.md)
44. [Chat система](44-chat-system.md)
45. [Поисковая система](45-poiskovaya-sistema.md)
46. [Nearby Friends / геолокация](46-nearby-friends-geolokaciya.md)

### Блок 9: Практические системы — Сложные
47. [Социальная сеть (Instagram/Twitter)](47-soczialnaya-set.md)
48. [Видео стриминг (YouTube/Netflix)](48-video-striming.md)
49. [Ride-sharing система (Uber/Lyft)](49-ride-sharing-sistema.md)
50. [Ticketing система и всплески нагрузки](50-ticketing-sistema.md)
51. [E-commerce платформа](51-e-commerce-platforma.md)
52. [Recommendation System](52-recommendation-system.md)

### Блок 10: Production & Operations
53. [Мониторинг и Observability](53-monitoring-i-observability.md)
54. [Deployment стратегии](54-deployment-strategii.md)
55. [Containers и Kubernetes](55-containers-i-kubernetes.md)
56. [Disaster Recovery и Backup](56-disaster-recovery-i-backup.md)
57. [Безопасность: Authentication, Authorization, Encryption](57-bezopasnost.md)
