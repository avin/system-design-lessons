# Урок 42: Autocomplete System (Typeahead)

## Введение

Autocomplete (также typeahead или search suggestion) — это система, которая предлагает варианты завершения запроса пользователя в реальном времени по мере ввода. Это критически важная функция для улучшения user experience в поисковых системах, e-commerce, социальных сетях.

В этом уроке мы спроектируем масштабируемую систему autocomplete, способную обрабатывать миллионы запросов в секунду с латентностью < 100ms.

**Примеры:**
- **Google Search**: поисковые подсказки
- **Amazon**: подсказки товаров
- **Facebook/Instagram**: поиск друзей, хэштегов
- **IDE**: автодополнение кода (VSCode IntelliSense)

## Требования к системе

### Функциональные требования

1. **Автодополнение**:
   - По мере ввода показывать топ-5 (или 10) релевантных подсказок
   - Поддержка prefix matching (`"ama"` → `"amazon"`, `"amazing"`)

2. **Ranking**:
   - Сортировка по популярности/релевантности
   - Персонализация (опционально)

3. **Real-time updates**:
   - Новые популярные запросы должны появляться в suggestions

### Нефункциональные требования

1. **Latency**: < 100ms (p99)
2. **Availability**: 99.99%
3. **Scalability**:
   - 10M DAU (Daily Active Users)
   - 10 searches per user per day = 100M searches/day
   - При вводе запроса — в среднем 5 символов → 5 API calls
   - Total: 500M requests/day ≈ 5,800 req/sec (average)
   - Peak (3x): 17,400 req/sec

4. **Storage**:
   - 100M unique queries
   - Average query length: 50 characters
   - Total: 100M × 50 bytes = 5 GB (сырые данные)
   - С метаданными (frequency, timestamp): ×3 = 15 GB

## Архитектура высокого уровня

```
┌─────────┐
│ Client  │
└────┬────┘
     │ GET /autocomplete?q=ama
     ↓
┌────────────────┐
│  API Gateway   │
│ (Rate Limiting)│
└───────┬────────┘
        │
        ↓
┌───────────────────┐
│ Autocomplete API  │
│   (Service)       │
└───────┬───────────┘
        │
        ├──────────────────┐
        │                  │
        ↓                  ↓
┌──────────────┐    ┌─────────────┐
│    Trie      │    │    Redis    │
│   Cache      │    │   (Cache)   │
│  (in-memory) │    └─────────────┘
└──────────────┘            │
                            ↓
                    ┌───────────────┐
                    │  Query Store  │
                    │  (Database)   │
                    └───────────────┘

Background:
┌──────────────────┐
│  Analytics       │
│  (Log Aggregator)│
└───────┬──────────┘
        │
        ↓
┌───────────────────┐
│  Trie Builder     │
│  (Batch Job)      │
└───────────────────┘
```

## Ключевая структура данных: Trie (Prefix Tree)

Trie — идеальная структура для prefix matching.

```
        root
       / | \
      a  b  c
     /
    m
   / \
  a   e
 /     \
z       r
|       |
o       i
|       |
n       c
        |
        a

Queries:
- "amazon" (frequency: 1000)
- "america" (frequency: 500)

Поиск "am":
1. Идём root → a → m
2. Собираем все слова в поддереве:
   - "amazon" (1000)
   - "america" (500)
3. Сортируем по frequency
4. Возвращаем топ-5
```

**Реализация Trie:**

