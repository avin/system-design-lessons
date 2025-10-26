# Consistent Hashing

## Проблема распределения данных

Представьте, что у вас есть distributed cache с 3 серверами. Как решить, на какой сервер положить каждый ключ?

### Наивный подход: Modulo Hashing

```python
server_index = hash(key) % num_servers

# Пример:
# hash("user_123") = 987654321
# 987654321 % 3 = 0 → Сервер 0
# hash("user_456") = 123456789
# 123456789 % 3 = 2 → Сервер 2
```

**Работает отлично... до тех пор, пока число серверов не меняется.**

### Проблема с Modulo Hashing

**Что происходит при добавлении/удалении сервера?**

```
Было 3 сервера:
hash("user_123") % 3 = 0 → Сервер 0
hash("user_456") % 3 = 2 → Сервер 2

Добавили 4-й сервер (теперь 4 сервера):
hash("user_123") % 4 = 1 → Сервер 1 (!)
hash("user_456") % 4 = 1 → Сервер 1 (!)
```

**Проблема**: Почти все ключи переместились на другие серверы!

**Последствия**:
- Cache miss для большинства ключей
- Необходимо rehash и переместить почти все данные
- Огромная нагрузка на БД
- Временная деградация performance

**Математика**:
- При изменении от N до N+1 серверов, **N/(N+1)** ключей переместятся
- 3→4 серверов: 75% ключей переместятся
- 10→11 серверов: 90% ключей переместятся

## Что такое Consistent Hashing?

**Consistent Hashing** — техника распределения данных, которая минимизирует количество перемещаемых ключей при изменении числа серверов.

**Ключевое свойство**: При добавлении/удалении сервера перемещается только ~1/N ключей (где N — число серверов).

### Как это работает?

#### 1. Hash Ring (Кольцо хэшей)

Представьте hash space как кольцо от 0 до 2^32-1 (или 2^64-1).

```
        0 (= 2^32)
           ↑
      359° ← → 1°
           |
    270° ← + → 90°
           |
         180°
```

#### 2. Размещение серверов на кольце

Хэшируем имя каждого сервера и помещаем его на кольцо.

```
hash("server_A") = 100
hash("server_B") = 200
hash("server_C") = 300

Кольцо:
0 ----100(A)---- 200(B)---- 300(C)---- 2^32 (→0)
```

#### 3. Размещение ключей

Хэшируем ключ и идем по часовой стрелке до первого сервера.

```
hash("user_123") = 150 → идем по часовой → попадаем на Server B (200)
hash("user_456") = 250 → идем по часовой → попадаем на Server C (300)
hash("user_789") = 350 → идем по часовой → попадаем на Server A (100, через 0)
```

### Что происходит при добавлении сервера?

```
Было:
0 ----100(A)---- 200(B)---- 300(C)---- 2^32

Добавили Server D с hash(D) = 250:
0 ----100(A)---- 200(B)--250(D)-- 300(C)---- 2^32
```

**Что переместилось?**
- Только ключи от 200 до 250 переместились с C на D
- Все остальные ключи остались на месте

**Результат**: Переместилось ~25% ключей (вместо 75% с modulo).

### Что происходит при удалении сервера?

```
Было:
0 ----100(A)---- 200(B)---- 300(C)---- 2^32

Удалили Server B:
0 ----100(A)------------ 300(C)---- 2^32
```

**Что переместилось?**
- Только ключи, которые были на B, переместились на C
- Все остальные ключи остались на месте

## Проблема: Неравномерное распределение

Если хэши серверов попадают близко друг к другу:

```
0 -- 100(A) 110(B) 120(C) ------------------------ 2^32

Server A: ключи от 120 до 100 (огромный диапазон ~2^32)
Server B: ключи от 100 до 110 (маленький диапазон)
Server C: ключи от 110 до 120 (маленький диапазон)
```

**Проблема**: Server A получит ~98% всех ключей, B и C почти ничего.

## Решение: Virtual Nodes (Virtual Replicas)

**Идея**: Каждый физический сервер представлен множеством виртуальных узлов на кольце.

