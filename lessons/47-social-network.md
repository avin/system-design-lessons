# Урок 47: Social Network (Полный дизайн)

## Введение

Social Network — это комплексная система, объединяющая множество компонентов: профили пользователей, френдшип, newsfeed, messaging, notifications, media storage. Это одна из самых сложных систем для проектирования из-за масштаба, разнообразия функций и требований к производительности.

В этом уроке мы спроектируем полноценную социальную сеть уровня Facebook/Instagram, способную обслуживать миллиарды пользователей.

**Компоненты:**
- User profiles & authentication
- Friend connections (social graph)
- Newsfeed
- Posts, comments, likes
- Messaging (1-on-1, groups)
- Notifications
- Media storage (photos, videos)
- Search

## Требования к системе

### Функциональные требования

1. **User Management**:
   - Регистрация, аутентификация
   - Профили с фото, bio, posts

2. **Social Graph**:
   - Friend requests (send, accept, decline)
   - Following/followers
   - Friendship suggestions

3. **Content**:
   - Create posts (text, photos, videos)
   - Like, comment, share
   - Newsfeed (personalized)

4. **Messaging**:
   - 1-on-1 chat
   - Group chats
   - Read receipts

5. **Notifications**:
   - Real-time notifications
   - Push, email, in-app

6. **Search**:
   - Search users, posts, hashtags

### Нефункциональные требования

1. **Scale**:
   - 2B users
   - 1B DAU (Daily Active Users)
   - 100M posts/day
   - 10B messages/day

2. **Performance**:
   - Newsfeed generation: < 500ms
   - Post creation: < 2 sec delivery
   - Message delivery: < 100ms

3. **Availability**: 99.99%
4. **Consistency**: Eventual consistency OK для newsfeed, strong для messaging

### Back-of-the-envelope

```
Users: 2B total, 1B DAU
Average friends: 300
Average posts per user per week: 2

Posts: 1B × 2/7 = 286M posts/day ≈ 3,300 posts/sec
Messages: 10B/day ≈ 115K msg/sec
Newsfeed reads: 1B users × 10 reads/day = 10B/day ≈ 115K req/sec

Storage:
- User profile: 1KB × 2B = 2 TB
- Posts metadata: 100B × 100M posts/day × 365 = 3.65 TB/year
- Messages: 500B × 10B/day × 365 = 1.8 PB/year
- Photos: 50% posts have photo (2MB avg)
  50M × 2MB × 365 = 36 PB/year
- Videos: 10% posts have video (50MB avg)
  10M × 50MB × 365 = 182 PB/year

Total storage/year: ~220 PB
```

## Архитектура высокого уровня

```
┌────────────────────────────────────────────────┐
│              Client Layer                      │
│  (Web, iOS, Android)                           │
└────────────────┬───────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────┐
│          API Gateway / Load Balancer           │
└────┬──────┬──────┬──────┬──────┬──────┬────────┘
     │      │      │      │      │      │
     ↓      ↓      ↓      ↓      ↓      ↓
┌────────┐┌────┐┌──────┐┌────┐┌──────┐┌────────┐
│  User  ││Feed││ Post ││Chat││Notif.││ Search │
│Service ││Svc ││Service││Svc ││ Svc  ││Service │
└───┬────┘└─┬──┘└──┬───┘└─┬──┘└──┬───┘└───┬────┘
    │       │      │      │      │        │
    └───────┴──────┴──────┴──────┴────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
    ↓              ↓              ↓
┌─────────┐  ┌──────────┐  ┌───────────┐
│  User   │  │  Posts   │  │ Messages  │
│Database │  │ Database │  │ Database  │
│(MySQL)  │  │(Cassandra)│ │(Cassandra)│
└─────────┘  └──────────┘  └───────────┘
    │              │              │
    ↓              ↓              ↓
┌─────────────────────────────────────┐
│         Cache Layer (Redis)         │
└─────────────────────────────────────┘
    │
    ↓
┌─────────────────────────────────────┐
│   Object Storage (S3) — Media       │
└─────────────────────────────────────┘
```

## Ключевые компоненты

### 1. User Service