```javascript
class TrieNode {
  constructor() {
    this.children = new Map(); // символ → TrieNode
    this.isEndOfWord = false;
    this.frequency = 0;
    this.query = null; // полный query (только для конечных узлов)
  }
}

class Trie {
  constructor() {
    this.root = new TrieNode();
  }

  insert(query, frequency = 1) {
    let node = this.root;

    for (const char of query.toLowerCase()) {
      if (!node.children.has(char)) {
        node.children.set(char, new TrieNode());
      }

      node = node.children.get(char);
    }

    node.isEndOfWord = true;
    node.query = query;
    node.frequency = frequency;
  }

  search(prefix, limit = 5) {
    let node = this.root;

    // Находим узел для prefix
    for (const char of prefix.toLowerCase()) {
      if (!node.children.has(char)) {
        return []; // prefix не найден
      }

      node = node.children.get(char);
    }

    // Собираем все queries в поддереве
    const results = [];
    this.collectQueries(node, results);

    // Сортируем по frequency (desc) и берём топ-N
    results.sort((a, b) => b.frequency - a.frequency);

    return results.slice(0, limit).map(r => ({
      query: r.query,
      frequency: r.frequency
    }));
  }

  collectQueries(node, results) {
    if (node.isEndOfWord) {
      results.push({
        query: node.query,
        frequency: node.frequency
      });
    }

    for (const [char, childNode] of node.children) {
      this.collectQueries(childNode, results);
    }
  }

  // Для больших Trie: храним top-K в каждом узле
  optimizedSearch(prefix, limit = 5) {
    let node = this.root;

    for (const char of prefix.toLowerCase()) {
      if (!node.children.has(char)) {
        return [];
      }

      node = node.children.get(char);
    }

    // Если в узле уже есть topK — возвращаем сразу
    return node.topK || [];
  }

  // Precompute topK для каждого узла (при построении Trie)
  precomputeTopK(limit = 5) {
    this.precomputeTopKHelper(this.root, limit);
  }

  precomputeTopKHelper(node, limit) {
    const allQueries = [];
    this.collectQueries(node, allQueries);

    allQueries.sort((a, b) => b.frequency - a.frequency);
    node.topK = allQueries.slice(0, limit);

    for (const [char, childNode] of node.children) {
      this.precomputeTopKHelper(childNode, limit);
    }
  }
}

// Использование
const trie = new Trie();

// Загружаем популярные queries
trie.insert('amazon', 1000);
trie.insert('amazon prime', 800);
trie.insert('amazing spider-man', 500);
trie.insert('america', 600);

// Precompute для оптимизации
trie.precomputeTopK(5);

// Поиск
const suggestions = trie.optimizedSearch('am', 5);
console.log(suggestions);
// [
//   { query: 'amazon', frequency: 1000 },
//   { query: 'amazon prime', frequency: 800 },
//   { query: 'america', frequency: 600 },
//   { query: 'amazing spider-man', frequency: 500 }
// ]
```

## Реализация Autocomplete Service

```javascript
const express = require('express');
const Redis = require('ioredis');
const redis = new Redis();

class AutocompleteService {
  constructor() {
    this.trie = new Trie();
    this.redis = redis;

    this.loadTrie();
    this.setupRoutes();
  }

  async loadTrie() {
    console.log('Loading Trie from database...');

    // Загружаем топ queries из БД
    const result = await db.query(
      `SELECT query, frequency
       FROM search_queries
       ORDER BY frequency DESC
       LIMIT 10000000` // топ 10M queries
    );

    for (const row of result.rows) {
      this.trie.insert(row.query, row.frequency);
    }

    // Precompute topK для быстрого поиска
    this.trie.precomputeTopK(10);

    console.log(`Trie loaded with ${result.rows.length} queries`);
  }

  setupRoutes() {
    const app = express();

    app.get('/autocomplete', async (req, res) => {
      try {
        const { q, limit = 5 } = req.query;

        if (!q || q.length === 0) {
          return res.json({ suggestions: [] });
        }

        const start = Date.now();

        // Проверяем Redis cache
        const cacheKey = `autocomplete:${q.toLowerCase()}:${limit}`;
        const cached = await this.redis.get(cacheKey);

        if (cached) {
          console.log(`Cache hit for "${q}"`);

          return res.json({
            suggestions: JSON.parse(cached),
            latency: Date.now() - start,
            cached: true
          });
        }

        // Ищем в Trie
        const suggestions = this.trie.optimizedSearch(q, parseInt(limit));

        // Кэшируем результат (TTL 1 hour)
        await this.redis.setex(cacheKey, 3600, JSON.stringify(suggestions));

        res.json({
          suggestions,
          latency: Date.now() - start,
          cached: false
        });

        // Асинхронно логируем query (для analytics)
        this.logQuery(q, suggestions.length > 0);
      } catch (error) {
        console.error('Autocomplete error:', error);
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3000, () => {
      console.log('Autocomplete service listening on port 3000');
    });
  }

  async logQuery(query, hasResults) {
    // Логируем в Kafka/analytics system
    await kafka.send({
      topic: 'search-queries',
      messages: [{
        value: JSON.stringify({
          query,
          hasResults,
          timestamp: Date.now()
        })
      }]
    });
  }
}

const service = new AutocompleteService();
```