```python
# Вместо одного hash(server_A)
# Создаем 100-500 виртуальных узлов:
hash("server_A#1")
hash("server_A#2")
hash("server_A#3")
...
hash("server_A#100")
```

### Кольцо с виртуальными узлами

```
0 -- A#1 -- B#2 -- C#1 -- A#2 -- B#1 -- C#2 -- A#3 ... -- 2^32

Каждый сервер:
- Server A: A#1, A#2, A#3, ..., A#100
- Server B: B#1, B#2, B#3, ..., B#100
- Server C: C#1, C#2, C#3, ..., C#100
```

**Преимущества**:
- Более равномерное распределение данных
- При удалении сервера его ключи распределяются между всеми остальными
- Больше виртуальных узлов = более равномерное распределение

**Типичное количество виртуальных узлов**: 100-500 на физический сервер.

## Реализация

### Простая имплементация на Python

```python
import hashlib
from bisect import bisect_right

class ConsistentHash:
    def __init__(self, num_virtual_nodes=100):
        self.num_virtual_nodes = num_virtual_nodes
        self.ring = {}  # hash_value -> server_name
        self.sorted_keys = []  # отсортированные hash values

    def _hash(self, key):
        """Hash function"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_server(self, server_name):
        """Добавить сервер с виртуальными узлами"""
        for i in range(self.num_virtual_nodes):
            virtual_key = f"{server_name}#{i}"
            hash_value = self._hash(virtual_key)
            self.ring[hash_value] = server_name
            self.sorted_keys.append(hash_value)

        # Сортируем после добавления
        self.sorted_keys.sort()

    def remove_server(self, server_name):
        """Удалить сервер и его виртуальные узлы"""
        for i in range(self.num_virtual_nodes):
            virtual_key = f"{server_name}#{i}"
            hash_value = self._hash(virtual_key)
            del self.ring[hash_value]
            self.sorted_keys.remove(hash_value)

    def get_server(self, key):
        """Найти сервер для ключа"""
        if not self.ring:
            return None

        hash_value = self._hash(key)

        # Найти первый сервер по часовой стрелке
        index = bisect_right(self.sorted_keys, hash_value)

        # Если вышли за пределы, wrap around к началу
        if index == len(self.sorted_keys):
            index = 0

        return self.ring[self.sorted_keys[index]]

# Использование
ch = ConsistentHash(num_virtual_nodes=150)

# Добавляем серверы
ch.add_server("server_A")
ch.add_server("server_B")
ch.add_server("server_C")

# Определяем, на каком сервере лежат ключи
print(ch.get_server("user_123"))  # server_B
print(ch.get_server("user_456"))  # server_A
print(ch.get_server("user_789"))  # server_C

# Добавляем новый сервер
ch.add_server("server_D")

# Только часть ключей переместится
print(ch.get_server("user_123"))  # может измениться
```

### Оптимизация: Binary Search

Для поиска следующего сервера используем binary search:
- Сложность: O(log N), где N — количество виртуальных узлов
- Быстро даже при тысячах узлов

## Математика Consistent Hashing

### Количество перемещаемых ключей

При добавлении/удалении одного сервера:

```
K = общее количество ключей
N = количество серверов

Перемещается: K/N ключей
```

**Пример**:
- 1,000,000 ключей
- 10 серверов
- Добавили 1 сервер → переместится ~91,000 ключей (9%)
- С modulo hashing → 909,000 ключей (91%)

### Стандартное отклонение распределения

Без виртуальных узлов:
```
σ = sqrt(1/N)
```

С V виртуальными узлами на сервер:
```
σ = sqrt(1/(N×V))
```

**Вывод**: Больше виртуальных узлов → меньше дисбаланс.

## Use Cases

### 1. Distributed Caching (Memcached, Redis Cluster)

**Проблема**: Распределить ключи между cache серверами.

**Решение**: Consistent hashing позволяет добавлять/удалять серверы без массового rehash.

