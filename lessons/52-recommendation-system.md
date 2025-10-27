# Урок 52: Recommendation System — Система рекомендаций

## Введение

Recommendation System (система рекомендаций) — это сервис, который предсказывает предпочтения пользователей и предлагает релевантный контент: товары, фильмы, музыку, статьи, друзей. Современные рекомендательные системы приносят огромную бизнес-ценность: 35% покупок на Amazon и 75% просмотров на Netflix происходят благодаря рекомендациям.

### Примеры в реальной жизни

- **Amazon**: "Customers who bought this item also bought..."
- **Netflix**: персональные рекомендации фильмов и сериалов
- **YouTube**: рекомендации видео на главной странице
- **Spotify**: Discover Weekly, Release Radar
- **LinkedIn**: "People You May Know"

### Зачем нужна система рекомендаций?

1. **Увеличение конверсии** — релевантные предложения повышают вероятность покупки
2. **Удержание пользователей** — персонализированный контент увеличивает вовлечённость
3. **Обнаружение контента** — помогает пользователям найти интересные товары/контент
4. **Решение проблемы выбора** — упрощает выбор из миллионов вариантов

---

## Требования

### Функциональные требования

1. **Персональные рекомендации** — генерация списка рекомендаций для конкретного пользователя
2. **Похожие товары** — рекомендации похожих товаров/контента (item-to-item)
3. **Real-time обновления** — учёт последних действий пользователя
4. **Холодный старт** — рекомендации для новых пользователей и товаров
5. **Разнообразие** — баланс между точностью и разнообразием рекомендаций
6. **Объяснимость** — пользователь должен понимать, почему получил эту рекомендацию

### Нефункциональные требования

1. **Низкая латентность** — генерация рекомендаций < 100ms (для real-time)
2. **Высокая доступность** — 99.9% uptime
3. **Масштабируемость** — поддержка 100M+ пользователей, 10M+ товаров
4. **Точность** — высокая релевантность рекомендаций (CTR, conversion)
5. **Freshness** — актуальность рекомендаций с учётом трендов

### Расчёты (Back-of-the-envelope)

**Дано:**
- 100M пользователей
- 10M товаров
- 1B взаимодействий в месяц (~400 requests/sec)
- Хранение матрицы user-item: 100M × 10M = 10^15 ячеек

**Проблема**: Матрица крайне разреженная (sparse) — большинство пользователей взаимодействовали с < 0.1% товаров.

**Решение**: Использовать разреженные представления и матричную факторизацию для сжатия.

**Хранение моделей:**
- User embeddings: 100M пользователей × 128 dimensions × 4 bytes = 51GB
- Item embeddings: 10M товаров × 128 dimensions × 4 bytes = 5GB
- Total: ~56GB — помещается в память одного сервера

**Latency бюджет:**
- Получение user embedding: 5ms (cache lookup)
- Вычисление similarity с топ-1000 товаров: 20ms
- Ranking и filtering: 10ms
- Total: ~35ms ✅

---

## High-Level Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────┐
│      API Gateway / Load Balancer    │
└──────────────┬──────────────────────┘
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
┌─────────────┐  ┌──────────────┐
│ Recomm API  │  │  Item-to-Item│
│  Service    │  │   Service    │
└──────┬──────┘  └──────┬───────┘
       │                │
       │    ┌───────────┴───────────┐
       │    │                       │
       ▼    ▼                       ▼
┌────────────────┐         ┌──────────────┐
│  Embedding     │         │   Ranking    │
│   Service      │◄────────│   Service    │
└────────┬───────┘         └──────┬───────┘
         │                        │
         ▼                        ▼
┌─────────────────┐      ┌─────────────────┐
│  Redis Cache    │      │  Feature Store  │
│  (embeddings)   │      │  (user features)│
└─────────────────┘      └─────────────────┘

         Offline Pipeline
         ================
┌─────────────┐     ┌─────────────┐
│  Event Log  │────►│  Spark Jobs │
│  (Kafka)    │     │  (training) │
└─────────────┘     └──────┬──────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │  Model Storage  │
                  │     (S3)        │
                  └─────────────────┘
```

---

## Основные алгоритмы

### 1. Collaborative Filtering

#### User-Based Collaborative Filtering

Идея: находим похожих пользователей и рекомендуем то, что понравилось им.

```javascript
class UserBasedCF {
  constructor(interactions) {
    // interactions: Map<userId, Set<itemId>>
    this.interactions = interactions;
  }

  // Вычисление similarity между двумя пользователями (Jaccard)
  jaccardSimilarity(userId1, userId2) {
    const items1 = this.interactions.get(userId1) || new Set();
    const items2 = this.interactions.get(userId2) || new Set();

    const intersection = new Set([...items1].filter(x => items2.has(x)));
    const union = new Set([...items1, ...items2]);

    return intersection.size / union.size;
  }

