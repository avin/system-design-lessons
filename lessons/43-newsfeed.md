# Урок 43: Newsfeed System

## Введение

Newsfeed (лента новостей) — это центральный компонент социальных сетей, который агрегирует и персонализирует контент от друзей, подписок и рекламодателей. Это одна из самых сложных систем для проектирования из-за требований к персонализации, масштабу и real-time обновлениям.

В этом уроке мы спроектируем newsfeed систему как в Facebook, Instagram или Twitter, способную обслуживать миллионы пользователей с высокой персонализацией.

**Примеры:**
- **Facebook**: News Feed с постами друзей
- **Instagram**: Feed с фото/видео
- **Twitter**: Timeline с твитами
- **LinkedIn**: Professional feed с обновлениями сети

## Требования к системе

### Функциональные требования

1. **Публикация контента**:
   - Пользователь создаёт пост (текст, фото, видео)
   - Пост доставляется в ленты подписчиков

2. **Просмотр ленты**:
   - Получение персонализированной ленты
   - Pagination (бесконечная прокрутка)
   - Real-time updates (новые посты появляются)

3. **Ranking & Filtering**:
   - Сортировка по релевантности (не просто хронология)
   - Фильтрация спама и неподходящего контента

### Нефункциональные требования

1. **Масштабируемость**:
   - 1B users
   - 500M DAU (Daily Active Users)
   - 200 posts/sec (новые посты)
   - 100K feed reads/sec

2. **Latency**:
   - Feed generation: < 500ms
   - Post publication: < 2 sec для доставки в ленты

3. **Availability**: 99.99%

4. **Consistency**: Eventual consistency OK (допустима задержка доставки постов)

### Back-of-the-envelope

```
Users: 1B total, 500M DAU
Average friends per user: 200
Posts per user per day: 1

New posts per day: 500M × 1 = 500M posts/day ≈ 5,800 posts/sec
Feed reads: 500M users × 10 reads/day = 5B reads/day ≈ 58K reads/sec

Storage:
- Post metadata: 500M posts/day × 1KB = 500 GB/day
- Media (photos/videos): 50% have media, avg 2MB
  250M × 2MB = 500 TB/day
- 1 year: 500 TB × 365 = 182 PB

Fanout:
- При публикации поста → delivery в ленты 200 friends
- 500M posts/day × 200 = 100B fanout operations/day
```

## Архитектура высокого уровня

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     ↓
┌─────────────────┐
│   API Gateway   │
│  (Load Balancer)│
└────┬─────┬──────┘
     │     │
     ↓     ↓
┌─────────┐ ┌──────────────┐
│  Post   │ │    Feed      │
│ Service │ │   Service    │
└────┬────┘ └──────┬───────┘
     │             │
     │             ↓
     │      ┌──────────────┐
     │      │  Feed Cache  │
     │      │   (Redis)    │
     │      └──────────────┘
     │             │
     ↓             ↓
┌─────────────────────────┐
│      Post Store         │
│  (Posts, Users, Graphs) │
└─────────────────────────┘
     ↑
     │
┌────┴─────────┐
│ Fanout Worker│
│  (Queue)     │
└──────────────┘
```

## Ключевые подходы к генерации ленты

### 1. Fanout on Write (Push Model)

При создании поста **сразу** записываем его в ленты всех подписчиков.

```
User A создаёт пост
    ↓
Post Service
    ↓
Fanout Worker
    ↓
┌─────┬─────┬─────┬─────┐
│Feed │Feed │Feed │ ... │  ← ленты всех friends
│  B  │  C  │  D  │     │
└─────┴─────┴─────┴─────┘

User B открывает ленту:
    → просто читает из своего Feed (уже готов!)
```

**Плюсы:**
- ✅ Быстрое чтение ленты (просто SELECT)
- ✅ Работает для read-heavy системы

**Минусы:**
- ❌ Медленная публикация поста (нужно записать в N лент)
- ❌ Не работает для celebrities (миллионы подписчиков)

### 2. Fanout on Read (Pull Model)

При открытии ленты **агрегируем** посты от всех друзей.

```
User B открывает ленту
    ↓