**Пример (Memcached)**:
```python
servers = ["cache1:11211", "cache2:11211", "cache3:11211"]
ch = ConsistentHash()
for server in servers:
    ch.add_server(server)

# Получаем значение
key = "user:123:profile"
server = ch.get_server(key)
cache_client = memcache.Client([server])
value = cache_client.get(key)
```

### 2. Load Balancing

**Проблема**: Распределить запросы между серверами, сохраняя sticky sessions.

**Решение**: Hash session ID и направляйте на тот же сервер.

**Пример**:
```python
# Пользователь с session_id всегда попадает на тот же app server
session_id = request.cookies.get("session_id")
server = ch.get_server(session_id)
proxy_to(server)
```

### 3. Data Partitioning (Cassandra, DynamoDB)

**Проблема**: Распределить данные между узлами кластера.

**Решение**: Consistent hashing определяет, на каких узлах хранить данные.

**Cassandra**:
- Каждый узел владеет диапазоном токенов на кольце
- Данные автоматически распределяются и реплицируются
- Добавление узла перераспределяет только часть данных

### 4. Content Delivery Network (CDN)

**Проблема**: Распределить контент между edge серверами.

**Решение**: Hash URL контента для определения серверов.

**Пример**:
```python
url = "https://example.com/images/photo.jpg"
servers = ch.get_servers(url, count=3)  # 3 ближайших сервера для репликации
```

### 5. Distributed Storage (Amazon S3, HDFS)

**Проблема**: Распределить файлы между storage узлами.

**Решение**: Hash file ID или path.

## Расширенные техники

### 1. Weighted Consistent Hashing

**Проблема**: Серверы имеют разную мощность.

**Решение**: Больше виртуальных узлов для мощных серверов.

```python
class WeightedConsistentHash(ConsistentHash):
    def add_server(self, server_name, weight=1):
        """Weight определяет количество виртуальных узлов"""
        num_vnodes = int(self.num_virtual_nodes * weight)
        for i in range(num_vnodes):
            virtual_key = f"{server_name}#{i}"
            hash_value = self._hash(virtual_key)
            self.ring[hash_value] = server_name
            self.sorted_keys.append(hash_value)
        self.sorted_keys.sort()

# Использование
ch = WeightedConsistentHash()
ch.add_server("server_A", weight=1)    # 100 vnodes
ch.add_server("server_B", weight=2)    # 200 vnodes (2x мощнее)
ch.add_server("server_C", weight=0.5)  # 50 vnodes (0.5x мощнее)
```

### 2. Jump Consistent Hash

**Google's Jump Hash** — альтернатива, не требующая хранения кольца.

**Преимущества**:
- O(1) память (не нужно хранить кольцо)
- O(log N) время для поиска сервера
- Идеально равномерное распределение

**Недостатки**:
- Серверы должны быть пронумерованы 0..N-1
- Нельзя удалять произвольный сервер (только последний)

**Use case**: Когда серверы добавляются/удаляются последовательно.

### 3. Rendezvous Hashing (HRW)

**Highest Random Weight** — альтернативный алгоритм.

**Принцип**:
1. Для каждого ключа вычисляем weight для всех серверов
2. Выбираем сервер с максимальным weight

```python
def get_server_hrw(key, servers):
    max_weight = -1
    selected_server = None

    for server in servers:
        weight = hash(key + server)
        if weight > max_weight:
            max_weight = weight
            selected_server = server

    return selected_server
```

**Преимущества**:
- Простая реализация
- Не требует хранения структур данных

**Недостатки**:
- O(N) для каждого запроса (где N — число серверов)

## Consistent Hashing в реальных системах

### Amazon DynamoDB

- Использует consistent hashing для партиционирования
- Каждый узел владеет диапазоном hash values
- Данные автоматически реплицируются на N узлов по часовой стрелке
- При добавлении узла данные автоматически rebalance

### Apache Cassandra

```
Token Ring:
Node A: tokens 0 - 100
Node B: tokens 100 - 200
Node C: tokens 200 - 300

Partition key "user_123" → hash = 150 → Node B (primary)
Replication factor 3 → также на Node C и Node A
```

### Memcached (с библиотеками типа ketama)