  // Cosine similarity (для rating-based данных)
  cosineSimilarity(userId1, userId2, ratings) {
    const r1 = ratings.get(userId1) || {};
    const r2 = ratings.get(userId2) || {};

    const commonItems = Object.keys(r1).filter(item => r2[item]);

    if (commonItems.length === 0) return 0;

    let dotProduct = 0;
    let norm1 = 0;
    let norm2 = 0;

    for (const item of commonItems) {
      dotProduct += r1[item] * r2[item];
      norm1 += r1[item] ** 2;
      norm2 += r2[item] ** 2;
    }

    return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
  }

  // Поиск топ-K похожих пользователей
  findSimilarUsers(userId, k = 10) {
    const similarities = [];

    for (const [otherUserId, _] of this.interactions) {
      if (otherUserId === userId) continue;

      const sim = this.jaccardSimilarity(userId, otherUserId);
      similarities.push({ userId: otherUserId, similarity: sim });
    }

    return similarities
      .sort((a, b) => b.similarity - a.similarity)
      .slice(0, k);
  }

  // Генерация рекомендаций
  recommend(userId, k = 10) {
    const userItems = this.interactions.get(userId) || new Set();
    const similarUsers = this.findSimilarUsers(userId, 50);

    const candidateScores = new Map();

    for (const { userId: simUserId, similarity } of similarUsers) {
      const simUserItems = this.interactions.get(simUserId) || new Set();

      for (const item of simUserItems) {
        if (userItems.has(item)) continue; // Skip already interacted

        const currentScore = candidateScores.get(item) || 0;
        candidateScores.set(item, currentScore + similarity);
      }
    }

    return Array.from(candidateScores.entries())
      .sort((a, b) => b[1] - a[1])
      .slice(0, k)
      .map(([itemId, score]) => ({ itemId, score }));
  }
}
```

**Недостатки:**
- Не масштабируется для миллионов пользователей (O(N²) для вычисления всех пар)
- Разреженность данных (sparsity problem)
- Холодный старт для новых пользователей

#### Item-Based Collaborative Filtering

Идея: находим похожие товары на основе паттернов взаимодействия.

```javascript
class ItemBasedCF {
  constructor(interactions) {
    // interactions: Map<itemId, Set<userId>>
    this.interactions = interactions;
    this.similarityCache = new Map();
  }

  // Предварительное вычисление similarity между товарами
  async buildSimilarityMatrix() {
    const items = Array.from(this.interactions.keys());

    for (let i = 0; i < items.length; i++) {
      for (let j = i + 1; j < items.length; j++) {
        const itemA = items[i];
        const itemB = items[j];

        const usersA = this.interactions.get(itemA) || new Set();
        const usersB = this.interactions.get(itemB) || new Set();

        const intersection = new Set([...usersA].filter(x => usersB.has(x)));
        const union = new Set([...usersA, ...usersB]);

        const similarity = intersection.size / union.size;

        this.similarityCache.set(`${itemA}:${itemB}`, similarity);
        this.similarityCache.set(`${itemB}:${itemA}`, similarity);
      }
    }
  }

  // Поиск похожих товаров
  findSimilarItems(itemId, k = 10) {
    const similarities = [];

    for (const [key, similarity] of this.similarityCache) {
      const [item1, item2] = key.split(':');
      if (item1 === itemId) {
        similarities.push({ itemId: item2, similarity });
      }
    }

    return similarities
      .sort((a, b) => b.similarity - a.similarity)
      .slice(0, k);
  }

  // Рекомендации для пользователя
  recommend(userId, userInteractions, k = 10) {
    const candidateScores = new Map();

    for (const interactedItem of userInteractions) {
      const similarItems = this.findSimilarItems(interactedItem, 20);

      for (const { itemId, similarity } of similarItems) {
        if (userInteractions.has(itemId)) continue;

        const currentScore = candidateScores.get(itemId) || 0;
        candidateScores.set(itemId, currentScore + similarity);
      }
    }

    return Array.from(candidateScores.entries())
      .sort((a, b) => b[1] - a[1])
      .slice(0, k)
      .map(([itemId, score]) => ({ itemId, score }));
  }
}
```

**Преимущества:**
- Лучше масштабируется (товаров обычно меньше, чем пользователей)
- Similarity можно предвычислить offline
- Работает лучше при разреженных данных

### 2. Matrix Factorization

Идея: представить матрицу user-item как произведение двух матриц меньшей размерности (embeddings).

```javascript
function randn(mean = 0, std = 1) {
  let u = 0;
  let v = 0;
  while (u === 0) u = Math.random();
  while (v === 0) v = Math.random();
  const num = Math.sqrt(-2 * Math.log(u)) * Math.cos(2 * Math.PI * v);
  return num * std + mean;
}

function shuffle(array) {
  for (let i = array.length - 1; i > 0; i -= 1) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
}

class MatrixFactorization {
  constructor(nUsers, nItems, { nFactors = 50, learningRate = 0.01, reg = 0.02 } = {}) {
    this.nFactors = nFactors;
    this.learningRate = learningRate;
    this.reg = reg;

    this.userFactors = Array.from({ length: nUsers }, () =>
      Array.from({ length: nFactors }, () => randn(0, 0.1)),
    );
    this.itemFactors = Array.from({ length: nItems }, () =>
      Array.from({ length: nFactors }, () => randn(0, 0.1)),
    );
    this.userBias = Array.from({ length: nUsers }, () => 0);
    this.itemBias = Array.from({ length: nItems }, () => 0);
    this.globalBias = 0;
  }