```javascript
const express = require('express');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

class UserService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Registration
    app.post('/auth/register', async (req, res) => {
      try {
        const { email, password, username, fullName } = req.body;

        // Validate
        if (await this.userExists(email, username)) {
          return res.status(400).json({ error: 'User already exists' });
        }

        // Hash password
        const passwordHash = await bcrypt.hash(password, 10);

        // Create user
        const userId = await this.createUser({
          email,
          passwordHash,
          username,
          fullName
        });

        // Generate token
        const token = this.generateToken(userId);

        res.json({ success: true, userId, token });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Login
    app.post('/auth/login', async (req, res) => {
      try {
        const { email, password } = req.body;

        const user = await this.getUserByEmail(email);

        if (!user) {
          return res.status(401).json({ error: 'Invalid credentials' });
        }

        // Verify password
        const valid = await bcrypt.compare(password, user.password_hash);

        if (!valid) {
          return res.status(401).json({ error: 'Invalid credentials' });
        }

        const token = this.generateToken(user.id);

        res.json({ success: true, userId: user.id, token });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Get profile
    app.get('/users/:userId', async (req, res) => {
      try {
        const { userId } = req.params;

        const profile = await this.getProfile(userId);

        res.json(profile);
      } catch (error) {
        res.status(404).json({ error: 'User not found' });
      }
    });

    // Update profile
    app.put('/users/:userId', async (req, res) => {
      try {
        const { userId } = req.params;
        const updates = req.body;

        await this.updateProfile(userId, updates);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3001, () => {
      console.log('User Service listening on port 3001');
    });
  }

  async createUser({ email, passwordHash, username, fullName }) {
    const userId = this.generateUserId();

    await this.db.query(
      `INSERT INTO users
       (id, email, password_hash, username, full_name, created_at)
       VALUES ($1, $2, $3, $4, $5, NOW())`,
      [userId, email, passwordHash, username, fullName]
    );

    return userId;
  }

  async userExists(email, username) {
    const result = await this.db.query(
      `SELECT id FROM users
       WHERE email = $1 OR username = $2`,
      [email, username]
    );

    return result.rows.length > 0;
  }

  async getUserByEmail(email) {
    const result = await this.db.query(
      'SELECT * FROM users WHERE email = $1',
      [email]
    );

    return result.rows[0];
  }

  async getProfile(userId) {
    // Проверяем cache
    const cacheKey = `profile:${userId}`;
    const cached = await this.redis.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // Загружаем из БД
    const result = await this.db.query(
      `SELECT id, username, full_name, bio, avatar_url,
              followers_count, following_count, posts_count,
              created_at
       FROM users
       WHERE id = $1`,
      [userId]
    );

    if (result.rows.length === 0) {
      throw new Error('User not found');
    }

    const profile = result.rows[0];

    // Кэшируем (1 hour)
    await this.redis.setex(cacheKey, 3600, JSON.stringify(profile));

    return profile;
  }

  async updateProfile(userId, updates) {
    const { fullName, bio, avatarUrl } = updates;

    await this.db.query(
      `UPDATE users
       SET full_name = COALESCE($1, full_name),
           bio = COALESCE($2, bio),
           avatar_url = COALESCE($3, avatar_url)
       WHERE id = $4`,
      [fullName, bio, avatarUrl, userId]
    );

    // Invalidate cache
    await this.redis.del(`profile:${userId}`);
  }

  generateToken(userId) {
    return jwt.sign({ userId }, process.env.JWT_SECRET, {
      expiresIn: '7d'
    });
  }

  generateUserId() {
    const Snowflake = require('./snowflake');
    return new Snowflake(1, 1).generate().toString();
  }
}

const service = new UserService();
```

### 2. Social Graph Service (Friendship)

