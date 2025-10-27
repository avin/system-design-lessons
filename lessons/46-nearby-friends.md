# Урок 46: Nearby Friends (Геолокационный поиск)

## Введение

Nearby Friends — это функция, которая показывает друзей или пользователей поблизости на основе их текущего местоположения. Это критически важная фича для location-based приложений: dating apps (Tinder), social networks (Facebook), food delivery (Uber Eats), gaming (Pokemon Go).

В этом уроке мы спроектируем масштабируемую систему для поиска пользователей в радиусе с учётом миллионов активных пользователей и частых обновлений локации.

**Примеры:**
- **Tinder**: nearby matches
- **Facebook**: nearby friends
- **Uber**: nearby drivers
- **Pokemon Go**: nearby players

## Требования к системе

### Функциональные требования

1. **Location Updates**:
   - Пользователь отправляет свою локацию (lat, lon)
   - Частота обновлений: каждые 30 секунд

2. **Nearby Search**:
   - Найти пользователей в радиусе N km
   - Вернуть топ-K ближайших пользователей

3. **Privacy**:
   - Пользователь может скрыть свою локацию
   - Показывать только друзей (или всех — настраиваемо)

### Нефункциональные требования

1. **Scalability**: 100M users, 10M active
2. **Latency**:
   - Location update: < 100ms
   - Nearby search: < 500ms
3. **Availability**: 99.9%
4. **Accuracy**: ~100m (GPS accuracy)
5. **Data freshness**: < 30 sec (stale location допустимо)

### Back-of-the-envelope

```
Active users: 10M
Location updates: 10M / 30 sec ≈ 333K updates/sec

Nearby searches: 10M users × 10 searches/day = 100M/day ≈ 1,157 searches/sec

Storage:
- User location: user_id (8 bytes) + lat (8 bytes) + lon (8 bytes) + timestamp (8 bytes) = 32 bytes
- 10M users × 32 bytes = 320 MB (легко в памяти!)

Grid cells (для partitioning):
- Земля: ~510M km²
- Cell size: 5 km × 5 km = 25 km²
- Total cells: 510M / 25 = 20M cells
```

## Ключевые концепции

### 1. Geohash

Geohash — это способ кодирования географических координат в строку.

```
Latitude:  37.7749 (San Francisco)
Longitude: -122.4194

Geohash: 9q8yy

Prefix sharing = близость:
9q8yy9 ← очень близко
9q8yy  ← близко
9q8    ← умеренно близко
9q     ← далеко
```

**Принцип работы:**

```
1. Разделяем мир пополам (lat/lon)
2. Кодируем каждое разделение: 0 или 1
3. Чередуем lon, lat биты
4. Конвертируем в base32

Пример:
Lon: -122.4194 → между [-180, 0] → 0
Lat: 37.7749  → между [0, 90]    → 1
...
Binary: 01101011100...
Base32: 9q8yy...
```

**Реализация:**