  static dot(a, b) {
    return a.reduce((sum, value, index) => sum + value * b[index], 0);
  }

  predict(userId, itemId) {
    return (
      this.globalBias +
      this.userBias[userId] +
      this.itemBias[itemId] +
      MatrixFactorization.dot(this.userFactors[userId], this.itemFactors[itemId])
    );
  }

  train(interactions, epochs = 20) {
    for (let epoch = 0; epoch < epochs; epoch += 1) {
      shuffle(interactions);
      let totalLoss = 0;

      for (const [userId, itemId, rating] of interactions) {
        const prediction = this.predict(userId, itemId);
        const error = rating - prediction;

        this.userBias[userId] += this.learningRate * (error - this.reg * this.userBias[userId]);
        this.itemBias[itemId] += this.learningRate * (error - this.reg * this.itemBias[itemId]);

        const userVector = this.userFactors[userId];
        const itemVector = this.itemFactors[itemId];
        const userVectorOld = [...userVector];

        for (let k = 0; k < this.nFactors; k += 1) {
          userVector[k] += this.learningRate * (error * itemVector[k] - this.reg * userVector[k]);
          itemVector[k] += this.learningRate * (error * userVectorOld[k] - this.reg * itemVector[k]);
        }

        totalLoss += error ** 2;
      }

      const rmse = Math.sqrt(totalLoss / interactions.length);
      console.log(`Epoch ${epoch + 1}/${epochs}, RMSE: ${rmse.toFixed(4)}`);
    }
  }

  recommend(userId, limit = 10, excludeItems = new Set()) {
    const scores = [];
    for (let itemId = 0; itemId < this.itemFactors.length; itemId += 1) {
      if (excludeItems.has(itemId)) {
        continue;
      }
      scores.push({ itemId, score: this.predict(userId, itemId) });
    }

    return scores
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);
  }
}
```

**Использование в production:**

```javascript
// Node.js сервис с предобученными embeddings
class EmbeddingService {
  constructor(redis) {
    this.redis = redis;
  }

  async getUserEmbedding(userId) {
    const cached = await this.redis.get(`user_emb:${userId}`);
    if (cached) {
      return JSON.parse(cached);
    }

    // Загрузка из Feature Store
    const embedding = await this.fetchFromFeatureStore(userId);
    await this.redis.setex(`user_emb:${userId}`, 3600, JSON.stringify(embedding));

    return embedding;
  }

  async getItemEmbedding(itemId) {
    const cached = await this.redis.get(`item_emb:${itemId}`);
    if (cached) {
      return JSON.parse(cached);
    }

    const embedding = await this.fetchItemEmbedding(itemId);
    await this.redis.setex(`item_emb:${itemId}`, 7200, JSON.stringify(embedding));

    return embedding;
  }

  dotProduct(vec1, vec2) {
    return vec1.reduce((sum, val, i) => sum + val * vec2[i], 0);
  }

  async computeSimilarity(userId, itemId) {
    const [userEmb, itemEmb] = await Promise.all([
      this.getUserEmbedding(userId),
      this.getItemEmbedding(itemId)
    ]);

    return this.dotProduct(userEmb, itemEmb);
  }
}
```

### 3. Content-Based Filtering

Идея: рекомендовать товары похожие на те, что пользователь уже оценил, на основе атрибутов.

```javascript
class ContentBasedRecommender {
  constructor(itemFeatures) {
    // itemFeatures: Map<itemId, { category, tags, price, ... }>
    this.itemFeatures = itemFeatures;
  }

  // TF-IDF для текстовых признаков
  computeTFIDF(items) {
    const termFrequency = new Map();
    const documentFrequency = new Map();
    const totalDocs = items.length;

    // Подсчёт TF и DF
    for (const item of items) {
      const terms = new Set(item.tags || []);

      for (const term of terms) {
        documentFrequency.set(term, (documentFrequency.get(term) || 0) + 1);
      }
    }

    const tfidfVectors = new Map();

    for (const item of items) {
      const vector = {};
      const terms = item.tags || [];
      const termCounts = {};

      for (const term of terms) {
        termCounts[term] = (termCounts[term] || 0) + 1;
      }

      for (const [term, count] of Object.entries(termCounts)) {
        const tf = count / terms.length;
        const idf = Math.log(totalDocs / (documentFrequency.get(term) || 1));
        vector[term] = tf * idf;
      }

      tfidfVectors.set(item.id, vector);
    }

    return tfidfVectors;
  }

  // Cosine similarity между векторами признаков
  cosineSimilarity(vec1, vec2) {
    const allTerms = new Set([...Object.keys(vec1), ...Object.keys(vec2)]);

    let dotProduct = 0;
    let norm1 = 0;
    let norm2 = 0;

    for (const term of allTerms) {
      const v1 = vec1[term] || 0;
      const v2 = vec2[term] || 0;

      dotProduct += v1 * v2;
      norm1 += v1 ** 2;
      norm2 += v2 ** 2;
    }

    if (norm1 === 0 || norm2 === 0) return 0;

    return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
  }