Feed Service
    ↓
Собирает посты от:
- Friend A
- Friend C
- Friend D
- ...
    ↓
Merge & Sort
    ↓
Return Feed
```

**Плюсы:**
- ✅ Быстрая публикация (просто INSERT post)
- ✅ Работает для celebrities

**Минусы:**
- ❌ Медленное чтение (нужно агрегировать N друзей)
- ❌ Повторные вычисления

### 3. Hybrid Approach (Рекомендуется)

Комбинация обоих подходов:

- **Fanout on Write** для обычных пользователей (< 10K followers)
- **Fanout on Read** для celebrities (> 10K followers)

```
Regular user → Fanout on Write
Celebrity → Fanout on Read (при запросе ленты)

User feed =
  [Pre-computed feed from Fanout on Write]
  + [Real-time fetch from Celebrity posts]
```

## Реализация компонентов

### 1. Post Service

```javascript
const express = require('express');
const app = express();

class PostService {
  constructor() {
    this.db = require('./database');
    this.queue = require('./queue'); // Kafka/RabbitMQ
    this.mediaService = require('./media-service');

    this.setupRoutes();
  }

  setupRoutes() {
    app.use(express.json());

    // Создание поста
    app.post('/posts', async (req, res) => {
      try {
        const { userId, content, mediaUrls, visibility } = req.body;

        const postId = await this.createPost({
          userId,
          content,
          mediaUrls,
          visibility: visibility || 'friends'
        });

        res.json({
          success: true,
          postId
        });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Получение поста
    app.get('/posts/:postId', async (req, res) => {
      try {
        const { postId } = req.params;

        const post = await this.getPost(postId);

        res.json(post);
      } catch (error) {
        res.status(404).json({ error: 'Post not found' });
      }
    });

    // Удаление поста
    app.delete('/posts/:postId', async (req, res) => {
      try {
        const { postId } = req.params;
        const { userId } = req.body;

        await this.deletePost(postId, userId);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Лайк
    app.post('/posts/:postId/like', async (req, res) => {
      try {
        const { postId } = req.params;
        const { userId } = req.body;

        await this.likePost(postId, userId);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
  }

  async createPost({ userId, content, mediaUrls, visibility }) {
    const postId = this.generateId();

    // Загружаем media (если есть)
    const processedMedia = [];

    for (const mediaUrl of mediaUrls || []) {
      const processed = await this.mediaService.process(mediaUrl);
      processedMedia.push(processed);
    }

    // Сохраняем пост в БД
    await this.db.query(
      `INSERT INTO posts
       (id, user_id, content, media, visibility, created_at)
       VALUES ($1, $2, $3, $4, $5, NOW())`,
      [postId, userId, content, JSON.stringify(processedMedia), visibility]
    );

    // Получаем список подписчиков для fanout
    const followers = await this.getFollowers(userId);

    // Проверяем: celebrity или нет
    const isCelebrity = followers.length > 10000;

    if (isCelebrity) {
      // Celebrity: помечаем пост, fanout не делаем
      await this.db.query(
        'UPDATE posts SET is_celebrity = true WHERE id = $1',
        [postId]
      );
    } else {
      // Regular user: запускаем fanout
      await this.queue.publish('post.fanout', {
        postId,
        userId,
        followers
      });
    }

    return postId;
  }

  async getPost(postId) {
    const result = await this.db.query(
      `SELECT p.*, u.username, u.avatar_url,
              (SELECT COUNT(*) FROM post_likes WHERE post_id = p.id) as likes_count,
              (SELECT COUNT(*) FROM comments WHERE post_id = p.id) as comments_count
       FROM posts p
       JOIN users u ON p.user_id = u.id
       WHERE p.id = $1`,
      [postId]
    );

    if (result.rows.length === 0) {
      throw new Error('Post not found');
    }

    return this.formatPost(result.rows[0]);
  }

  async deletePost(postId, userId) {
    // Проверяем ownership
    const result = await this.db.query(
      'SELECT user_id FROM posts WHERE id = $1',
      [postId]
    );

    if (result.rows.length === 0 || result.rows[0].user_id !== userId) {
      throw new Error('Unauthorized');
    }

    // Удаляем пост
    await this.db.query('DELETE FROM posts WHERE id = $1', [postId]);

    // Удаляем из лент (асинхронно)
    await this.queue.publish('post.delete', { postId });
  }

  async likePost(postId, userId) {
    await this.db.query(
      `INSERT INTO post_likes (post_id, user_id, created_at)
       VALUES ($1, $2, NOW())
       ON CONFLICT (post_id, user_id) DO NOTHING`,
      [postId, userId]
    );

    // Инкрементируем счётчик
    await this.db.query(
      'UPDATE posts SET likes_count = likes_count + 1 WHERE id = $1',
      [postId]
    );
  }

  async getFollowers(userId) {
    const result = await this.db.query(
      `SELECT follower_id
       FROM follows
       WHERE following_id = $1`,
      [userId]
    );

    return result.rows.map(r => r.follower_id);
  }

  formatPost(row) {
    return {
      id: row.id,
      userId: row.user_id,
      username: row.username,
      avatarUrl: row.avatar_url,
      content: row.content,
      media: JSON.parse(row.media || '[]'),
      likesCount: parseInt(row.likes_count),
      commentsCount: parseInt(row.comments_count),
      createdAt: row.created_at
    };
  }

  generateId() {
    return require('crypto').randomUUID();
  }
}

const service = new PostService();
app.listen(3001, () => {
  console.log('Post Service listening on port 3001');
});
```

### 2. Fanout Worker

```javascript
class FanoutWorker {
  constructor() {
    this.queue = require('./queue');
    this.redis = require('ioredis');
    this.cache = new this.redis();
    this.db = require('./database');

    this.batchSize = 1000; // fanout по 1000 пользователей за раз
  }

  async start() {
    console.log('Fanout worker started');

    await this.queue.subscribe('post.fanout', async (message) => {
      try {
        await this.processFanout(message);
      } catch (error) {
        console.error('Fanout error:', error);
      }
    });
  }

  async processFanout(message) {
    const { postId, userId, followers } = message;

    console.log(`Fanout for post ${postId} to ${followers.length} followers`);

    const start = Date.now();

    // Обрабатываем по батчам
    for (let i = 0; i < followers.length; i += this.batchSize) {
      const batch = followers.slice(i, i + this.batchSize);

      await this.fanoutBatch(postId, batch);
    }

    console.log(`Fanout completed in ${Date.now() - start}ms`);
  }

  async fanoutBatch(postId, followerIds) {
    // Используем Redis для хранения лент
    const pipeline = this.cache.pipeline();

    for (const followerId of followerIds) {
      const feedKey = `feed:${followerId}`;

      // Добавляем пост в начало ленты (Sorted Set по timestamp)
      const score = Date.now();
      pipeline.zadd(feedKey, score, postId);

      // Ограничиваем размер ленты (храним последние 1000 постов)
      pipeline.zremrangebyrank(feedKey, 0, -1001);

      // TTL: 30 дней
      pipeline.expire(feedKey, 30 * 24 * 3600);
    }

    await pipeline.exec();
  }
}

const worker = new FanoutWorker();
worker.start();
```

### 3. Feed Service

```javascript
class FeedService {
  constructor() {
    this.redis = require('ioredis');
    this.cache = new this.redis();
    this.db = require('./database');
    this.rankingService = new RankingService();

    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Получение ленты
    app.get('/feed', async (req, res) => {
      try {
        const { userId, cursor, limit = 20 } = req.query;

        const feed = await this.getFeed(userId, cursor, parseInt(limit));

        res.json(feed);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3002, () => {
      console.log('Feed Service listening on port 3002');
    });
  }

  async getFeed(userId, cursor, limit) {
    const start = Date.now();

    // Получаем pre-computed feed из Redis (Fanout on Write)
    const precomputedPosts = await this.getPrecomputedFeed(userId, cursor, limit);

    // Получаем посты от celebrities (Fanout on Read)
    const celebrityPosts = await this.getCelebrityPosts(userId, limit);

    // Объединяем
    const allPosts = [...precomputedPosts, ...celebrityPosts];

    // Ranking (сортировка по релевантности, а не только по времени)
    const rankedPosts = await this.rankingService.rank(userId, allPosts);

    // Берём топ N
    const feedPosts = rankedPosts.slice(0, limit);

    // Загружаем детали постов
    const postsWithDetails = await this.loadPostDetails(feedPosts);

    // Генерируем cursor для pagination
    const nextCursor = feedPosts.length > 0
      ? feedPosts[feedPosts.length - 1].timestamp
      : null;

    return {
      posts: postsWithDetails,
      nextCursor,
      latency: Date.now() - start
    };
  }

  async getPrecomputedFeed(userId, cursor, limit) {
    const feedKey = `feed:${userId}`;

    // Получаем post IDs из Sorted Set
    const maxScore = cursor ? parseInt(cursor) : '+inf';

    const postIds = await this.cache.zrevrangebyscore(
      feedKey,
      maxScore,
      '-inf',
      'LIMIT',
      0,
      limit
    );

    return postIds.map(postId => ({
      postId,
      source: 'precomputed'
    }));
  }

  async getCelebrityPosts(userId, limit) {
    // Получаем список celebrities, на которых подписан пользователь
    const celebrities = await this.getCelebritiesFollowing(userId);

    if (celebrities.length === 0) {
      return [];
    }

    // Получаем последние посты от celebrities
    const result = await this.db.query(
      `SELECT id as post_id, created_at
       FROM posts
       WHERE user_id = ANY($1)
         AND is_celebrity = true
         AND created_at > NOW() - INTERVAL '7 days'
       ORDER BY created_at DESC
       LIMIT $2`,
      [celebrities, limit]
    );

    return result.rows.map(r => ({
      postId: r.post_id,
      timestamp: r.created_at.getTime(),
      source: 'celebrity'
    }));
  }

  async getCelebritiesFollowing(userId) {
    const result = await this.db.query(
      `SELECT DISTINCT p.user_id
       FROM follows f
       JOIN posts p ON f.following_id = p.user_id
       WHERE f.follower_id = $1
         AND p.is_celebrity = true`,
      [userId]
    );

    return result.rows.map(r => r.user_id);
  }

  async loadPostDetails(feedPosts) {
    if (feedPosts.length === 0) return [];

    const postIds = feedPosts.map(p => p.postId);

    // Batch fetch из БД
    const result = await this.db.query(
      `SELECT p.*, u.username, u.avatar_url
       FROM posts p
       JOIN users u ON p.user_id = u.id
       WHERE p.id = ANY($1)`,
      [postIds]
    );

    // Сохраняем порядок из feedPosts
    const postsMap = new Map();
    result.rows.forEach(row => {
      postsMap.set(row.id, this.formatPost(row));
    });

    return feedPosts
      .map(fp => postsMap.get(fp.postId))
      .filter(Boolean);
  }

  formatPost(row) {
    return {
      id: row.id,
      userId: row.user_id,
      username: row.username,
      avatarUrl: row.avatar_url,
      content: row.content,
      media: JSON.parse(row.media || '[]'),
      likesCount: parseInt(row.likes_count || 0),
      commentsCount: parseInt(row.comments_count || 0),
      createdAt: row.created_at,
      timestamp: row.created_at.getTime()
    };
  }
}

const service = new FeedService();
```

### 4. Ranking Service

Ранжирование постов по релевантности (Machine Learning модель или эвристики).

```javascript
class RankingService {
  async rank(userId, posts) {
    // Получаем user preferences и engagement data
    const userProfile = await this.getUserProfile(userId);

    // Вычисляем score для каждого поста
    const scoredPosts = await Promise.all(
      posts.map(async (post) => {
        const score = await this.calculateScore(post, userProfile);

        return { ...post, score };
      })
    );

    // Сортируем по score (desc)
    scoredPosts.sort((a, b) => b.score - a.score);

    return scoredPosts;
  }

  async calculateScore(post, userProfile) {
    let score = 0;

    // 1. Recency (свежесть поста)
    const ageHours = (Date.now() - post.timestamp) / 3600000;
    const recencyScore = Math.max(0, 100 - ageHours); // max 100 points
    score += recencyScore * 0.3;

    // 2. Engagement (лайки, комментарии)
    const engagementScore = Math.log10(post.likesCount + 1) * 10 +
                           Math.log10(post.commentsCount + 1) * 20;
    score += engagementScore * 0.4;

    // 3. Author relationship (близость с автором)
    const authorScore = await this.getAuthorScore(post.userId, userProfile);
    score += authorScore * 0.3;

    // 4. Content type preference (фото vs текст)
    if (post.media && post.media.length > 0 && userProfile.prefersMedia) {
      score += 20;
    }

    return score;
  }

  async getAuthorScore(authorId, userProfile) {
    // Проверяем: close friend, regular friend, or acquaintance
    const relationship = userProfile.relationships.get(authorId);

    if (!relationship) return 0;

    // Баллы на основе взаимодействия
    const interactions = relationship.interactions || 0;

    return Math.min(interactions, 100); // max 100 points
  }

  async getUserProfile(userId) {
    // Кэшируем профиль
    const cacheKey = `user_profile:${userId}`;
    const cached = await redis.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // Загружаем из БД
    const result = await db.query(
      `SELECT preferences, relationships
       FROM user_profiles
       WHERE user_id = $1`,
      [userId]
    );

    if (result.rows.length === 0) {
      return { relationships: new Map(), prefersMedia: true };
    }

    const profile = {
      preferences: result.rows[0].preferences,
      relationships: new Map(Object.entries(result.rows[0].relationships || {})),
      prefersMedia: result.rows[0].preferences?.mediaPreference || true
    };

    // Кэшируем на 1 час
    await redis.setex(cacheKey, 3600, JSON.stringify(profile));

    return profile;
  }
}
```

## Оптимизации

### 1. Feed Pagination с Cursor

```javascript
async getFeed(userId, cursor, limit) {
  // Cursor = timestamp последнего поста
  const feedKey = `feed:${userId}`;

  const maxScore = cursor || Date.now();

  // Получаем посты с timestamp < cursor
  const postIds = await redis.zrevrangebyscore(
    feedKey,
    `(${maxScore}`, // exclusive
    '-inf',
    'WITHSCORES',
    'LIMIT',
    0,
    limit
  );

  // postIds = [postId1, score1, postId2, score2, ...]
  const posts = [];
  for (let i = 0; i < postIds.length; i += 2) {
    posts.push({
      postId: postIds[i],
      timestamp: parseInt(postIds[i + 1])
    });
  }

  // Генерируем следующий cursor
  const nextCursor = posts.length > 0
    ? posts[posts.length - 1].timestamp
    : null;

  return { posts, nextCursor };
}
```

### 2. Cache Warming

Предзагрузка лент для активных пользователей.

```javascript
class FeedCacheWarmer {
  async warmCache(userId) {
    // Проверяем: есть ли уже feed в кэше
    const feedKey = `feed:${userId}`;
    const exists = await redis.exists(feedKey);

    if (exists) {
      return; // уже есть
    }

    console.log(`Warming cache for user ${userId}`);

    // Генерируем feed
    const friends = await this.getFriends(userId);

    // Получаем последние N постов от друзей
    const result = await db.query(
      `SELECT id, created_at
       FROM posts
       WHERE user_id = ANY($1)
         AND created_at > NOW() - INTERVAL '30 days'
       ORDER BY created_at DESC
       LIMIT 1000`,
      [friends]
    );

    // Заполняем Redis
    const pipeline = redis.pipeline();

    for (const post of result.rows) {
      const score = post.created_at.getTime();
      pipeline.zadd(feedKey, score, post.id);
    }

    pipeline.expire(feedKey, 30 * 24 * 3600);

    await pipeline.exec();

    console.log(`Cache warmed for user ${userId}: ${result.rows.length} posts`);
  }
}
```

### 3. Real-time Updates (WebSocket)

```javascript
const io = require('socket.io')(server);

io.on('connection', (socket) => {
  const userId = socket.handshake.auth.userId;

  console.log(`User ${userId} connected`);

  // Подписываем на обновления ленты
  socket.join(`feed:${userId}`);

  socket.on('disconnect', () => {
    console.log(`User ${userId} disconnected`);
  });
});

// В Fanout Worker:
async fanoutBatch(postId, followerIds) {
  // ... добавление в Redis ...

  // Уведомляем подключённых пользователей
  for (const followerId of followerIds) {
    io.to(`feed:${followerId}`).emit('new_post', {
      postId,
      message: 'New post available'
    });
  }
}
```

### 4. Sharding Strategy

Шардирование по user ID:

```javascript
class ShardedFeedService {
  constructor(shards) {
    this.shards = shards; // массив Redis instances
  }

  getShard(userId) {
    const hash = crypto.createHash('md5').update(userId).digest();
    const hashInt = hash.readUInt32BE(0);
    const shardIndex = hashInt % this.shards.length;

    return this.shards[shardIndex];
  }

  async getFeed(userId, cursor, limit) {
    const shard = this.getShard(userId);
    const feedKey = `feed:${userId}`;

    // Работаем с конкретным шардом
    const postIds = await shard.zrevrangebyscore(
      feedKey,
      cursor || '+inf',
      '-inf',
      'LIMIT',
      0,
      limit
    );

    return postIds;
  }
}
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const feedRequests = new prometheus.Counter({
  name: 'feed_requests_total',
  help: 'Total feed requests',
  labelNames: ['status']
});

const feedLatency = new prometheus.Histogram({
  name: 'feed_generation_duration_seconds',
  help: 'Feed generation time',
  buckets: [0.1, 0.2, 0.5, 1, 2, 5]
});

const fanoutDuration = new prometheus.Histogram({
  name: 'fanout_duration_seconds',
  help: 'Time to fanout a post',
  labelNames: ['followers_count'],
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60]
});

const cacheHitRate = new prometheus.Counter({
  name: 'feed_cache_hits_total',
  help: 'Feed cache hits',
  labelNames: ['result'] // hit, miss
});
```

## Schema Design

```sql
-- Posts table
CREATE TABLE posts (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  content TEXT,
  media JSONB,
  visibility VARCHAR(20) DEFAULT 'friends',
  is_celebrity BOOLEAN DEFAULT FALSE,
  likes_count INTEGER DEFAULT 0,
  comments_count INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  INDEX idx_user_created (user_id, created_at DESC),
  INDEX idx_celebrity (is_celebrity, created_at DESC)
);

-- Follows (social graph)
CREATE TABLE follows (
  follower_id UUID NOT NULL,
  following_id UUID NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (follower_id, following_id),
  INDEX idx_following (following_id)
);

-- User profiles
CREATE TABLE user_profiles (
  user_id UUID PRIMARY KEY,
  preferences JSONB,
  relationships JSONB
);

-- Post likes
CREATE TABLE post_likes (
  post_id UUID NOT NULL,
  user_id UUID NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (post_id, user_id)
);
```

## Trade-offs

| Аспект | Fanout on Write | Fanout on Read | Hybrid |
|--------|----------------|---------------|--------|
| Read Speed | Очень быстро | Медленно | Быстро |
| Write Speed | Медленно | Быстро | Средне |
| Storage | Высокое (N копий) | Низкое | Средне |
| Celebrity Support | Плохо | Хорошо | Хорошо |
| Real-time Updates | Сложно | Легко | Средне |
| Рекомендация | Small networks | Read-heavy | **Production** |

## Что читать дальше

- **Урок 44**: Chat System — ещё одна система с real-time требованиями
- **Урок 47**: Social Network — полный design с profiles, messages, notifications
- **Урок 22**: Kafka & Event Streaming — для fanout worker

## Проверь себя

1. Почему Hybrid approach лучше чем чистый Fanout on Write?
2. Как обрабатывать deletion поста из лент миллионов подписчиков?
3. Спроектируйте ranking алгоритм для новостной ленты.
4. Сколько памяти потребуется Redis для хранения лент 500M пользователей (1000 постов каждому)?
5. Как реализовать "Stories" (исчезающий контент через 24 часа)?

---

[← Урок 42: Autocomplete System](42-autocomplete.md) | [Урок 44: Chat System →](44-chat.md)
