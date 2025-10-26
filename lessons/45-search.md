# Урок 45: Search System

## Введение

Search System (поисковая система) — это сложный комплекс компонентов для индексации, хранения и поиска по большим объёмам данных. Поиск — критически важная функция для e-commerce, документации, социальных сетей и любых data-driven приложений.

В этом уроке мы спроектируем масштабируемую систему поиска, способную обрабатывать миллионы документов и тысячи запросов в секунду с низкой латентностью.

**Примеры:**
- **Google**: web search
- **Amazon**: product search
- **Airbnb**: listings search
- **GitHub**: code search

## Требования к системе

### Функциональные требования

1. **Indexing**:
   - Индексация документов (add, update, delete)
   - Support для различных типов данных (text, numbers, dates)

2. **Searching**:
   - Full-text search
   - Filters (price range, category, etc.)
   - Faceted search (aggregations)
   - Fuzzy matching (опечатки)
   - Autocomplete (covered in урок 42)

3. **Ranking**:
   - Relevance scoring (TF-IDF, BM25)
   - Custom ranking signals (popularity, freshness)

### Нефункциональные требования

1. **Latency**: < 100ms (p95)
2. **Throughput**: 10K queries/sec
3. **Scalability**: 100M+ documents
4. **Availability**: 99.9%
5. **Freshness**: Near real-time indexing (< 1 sec)

### Back-of-the-envelope

```
Documents: 100M
Average document size: 5 KB
Total storage: 100M × 5 KB = 500 GB

With inverted index (1.5x overhead):
500 GB × 1.5 = 750 GB

Queries: 10K/sec
Average query latency: 50ms
Concurrent queries: 10K × 0.05 = 500

Index updates: 1000 docs/sec
Indexing throughput: 1000 × 5 KB = 5 MB/sec
```

## Ключевые концепции

### 1. Inverted Index

Основная структура данных для full-text search.

```
Documents:
Doc1: "The quick brown fox"
Doc2: "Quick brown dogs"
Doc3: "The lazy dog"

Inverted Index:
┌─────────┬──────────────────┐
│  Term   │   Postings List  │
├─────────┼──────────────────┤
│ the     │ [Doc1, Doc3]     │
│ quick   │ [Doc1, Doc2]     │
│ brown   │ [Doc1, Doc2]     │
│ fox     │ [Doc1]           │
│ dogs    │ [Doc2]           │
│ lazy    │ [Doc3]           │
│ dog     │ [Doc3]           │
└─────────┴──────────────────┘

Query: "brown dog"
→ terms: ["brown", "dog"]
→ postings: brown=[Doc1, Doc2], dog=[Doc3]
→ union: [Doc1, Doc2, Doc3]
→ rank by relevance
```

**Реализация (упрощённая):**

```javascript
class InvertedIndex {
  constructor() {
    this.index = new Map(); // term → Set<docId>
    this.documents = new Map(); // docId → document
  }

  addDocument(docId, text) {
    const terms = this.tokenize(text);

    // Добавляем в inverted index
    for (const term of terms) {
      if (!this.index.has(term)) {
        this.index.set(term, new Set());
      }

      this.index.get(term).add(docId);
    }

    // Сохраняем оригинальный документ
    this.documents.set(docId, text);
  }

  search(query) {
    const terms = this.tokenize(query);

    if (terms.length === 0) return [];

    // Получаем postings lists для всех terms
    const postingsLists = terms
      .map(term => this.index.get(term) || new Set())
      .filter(set => set.size > 0);

    if (postingsLists.length === 0) return [];

    // Находим документы, содержащие хотя бы один term (OR)
    const resultSet = new Set();

    for (const postings of postingsLists) {
      for (const docId of postings) {
        resultSet.add(docId);
      }
    }

    // Ранжируем по релевантности
    const results = Array.from(resultSet).map(docId => ({
      docId,
      score: this.calculateScore(docId, terms)
    }));

    results.sort((a, b) => b.score - a.score);

    return results;
  }

  calculateScore(docId, queryTerms) {
    let score = 0;

    const doc = this.documents.get(docId);
    const docTerms = this.tokenize(doc);

    // TF (Term Frequency)
    for (const queryTerm of queryTerms) {
      const tf = docTerms.filter(t => t === queryTerm).length / docTerms.length;

      // IDF (Inverse Document Frequency)
      const docsWithTerm = (this.index.get(queryTerm) || new Set()).size;
      const idf = Math.log(this.documents.size / (docsWithTerm + 1));

      score += tf * idf;
    }

    return score;
  }

  tokenize(text) {
    return text
      .toLowerCase()
      .replace(/[^\w\s]/g, '') // удаляем пунктуацию
      .split(/\s+/)
      .filter(word => word.length > 0);
  }

  deleteDocument(docId) {
    // Удаляем из inverted index
    for (const [term, postings] of this.index) {
      postings.delete(docId);

      if (postings.size === 0) {
        this.index.delete(term);
      }
    }

    this.documents.delete(docId);
  }
}

// Использование
const index = new InvertedIndex();

index.addDocument('doc1', 'The quick brown fox jumps over the lazy dog');
index.addDocument('doc2', 'Quick brown dogs are friendly');
index.addDocument('doc3', 'The lazy cat sleeps all day');

const results = index.search('quick brown');
console.log(results);
// [
//   { docId: 'doc1', score: 0.45 },
//   { docId: 'doc2', score: 0.38 }
// ]
```