  // Рекомендации на основе профиля пользователя
  recommend(userProfile, k = 10) {
    const items = Array.from(this.itemFeatures.values());
    const tfidfVectors = this.computeTFIDF(items);

    const scores = [];

    for (const item of items) {
      if (userProfile.interactedItems.has(item.id)) continue;

      const itemVector = tfidfVectors.get(item.id);
      const similarity = this.cosineSimilarity(userProfile.featureVector, itemVector);

      scores.push({ itemId: item.id, score: similarity });
    }

    return scores
      .sort((a, b) => b.score - a.score)
      .slice(0, k);
  }
}
```

### 4. Two-Tower Neural Network (Deep Learning подход)

Современный подход: обучить две нейросети — одну для user embeddings, другую для item embeddings.

```javascript
const tf = require('@tensorflow/tfjs-node');

class TwoTowerModel {
  constructor(userVocabSize, itemVocabSize, embeddingDim = 128) {
    // User tower
    const userInput = tf.input({ shape: [], dtype: 'int32', name: 'user_id' });
    let userVector = tf.layers.embedding({ inputDim: userVocabSize, outputDim: 64 })(userInput);
    userVector = tf.layers.dense({ units: 128, activation: 'relu' })(userVector);
    userVector = tf.layers.dense({ units: embeddingDim })(userVector);
    userVector = tf.layers.lambda(({ x }) => tf.linalg.l2Normalize(x, -1))({ x: userVector });

    // Item tower
    const itemInput = tf.input({ shape: [], dtype: 'int32', name: 'item_id' });
    let itemVector = tf.layers.embedding({ inputDim: itemVocabSize, outputDim: 64 })(itemInput);
    itemVector = tf.layers.dense({ units: 128, activation: 'relu' })(itemVector);
    itemVector = tf.layers.dense({ units: embeddingDim })(itemVector);
    itemVector = tf.layers.lambda(({ x }) => tf.linalg.l2Normalize(x, -1))({ x: itemVector });

    const dotProduct = tf.layers.dot({ axes: -1 })([userVector, itemVector]);

    this.model = tf.model({ inputs: [userInput, itemInput], outputs: dotProduct });
    this.model.compile({
      optimizer: tf.train.adam(),
      loss: tf.losses.sigmoidCrossEntropy,
      metrics: ['accuracy'],
    });
  }

  async train(dataset, epochs = 10, validationData) {
    return this.model.fitDataset(dataset, { epochs, validationData });
  }
}

// Training data: (user_id, item_id, label)
// label = 1 if interaction, 0 if negative sample
const model = new TwoTowerModel(100_000, 50_000);
await model.train(trainDataset, 10, valDataset);
```

---

## Ranking Service

После генерации кандидатов нужно их ранжировать с учётом множества факторов.

```javascript
class RankingService {
  constructor(db, featureStore) {
    this.db = db;
    this.featureStore = featureStore;
  }

  async extractFeatures(userId, itemId, context) {
    const [userFeatures, itemFeatures, contextFeatures] = await Promise.all([
      this.featureStore.getUserFeatures(userId),
      this.featureStore.getItemFeatures(itemId),
      this.extractContextFeatures(context)
    ]);

    return {
      // User features
      user_age: userFeatures.age,
      user_gender: userFeatures.gender,
      user_avg_price: userFeatures.avgPurchasePrice,
      user_category_affinity: userFeatures.categoryAffinity,

      // Item features
      item_price: itemFeatures.price,
      item_category: itemFeatures.category,
      item_popularity: itemFeatures.viewCount,
      item_rating: itemFeatures.averageRating,
      item_recency: Date.now() - itemFeatures.publishedAt,

      // Context features
      hour_of_day: context.hour,
      day_of_week: context.dayOfWeek,
      device_type: context.deviceType,

      // Cross features
      price_affinity: Math.abs(userFeatures.avgPurchasePrice - itemFeatures.price),
      category_match: userFeatures.categoryAffinity[itemFeatures.category] || 0
    };
  }

  async rankCandidates(userId, candidates, context) {
    const rankedItems = [];

    for (const candidate of candidates) {
      const features = await this.extractFeatures(userId, candidate.itemId, context);

      // Простая линейная модель (в production — ML модель)
      const score = this.computeScore(features, candidate.score);

      rankedItems.push({
        itemId: candidate.itemId,
        score,
        features // для debugging/logging
      });
    }

    return rankedItems
      .sort((a, b) => b.score - a.score)
      .map(item => ({
        itemId: item.itemId,
        score: item.score,
        reason: this.explainRecommendation(item.features)
      }));
  }