```javascript
class SocialGraphService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Send friend request
    app.post('/friends/request', async (req, res) => {
      try {
        const { fromUserId, toUserId } = req.body;

        await this.sendFriendRequest(fromUserId, toUserId);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Accept friend request
    app.post('/friends/accept', async (req, res) => {
      try {
        const { requestId, userId } = req.body;

        await this.acceptFriendRequest(requestId, userId);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Get friends list
    app.get('/friends/:userId', async (req, res) => {
      try {
        const { userId } = req.params;
        const { limit = 50, offset = 0 } = req.query;

        const friends = await this.getFriends(userId, limit, offset);

        res.json({ friends });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Friend suggestions
    app.get('/friends/suggestions/:userId', async (req, res) => {
      try {
        const { userId } = req.params;

        const suggestions = await this.getFriendSuggestions(userId);

        res.json({ suggestions });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3002, () => {
      console.log('Social Graph Service listening on port 3002');
    });
  }

  async sendFriendRequest(fromUserId, toUserId) {
    // Проверяем: не друзья ли уже
    const areFriends = await this.checkFriendship(fromUserId, toUserId);

    if (areFriends) {
      throw new Error('Already friends');
    }

    // Проверяем: нет ли pending request
    const existingRequest = await this.getPendingRequest(fromUserId, toUserId);

    if (existingRequest) {
      throw new Error('Request already sent');
    }

    // Создаём friend request
    const requestId = this.generateRequestId();

    await this.db.query(
      `INSERT INTO friend_requests
       (id, from_user_id, to_user_id, status, created_at)
       VALUES ($1, $2, $3, 'pending', NOW())`,
      [requestId, fromUserId, toUserId]
    );

    // Отправляем notification
    await this.notifyFriendRequest(fromUserId, toUserId);

    return requestId;
  }

  async acceptFriendRequest(requestId, userId) {
    // Получаем request
    const request = await this.getRequest(requestId);

    if (!request || request.to_user_id !== userId) {
      throw new Error('Invalid request');
    }

    // Обновляем статус
    await this.db.query(
      `UPDATE friend_requests
       SET status = 'accepted', updated_at = NOW()
       WHERE id = $1`,
      [requestId]
    );

    // Создаём friendship (двунаправленная связь)
    await this.createFriendship(request.from_user_id, request.to_user_id);

    // Обновляем счётчики
    await this.incrementFriendsCount(request.from_user_id);
    await this.incrementFriendsCount(request.to_user_id);

    // Уведомляем
    await this.notifyFriendAccepted(request.from_user_id, request.to_user_id);
  }

  async createFriendship(userId1, userId2) {
    await this.db.query('BEGIN');

    try {
      // Обе стороны
      await this.db.query(
        `INSERT INTO friendships (user_id, friend_id, created_at)
         VALUES ($1, $2, NOW()), ($2, $1, NOW())`,
        [userId1, userId2]
      );

      // Добавляем в Redis (для быстрого доступа)
      await this.redis.sadd(`friends:${userId1}`, userId2);
      await this.redis.sadd(`friends:${userId2}`, userId1);

      await this.db.query('COMMIT');
    } catch (error) {
      await this.db.query('ROLLBACK');
      throw error;
    }
  }

  async getFriends(userId, limit, offset) {
    // Проверяем cache
    const cacheKey = `friends:${userId}`;
    const cachedIds = await this.redis.smembers(cacheKey);

    if (cachedIds.length > 0) {
      // Загружаем профили друзей
      const friends = await this.loadUserProfiles(cachedIds.slice(offset, offset + limit));
      return friends;
    }

    // Загружаем из БД
    const result = await this.db.query(
      `SELECT u.id, u.username, u.full_name, u.avatar_url
       FROM friendships f
       JOIN users u ON f.friend_id = u.id
       WHERE f.user_id = $1
       ORDER BY f.created_at DESC
       LIMIT $2 OFFSET $3`,
      [userId, limit, offset]
    );

    return result.rows;
  }

  async getFriendSuggestions(userId) {
    // Алгоритм: Friends of Friends (FoF)
    const result = await this.db.query(
      `SELECT u.id, u.username, u.full_name, u.avatar_url,
              COUNT(*) as mutual_friends
       FROM friendships f1
       JOIN friendships f2 ON f1.friend_id = f2.user_id
       JOIN users u ON f2.friend_id = u.id
       WHERE f1.user_id = $1
         AND f2.friend_id != $1
         AND f2.friend_id NOT IN (
           SELECT friend_id FROM friendships WHERE user_id = $1
         )
       GROUP BY u.id, u.username, u.full_name, u.avatar_url
       ORDER BY mutual_friends DESC, RANDOM()
       LIMIT 10`,
      [userId]
    );

    return result.rows;
  }

  async checkFriendship(userId1, userId2) {
    // Проверяем в Redis (быстро)
    const exists = await this.redis.sismember(`friends:${userId1}`, userId2);

    return exists === 1;
  }

  async notifyFriendRequest(fromUserId, toUserId) {
    // Публикуем в notification service
    const kafka = require('./kafka');

    await kafka.send({
      topic: 'notifications',
      messages: [{
        value: JSON.stringify({
          type: 'friend_request',
          fromUserId,
          toUserId,
          timestamp: Date.now()
        })
      }]
    });
  }

  generateRequestId() {
    return require('crypto').randomUUID();
  }
}

const service = new SocialGraphService();
```