### 2. BM25 Ranking Algorithm

Улучшенная версия TF-IDF.

```javascript
class BM25Scorer {
  constructor(k1 = 1.5, b = 0.75) {
    this.k1 = k1; // term saturation parameter
    this.b = b;   // length normalization parameter
  }

  score(doc, query, avgDocLength, totalDocs, docFreq) {
    const docLength = doc.terms.length;
    let score = 0;

    for (const term of query.terms) {
      const tf = doc.termFrequency[term] || 0;
      const df = docFreq[term] || 0;

      // IDF component
      const idf = Math.log(
        (totalDocs - df + 0.5) / (df + 0.5) + 1
      );

      // Normalized TF component
      const numerator = tf * (this.k1 + 1);
      const denominator = tf + this.k1 * (
        1 - this.b + this.b * (docLength / avgDocLength)
      );

      score += idf * (numerator / denominator);
    }

    return score;
  }
}
```

## Архитектура с Elasticsearch

```
┌──────────┐
│ Client   │
└────┬─────┘
     │ POST /search
     ↓
┌────────────────┐
│  Search API    │
│  (Node.js)     │
└────┬───────────┘
     │
     ↓
┌─────────────────────────────┐
│   Elasticsearch Cluster     │
│  ┌──────┐ ┌──────┐ ┌──────┐│
│  │Node 1│ │Node 2│ │Node 3││
│  │Shard │ │Shard │ │Shard ││
│  │ 0,1  │ │ 2,3  │ │ 4,5  ││
│  └──────┘ └──────┘ └──────┘│
└─────────────────────────────┘
     ↑
     │ Index updates
┌────┴──────────┐
│  Indexer      │
│  (Worker)     │
└───────────────┘
     ↑
     │
┌────┴──────────┐
│  Data Source  │
│  (Database)   │
└───────────────┘
```

## Реализация компонентов

### 1. Search API