```python
# Ketama algorithm — популярная реализация consistent hashing
import pylibmc

mc = pylibmc.Client(
    ["cache1:11211", "cache2:11211", "cache3:11211"],
    binary=True,
    behaviors={"ketama": True}  # включаем consistent hashing
)

mc.set("key", "value")
```

### Redis Cluster

- 16384 hash slots
- Каждый узел владеет частью slots
- Hash slot = CRC16(key) mod 16384
- Похоже на consistent hashing, но с фиксированным числом slots

## Trade-offs

### Преимущества

✅ Минимальное перемещение данных при изменении числа узлов
✅ Масштабируемость (легко добавлять/удалять узлы)
✅ Decentralized (не нужен центральный coordinator)
✅ Fault tolerance (при падении узла ключи распределяются между оставшимися)

### Недостатки

❌ Сложнее простого modulo hashing
❌ Требует хранения кольца (память)
❌ Не идеально равномерное распределение (даже с виртуальными узлами)
❌ Cascade failures возможны при одновременном падении узлов

### Альтернативы

| Метод | Когда использовать |
|-------|-------------------|
| Modulo hashing | Фиксированное число серверов, rehash допустим |
| Consistent hashing | Динамическое число серверов, минимальный rehash |
| Jump hash | Серверы добавляются последовательно, идеальный balance |
| Rendezvous hashing | Малое число серверов, простота важнее performance |

## Практические упражнения

### Задача 1: Анализ перемещения ключей

У вас 5 серверов и 1,000,000 ключей.
- Сколько ключей переместится при добавлении 6-го сервера?
- С modulo hashing?
- С consistent hashing?

<details>
<summary>Решение</summary>

**Modulo hashing**:
```
Было: hash(key) % 5
Стало: hash(key) % 6
Переместится: ~833,333 ключей (83%)
```

**Consistent hashing**:
```
Переместится: 1,000,000 / 6 ≈ 166,667 ключей (17%)
5x меньше!
```
</details>

### Задача 2: Выбор числа виртуальных узлов

У вас 10 серверов. Сколько виртуальных узлов нужно для стандартного отклонения < 5%?

<details>
<summary>Решение</summary>

```
σ = sqrt(1/(N×V)) < 0.05
sqrt(1/(10×V)) < 0.05
1/(10×V) < 0.0025
10×V > 400
V > 40

Нужно минимум 40 виртуальных узлов на сервер.
На практике используют 100-150.
```
</details>

## Что почитать дальше

### Papers
- "Consistent Hashing and Random Trees" by Karger et al. (1997) — оригинальная статья
- "A Fast, Minimal Memory, Consistent Hash Algorithm" (Jump Hash) by Google

### Статьи
- [Consistent Hashing Explained](https://www.toptal.com/big-data/consistent-hashing)
- [Amazon's Dynamo Paper](https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html)

## Проверьте себя

1. В чем главная проблема modulo hashing?
2. Как работает consistent hashing с точки зрения hash ring?
3. Зачем нужны виртуальные узлы?
4. Сколько ключей переместится при добавлении N+1 сервера к N серверам?
5. Где используется consistent hashing в реальных системах?
6. В чем разница между consistent hashing и jump hash?
7. Как реализовать weighted consistent hashing?

## Ключевые выводы

- Consistent hashing решает проблему rehashing при изменении числа серверов
- Минимизирует перемещение данных: ~K/N ключей вместо ~K×(N-1)/N
- Виртуальные узлы обеспечивают равномерное распределение
- Широко используется в distributed systems (Cassandra, DynamoDB, Memcached)
- Типичное число виртуальных узлов: 100-500 на физический сервер
- Binary search делает поиск сервера O(log N)
- Trade-off: сложность реализации vs минимальный rehash
- Не идеально равномерное распределение (даже с vnodes)

---

**Предыдущий урок**: [CAP теорема, ACID vs BASE](05-cap-teorema-acid-vs-base.md)
**Следующий урок**: [DNS и основы работы интернета](07-dns-i-osnovy-raboty-interneta.md)