```javascript
class Geohash {
  static encode(lat, lon, precision = 9) {
    const BASE32 = '0123456789bcdefghjkmnpqrstuvwxyz';

    let idx = 0;
    let bit = 0;
    let evenBit = true;
    let geohash = '';

    let latRange = [-90, 90];
    let lonRange = [-180, 180];

    while (geohash.length < precision) {
      if (evenBit) {
        // Longitude
        const mid = (lonRange[0] + lonRange[1]) / 2;

        if (lon > mid) {
          idx = (idx << 1) + 1;
          lonRange[0] = mid;
        } else {
          idx = idx << 1;
          lonRange[1] = mid;
        }
      } else {
        // Latitude
        const mid = (latRange[0] + latRange[1]) / 2;

        if (lat > mid) {
          idx = (idx << 1) + 1;
          latRange[0] = mid;
        } else {
          idx = idx << 1;
          latRange[1] = mid;
        }
      }

      evenBit = !evenBit;

      if (++bit === 5) {
        geohash += BASE32[idx];
        bit = 0;
        idx = 0;
      }
    }

    return geohash;
  }

  static decode(geohash) {
    const BASE32 = '0123456789bcdefghjkmnpqrstuvwxyz';

    let evenBit = true;
    let latRange = [-90, 90];
    let lonRange = [-180, 180];

    for (const char of geohash) {
      const idx = BASE32.indexOf(char);

      for (let i = 4; i >= 0; i--) {
        const bit = (idx >> i) & 1;

        if (evenBit) {
          const mid = (lonRange[0] + lonRange[1]) / 2;

          if (bit === 1) {
            lonRange[0] = mid;
          } else {
            lonRange[1] = mid;
          }
        } else {
          const mid = (latRange[0] + latRange[1]) / 2;

          if (bit === 1) {
            latRange[0] = mid;
          } else {
            latRange[1] = mid;
          }
        }

        evenBit = !evenBit;
      }
    }

    const lat = (latRange[0] + latRange[1]) / 2;
    const lon = (lonRange[0] + lonRange[1]) / 2;

    return { lat, lon };
  }

  static neighbors(geohash) {
    // Возвращает 8 соседних geohash cells
    // (упрощённая версия — используйте библиотеку ngeohash для production)
    const lat_lon = this.decode(geohash);
    const precision = geohash.length;

    // Примерный размер cell для данного precision
    const cellSize = this.getCellSize(precision);

    const neighbors = [];
    const offsets = [
      [-1, -1], [0, -1], [1, -1],
      [-1, 0],           [1, 0],
      [-1, 1],  [0, 1],  [1, 1]
    ];

    for (const [dx, dy] of offsets) {
      const neighborLat = lat_lon.lat + dy * cellSize.lat;
      const neighborLon = lat_lon.lon + dx * cellSize.lon;

      neighbors.push(this.encode(neighborLat, neighborLon, precision));
    }

    return neighbors;
  }

  static getCellSize(precision) {
    // Approximate cell size for each precision level
    const sizes = {
      1: { lat: 22.5, lon: 45 },
      2: { lat: 5.625, lon: 5.625 },
      3: { lat: 1.40625, lon: 1.40625 },
      4: { lat: 0.3515625, lon: 0.17578125 },
      5: { lat: 0.0439453125, lon: 0.0439453125 },
      6: { lat: 0.010986328125, lon: 0.005493164063 },
      7: { lat: 0.001373291016, lon: 0.001373291016 },
      8: { lat: 0.000343322754, lon: 0.000171661377 }
    };

    return sizes[precision] || sizes[8];
  }
}

// Использование
const geohash = Geohash.encode(37.7749, -122.4194, 7);
console.log('Geohash:', geohash); // 9q8yyk

const coords = Geohash.decode(geohash);
console.log('Decoded:', coords); // { lat: 37.7749, lon: -122.4194 }

const neighbors = Geohash.neighbors(geohash);
console.log('Neighbors:', neighbors);
```

### 2. Quadtree

Альтернатива Geohash — иерархическая структура данных.

```
        World
       /  |  \  \
     NW  NE SW SE
    / |  ..
   NW NE
   ..

Каждая нода делится на 4 квадранта:
- NorthWest (NW)
- NorthEast (NE)
- SouthWest (SW)
- SouthEast (SE)
```

**Реализация:**