  computeScore(features, baseScore) {
    // В production используется gradient boosting или нейросеть
    const weights = {
      base: 1.0,
      item_popularity: 0.2,
      item_rating: 0.3,
      category_match: 0.5,
      price_affinity: -0.1,
      item_recency: 0.15
    };

    return (
      baseScore * weights.base +
      Math.log(features.item_popularity + 1) * weights.item_popularity +
      features.item_rating * weights.item_rating +
      features.category_match * weights.category_match +
      features.price_affinity * weights.price_affinity +
      Math.log(features.item_recency + 1) * weights.item_recency
    );
  }

  explainRecommendation(features) {
    if (features.category_match > 0.7) {
      return 'Based on your interest in this category';
    }
    if (features.item_popularity > 1000) {
      return 'Trending now';
    }
    if (features.item_rating > 4.5) {
      return 'Highly rated';
    }
    return 'Recommended for you';
  }
}
```

---

## Полная имплементация Recommendation API

```javascript
const express = require('express');
const Redis = require('ioredis');
const { Pool } = require('pg');

class RecommendationService {
  constructor(redis, db, embeddingService, rankingService) {
    this.redis = redis;
    this.db = db;
    this.embeddingService = embeddingService;
    this.rankingService = rankingService;
  }

  async getPersonalizedRecommendations(userId, options = {}) {
    const {
      limit = 20,
      excludeItems = [],
      context = {}
    } = options;

    // 1. Проверка кэша
    const cacheKey = `reco:${userId}:${JSON.stringify(context)}`;
    const cached = await this.redis.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // 2. Получение user interaction history
    const userHistory = await this.getUserHistory(userId);

    // 3. Candidate Generation из нескольких источников
    const [
      collaborativeCandidates,
      contentCandidates,
      trendingItems,
      newItems
    ] = await Promise.all([
      this.getCollaborativeFilteringCandidates(userId, userHistory, 100),
      this.getContentBasedCandidates(userId, userHistory, 100),
      this.getTrendingItems(50),
      this.getNewItems(20)
    ]);

    // 4. Объединение кандидатов
    const allCandidates = this.mergeCandidates([
      collaborativeCandidates,
      contentCandidates,
      trendingItems,
      newItems
    ], excludeItems.concat(userHistory));

    // 5. Ranking
    const rankedItems = await this.rankingService.rankCandidates(
      userId,
      allCandidates,
      context
    );

    // 6. Diversification (разнообразие по категориям)
    const diversifiedItems = this.diversify(rankedItems, limit);

    // 7. Кэширование на 1 час
    await this.redis.setex(cacheKey, 3600, JSON.stringify(diversifiedItems));

    return diversifiedItems;
  }

  async getUserHistory(userId) {
    const result = await this.db.query(
      `SELECT item_id, interaction_type, created_at
       FROM user_interactions
       WHERE user_id = $1
       ORDER BY created_at DESC
       LIMIT 100`,
      [userId]
    );

    return result.rows.map(r => r.item_id);
  }

  async getCollaborativeFilteringCandidates(userId, userHistory, limit) {
    // Используем precomputed item-to-item similarity
    const candidates = new Map();

    for (const itemId of userHistory.slice(0, 10)) { // Последние 10
      const similarItems = await this.redis.zrevrange(
        `similar:${itemId}`,
        0,
        19,
        'WITHSCORES'
      );

      for (let i = 0; i < similarItems.length; i += 2) {
        const candidateId = similarItems[i];
        const score = parseFloat(similarItems[i + 1]);

        candidates.set(candidateId, (candidates.get(candidateId) || 0) + score);
      }
    }

    return Array.from(candidates.entries())
      .sort((a, b) => b[1] - a[1])
      .slice(0, limit)
      .map(([itemId, score]) => ({ itemId, score, source: 'collaborative' }));
  }

  async getContentBasedCandidates(userId, userHistory, limit) {
    // Используем user embedding × item embeddings
    const userEmbedding = await this.embeddingService.getUserEmbedding(userId);

    // ANN search (Approximate Nearest Neighbor) в векторной БД
    // Здесь упрощённо — в production используется Faiss, Annoy, или Milvus
    const candidates = await this.redis.zrevrange(
      `user_reco:${userId}`,
      0,
      limit - 1,
      'WITHSCORES'
    );

    return candidates
      .filter((_, i) => i % 2 === 0)
      .map((itemId, idx) => ({
        itemId,
        score: parseFloat(candidates[idx * 2 + 1]),
        source: 'content'
      }));
  }

  async getTrendingItems(limit) {
    const trending = await this.redis.zrevrange('trending:24h', 0, limit - 1, 'WITHSCORES');

    return trending
      .filter((_, i) => i % 2 === 0)
      .map((itemId, idx) => ({
        itemId,
        score: parseFloat(trending[idx * 2 + 1]) * 0.5, // Lower weight
        source: 'trending'
      }));
  }

  async getNewItems(limit) {
    const result = await this.db.query(
      `SELECT id FROM items
       WHERE created_at > NOW() - INTERVAL '7 days'
       ORDER BY created_at DESC
       LIMIT $1`,
      [limit]
    );

    return result.rows.map(r => ({
      itemId: r.id,
      score: 0.3, // Low base score
      source: 'new'
    }));
  }