```javascript
const express = require('express');
const { Client } = require('@elastic/elasticsearch');

const app = express();
const esClient = new Client({ node: 'http://elasticsearch:9200' });

class SearchService {
  constructor() {
    this.setupRoutes();
  }

  setupRoutes() {
    app.use(express.json());

    // Search endpoint
    app.post('/search', async (req, res) => {
      try {
        const {
          query,
          filters = {},
          page = 1,
          pageSize = 20,
          sort = '_score'
        } = req.body;

        const results = await this.search({
          query,
          filters,
          page,
          pageSize,
          sort
        });

        res.json(results);
      } catch (error) {
        console.error('Search error:', error);
        res.status(500).json({ error: error.message });
      }
    });

    // Faceted search (aggregations)
    app.post('/search/facets', async (req, res) => {
      try {
        const { query, facets } = req.body;

        const results = await this.facetedSearch(query, facets);

        res.json(results);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Index document
    app.post('/index', async (req, res) => {
      try {
        const { index, id, document } = req.body;

        await this.indexDocument(index, id, document);

        res.json({ success: true, id });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3000, () => {
      console.log('Search API listening on port 3000');
    });
  }

  async search({ query, filters, page, pageSize, sort }) {
    const from = (page - 1) * pageSize;

    // Build Elasticsearch query
    const esQuery = {
      bool: {
        must: [
          {
            multi_match: {
              query,
              fields: ['title^3', 'description^2', 'content'],
              type: 'best_fields',
              fuzziness: 'AUTO'
            }
          }
        ],
        filter: []
      }
    };

    // Apply filters
    if (filters.category) {
      esQuery.bool.filter.push({
        term: { category: filters.category }
      });
    }

    if (filters.price) {
      esQuery.bool.filter.push({
        range: {
          price: {
            gte: filters.price.min,
            lte: filters.price.max
          }
        }
      });
    }

    if (filters.dateRange) {
      esQuery.bool.filter.push({
        range: {
          created_at: {
            gte: filters.dateRange.from,
            lte: filters.dateRange.to
          }
        }
      });
    }

    // Execute search
    const result = await esClient.search({
      index: 'products',
      body: {
        query: esQuery,
        from,
        size: pageSize,
        sort: this.buildSort(sort),
        highlight: {
          fields: {
            title: {},
            description: {},
            content: {}
          },
          pre_tags: ['<strong>'],
          post_tags: ['</strong>']
        }
      }
    });

    return {
      total: result.hits.total.value,
      hits: result.hits.hits.map(hit => ({
        id: hit._id,
        score: hit._score,
        ...hit._source,
        highlights: hit.highlight
      })),
      page,
      pageSize,
      totalPages: Math.ceil(result.hits.total.value / pageSize)
    };
  }

  async facetedSearch(query, facets) {
    const aggs = {};

    // Build aggregations
    for (const facet of facets) {
      switch (facet.type) {
        case 'terms':
          aggs[facet.name] = {
            terms: {
              field: facet.field,
              size: facet.size || 10
            }
          };
          break;

        case 'range':
          aggs[facet.name] = {
            range: {
              field: facet.field,
              ranges: facet.ranges
            }
          };
          break;

        case 'histogram':
          aggs[facet.name] = {
            histogram: {
              field: facet.field,
              interval: facet.interval
            }
          };
          break;
      }
    }

    const result = await esClient.search({
      index: 'products',
      body: {
        query: {
          multi_match: {
            query,
            fields: ['title', 'description', 'content']
          }
        },
        size: 0, // только aggregations, без документов
        aggs
      }
    });

    return {
      query,
      facets: result.aggregations
    };
  }

  async indexDocument(index, id, document) {
    await esClient.index({
      index,
      id,
      document,
      refresh: 'wait_for' // для immediate visibility
    });
  }

  async deleteDocument(index, id) {
    await esClient.delete({
      index,
      id
    });
  }

  buildSort(sort) {
    if (sort === '_score') {
      return [{ _score: 'desc' }];
    }

    const [field, order] = sort.split(':');

    return [{ [field]: order || 'asc' }];
  }
}

const service = new SearchService();
```

### 2. Indexing Worker

