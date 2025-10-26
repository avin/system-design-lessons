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

```python
import numpy as np

class MatrixFactorization:
    def __init__(self, n_users, n_items, n_factors=50, learning_rate=0.01, reg=0.02):
        self.n_factors = n_factors
        self.lr = learning_rate
        self.reg = reg

        # Инициализация embeddings
        self.user_factors = np.random.normal(0, 0.1, (n_users, n_factors))
        self.item_factors = np.random.normal(0, 0.1, (n_items, n_factors))
        self.user_bias = np.zeros(n_users)
        self.item_bias = np.zeros(n_items)
        self.global_bias = 0

    def predict(self, user_id, item_id):
        """Предсказание рейтинга"""
        prediction = (
            self.global_bias +
            self.user_bias[user_id] +
            self.item_bias[item_id] +
            np.dot(self.user_factors[user_id], self.item_factors[item_id])
        )
        return prediction

    def train(self, interactions, n_epochs=20):
        """
        interactions: list of (user_id, item_id, rating)
        """
        for epoch in range(n_epochs):
            np.random.shuffle(interactions)
            total_loss = 0

            for user_id, item_id, rating in interactions:
                # Предсказание
                pred = self.predict(user_id, item_id)
                error = rating - pred

                # Градиентный спуск
                self.user_bias[user_id] += self.lr * (error - self.reg * self.user_bias[user_id])
                self.item_bias[item_id] += self.lr * (error - self.reg * self.item_bias[item_id])

                user_factor_old = self.user_factors[user_id].copy()

                self.user_factors[user_id] += self.lr * (
                    error * self.item_factors[item_id] -
                    self.reg * self.user_factors[user_id]
                )

                self.item_factors[item_id] += self.lr * (
                    error * user_factor_old -
                    self.reg * self.item_factors[item_id]
                )

                total_loss += error ** 2

            rmse = np.sqrt(total_loss / len(interactions))
            print(f"Epoch {epoch + 1}/{n_epochs}, RMSE: {rmse:.4f}")

    def recommend(self, user_id, n=10, exclude_items=None):
        """Генерация рекомендаций для пользователя"""
        if exclude_items is None:
            exclude_items = set()

        scores = []
        for item_id in range(len(self.item_factors)):
            if item_id in exclude_items:
                continue

            score = self.predict(user_id, item_id)
            scores.append((item_id, score))

        scores.sort(key=lambda x: x[1], reverse=True)
        return scores[:n]
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

```python
import tensorflow as tf

class TwoTowerModel(tf.keras.Model):
    def __init__(self, user_vocab_size, item_vocab_size, embedding_dim=128):
        super().__init__()

        # User Tower
        self.user_embedding = tf.keras.layers.Embedding(user_vocab_size, 64)
        self.user_dense1 = tf.keras.layers.Dense(128, activation='relu')
        self.user_dense2 = tf.keras.layers.Dense(embedding_dim)

        # Item Tower
        self.item_embedding = tf.keras.layers.Embedding(item_vocab_size, 64)
        self.item_dense1 = tf.keras.layers.Dense(128, activation='relu')
        self.item_dense2 = tf.keras.layers.Dense(embedding_dim)

    def call(self, inputs):
        user_id = inputs['user_id']
        item_id = inputs['item_id']

        # User Tower
        user_emb = self.user_embedding(user_id)
        user_emb = self.user_dense1(user_emb)
        user_vector = self.user_dense2(user_emb)
        user_vector = tf.nn.l2_normalize(user_vector, axis=1)

        # Item Tower
        item_emb = self.item_embedding(item_id)
        item_emb = self.item_dense1(item_emb)
        item_vector = self.item_dense2(item_emb)
        item_vector = tf.nn.l2_normalize(item_vector, axis=1)

        # Dot product
        logits = tf.reduce_sum(user_vector * item_vector, axis=1)

        return logits

# Обучение
model = TwoTowerModel(user_vocab_size=100000, item_vocab_size=50000)
model.compile(
    optimizer='adam',
    loss=tf.keras.losses.BinaryCrossentropy(from_logits=True),
    metrics=['accuracy']
)