  mergeCandidates(candidateSets, excludeItems) {
    const merged = new Map();
    const excludeSet = new Set(excludeItems);

    for (const candidates of candidateSets) {
      for (const { itemId, score, source } of candidates) {
        if (excludeSet.has(itemId)) continue;

        const existing = merged.get(itemId);
        if (!existing || existing.score < score) {
          merged.set(itemId, { itemId, score, source });
        }
      }
    }

    return Array.from(merged.values());
  }

  diversify(items, limit) {
    // MMR (Maximal Marginal Relevance) для разнообразия
    const selected = [];
    const remaining = [...items];
    const categories = new Map();

    while (selected.length < limit && remaining.length > 0) {
      let bestIdx = 0;
      let bestScore = -Infinity;

      for (let i = 0; i < remaining.length; i++) {
        const item = remaining[i];
        const categoryCount = categories.get(item.category) || 0;

        // Penalty за повторение категории
        const diversityPenalty = categoryCount * 0.2;
        const adjustedScore = item.score - diversityPenalty;

        if (adjustedScore > bestScore) {
          bestScore = adjustedScore;
          bestIdx = i;
        }
      }

      const selected_item = remaining.splice(bestIdx, 1)[0];
      selected.push(selected_item);
      categories.set(selected_item.category, (categories.get(selected_item.category) || 0) + 1);
    }

    return selected;
  }

  // Решение проблемы холодного старта
  async getColdStartRecommendations(userId) {
    // Для нового пользователя без истории
    const [trending, popular, diverse] = await Promise.all([
      this.getTrendingItems(10),
      this.getPopularByCategory(5), // Топ по каждой категории
      this.getDiverseItems(10)
    ]);

    return this.mergeCandidates([trending, popular, diverse], []);
  }

  async getPopularByCategory(limitPerCategory) {
    // Получаем топ товары из каждой категории
    const result = await this.db.query(`
      WITH ranked AS (
        SELECT id, category,
               ROW_NUMBER() OVER (PARTITION BY category ORDER BY popularity DESC) as rn
        FROM items
      )
      SELECT id, category FROM ranked WHERE rn <= $1
    `, [limitPerCategory]);

    return result.rows.map(r => ({
      itemId: r.id,
      score: 0.6,
      source: 'popular'
    }));
  }

  async getDiverseItems(limit) {
    // Случайные товары из разных категорий
    const result = await this.db.query(`
      SELECT DISTINCT ON (category) id, category
      FROM items
      ORDER BY category, RANDOM()
      LIMIT $1
    `, [limit]);

    return result.rows.map(r => ({
      itemId: r.id,
      score: 0.4,
      source: 'diverse'
    }));
  }
}

// API Routes
const app = express();
const redis = new Redis();
const db = new Pool({ connectionString: process.env.DATABASE_URL });

const embeddingService = new EmbeddingService(redis);
const rankingService = new RankingService(db, new FeatureStore(redis));
const recoService = new RecommendationService(redis, db, embeddingService, rankingService);

app.get('/api/recommendations/:userId', async (req, res) => {
  try {
    const { userId } = req.params;
    const { limit = 20, exclude } = req.query;

    const context = {
      hour: new Date().getHours(),
      dayOfWeek: new Date().getDay(),
      deviceType: req.headers['user-agent']
    };

    const excludeItems = exclude ? exclude.split(',') : [];

    const recommendations = await recoService.getPersonalizedRecommendations(
      userId,
      { limit: parseInt(limit), excludeItems, context }
    );

    res.json({
      userId,
      recommendations,
      generatedAt: new Date().toISOString()
    });
  } catch (error) {
    console.error('Recommendation error:', error);
    res.status(500).json({ error: 'Failed to generate recommendations' });
  }
});

// Item-to-Item recommendations
app.get('/api/similar/:itemId', async (req, res) => {
  try {
    const { itemId } = req.params;
    const { limit = 10 } = req.query;

    const similarItems = await redis.zrevrange(
      `similar:${itemId}`,
      0,
      limit - 1,
      'WITHSCORES'
    );

    const results = [];
    for (let i = 0; i < similarItems.length; i += 2) {
      results.push({
        itemId: similarItems[i],
        similarity: parseFloat(similarItems[i + 1])
      });
    }

    res.json({ itemId, similar: results });
  } catch (error) {
    res.status(500).json({ error: 'Failed to get similar items' });
  }
});

app.listen(3000, () => console.log('Recommendation API running on port 3000'));
```

---

## Offline Training Pipeline (Spark)

```javascript
const fs = require('fs/promises');
const path = require('path');
const { Redis } = require('ioredis');
const pl = require('@polars/wasm');

class OfflineTrainingPipeline {
  constructor({ dataRoot, redisUrl }) {
    this.dataRoot = dataRoot;
    this.redis = new Redis(redisUrl);
  }