```javascript
class QuadTreeNode {
  constructor(boundary, capacity = 4) {
    this.boundary = boundary; // { minLat, maxLat, minLon, maxLon }
    this.capacity = capacity;
    this.points = []; // { userId, lat, lon }
    this.divided = false;
    this.nw = null;
    this.ne = null;
    this.sw = null;
    this.se = null;
  }

  insert(point) {
    if (!this.contains(point)) {
      return false;
    }

    if (this.points.length < this.capacity) {
      this.points.push(point);
      return true;
    }

    if (!this.divided) {
      this.subdivide();
    }

    return (
      this.nw.insert(point) ||
      this.ne.insert(point) ||
      this.sw.insert(point) ||
      this.se.insert(point)
    );
  }

  subdivide() {
    const { minLat, maxLat, minLon, maxLon } = this.boundary;
    const midLat = (minLat + maxLat) / 2;
    const midLon = (minLon + maxLon) / 2;

    this.nw = new QuadTreeNode({
      minLat: midLat,
      maxLat,
      minLon,
      maxLon: midLon
    }, this.capacity);

    this.ne = new QuadTreeNode({
      minLat: midLat,
      maxLat,
      minLon: midLon,
      maxLon
    }, this.capacity);

    this.sw = new QuadTreeNode({
      minLat,
      maxLat: midLat,
      minLon,
      maxLon: midLon
    }, this.capacity);

    this.se = new QuadTreeNode({
      minLat,
      maxLat: midLat,
      minLon: midLon,
      maxLon
    }, this.capacity);

    this.divided = true;
  }

  query(range, found = []) {
    if (!this.intersects(range)) {
      return found;
    }

    for (const point of this.points) {
      if (this.inRange(point, range)) {
        found.push(point);
      }
    }

    if (this.divided) {
      this.nw.query(range, found);
      this.ne.query(range, found);
      this.sw.query(range, found);
      this.se.query(range, found);
    }

    return found;
  }

  contains(point) {
    const { minLat, maxLat, minLon, maxLon } = this.boundary;

    return (
      point.lat >= minLat &&
      point.lat <= maxLat &&
      point.lon >= minLon &&
      point.lon <= maxLon
    );
  }

  intersects(range) {
    const { minLat, maxLat, minLon, maxLon } = this.boundary;
    const { lat, lon, radius } = range;

    // Упрощённая проверка (можно улучшить)
    return !(
      maxLat < lat - radius ||
      minLat > lat + radius ||
      maxLon < lon - radius ||
      minLon > lon + radius
    );
  }

  inRange(point, range) {
    const distance = this.haversineDistance(
      point.lat,
      point.lon,
      range.lat,
      range.lon
    );

    return distance <= range.radius;
  }

  haversineDistance(lat1, lon1, lat2, lon2) {
    const R = 6371; // Earth radius in km
    const dLat = this.toRadians(lat2 - lat1);
    const dLon = this.toRadians(lon2 - lon1);

    const a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(this.toRadians(lat1)) *
      Math.cos(this.toRadians(lat2)) *
      Math.sin(dLon / 2) *
      Math.sin(dLon / 2);

    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

    return R * c;
  }

  toRadians(degrees) {
    return degrees * (Math.PI / 180);
  }
}

// Использование
const quadtree = new QuadTreeNode({
  minLat: -90,
  maxLat: 90,
  minLon: -180,
  maxLon: 180
});

// Добавляем пользователей
quadtree.insert({ userId: 'user1', lat: 37.7749, lon: -122.4194 });
quadtree.insert({ userId: 'user2', lat: 37.7849, lon: -122.4094 });

// Поиск в радиусе
const nearby = quadtree.query({
  lat: 37.7749,
  lon: -122.4194,
  radius: 5 // km
});

console.log('Nearby users:', nearby);
```

## Архитектура системы

```
┌──────────┐
│  Client  │
└────┬─────┘
     │ POST /location/update
     │ GET  /nearby
     ↓
┌─────────────────┐
│   API Gateway   │
└────┬────────────┘
     │
     ├────────────┬────────────┐
     ↓            ↓            ↓
┌─────────┐ ┌──────────┐ ┌─────────┐
│Location │ │ Nearby   │ │  User   │
│ Service │ │ Service  │ │ Service │
└────┬────┘ └────┬─────┘ └─────────┘
     │           │
     ↓           ↓
┌──────────────────────┐
│   Redis (In-Memory)  │
│  Geospatial Index    │
│  GEOADD, GEORADIUS   │
└──────────────────────┘
```

## Реализация компонентов

### 1. Location Service