```javascript
const { Kafka } = require('kafkajs');

class IndexingWorker {
  constructor() {
    this.kafka = new Kafka({
      clientId: 'indexer',
      brokers: ['kafka:9092']
    });

    this.consumer = this.kafka.consumer({ groupId: 'indexers' });
    this.esClient = new Client({ node: 'http://elasticsearch:9200' });

    this.batchSize = 1000;
    this.batch = [];
  }

  async start() {
    await this.consumer.connect();
    await this.consumer.subscribe({ topic: 'index-updates' });

    console.log('Indexing worker started');

    await this.consumer.run({
      eachMessage: async ({ message }) => {
        try {
          const update = JSON.parse(message.value.toString());
          await this.handleUpdate(update);
        } catch (error) {
          console.error('Indexing error:', error);
        }
      }
    });

    // Периодический flush batch
    setInterval(() => this.flushBatch(), 1000);
  }

  async handleUpdate(update) {
    const { operation, index, id, document } = update;

    switch (operation) {
      case 'index':
      case 'update':
        this.batch.push({
          index: { _index: index, _id: id }
        });
        this.batch.push(document);
        break;

      case 'delete':
        this.batch.push({
          delete: { _index: index, _id: id }
        });
        break;
    }

    if (this.batch.length >= this.batchSize * 2) {
      await this.flushBatch();
    }
  }

  async flushBatch() {
    if (this.batch.length === 0) return;

    console.log(`Flushing batch of ${this.batch.length / 2} operations`);

    const start = Date.now();

    try {
      const result = await this.esClient.bulk({
        body: this.batch,
        refresh: true
      });

      if (result.errors) {
        console.error('Bulk indexing errors:', result.items.filter(item => item.index?.error));
      }

      console.log(`Batch indexed in ${Date.now() - start}ms`);
    } catch (error) {
      console.error('Bulk indexing failed:', error);
    } finally {
      this.batch = [];
    }
  }
}

const worker = new IndexingWorker();
worker.start();
```

### 3. Sync Service (Database → Elasticsearch)

```javascript
class SyncService {
  constructor() {
    this.db = require('./database');
    this.kafka = require('./kafka');
  }

  async syncAll() {
    console.log('Starting full sync...');

    const stream = this.db.query('SELECT * FROM products').stream();

    let batch = [];
    const BATCH_SIZE = 1000;

    for await (const row of stream) {
      batch.push({
        operation: 'index',
        index: 'products',
        id: row.id,
        document: this.transformDocument(row)
      });

      if (batch.length >= BATCH_SIZE) {
        await this.publishBatch(batch);
        batch = [];
      }
    }

    if (batch.length > 0) {
      await this.publishBatch(batch);
    }

    console.log('Full sync completed');
  }

  async publishBatch(batch) {
    const messages = batch.map(item => ({
      value: JSON.stringify(item)
    }));

    await this.kafka.send({
      topic: 'index-updates',
      messages
    });
  }

  transformDocument(row) {
    return {
      title: row.title,
      description: row.description,
      price: parseFloat(row.price),
      category: row.category,
      created_at: row.created_at,
      rating: row.rating
    };
  }

  // Incremental sync (Change Data Capture)
  async watchChanges() {
    // Используем pg_notify или Debezium для CDC
    this.db.query('LISTEN product_changes');

    this.db.on('notification', async (msg) => {
      const { operation, id } = JSON.parse(msg.payload);

      switch (operation) {
        case 'INSERT':
        case 'UPDATE':
          const product = await this.db.query('SELECT * FROM products WHERE id = $1', [id]);

          await this.kafka.send({
            topic: 'index-updates',
            messages: [{
              value: JSON.stringify({
                operation: 'index',
                index: 'products',
                id,
                document: this.transformDocument(product.rows[0])
              })
            }]
          });
          break;

        case 'DELETE':
          await this.kafka.send({
            topic: 'index-updates',
            messages: [{
              value: JSON.stringify({
                operation: 'delete',
                index: 'products',
                id
              })
            }]
          });
          break;
      }
    });
  }
}
```

## Продвинутые фичи

### 1. Synonym Search

```javascript
// Elasticsearch mapping с синонимами
const mappingWithSynonyms = {
  settings: {
    analysis: {
      filter: {
        synonym_filter: {
          type: 'synonym',
          synonyms: [
            'laptop, notebook, computer',
            'phone, smartphone, mobile',
            'tv, television'
          ]
        }
      },
      analyzer: {
        synonym_analyzer: {
          tokenizer: 'standard',
          filter: ['lowercase', 'synonym_filter']
        }
      }
    }
  },
  mappings: {
    properties: {
      title: {
        type: 'text',
        analyzer: 'synonym_analyzer'
      },
      description: {
        type: 'text',
        analyzer: 'synonym_analyzer'
      }
    }
  }
};

await esClient.indices.create({
  index: 'products',
  body: mappingWithSynonyms
});
```

### 2. Geospatial Search