## Data Collection & Analytics

### 1. Log Aggregator

```javascript
const { Kafka } = require('kafkajs');

class SearchQueryAggregator {
  constructor() {
    this.kafka = new Kafka({
      clientId: 'query-aggregator',
      brokers: ['kafka:9092']
    });

    this.consumer = this.kafka.consumer({ groupId: 'aggregators' });
    this.counts = new Map(); // query → count (in-memory buffer)
    this.flushInterval = 60000; // flush каждую минуту
  }

  async start() {
    await this.consumer.connect();
    await this.consumer.subscribe({ topic: 'search-queries' });

    console.log('Query aggregator started');

    await this.consumer.run({
      eachMessage: async ({ message }) => {
        const { query } = JSON.parse(message.value.toString());

        // Агрегируем в памяти
        const current = this.counts.get(query) || 0;
        this.counts.set(query, current + 1);
      }
    });

    // Периодический flush в БД
    setInterval(() => this.flush(), this.flushInterval);
  }

  async flush() {
    if (this.counts.size === 0) return;

    console.log(`Flushing ${this.counts.size} queries to database`);

    const queries = Array.from(this.counts.entries());

    // Batch update в БД
    await db.query('BEGIN');

    for (const [query, count] of queries) {
      await db.query(
        `INSERT INTO search_queries (query, frequency, last_updated)
         VALUES ($1, $2, NOW())
         ON CONFLICT (query) DO UPDATE SET
           frequency = search_queries.frequency + $2,
           last_updated = NOW()`,
        [query, count]
      );
    }

    await db.query('COMMIT');

    this.counts.clear();
  }
}

const aggregator = new SearchQueryAggregator();
aggregator.start();
```

### 2. Trie Rebuilder (Background Job)

```javascript
class TrieRebuilder {
  constructor() {
    this.rebuildInterval = 3600000; // каждый час
  }

  start() {
    console.log('Trie rebuilder started');

    setInterval(async () => {
      await this.rebuild();
    }, this.rebuildInterval);
  }

  async rebuild() {
    console.log('Rebuilding Trie...');

    const start = Date.now();

    // Создаём новый Trie
    const newTrie = new Trie();

    // Загружаем обновлённые данные
    const result = await db.query(
      `SELECT query, frequency
       FROM search_queries
       WHERE frequency > 10  -- минимальный порог
       ORDER BY frequency DESC
       LIMIT 10000000`
    );

    for (const row of result.rows) {
      newTrie.insert(row.query, row.frequency);
    }

    newTrie.precomputeTopK(10);

    // Atomic swap: заменяем старый Trie на новый
    global.trie = newTrie;

    // Инвалидируем Redis cache
    await this.invalidateCache();

    console.log(`Trie rebuilt in ${Date.now() - start}ms with ${result.rows.length} queries`);
  }

  async invalidateCache() {
    // Удаляем все autocomplete кэши
    const keys = await redis.keys('autocomplete:*');

    if (keys.length > 0) {
      await redis.del(...keys);
      console.log(`Invalidated ${keys.length} cache keys`);
    }
  }
}

const rebuilder = new TrieRebuilder();
rebuilder.start();
```

## Оптимизации

### 1. Serialization Trie для быстрой загрузки