### 3. Post Service (использует код из урока 43)

```javascript
class PostService {
  async createPost(userId, content, mediaUrls) {
    const postId = this.generatePostId();

    // Сохраняем пост
    await this.db.execute(
      `INSERT INTO posts
       (id, user_id, content, media_urls, created_at)
       VALUES (?, ?, ?, ?, ?)`,
      [postId, userId, content, JSON.stringify(mediaUrls), Date.now()]
    );

    // Инкрементируем posts_count
    await this.db.query(
      'UPDATE users SET posts_count = posts_count + 1 WHERE id = $1',
      [userId]
    );

    // Публикуем для fanout worker
    await this.publishToFanout(postId, userId);

    return postId;
  }

  async likePost(postId, userId) {
    // Проверяем: уже лайкнуто?
    const alreadyLiked = await this.redis.sismember(`likes:${postId}`, userId);

    if (alreadyLiked) {
      return;
    }

    // Добавляем like
    await this.redis.sadd(`likes:${postId}`, userId);

    // Инкрементируем счётчик
    await this.db.query(
      'UPDATE posts SET likes_count = likes_count + 1 WHERE id = $1',
      [postId]
    );

    // Уведомляем автора
    const post = await this.getPost(postId);

    if (post.user_id !== userId) {
      await this.notifyLike(post.user_id, userId, postId);
    }
  }

  async addComment(postId, userId, content) {
    const commentId = this.generateCommentId();

    await this.db.execute(
      `INSERT INTO comments
       (id, post_id, user_id, content, created_at)
       VALUES (?, ?, ?, ?, ?)`,
      [commentId, postId, userId, content, Date.now()]
    );

    // Инкрементируем comments_count
    await this.db.query(
      'UPDATE posts SET comments_count = comments_count + 1 WHERE id = $1',
      [postId]
    );

    return commentId;
  }
}
```

### 4. Feed Service (из урока 43)

Полная реализация в уроке 43 (Newsfeed System).

### 5. Chat Service (из урока 44)

Полная реализация в уроке 44 (Chat System).

## Schema Design

### PostgreSQL (Relational Data)

```sql
-- Users
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  username VARCHAR(50) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  full_name VARCHAR(255),
  bio TEXT,
  avatar_url TEXT,
  followers_count INT DEFAULT 0,
  following_count INT DEFAULT 0,
  posts_count INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  INDEX idx_username (username),
  INDEX idx_email (email)
);

-- Friendships (bidirectional)
CREATE TABLE friendships (
  user_id BIGINT NOT NULL,
  friend_id BIGINT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (user_id, friend_id),
  INDEX idx_friend (friend_id)
);

-- Friend requests
CREATE TABLE friend_requests (
  id UUID PRIMARY KEY,
  from_user_id BIGINT NOT NULL,
  to_user_id BIGINT NOT NULL,
  status VARCHAR(20) DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP,
  INDEX idx_to_user (to_user_id, status)
);
```

### Cassandra (High-throughput Data)

```sql
-- Posts
CREATE TABLE posts (
  id TEXT PRIMARY KEY,
  user_id TEXT,
  content TEXT,
  media_urls TEXT, -- JSON array
  likes_count INT,
  comments_count INT,
  created_at BIGINT
);

CREATE TABLE posts_by_user (
  user_id TEXT,
  created_at BIGINT,
  post_id TEXT,
  PRIMARY KEY (user_id, created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Comments
CREATE TABLE comments (
  id TEXT,
  post_id TEXT,
  user_id TEXT,
  content TEXT,
  created_at BIGINT,
  PRIMARY KEY (post_id, created_at, id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Messages (from урок 44)
CREATE TABLE messages (
  id TEXT,
  sender_id TEXT,
  recipient_id TEXT,
  content TEXT,
  created_at BIGINT,
  PRIMARY KEY ((sender_id, recipient_id), created_at, id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

## Продвинутые фичи

### 1. Stories (24-hour content)

```javascript
class StoriesService {
  async createStory(userId, mediaUrl, duration = 86400) {
    const storyId = this.generateStoryId();

    // Сохраняем в Redis с TTL 24 hours
    await this.redis.setex(
      `story:${storyId}`,
      duration,
      JSON.stringify({
        userId,
        mediaUrl,
        createdAt: Date.now()
      })
    );

    // Добавляем в список stories пользователя
    await this.redis.zadd(
      `stories:${userId}`,
      Date.now(),
      storyId
    );

    await this.redis.expire(`stories:${userId}`, duration);

    // Уведомляем друзей
    await this.notifyFriendsAboutStory(userId, storyId);

    return storyId;
  }

