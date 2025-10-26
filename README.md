# Учебник по System Design

Добро пожаловать в практический курс по системному дизайну. Материалы собраны в виде коротких, но ёмких уроков, которые постепенно проведут вас от фундаментальных понятий до проектирования сложных распределённых систем и эксплуатации их в продакшене.

## Как пользоваться
- Изучайте блоки последовательно: каждый блок опирается на ключевые идеи предыдущего.
- В конце каждого урока ищите разделы «Что почитать дальше» и «Проверьте себя», чтобы закрепить материал.
- Возвращайтесь к файлам как к справочнику перед собеседованиями или при проектировании новых систем.

## Структура курса

### Блок 1: Фундамент
1. [Введение в System Design](lessons/01-introduction-to-system-design.md)
2. [Back-of-the-envelope calculations](lessons/02-back-of-the-envelope-calculations.md)
3. [Latency numbers](lessons/03-latency-numbers.md)
4. [Функциональные и нефункциональные требования](lessons/04-functional-vs-nonfunctional-requirements.md)
5. [CAP теорема, ACID vs BASE](lessons/05-cap-theorem-acid-vs-base.md)
6. [Consistent Hashing](lessons/06-consistent-hashing.md)
7. [DNS и основы работы интернета](lessons/07-dns-and-internet-basics.md)

### Блок 2: Networking & Communication
8. [API Design: REST, GraphQL, gRPC](lessons/08-api-design-rest-graphql-grpc.md)
9. [WebSockets, SSE и long polling](lessons/09-websockets-sse-long-polling.md)
10. [Load Balancing: алгоритмы, L4 vs L7](lessons/10-load-balancing-algorithms-l4-vs-l7.md)
11. [API Gateway и reverse proxy](lessons/11-api-gateway-and-reverse-proxy.md)
12. [Rate Limiting: алгоритмы и стратегии](lessons/12-rate-limiting-algorithms.md)

### Блок 3: Storage & Data
13. [SQL vs NoSQL](lessons/13-sql-vs-nosql.md)
14. [Database Indexes](lessons/14-database-indexes.md)
15. [Репликация: master-slave и master-master](lessons/15-master-slave-and-multi-master-replication.md)
16. [Шардирование: стратегии](lessons/16-sharding-strategies.md)
17. [Денормализация](lessons/17-denormalization.md)
18. [Blob Storage и CDN](lessons/18-blob-storage-and-cdn.md)
19. [Кеширование: стратегии](lessons/19-caching-strategies.md)
20. [Distributed Caching](lessons/20-distributed-caching.md)

### Блок 4: Асинхронная обработка
21. [Message Queues](lessons/21-message-queues.md)
22. [Kafka и event streaming](lessons/22-kafka-event-streaming.md)
23. [Async workers и фоновые задачи](lessons/23-async-workers.md)
24. [WebSockets в продакшене](lessons/24-websockets-production.md)

### Блок 5: Микросервисы и паттерны
25. [Microservices: когда и зачем](lessons/25-microservices-when-why.md)
26. [Service Mesh](lessons/26-service-mesh.md)
27. [Event-driven architecture](lessons/27-event-driven-architecture.md)
28. [API Gateway patterns](lessons/28-api-gateway-patterns.md)
29. [Backend for Frontend](lessons/29-backend-for-frontend.md)
30. [Service Discovery](lessons/30-service-discovery.md)

### Блок 6: Distributed Systems
31. [Consensus algorithms](lessons/31-consensus-algorithms.md)
32. [Distributed transactions](lessons/32-distributed-transactions.md)
33. [Data replication](lessons/33-data-replication.md)
34. [Eventual consistency](lessons/34-eventual-consistency.md)

### Блок 7: Практические системы — Простые
35. [Design URL Shortener](lessons/35-design-url-shortener.md)
36. [Pastebin](lessons/36-pastebin.md)
37. [Rate Limiter System](lessons/37-rate-limiter.md)
38. [Distributed Key-Value Store](lessons/38-distributed-kv-store.md)
39. [Unique ID Generator](lessons/39-unique-id-generator.md)

### Блок 8: Практические системы — Средние
40. [Web Crawler](lessons/40-web-crawler.md)
41. [Notification System](lessons/41-notification-system.md)
42. [Autocomplete System](lessons/42-autocomplete.md)
43. [Newsfeed System](lessons/43-newsfeed.md)
44. [Chat System](lessons/44-chat.md)
45. [Search Engine](lessons/45-search.md)
46. [Nearby Friends](lessons/46-nearby-friends.md)

### Блок 9: Практические системы — Сложные
47. [Social Network](lessons/47-social-network.md)
48. [Ticketing System](lessons/48-ticketing-system.md)
49. [Video Streaming](lessons/49-video-streaming.md)
50. [Ride-sharing System](lessons/50-ride-sharing.md)
51. [E-commerce Platform](lessons/51-ecommerce.md)
52. [Recommendation System](lessons/52-recommendation-system.md)

### Блок 10: Production & Operations
53. [Monitoring и Observability](lessons/53-monitoring.md)
54. [Deployment Strategies](lessons/54-deployment.md)
55. [Kubernetes в Production](lessons/55-kubernetes.md)
56. [Disaster Recovery](lessons/56-disaster-recovery.md)
57. [Security Best Practices](lessons/57-security.md)