  async loadInteractions(dateFrom, dateTo) {
    const filePath = path.join(this.dataRoot, `dt=${dateFrom}`);
    const frame = await pl.readParquet(`${filePath}/interactions.parquet`);

    return frame
      .filter(pl.col('created_at').isBetween(dateFrom, dateTo))
      .filter(pl.col('interaction_type').isIn(['view', 'purchase', 'add_to_cart']))
      .select([
        pl.col('user_id').cast(pl.Int32).alias('userId'),
        pl.col('item_id').cast(pl.Int32).alias('itemId'),
        pl.col('rating').cast(pl.Float64),
      ])
      .toArray();
  }

  async trainModel(interactions) {
    const userIds = new Set();
    const itemIds = new Set();
    for (const row of interactions) {
      userIds.add(row.userId);
      itemIds.add(row.itemId);
    }

    const model = new MatrixFactorization(userIds.size, itemIds.size, {
      nFactors: 64,
      learningRate: 0.01,
      reg: 0.02,
    });

    model.train(interactions.map((row) => [row.userId, row.itemId, row.rating]), 15);
    return model;
  }

  generateItemSimilarities(model, threshold = 0.5, topK = 20) {
    const similarities = new Map();
    const items = model.itemFactors.length;

    for (let i = 0; i < items; i += 1) {
      const scores = [];
      for (let j = i + 1; j < items; j += 1) {
        const score = MatrixFactorization.dot(model.itemFactors[i], model.itemFactors[j]);
        if (score > threshold) {
          scores.push({ itemId: j, score });
        }
      }

      similarities.set(
        i,
        scores
          .sort((a, b) => b.score - a.score)
          .slice(0, topK),
      );
    }

    return similarities;
  }

  async exportToRedis(similarities) {
    const pipeline = this.redis.pipeline();
    let count = 0;

    for (const [itemId, neighbours] of similarities.entries()) {
      if (!neighbours.length) continue;
      const payload = neighbours.reduce((acc, neighbour) => {
        acc[neighbour.itemId] = neighbour.score;
        return acc;
      }, {});

      pipeline.zadd(`similar:${itemId}`, payload);
      count += neighbours.length;
    }

    await pipeline.exec();
    console.log(`Exported ${count} similarities to Redis`);
  }

  async saveModel(model, outputPath) {
    await fs.mkdir(outputPath, { recursive: true });
    await fs.writeFile(path.join(outputPath, 'userFactors.json'), JSON.stringify(model.userFactors));
    await fs.writeFile(path.join(outputPath, 'itemFactors.json'), JSON.stringify(model.itemFactors));
    await fs.writeFile(path.join(outputPath, 'metadata.json'), JSON.stringify({
      userBias: model.userBias,
      itemBias: model.itemBias,
      globalBias: model.globalBias,
    }));
  }

  async run() {
    const interactions = await this.loadInteractions('2025-09-01', '2025-10-26');
    const model = await this.trainModel(interactions);
    const similarities = this.generateItemSimilarities(model);
    await this.exportToRedis(similarities);
    await this.saveModel(model, path.join('models', 'als-model-2025-10-26'));
  }
}

const pipeline = new OfflineTrainingPipeline({
  dataRoot: 's3://datalake/interactions',
  redisUrl: 'redis://redis.example.com:6379',
});

await pipeline.run();
```

---

## A/B Testing для рекомендаций

```javascript
class ABTestingService {
  constructor(redis) {
    this.redis = redis;
  }

  async assignVariant(userId) {
    // Консистентное распределение пользователей по вариантам
    const hash = this.hashUserId(userId);
    const variant = hash % 100 < 50 ? 'control' : 'treatment';

    await this.redis.hset('ab_test:variant', userId, variant);
    return variant;
  }

  hashUserId(userId) {
    // Simple hash function (в production использовать crypto)
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) - hash) + userId.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  async getRecommendations(userId, recoService) {
    const variant = await this.redis.hget('ab_test:variant', userId) ||
                     await this.assignVariant(userId);

    let recommendations;

    if (variant === 'control') {
      // Baseline алгоритм
      recommendations = await recoService.getCollaborativeFilteringCandidates(userId, [], 20);
    } else {
      // Новый алгоритм (например, с deep learning)
      recommendations = await recoService.getPersonalizedRecommendations(userId, { limit: 20 });
    }

    // Логирование для анализа
    await this.logRecommendation(userId, variant, recommendations);

    return { recommendations, variant };
  }

  async logRecommendation(userId, variant, recommendations) {
    const event = {
      userId,
      variant,
      recommendations: recommendations.map(r => r.itemId),
      timestamp: Date.now()
    };

    await this.redis.lpush('ab_test:events', JSON.stringify(event));
  }

  async trackInteraction(userId, itemId, interactionType) {
    const variant = await this.redis.hget('ab_test:variant', userId);

    const event = {
      userId,
      variant,
      itemId,
      interactionType, // 'click', 'purchase', etc.
      timestamp: Date.now()
    };

    await this.redis.lpush('ab_test:interactions', JSON.stringify(event));
  }

  async computeMetrics() {
    // Анализ метрик для каждого варианта
    const events = await this.redis.lrange('ab_test:interactions', 0, -1);

    const metrics = {
      control: { impressions: 0, clicks: 0, purchases: 0 },
      treatment: { impressions: 0, clicks: 0, purchases: 0 }
    };

    for (const eventStr of events) {
      const event = JSON.parse(eventStr);
      const variant = event.variant;

      if (event.interactionType === 'impression') {
        metrics[variant].impressions++;
      } else if (event.interactionType === 'click') {
        metrics[variant].clicks++;
      } else if (event.interactionType === 'purchase') {
        metrics[variant].purchases++;
      }
    }

    // Вычисление CTR и Conversion Rate
    for (const variant of ['control', 'treatment']) {
      const m = metrics[variant];
      m.ctr = m.impressions > 0 ? m.clicks / m.impressions : 0;
      m.conversionRate = m.clicks > 0 ? m.purchases / m.clicks : 0;
    }

    return metrics;
  }
}
```

---

## Monitoring

```yaml
# prometheus.yml
- job_name: 'recommendation-service'
  static_configs:
    - targets: ['recommendation-api:3000']