```javascript
class TrieSerializer {
  static serialize(trie) {
    return JSON.stringify(this.serializeNode(trie.root));
  }

  static serializeNode(node) {
    const obj = {
      c: {}, // children
      e: node.isEndOfWord,
      q: node.query,
      f: node.frequency,
      t: node.topK
    };

    for (const [char, childNode] of node.children) {
      obj.c[char] = this.serializeNode(childNode);
    }

    return obj;
  }

  static deserialize(json) {
    const trie = new Trie();
    trie.root = this.deserializeNode(JSON.parse(json));
    return trie;
  }

  static deserializeNode(obj) {
    const node = new TrieNode();
    node.isEndOfWord = obj.e;
    node.query = obj.q;
    node.frequency = obj.f;
    node.topK = obj.t;

    for (const [char, childObj] of Object.entries(obj.c)) {
      node.children.set(char, this.deserializeNode(childObj));
    }

    return node;
  }

  static async saveToFile(trie, filename) {
    const serialized = this.serialize(trie);
    const compressed = await gzip(serialized);

    await fs.promises.writeFile(filename, compressed);
  }

  static async loadFromFile(filename) {
    const compressed = await fs.promises.readFile(filename);
    const serialized = await gunzip(compressed);

    return this.deserialize(serialized.toString());
  }
}

// Использование
await TrieSerializer.saveToFile(trie, './trie-snapshot.json.gz');

// При старте сервиса
const trie = await TrieSerializer.loadFromFile('./trie-snapshot.json.gz');
```

### 2. Sharding Trie по первому символу

Для очень больших datasets (billions queries):

```javascript
class ShardedTrie {
  constructor() {
    this.shards = new Map(); // первый символ → Trie

    // Создаём Trie для каждого символа
    for (let i = 97; i <= 122; i++) {
      // a-z
      this.shards.set(String.fromCharCode(i), new Trie());
    }

    // Отдельный shard для цифр и специальных символов
    this.shards.set('0', new Trie()); // для queries начинающихся с цифр
  }

  insert(query, frequency) {
    const firstChar = query[0].toLowerCase();
    const shard = this.shards.get(firstChar) || this.shards.get('0');

    shard.insert(query, frequency);
  }

  search(prefix, limit) {
    const firstChar = prefix[0].toLowerCase();
    const shard = this.shards.get(firstChar) || this.shards.get('0');

    return shard.optimizedSearch(prefix, limit);
  }

  precomputeTopK(limit) {
    for (const [char, shard] of this.shards) {
      console.log(`Precomputing shard '${char}'`);
      shard.precomputeTopK(limit);
    }
  }
}
```

### 3. Distributed Trie (Multi-node)

```javascript
class DistributedAutocomplete {
  constructor(nodes) {
    this.nodes = nodes; // массив адресов нод
    this.ring = new ConsistentHashing(nodes);
  }

  async search(prefix, limit) {
    // Определяем ноду по первому символу
    const node = this.ring.getNode(prefix[0]);

    // Отправляем запрос на эту ноду
    try {
      const response = await axios.get(`http://${node.host}/autocomplete`, {
        params: { q: prefix, limit },
        timeout: 100
      });

      return response.data.suggestions;
    } catch (error) {
      console.error(`Failed to query node ${node.host}:`, error.message);

      // Fallback на другую ноду
      const fallbackNode = this.ring.getNode(prefix[0] + prefix[1]);
      const response = await axios.get(`http://${fallbackNode.host}/autocomplete`, {
        params: { q: prefix, limit },
        timeout: 100
      });

      return response.data.suggestions;
    }
  }
}
```

### 4. Fuzzy Matching (опционально)

Для исправления опечаток (Levenshtein distance):

```javascript
function levenshteinDistance(a, b) {
  const matrix = [];

  for (let i = 0; i <= b.length; i++) {
    matrix[i] = [i];
  }

  for (let j = 0; j <= a.length; j++) {
    matrix[0][j] = j;
  }

  for (let i = 1; i <= b.length; i++) {
    for (let j = 1; j <= a.length; j++) {
      if (b.charAt(i - 1) === a.charAt(j - 1)) {
        matrix[i][j] = matrix[i - 1][j - 1];
      } else {
        matrix[i][j] = Math.min(
          matrix[i - 1][j - 1] + 1, // substitution
          matrix[i][j - 1] + 1,     // insertion
          matrix[i - 1][j] + 1      // deletion
        );
      }
    }
  }

  return matrix[b.length][a.length];
}

class FuzzyTrie extends Trie {
  fuzzySearch(prefix, limit = 5, maxDistance = 2) {
    const results = [];

    this.fuzzySearchHelper(this.root, prefix, '', 0, maxDistance, results);

    results.sort((a, b) => {
      // Сначала по distance, потом по frequency
      if (a.distance !== b.distance) {
        return a.distance - b.distance;
      }

      return b.frequency - a.frequency;
    });

    return results.slice(0, limit);
  }