```javascript
async searchNearby(location, radius, query) {
  const result = await esClient.search({
    index: 'places',
    body: {
      query: {
        bool: {
          must: [
            {
              match: { name: query }
            }
          ],
          filter: [
            {
              geo_distance: {
                distance: `${radius}km`,
                location: {
                  lat: location.lat,
                  lon: location.lon
                }
              }
            }
          ]
        }
      },
      sort: [
        {
          _geo_distance: {
            location: {
              lat: location.lat,
              lon: location.lon
            },
            order: 'asc',
            unit: 'km'
          }
        }
      ]
    }
  });

  return result.hits.hits;
}
```

### 3. Autocomplete (опционально, см. урок 42)

```javascript
const autocompleteMapping = {
  mappings: {
    properties: {
      title: {
        type: 'text',
        fields: {
          suggest: {
            type: 'completion'
          }
        }
      }
    }
  }
};

// Search suggestions
async suggest(prefix) {
  const result = await esClient.search({
    index: 'products',
    body: {
      suggest: {
        title_suggest: {
          prefix,
          completion: {
            field: 'title.suggest',
            size: 5,
            skip_duplicates: true
          }
        }
      }
    }
  });

  return result.suggest.title_suggest[0].options;
}
```

### 4. Персонализация

```javascript
async personalizedSearch(userId, query, page, pageSize) {
  // Получаем user profile
  const userProfile = await this.getUserProfile(userId);

  // Boosting на основе preferences
  const result = await esClient.search({
    index: 'products',
    body: {
      query: {
        function_score: {
          query: {
            multi_match: {
              query,
              fields: ['title', 'description']
            }
          },
          functions: [
            // Boost по категории preference
            {
              filter: { term: { category: userProfile.preferredCategory } },
              weight: 2
            },
            // Boost по price range
            {
              gauss: {
                price: {
                  origin: userProfile.avgPurchasePrice,
                  scale: '100'
                }
              },
              weight: 1.5
            },
            // Recency boost
            {
              exp: {
                created_at: {
                  origin: 'now',
                  scale: '30d',
                  decay: 0.5
                }
              }
            }
          ],
          score_mode: 'multiply',
          boost_mode: 'multiply'
        }
      },
      from: (page - 1) * pageSize,
      size: pageSize
    }
  });

  return result.hits.hits;
}
```

## Оптимизации

### 1. Caching

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function cachedSearch(query, filters) {
  const cacheKey = `search:${JSON.stringify({ query, filters })}`;

  // Проверяем cache
  const cached = await redis.get(cacheKey);

  if (cached) {
    return JSON.parse(cached);
  }

  // Выполняем поиск
  const results = await esClient.search({ ... });

  // Кэшируем (TTL 5 minutes)
  await redis.setex(cacheKey, 300, JSON.stringify(results));

  return results;
}
```

### 2. Sharding Strategy

```javascript
// При создании индекса
await esClient.indices.create({
  index: 'products',
  body: {
    settings: {
      number_of_shards: 5,
      number_of_replicas: 2,
      routing: {
        allocation: {
          include: {
            _tier_preference: 'data_hot'
          }
        }
      }
    }
  }
});
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const searchRequests = new prometheus.Counter({
  name: 'search_requests_total',
  help: 'Total search requests',
  labelNames: ['status']
});

const searchLatency = new prometheus.Histogram({
  name: 'search_latency_seconds',
  help: 'Search query latency',
  buckets: [0.01, 0.05, 0.1, 0.5, 1]
});

const indexingRate = new prometheus.Counter({
  name: 'documents_indexed_total',
  help: 'Total documents indexed'
});
```

## Что читать дальше

- **Урок 42**: Autocomplete — специализированный случай search
- **Урок 40**: Web Crawler — сбор данных для индексации
- **Урок 46**: Nearby Friends — geospatial search

## Проверь себя

1. В чём разница между TF-IDF и BM25?
2. Как работает inverted index?
3. Спроектируйте search для 1B documents с < 50ms latency.
4. Как обеспечить near real-time indexing?
5. Реализуйте fuzzy search с Levenshtein distance.

---

[← Урок 44: Chat System](44-chat.md) | [Урок 46: Nearby Friends →](46-nearby-friends.md)