  async getStoriesByUser(userId) {
    // Получаем story IDs
    const storyIds = await this.redis.zrevrange(`stories:${userId}`, 0, -1);

    // Загружаем детали
    const stories = await Promise.all(
      storyIds.map(async (id) => {
        const data = await this.redis.get(`story:${id}`);
        return data ? JSON.parse(data) : null;
      })
    );

    return stories.filter(Boolean);
  }

  async getFriendsStories(userId) {
    // Получаем друзей
    const friends = await this.redis.smembers(`friends:${userId}`);

    // Загружаем stories от каждого друга
    const allStories = [];

    for (const friendId of friends) {
      const stories = await this.getStoriesByUser(friendId);

      if (stories.length > 0) {
        allStories.push({
          userId: friendId,
          stories
        });
      }
    }

    return allStories;
  }
}
```

### 2. Live Streaming

```javascript
class LiveStreamService {
  async startStream(userId, title) {
    const streamId = this.generateStreamId();

    // Создаём stream session
    await this.db.query(
      `INSERT INTO live_streams
       (id, user_id, title, status, started_at)
       VALUES ($1, $2, $3, 'live', NOW())`,
      [streamId, userId, title]
    );

    // Добавляем в Redis (для быстрого доступа к active streams)
    await this.redis.sadd('active_streams', streamId);
    await this.redis.hset(`stream:${streamId}`, {
      userId,
      title,
      viewers: 0
    });

    // Уведомляем подписчиков
    await this.notifyFollowersAboutStream(userId, streamId);

    // Получаем RTMP endpoint для стриминга
    const rtmpUrl = await this.allocateStreamingEndpoint(streamId);

    return { streamId, rtmpUrl };
  }

  async joinStream(streamId, viewerId) {
    // Инкрементируем viewers count
    await this.redis.hincrby(`stream:${streamId}`, 'viewers', 1);
    await this.redis.sadd(`stream:${streamId}:viewers`, viewerId);

    // Получаем HLS/DASH playback URL
    const playbackUrl = `https://cdn.example.com/stream/${streamId}/index.m3u8`;

    return { playbackUrl };
  }

  async endStream(streamId, userId) {
    // Обновляем статус
    await this.db.query(
      `UPDATE live_streams
       SET status = 'ended', ended_at = NOW()
       WHERE id = $1 AND user_id = $2`,
      [streamId, userId]
    );

    // Удаляем из active
    await this.redis.srem('active_streams', streamId);
    await this.redis.del(`stream:${streamId}`);
  }
}
```

### 3. Trending Topics

```javascript
class TrendingService {
  async trackHashtag(hashtag, postId) {
    const now = Date.now();

    // Добавляем в sorted set по времени
    await this.redis.zadd(`hashtag:${hashtag}`, now, postId);

    // Инкрементируем счётчик за последний час
    const hourKey = `trending:${Math.floor(now / 3600000)}`;
    await this.redis.zincrby(hourKey, 1, hashtag);
    await this.redis.expire(hourKey, 7200); // 2 hours TTL
  }