```javascript
const express = require('express');
const Redis = require('ioredis');
const redis = new Redis();

class LocationService {
  constructor() {
    this.redis = redis;
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Update location
    app.post('/location/update', async (req, res) => {
      try {
        const { userId, lat, lon, timestamp } = req.body;

        await this.updateLocation(userId, lat, lon, timestamp);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Get user location
    app.get('/location/:userId', async (req, res) => {
      try {
        const { userId } = req.params;

        const location = await this.getLocation(userId);

        if (!location) {
          return res.status(404).json({ error: 'Location not found' });
        }

        res.json(location);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3001, () => {
      console.log('Location Service listening on port 3001');
    });
  }

  async updateLocation(userId, lat, lon, timestamp) {
    // Проверяем privacy settings
    const privacySettings = await this.getPrivacySettings(userId);

    if (!privacySettings.shareLocation) {
      console.log(`User ${userId} has location sharing disabled`);
      return;
    }

    // Вычисляем geohash
    const geohash = Geohash.encode(lat, lon, 7);

    // Сохраняем в Redis Geospatial
    await this.redis.geoadd(
      'user_locations',
      lon,
      lat,
      userId
    );

    // Сохраняем метаданные
    await this.redis.hset(
      `location:${userId}`,
      'lat', lat,
      'lon', lon,
      'geohash', geohash,
      'timestamp', timestamp
    );

    // TTL: 5 minutes (автоочистка stale locations)
    await this.redis.expire(`location:${userId}`, 300);

    console.log(`Location updated for user ${userId}: ${lat}, ${lon}`);
  }

  async getLocation(userId) {
    const location = await this.redis.hgetall(`location:${userId}`);

    if (Object.keys(location).length === 0) {
      return null;
    }

    return {
      userId,
      lat: parseFloat(location.lat),
      lon: parseFloat(location.lon),
      geohash: location.geohash,
      timestamp: parseInt(location.timestamp)
    };
  }

  async getPrivacySettings(userId) {
    // Заглушка — в реальности из БД
    return { shareLocation: true };
  }
}

const service = new LocationService();
```

### 2. Nearby Service

```javascript
class NearbyService {
  constructor() {
    this.redis = new Redis();
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Find nearby users
    app.get('/nearby', async (req, res) => {
      try {
        const { userId, radius = 5, limit = 20 } = req.query;

        const nearby = await this.findNearby(
          userId,
          parseFloat(radius),
          parseInt(limit)
        );

        res.json({ nearby });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3002, () => {
      console.log('Nearby Service listening on port 3002');
    });
  }

  async findNearby(userId, radiusKm, limit) {
    // Получаем локацию пользователя
    const userLocation = await this.redis.hgetall(`location:${userId}`);

    if (!userLocation.lat || !userLocation.lon) {
      throw new Error('User location not found');
    }

    const lat = parseFloat(userLocation.lat);
    const lon = parseFloat(userLocation.lon);

    // Redis GEORADIUS command
    const nearby = await this.redis.georadius(
      'user_locations',
      lon,
      lat,
      radiusKm,
      'km',
      'WITHDIST',
      'ASC',
      'COUNT',
      limit + 1 // +1 потому что включает самого пользователя
    );

    // Фильтруем самого пользователя и загружаем детали
    const results = [];

    for (const [nearbyUserId, distance] of nearby) {
      if (nearbyUserId === userId) continue;

      // Проверяем friendship (опционально)
      const isFriend = await this.checkFriendship(userId, nearbyUserId);

      if (!isFriend) continue;

      // Загружаем user info
      const userInfo = await this.getUserInfo(nearbyUserId);

      results.push({
        userId: nearbyUserId,
        username: userInfo.username,
        avatarUrl: userInfo.avatarUrl,
        distance: parseFloat(distance),
        location: await this.getLocation(nearbyUserId)
      });

      if (results.length >= limit) break;
    }

    return results;
  }

  async checkFriendship(userId1, userId2) {
    // Проверка дружбы — из БД или Redis
    const result = await db.query(
      `SELECT 1 FROM friendships
       WHERE (user_id = $1 AND friend_id = $2)
          OR (user_id = $2 AND friend_id = $1)`,
      [userId1, userId2]
    );

    return result.rows.length > 0;
  }

  async getUserInfo(userId) {
    // Заглушка — из БД или cache
    return {
      username: `user_${userId}`,
      avatarUrl: `https://example.com/avatar/${userId}.jpg`
    };
  }

  async getLocation(userId) {
    const location = await this.redis.hgetall(`location:${userId}`);

    return {
      lat: parseFloat(location.lat),
      lon: parseFloat(location.lon),
      timestamp: parseInt(location.timestamp)
    };
  }
}