  fuzzySearchHelper(node, prefix, currentWord, distance, maxDistance, results) {
    if (node.isEndOfWord && currentWord.startsWith(prefix[0])) {
      const dist = levenshteinDistance(prefix, currentWord);

      if (dist <= maxDistance) {
        results.push({
          query: node.query,
          frequency: node.frequency,
          distance: dist
        });
      }
    }

    if (distance > maxDistance) return;

    for (const [char, childNode] of node.children) {
      this.fuzzySearchHelper(
        childNode,
        prefix,
        currentWord + char,
        distance,
        maxDistance,
        results
      );
    }
  }
}
```

## Персонализация

```javascript
class PersonalizedAutocomplete {
  async search(userId, prefix, limit) {
    // Получаем персональную историю
    const userHistory = await this.getUserHistory(userId);

    // Ищем в global Trie
    const globalResults = trie.optimizedSearch(prefix, limit * 2);

    // Boost результаты из истории пользователя
    const boosted = globalResults.map(r => {
      const inHistory = userHistory.includes(r.query);

      return {
        ...r,
        score: inHistory ? r.frequency * 2 : r.frequency
      };
    });

    // Сортируем по score и возвращаем топ-N
    boosted.sort((a, b) => b.score - a.score);

    return boosted.slice(0, limit);
  }

  async getUserHistory(userId) {
    const result = await db.query(
      `SELECT DISTINCT query
       FROM user_search_history
       WHERE user_id = $1
       ORDER BY created_at DESC
       LIMIT 100`,
      [userId]
    );

    return result.rows.map(r => r.query);
  }
}
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const autocompleteRequests = new prometheus.Counter({
  name: 'autocomplete_requests_total',
  help: 'Total autocomplete requests',
  labelNames: ['status'] // success, error, empty
});

const autocompleteLatency = new prometheus.Histogram({
  name: 'autocomplete_latency_seconds',
  help: 'Autocomplete request latency',
  buckets: [0.01, 0.05, 0.1, 0.2, 0.5]
});

const cacheHitRate = new prometheus.Counter({
  name: 'autocomplete_cache_hits_total',
  help: 'Cache hits vs misses',
  labelNames: ['result'] // hit, miss
});

// В API endpoint
app.get('/autocomplete', async (req, res) => {
  const start = process.hrtime.bigint();

  try {
    // ... логика ...

    if (cached) {
      cacheHitRate.inc({ result: 'hit' });
    } else {
      cacheHitRate.inc({ result: 'miss' });
    }

    if (suggestions.length > 0) {
      autocompleteRequests.inc({ status: 'success' });
    } else {
      autocompleteRequests.inc({ status: 'empty' });
    }
  } catch (error) {
    autocompleteRequests.inc({ status: 'error' });
    throw error;
  } finally {
    const duration = Number(process.hrtime.bigint() - start) / 1e9;
    autocompleteLatency.observe(duration);
  }
});
```

## Trade-offs

| Аспект | Trie | Database (Indexed) | Elasticsearch |
|--------|------|-------------------|---------------|
| Latency | < 10ms | 50-100ms | 20-50ms |
| Memory | High (in-memory) | Low (disk) | Medium |
| Freshness | Batch update (hourly) | Real-time | Near real-time |
| Scalability | Vertical (RAM) | Horizontal | Horizontal |
| Complexity | Medium | Low | High |

## Что читать дальше

- **Урок 43**: Newsfeed System — ещё одна система с ranking
- **Урок 45**: Search Engine — full-text search vs autocomplete
- **Урок 16**: Caching Strategies — оптимизация latency

## Проверь себя

1. Почему Trie лучше чем B-Tree для autocomplete?
2. Как обновлять Trie без downtime?
3. Спроектируйте autocomplete для 1B queries с personalization.
4. Сколько памяти потребуется Trie для 10M queries (avg length 50 chars)?
5. Как реализовать multi-language autocomplete?

---
**Предыдущий урок**: [Урок 41: Notification System](41-notification-system.md)
**Следующий урок**: [Урок 43: Newsfeed System](43-newsfeed.md)