  async getTrendingHashtags(limit = 10) {
    const currentHour = Math.floor(Date.now() / 3600000);

    // Получаем топ хэштеги за последние 3 часа
    const trending = [];

    for (let i = 0; i < 3; i++) {
      const hourKey = `trending:${currentHour - i}`;
      const tags = await this.redis.zrevrange(hourKey, 0, limit - 1, 'WITHSCORES');

      trending.push(...tags);
    }

    // Агрегируем и сортируем
    const aggregated = new Map();

    for (let i = 0; i < trending.length; i += 2) {
      const tag = trending[i];
      const score = parseFloat(trending[i + 1]);

      aggregated.set(tag, (aggregated.get(tag) || 0) + score);
    }

    const sorted = Array.from(aggregated.entries())
      .sort((a, b) => b[1] - a[1])
      .slice(0, limit);

    return sorted.map(([tag, count]) => ({ tag, count }));
  }
}
```

## Оптимизации

### 1. CDN для Media

```javascript
class MediaService {
  async uploadMedia(file, userId) {
    // Загружаем в S3
    const key = `media/${userId}/${Date.now()}_${file.name}`;

    await s3.putObject({
      Bucket: process.env.S3_BUCKET,
      Key: key,
      Body: file.buffer,
      ContentType: file.mimetype
    }).promise();

    // Генерируем CloudFront URL
    const cdnUrl = `https://cdn.example.com/${key}`;

    return cdnUrl;
  }

  async generateThumbnail(imageUrl) {
    // Используем Lambda или image processing service
    const thumbnail = await imageProcessor.resize(imageUrl, {
      width: 150,
      height: 150,
      fit: 'cover'
    });

    return thumbnail;
  }
}
```

### 2. Database Sharding

```javascript
class ShardedDatabase {
  constructor() {
    this.shards = [
      { id: 0, connection: new Pool({ host: 'db-shard-0' }) },
      { id: 1, connection: new Pool({ host: 'db-shard-1' }) },
      { id: 2, connection: new Pool({ host: 'db-shard-2' }) },
      { id: 3, connection: new Pool({ host: 'db-shard-3' }) }
    ];
  }

  getShard(userId) {
    const hash = crypto.createHash('md5').update(userId).digest();
    const shardId = hash.readUInt32BE(0) % this.shards.length;

    return this.shards[shardId].connection;
  }

  async query(userId, sql, params) {
    const shard = this.getShard(userId);
    return await shard.query(sql, params);
  }
}
```

### 3. Read Replicas

```javascript
class DatabaseRouter {
  constructor() {
    this.master = new Pool({ host: 'db-master' });
    this.replicas = [
      new Pool({ host: 'db-replica-1' }),
      new Pool({ host: 'db-replica-2' }),
      new Pool({ host: 'db-replica-3' })
    ];
    this.replicaIndex = 0;
  }

  async write(sql, params) {
    return await this.master.query(sql, params);
  }

  async read(sql, params) {
    // Round-robin across replicas
    const replica = this.replicas[this.replicaIndex];
    this.replicaIndex = (this.replicaIndex + 1) % this.replicas.length;

    return await replica.query(sql, params);
  }
}
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

// User metrics
const activeUsers = new prometheus.Gauge({
  name: 'social_active_users',
  help: 'Number of active users'
});

const registrations = new prometheus.Counter({
  name: 'social_registrations_total',
  help: 'Total user registrations'
});

// Content metrics
const postsCreated = new prometheus.Counter({
  name: 'social_posts_created_total',
  help: 'Total posts created'
});

const feedLatency = new prometheus.Histogram({
  name: 'social_feed_generation_seconds',
  help: 'Feed generation latency',
  buckets: [0.1, 0.2, 0.5, 1, 2, 5]
});

// Social graph metrics
const friendships = new prometheus.Counter({
  name: 'social_friendships_total',
  help: 'Total friendships created'
});
```

## Что читать дальше

- **Урок 43**: Newsfeed System — детали реализации ленты
- **Урок 44**: Chat System — messaging компонент
- **Урок 41**: Notification System — уведомления
- **Урок 52**: Recommendation System — friend suggestions, content recommendations

## Проверь себя

1. Как обеспечить консистентность при создании двунаправленной дружбы?
2. Спроектируйте систему для 2B users с minimal latency для newsfeed.
3. Как реализовать privacy controls (public/friends-only/private posts)?
4. Какую стратегию sharding использовать для users vs posts?
5. Как обрабатывать viral posts (миллионы лайков/комментов)?

---
**Предыдущий урок**: [Урок 46: Nearby Friends (Геолокационный поиск)](46-nearby-friends.md)
**Следующий урок**: [Урок 48: Ticketing System (Система продажи билетов)](48-ticketing-system.md)