const service = new NearbyService();
```

### 3. Geohash-based Approach (Альтернатива Redis)

```javascript
class GeohashNearbyService {
  async findNearbyWithGeohash(lat, lon, radiusKm, precision = 6) {
    const centerGeohash = Geohash.encode(lat, lon, precision);

    // Получаем соседние cells
    const neighbors = Geohash.neighbors(centerGeohash);
    const cellsToSearch = [centerGeohash, ...neighbors];

    console.log('Searching cells:', cellsToSearch);

    // Получаем пользователей из всех cells
    const pipeline = this.redis.pipeline();

    for (const cell of cellsToSearch) {
      pipeline.smembers(`geohash:${cell}`);
    }

    const results = await pipeline.exec();

    // Собираем всех пользователей
    const allUserIds = new Set();

    for (const [err, userIds] of results) {
      if (!err) {
        userIds.forEach(id => allUserIds.add(id));
      }
    }

    // Фильтруем по точному расстоянию
    const nearbyUsers = [];

    for (const userId of allUserIds) {
      const userLocation = await this.getLocation(userId);

      if (!userLocation) continue;

      const distance = this.haversineDistance(
        lat,
        lon,
        userLocation.lat,
        userLocation.lon
      );

      if (distance <= radiusKm) {
        nearbyUsers.push({
          userId,
          distance,
          location: userLocation
        });
      }
    }

    // Сортируем по расстоянию
    nearbyUsers.sort((a, b) => a.distance - b.distance);

    return nearbyUsers;
  }

  async updateLocationWithGeohash(userId, lat, lon) {
    const geohash = Geohash.encode(lat, lon, 6);

    // Удаляем из старого geohash cell (если был)
    const oldGeohash = await this.redis.hget(`location:${userId}`, 'geohash');

    if (oldGeohash) {
      await this.redis.srem(`geohash:${oldGeohash}`, userId);
    }

    // Добавляем в новый geohash cell
    await this.redis.sadd(`geohash:${geohash}`, userId);

    // Обновляем location metadata
    await this.redis.hset(
      `location:${userId}`,
      'lat', lat,
      'lon', lon,
      'geohash', geohash,
      'timestamp', Date.now()
    );
  }

  haversineDistance(lat1, lon1, lat2, lon2) {
    const R = 6371; // km
    const dLat = this.toRadians(lat2 - lat1);
    const dLon = this.toRadians(lon2 - lon1);

    const a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(this.toRadians(lat1)) *
      Math.cos(this.toRadians(lat2)) *
      Math.sin(dLon / 2) *
      Math.sin(dLon / 2);

    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

    return R * c;
  }