# Training data: (user_id, item_id, label)
# label = 1 if interaction, 0 if negative sample
model.fit(train_dataset, epochs=10, validation_data=val_dataset)
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

```python
from pyspark.sql import SparkSession
from pyspark.ml.recommendation import ALS
from pyspark.ml.evaluation import RegressionEvaluator

class OfflineTrainingPipeline:
    def __init__(self):
        self.spark = SparkSession.builder \
            .appName("RecommendationTraining") \
            .getOrCreate()

    def load_interactions(self, date_from, date_to):
        """Загрузка interactions из data lake"""
        df = self.spark.read.parquet(f"s3://datalake/interactions/dt={date_from}/*")

        # Фильтрация и преобразование
        interactions = df.filter(
            (df.interaction_type.isin(['view', 'purchase', 'add_to_cart'])) &
            (df.created_at >= date_from) &
            (df.created_at < date_to)
        ).select(
            df.user_id.cast('integer').alias('userId'),
            df.item_id.cast('integer').alias('itemId'),
            df.rating.cast('float')  # implicit feedback → rating
        )

        return interactions

    def train_als_model(self, interactions):
        """Обучение ALS (Alternating Least Squares)"""
        train, test = interactions.randomSplit([0.8, 0.2], seed=42)

        als = ALS(
            maxIter=10,
            regParam=0.1,
            userCol='userId',
            itemCol='itemId',
            ratingCol='rating',
            coldStartStrategy='drop',
            nonnegative=True,
            implicitPrefs=True  # implicit feedback
        )

        model = als.fit(train)

        # Evaluation
        predictions = model.transform(test)
        evaluator = RegressionEvaluator(
            metricName='rmse',
            labelCol='rating',
            predictionCol='prediction'
        )
        rmse = evaluator.evaluate(predictions)
        print(f"RMSE on test set: {rmse}")

        return model

    def generate_item_similarities(self, model):
        """Генерация item-to-item similarities"""
        item_factors = model.itemFactors

        # Cross join для вычисления всех пар (оптимизировать для production)
        from pyspark.sql.functions import col, udf
        from pyspark.sql.types import FloatType
        import numpy as np

        def cosine_sim(vec1, vec2):
            return float(np.dot(vec1, vec2) / (np.linalg.norm(vec1) * np.linalg.norm(vec2)))

        cosine_udf = udf(cosine_sim, FloatType())

        similarities = item_factors.alias('a').join(
            item_factors.alias('b'),
            col('a.id') < col('b.id')
        ).select(
            col('a.id').alias('item1'),
            col('b.id').alias('item2'),
            cosine_udf(col('a.features'), col('b.features')).alias('similarity')
        ).filter(col('similarity') > 0.5)  # Только значимые

        return similarities

    def export_to_redis(self, similarities):
        """Экспорт similarities в Redis"""
        # Группировка топ-20 для каждого товара
        from pyspark.sql.window import Window
        from pyspark.sql.functions import row_number

        window = Window.partitionBy('item1').orderBy(col('similarity').desc())

        top_similar = similarities.withColumn('rank', row_number().over(window)) \
            .filter(col('rank') <= 20) \
            .collect()

        # Bulk insert в Redis
        import redis
        r = redis.Redis(host='redis.example.com')

        pipe = r.pipeline()
        for row in top_similar:
            pipe.zadd(f"similar:{row.item1}", {row.item2: row.similarity})
        pipe.execute()

        print(f"Exported {len(top_similar)} similarities to Redis")

    def run_pipeline(self):
        # 1. Load data
        interactions = self.load_interactions('2025-09-01', '2025-10-26')

        # 2. Train model
        model = self.train_als_model(interactions)

        # 3. Generate similarities
        similarities = self.generate_item_similarities(model)

        # 4. Export to Redis
        self.export_to_redis(similarities)

        # 5. Save model
        model.save("s3://models/als-model-2025-10-26")

        print("Pipeline completed successfully")

# Запуск
pipeline = OfflineTrainingPipeline()
pipeline.run_pipeline()
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

[← Урок 51: E-commerce Platform](./51-ecommerce.md) | [Урок 53: Monitoring и Observability →](./53-monitoring.md)