```

```javascript
const promClient = require('prom-client');

// Метрики
const recommendationLatency = new promClient.Histogram({
  name: 'recommendation_latency_ms',
  help: 'Latency of recommendation generation',
  labelNames: ['userId', 'variant'],
  buckets: [10, 50, 100, 200, 500, 1000, 2000]
});

const recommendationCacheHitRate = new promClient.Counter({
  name: 'recommendation_cache_hits_total',
  help: 'Number of cache hits',
  labelNames: ['hit']
});

const recommendationDiversity = new promClient.Gauge({
  name: 'recommendation_diversity',
  help: 'Diversity score of recommendations',
  labelNames: ['userId']
});

// Middleware для метрик
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    recommendationLatency.observe({ userId: req.params.userId }, duration);
  });

  next();
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});
```

---

## Best Practices

### 1. Candidate Generation → Ranking (Двухстадийный подход)

- **Stage 1 (Candidate Generation)**: быстро генерируем 100-1000 кандидатов из нескольких источников
- **Stage 2 (Ranking)**: точное ранжирование с помощью сложной ML модели

### 2. Diversification

Не показывайте только самые релевантные товары — добавьте разнообразие:
- MMR (Maximal Marginal Relevance)
- Разные категории
- Serendipity (неожиданные, но интересные рекомендации)

### 3. Exploration vs Exploitation

- **Exploitation**: рекомендуем то, что точно понравится (на основе модели)
- **Exploration**: пробуем новые товары для сбора данных (epsilon-greedy, Thompson Sampling)

### 4. Real-time Personalization

Учитывайте последние действия пользователя в текущей сессии:
```javascript
async getRealTimeRecommendations(userId, sessionItems) {
  const baseRecommendations = await this.getPersonalizedRecommendations(userId);

  // Boost items similar to what user viewed in session
  for (const sessionItem of sessionItems) {
    const similarItems = await this.getSimilarItems(sessionItem);
    // Adjust scores based on recency
  }

  return reranked;
}
```

### 5. Cold Start Solutions

- **Новые пользователи**: trending items, популярные товары, опрос предпочтений при регистрации
- **Новые товары**: content-based filtering на основе атрибутов, boosting новых товаров

### 6. Negative Feedback

Учитывайте негативные сигналы:
- Пользователь скрыл рекомендацию
- Быстро пролистал (dwell time < 1s)
- Не купил после нескольких показов

---

## Trade-offs

| Аспект | Вариант A | Вариант B |
|--------|-----------|-----------|
| **Алгоритм** | Collaborative Filtering | Content-Based |
| Точность | Высокая (если много данных) | Средняя |
| Cold Start | Плохо | Хорошо |
| Serendipity | Высокая | Низкая |
| Scalability | Средняя (O(N²)) | Хорошая |
| **Approach** | Real-time | Batch precomputation |
| Latency | Выше | Ниже |
| Freshness | Высокая | Средняя |
| Cost | Выше (compute) | Ниже |
| **Model** | Simple (CF, CB) | Deep Learning |
| Accuracy | Средняя | Высокая |
| Latency | Низкая | Средняя |
| Training Cost | Низкая | Высокая |
| Interpretability | Высокая | Низкая |

---

## Что почитать дальше?

1. **Урок 44: Search Engine** — BM25 ranking, Elasticsearch
2. **Урок 46: Newsfeed** — персонализированная лента
3. **Урок 53: Monitoring и Observability** — мониторинг ML моделей

---

## Проверь себя

1. В чём разница между User-Based и Item-Based Collaborative Filtering?
2. Что такое Matrix Factorization и как она решает проблему разреженности?
3. Как решать проблему холодного старта для новых пользователей и товаров?
4. Что такое двухстадийный подход (Candidate Generation + Ranking)?
5. Как обеспечить diversification рекомендаций?
6. Зачем нужен Exploration в рекомендательных системах?
7. Как A/B тестировать новые алгоритмы рекомендаций?
8. Какие метрики использовать для оценки качества рекомендаций (CTR, Conversion, Diversity)?

---
**Предыдущий урок**: [Урок 51: E-commerce Platform](51-ecommerce.md)
**Следующий урок**: [Урок 53: Monitoring и Observability](53-monitoring.md)