  toRadians(degrees) {
    return degrees * (Math.PI / 180);
  }
}
```

## Оптимизации

### 1. Location Throttling

```javascript
class LocationThrottler {
  async shouldUpdate(userId, lat, lon) {
    const lastLocation = await redis.hgetall(`location:${userId}`);

    if (!lastLocation.lat || !lastLocation.lon) {
      return true; // первое обновление
    }

    // Проверяем: достаточно ли сдвинулся пользователь
    const distance = this.haversineDistance(
      parseFloat(lastLocation.lat),
      parseFloat(lastLocation.lon),
      lat,
      lon
    );

    const MIN_DISTANCE = 0.1; // km (100 meters)

    if (distance < MIN_DISTANCE) {
      console.log(`User ${userId} moved only ${distance}km, skipping update`);
      return false;
    }

    // Проверяем: прошло ли достаточно времени
    const lastUpdate = parseInt(lastLocation.timestamp);
    const now = Date.now();
    const MIN_INTERVAL = 10000; // 10 seconds

    if (now - lastUpdate < MIN_INTERVAL) {
      console.log(`User ${userId} updated too recently, skipping`);
      return false;
    }

    return true;
  }
}
```

### 2. Caching Nearby Results

```javascript
async findNearbyWithCache(userId, radiusKm, limit) {
  const cacheKey = `nearby:${userId}:${radiusKm}:${limit}`;

  // Проверяем cache
  const cached = await redis.get(cacheKey);

  if (cached) {
    return JSON.parse(cached);
  }

  // Выполняем поиск
  const nearby = await this.findNearby(userId, radiusKm, limit);

  // Кэшируем (TTL 30 sec)
  await redis.setex(cacheKey, 30, JSON.stringify(nearby));

  return nearby;
}
```

### 3. Sharding by Region

```javascript
class RegionalLocationService {
  constructor() {
    this.regions = {
      'us-west': new Redis({ host: 'redis-us-west' }),
      'us-east': new Redis({ host: 'redis-us-east' }),
      'eu': new Redis({ host: 'redis-eu' }),
      'asia': new Redis({ host: 'redis-asia' })
    };
  }

  getRegion(lat, lon) {
    // Упрощённая логика — можно улучшить
    if (lon < -100) return 'us-west';
    if (lon < -50) return 'us-east';
    if (lon < 60) return 'eu';
    return 'asia';
  }

  async updateLocation(userId, lat, lon) {
    const region = this.getRegion(lat, lon);
    const redis = this.regions[region];

    await redis.geoadd('user_locations', lon, lat, userId);
  }
}
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const locationUpdates = new prometheus.Counter({
  name: 'location_updates_total',
  help: 'Total location updates'
});

const nearbySearches = new prometheus.Counter({
  name: 'nearby_searches_total',
  help: 'Total nearby searches'
});

const nearbySearchLatency = new prometheus.Histogram({
  name: 'nearby_search_duration_seconds',
  help: 'Nearby search latency',
  buckets: [0.01, 0.05, 0.1, 0.5, 1]
});

const activeLocations = new prometheus.Gauge({
  name: 'active_locations_count',
  help: 'Number of active user locations'
});

// Периодически обновляем
setInterval(async () => {
  const count = await redis.zcard('user_locations');
  activeLocations.set(count);
}, 10000);
```

## Сравнение подходов

| Подход | Latency | Scalability | Accuracy | Complexity |
|--------|---------|-------------|----------|------------|
| Redis GEORADIUS | < 10ms | Отличная (in-memory) | Высокая | Низкая |
| Geohash | < 50ms | Хорошая | Средняя | Средняя |
| QuadTree | < 100ms | Средняя (rebuild cost) | Высокая | Высокая |
| Database (PostGIS) | 100-500ms | Средняя | Очень высокая | Низкая |

**Рекомендация**: Redis GEORADIUS для production.

## Что читать дальше

- **Урок 45**: Search System — geospatial search с Elasticsearch
- **Урок 49**: Ride-sharing System — matching drivers & riders
- **Урок 16**: Caching Strategies — кэширование результатов

## Проверь себя

1. Как работает Geohash? В чём его преимущество перед обычными координатами?
2. Почему Redis GEORADIUS лучше чем SQL query с PostGIS для real-time поиска?
3. Спроектируйте nearby search для 100M active users.
4. Как обработать edge case: пользователи на границе geohash cells?
5. Реализуйте privacy zones (user не виден в определённых локациях, например дома).

---
**Предыдущий урок**: [Урок 45: Search System](45-search.md)
**Следующий урок**: [Урок 47: Social Network (Полный дизайн)](47-social-network.md)
