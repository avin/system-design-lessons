# Урок 50: Ride-sharing System (Uber/Lyft)

## Введение

Ride-sharing System — это комплексная real-time система, которая соединяет водителей и пассажиров, оптимизирует маршруты, управляет ценообразованием и обрабатывает платежи. Это одна из самых сложных geospatial систем с требованиями к ultra-low latency и высокой точности.

В этом уроке мы спроектируем систему уровня Uber/Lyft, способную обслуживать миллионы поездок в день по всему миру.

**Компоненты:**
- Real-time location tracking
- Driver-rider matching
- Route optimization
- Dynamic pricing (surge)
- Payment processing
- Trip management

## Требования к системе

### Функциональные требования

1. **Rider App**:
   - Запрос поездки (pickup, destination)
   - Просмотр доступных водителей
   - Отслеживание водителя в реальном времени
   - Оплата

2. **Driver App**:
   - Обновление локации
   - Приём/отклонение запросов
   - Навигация к пассажиру и destination
   - Earnings tracking

3. **Matching**:
   - Поиск ближайших доступных водителей
   - Оптимальное назначение (минимизация ETA)

4. **Pricing**:
   - Базовый расчёт стоимости
   - Surge pricing (динамическое повышение цены)

5. **Trip Management**:
   - Start trip, end trip
   - История поездок
   - Рейтинги

### Нефункциональные требования

1. **Scalability**:
   - 100M users
   - 10M active drivers
   - 50M trips/day ≈ 578 trips/sec

2. **Latency**:
   - Location update: < 100ms
   - Driver matching: < 5 sec
   - ETA calculation: < 1 sec

3. **Availability**: 99.99%

4. **Accuracy**:
   - Location: ~10 meters
   - ETA: ±2 minutes

5. **Real-time**:
   - Location updates: every 3-5 seconds
   - Live tracking during trip

### Back-of-the-envelope

```
Trips: 50M/day ≈ 578 trips/sec
Active drivers: 10M (1M online at any time)
Location updates: 1M drivers × 20 updates/min = 20M updates/min ≈ 333K updates/sec

Storage:
- Trip data: 1KB × 50M/day = 50 GB/day
- Location history: 100 bytes × 20M updates/min × 1440 min = 2.88 TB/day
- 1 year: 50 GB × 365 + 2.88 TB × 365 ≈ 1 PB

Geospatial queries:
- Rider requests ride → search nearby drivers (5 km radius)
- Average: 100 drivers in radius
- QPS: 578 trip requests/sec
```

## Архитектура высокого уровня

```
┌──────────┐          ┌──────────┐
│  Rider   │          │  Driver  │
│   App    │          │   App    │
└────┬─────┘          └────┬─────┘
     │                     │
     │  WebSocket          │  WebSocket
     │                     │
     ↓                     ↓
┌────────────────────────────────────┐
│       API Gateway / Load Balancer  │
└────┬──────┬──────┬──────┬──────────┘
     │      │      │      │
     ↓      ↓      ↓      ↓
┌────────┐┌────┐┌──────┐┌────────┐
│Location││Match││Route││Payment │
│Service ││Svc  ││Svc  ││Service │
└───┬────┘└─┬──┘└──┬───┘└────────┘
    │       │      │
    ↓       ↓      ↓
┌──────────────────────┐
│   Redis Geospatial   │
│  (Driver Locations)  │
└──────────────────────┘
    │
    ↓
┌──────────────────────┐
│   Trip Database      │
│  (PostgreSQL/MySQL)  │
└──────────────────────┘
```

## Ключевые компоненты

### 1. Location Service

```javascript
const express = require('express');
const Redis = require('ioredis');
const WebSocket = require('ws');

class LocationService {
  constructor() {
    this.redis = new Redis();
    this.wss = new WebSocket.Server({ port: 8080 });
    this.driverConnections = new Map(); // driverId → WebSocket

    this.setupWebSocket();
    this.setupHTTP();
  }

  setupWebSocket() {
    this.wss.on('connection', (ws, req) => {
      ws.on('message', async (data) => {
        try {
          const message = JSON.parse(data);

          if (message.type === 'auth') {
            await this.handleAuth(ws, message);
          } else if (message.type === 'location_update') {
            await this.handleLocationUpdate(ws, message);
          }
        } catch (error) {
          console.error('WebSocket error:', error);
        }
      });

      ws.on('close', () => {
        if (ws.driverId) {
          this.handleDisconnect(ws.driverId);
        }
      });
    });
  }

  setupHTTP() {
    const app = express();
    app.use(express.json());

    // Get nearby drivers
    app.get('/drivers/nearby', async (req, res) => {
      try {
        const { lat, lon, radius = 5 } = req.query;

        const drivers = await this.getNearbyDrivers(
          parseFloat(lat),
          parseFloat(lon),
          parseFloat(radius)
        );

        res.json({ drivers });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Get driver location
    app.get('/drivers/:driverId/location', async (req, res) => {
      try {
        const { driverId } = req.params;

        const location = await this.getDriverLocation(driverId);

        res.json(location);
      } catch (error) {
        res.status(404).json({ error: 'Driver not found' });
      }
    });

    app.listen(3001, () => {
      console.log('Location Service listening on port 3001');
    });
  }

  async handleAuth(ws, message) {
    const { driverId, token } = message;

    // Verify token
    const valid = await this.verifyToken(token);

    if (!valid) {
      ws.close();
      return;
    }

    ws.driverId = driverId;
    this.driverConnections.set(driverId, ws);

    ws.send(JSON.stringify({ type: 'auth', success: true }));
  }

  async handleLocationUpdate(ws, message) {
    const { lat, lon, heading, speed } = message;
    const driverId = ws.driverId;

    if (!driverId) return;

    // Обновляем локацию в Redis Geospatial
    await this.updateDriverLocation(driverId, lat, lon, {
      heading,
      speed,
      timestamp: Date.now()
    });

    // Если водитель в поездке — отправляем локацию пассажиру
    const activeTrip = await this.getActiveTrip(driverId);

    if (activeTrip) {
      await this.notifyRider(activeTrip.riderId, {
        type: 'driver_location',
        driverId,
        lat,
        lon,
        heading,
        eta: await this.calculateETA(driverId, activeTrip)
      });
    }
  }

  async updateDriverLocation(driverId, lat, lon, metadata) {
    // Redis GEOADD для geospatial indexing
    await this.redis.geoadd('drivers:online', lon, lat, driverId);

    // Сохраняем метаданные
    await this.redis.hset(`driver:${driverId}`, {
      lat,
      lon,
      heading: metadata.heading,
      speed: metadata.speed,
      timestamp: metadata.timestamp
    });

    // TTL: 60 секунд (если нет обновлений — считаем offline)
    await this.redis.expire(`driver:${driverId}`, 60);
  }

  async getNearbyDrivers(lat, lon, radiusKm) {
    // Redis GEORADIUS
    const result = await this.redis.georadius(
      'drivers:online',
      lon,
      lat,
      radiusKm,
      'km',
      'WITHDIST',
      'WITHCOORD',
      'ASC',
      'COUNT',
      50
    );

    // Загружаем детали водителей
    const drivers = [];

    for (const [driverId, distance, coords] of result) {
      const details = await this.redis.hgetall(`driver:${driverId}`);

      if (Object.keys(details).length === 0) continue;

      // Проверяем статус водителя
      const status = await this.getDriverStatus(driverId);

      if (status !== 'available') continue;

      drivers.push({
        driverId,
        lat: parseFloat(coords[1]),
        lon: parseFloat(coords[0]),
        distance: parseFloat(distance),
        heading: parseInt(details.heading),
        rating: await this.getDriverRating(driverId)
      });
    }

    return drivers;
  }

  async getDriverLocation(driverId) {
    const details = await this.redis.hgetall(`driver:${driverId}`);

    if (Object.keys(details).length === 0) {
      throw new Error('Driver not found');
    }

    return {
      lat: parseFloat(details.lat),
      lon: parseFloat(details.lon),
      heading: parseInt(details.heading),
      speed: parseFloat(details.speed),
      timestamp: parseInt(details.timestamp)
    };
  }

  async getDriverStatus(driverId) {
    return await this.redis.get(`driver:${driverId}:status`) || 'offline';
  }

  async getDriverRating(driverId) {
    const rating = await this.redis.get(`driver:${driverId}:rating`);
    return rating ? parseFloat(rating) : 4.5;
  }

  async getActiveTrip(driverId) {
    const tripId = await this.redis.get(`driver:${driverId}:active_trip`);

    if (!tripId) return null;

    const trip = await db.query('SELECT * FROM trips WHERE id = $1', [tripId]);
    return trip.rows[0];
  }

  async notifyRider(riderId, message) {
    // Отправляем через WebSocket или push notification
    // (реализация зависит от подключения)
  }

  async calculateETA(driverId, trip) {
    // Используем маршрутизацию для расчёта ETA
    const routeService = require('./route-service');

    const driverLocation = await this.getDriverLocation(driverId);

    const destination = trip.status === 'picking_up'
      ? { lat: trip.pickup_lat, lon: trip.pickup_lon }
      : { lat: trip.destination_lat, lon: trip.destination_lon };

    const route = await routeService.getRoute(driverLocation, destination);

    return Math.ceil(route.duration / 60); // minutes
  }

  handleDisconnect(driverId) {
    this.driverConnections.delete(driverId);

    // Помечаем водителя как offline (не удаляем из Redis — TTL сделает это)
    console.log(`Driver ${driverId} disconnected`);
  }

  async verifyToken(token) {
    // JWT verification
    const jwt = require('jsonwebtoken');

    try {
      jwt.verify(token, process.env.JWT_SECRET);
      return true;
    } catch (error) {
      return false;
    }
  }
}

const service = new LocationService();
```

### 2. Matching Service

```javascript
class MatchingService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
    this.locationService = require('./location-service');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Request ride
    app.post('/rides/request', async (req, res) => {
      try {
        const {
          riderId,
          pickupLat,
          pickupLon,
          destinationLat,
          destinationLon,
          rideType
        } = req.body;

        const rideRequest = await this.requestRide({
          riderId,
          pickupLat,
          pickupLon,
          destinationLat,
          destinationLon,
          rideType
        });

        res.json(rideRequest);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Cancel ride request
    app.post('/rides/:rideRequestId/cancel', async (req, res) => {
      try {
        const { rideRequestId } = req.params;

        await this.cancelRideRequest(rideRequestId);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3002, () => {
      console.log('Matching Service listening on port 3002');
    });
  }

  async requestRide({
    riderId,
    pickupLat,
    pickupLon,
    destinationLat,
    destinationLon,
    rideType
  }) {
    const rideRequestId = this.generateRideRequestId();

    // Сохраняем ride request
    await this.db.query(
      `INSERT INTO ride_requests
       (id, rider_id, pickup_lat, pickup_lon, destination_lat, destination_lon,
        ride_type, status, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7, 'searching', NOW())`,
      [
        rideRequestId,
        riderId,
        pickupLat,
        pickupLon,
        destinationLat,
        destinationLon,
        rideType
      ]
    );

    // Вычисляем примерную стоимость
    const fareEstimate = await this.calculateFare(
      pickupLat,
      pickupLon,
      destinationLat,
      destinationLon,
      rideType
    );

    // Запускаем поиск водителя (асинхронно)
    setImmediate(() => this.findDriver(rideRequestId));

    return {
      rideRequestId,
      fareEstimate,
      status: 'searching'
    };
  }

  async findDriver(rideRequestId) {
    console.log(`Finding driver for ride request ${rideRequestId}`);

    // Получаем детали запроса
    const request = await this.getRideRequest(rideRequestId);

    // Находим nearby drivers
    const nearbyDrivers = await this.locationService.getNearbyDrivers(
      request.pickup_lat,
      request.pickup_lon,
      10 // 10 km radius
    );

    if (nearbyDrivers.length === 0) {
      // Нет доступных водителей — увеличиваем радиус или уведомляем rider
      await this.notifyNoDriversAvailable(request.rider_id);
      return;
    }

    // Сортируем по оптимальности (расстояние + rating)
    const scored = nearbyDrivers.map(driver => ({
      ...driver,
      score: this.calculateDriverScore(driver)
    }));

    scored.sort((a, b) => b.score - a.score);

    // Отправляем предложение лучшим водителям (параллельно)
    const MAX_OFFERS = 5;
    const offers = scored.slice(0, MAX_OFFERS);

    for (const driver of offers) {
      await this.sendRideOffer(rideRequestId, driver.driverId);
    }

    // Ждём подтверждения от водителя (timeout 30 секунд)
    setTimeout(() => {
      this.checkMatchStatus(rideRequestId);
    }, 30000);
  }

  calculateDriverScore(driver) {
    // Score = weighted combination of factors
    const distanceScore = Math.max(0, 10 - driver.distance); // closer = better
    const ratingScore = driver.rating * 2; // rating weight

    return distanceScore + ratingScore;
  }

  async sendRideOffer(rideRequestId, driverId) {
    // Отправляем через WebSocket или push notification
    const request = await this.getRideRequest(rideRequestId);

    const offer = {
      type: 'ride_offer',
      rideRequestId,
      pickupLat: request.pickup_lat,
      pickupLon: request.pickup_lon,
      destinationLat: request.destination_lat,
      destinationLon: request.destination_lon,
      fareEstimate: await this.calculateFare(
        request.pickup_lat,
        request.pickup_lon,
        request.destination_lat,
        request.destination_lon,
        request.ride_type
      )
    };

    // Сохраняем offer (для tracking)
    await this.redis.setex(
      `offer:${rideRequestId}:${driverId}`,
      30,
      JSON.stringify(offer)
    );

    // Отправляем водителю
    await this.notifyDriver(driverId, offer);
  }

  async acceptRideOffer(rideRequestId, driverId) {
    // Проверяем: ride request ещё активен?
    const request = await this.getRideRequest(rideRequestId);

    if (request.status !== 'searching') {
      return { success: false, reason: 'Ride already matched' };
    }

    // Атомарно обновляем статус
    const result = await this.db.query(
      `UPDATE ride_requests
       SET status = 'matched', matched_driver_id = $1
       WHERE id = $2 AND status = 'searching'
       RETURNING *`,
      [driverId, rideRequestId]
    );

    if (result.rows.length === 0) {
      return { success: false, reason: 'Ride already matched by another driver' };
    }

    // Создаём trip
    const tripId = await this.createTrip(rideRequestId, driverId);

    // Обновляем статус водителя
    await this.redis.set(`driver:${driverId}:status`, 'on_trip');
    await this.redis.set(`driver:${driverId}:active_trip`, tripId);

    // Уведомляем rider
    await this.notifyRider(request.rider_id, {
      type: 'driver_matched',
      tripId,
      driver: await this.getDriverInfo(driverId)
    });

    return { success: true, tripId };
  }

  async createTrip(rideRequestId, driverId) {
    const request = await this.getRideRequest(rideRequestId);
    const tripId = this.generateTripId();

    await this.db.query(
      `INSERT INTO trips
       (id, rider_id, driver_id, pickup_lat, pickup_lon,
        destination_lat, destination_lon, status, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7, 'picking_up', NOW())`,
      [
        tripId,
        request.rider_id,
        driverId,
        request.pickup_lat,
        request.pickup_lon,
        request.destination_lat,
        request.destination_lon
      ]
    );

    return tripId;
  }

  async calculateFare(pickupLat, pickupLon, destLat, destLon, rideType) {
    // Получаем расстояние и время маршрута
    const routeService = require('./route-service');

    const route = await routeService.getRoute(
      { lat: pickupLat, lon: pickupLon },
      { lat: destLat, lon: destLon }
    );

    // Базовая стоимость
    const baseFare = 5.0;
    const perKm = 1.5;
    const perMinute = 0.3;

    let fare = baseFare +
               (route.distance * perKm) +
               (route.duration / 60 * perMinute);

    // Ride type multiplier
    const multipliers = {
      'economy': 1.0,
      'comfort': 1.5,
      'premium': 2.0
    };

    fare *= multipliers[rideType] || 1.0;

    // Surge pricing
    const surgeMultiplier = await this.getSurgeMultiplier(pickupLat, pickupLon);
    fare *= surgeMultiplier;

    return {
      baseFare: fare.toFixed(2),
      surgeMultiplier,
      estimatedDistance: route.distance.toFixed(2),
      estimatedDuration: Math.ceil(route.duration / 60)
    };
  }

  async getSurgeMultiplier(lat, lon) {
    // Surge pricing на основе demand/supply в области
    const geohash = this.getGeohash(lat, lon, 5); // precision 5 (~5km cell)

    const demandKey = `demand:${geohash}`;
    const supplyKey = `supply:${geohash}`;

    const demand = parseInt(await this.redis.get(demandKey)) || 0;
    const supply = parseInt(await this.redis.get(supplyKey)) || 1;

    const ratio = demand / supply;

    // Surge multiplier: 1.0 (normal) до 3.0 (high demand)
    if (ratio < 1) return 1.0;
    if (ratio < 2) return 1.2;
    if (ratio < 3) return 1.5;
    if (ratio < 5) return 2.0;
    return 3.0;
  }

  async getRideRequest(rideRequestId) {
    const result = await this.db.query(
      'SELECT * FROM ride_requests WHERE id = $1',
      [rideRequestId]
    );

    return result.rows[0];
  }

  async notifyDriver(driverId, message) {
    // WebSocket или push notification
  }

  async notifyRider(riderId, message) {
    // WebSocket или push notification
  }

  getGeohash(lat, lon, precision) {
    const Geohash = require('./geohash');
    return Geohash.encode(lat, lon, precision);
  }

  generateRideRequestId() {
    return require('crypto').randomUUID();
  }

  generateTripId() {
    const Snowflake = require('./snowflake');
    return new Snowflake(1, 1).generate().toString();
  }
}

const service = new MatchingService();
```

### 3. Trip Service

```javascript
class TripService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Start trip
    app.post('/trips/:tripId/start', async (req, res) => {
      try {
        const { tripId } = req.params;
        const { driverId } = req.body;

        await this.startTrip(tripId, driverId);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // End trip
    app.post('/trips/:tripId/end', async (req, res) => {
      try {
        const { tripId } = req.params;
        const { driverId, endLat, endLon } = req.body;

        const fare = await this.endTrip(tripId, driverId, endLat, endLon);

        res.json({ success: true, fare });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Get trip details
    app.get('/trips/:tripId', async (req, res) => {
      try {
        const { tripId } = req.params;

        const trip = await this.getTrip(tripId);

        res.json(trip);
      } catch (error) {
        res.status(404).json({ error: 'Trip not found' });
      }
    });

    // Rate trip
    app.post('/trips/:tripId/rate', async (req, res) => {
      try {
        const { tripId } = req.params;
        const { userId, rating, comment } = req.body;

        await this.rateTrip(tripId, userId, rating, comment);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3003, () => {
      console.log('Trip Service listening on port 3003');
    });
  }

  async startTrip(tripId, driverId) {
    // Проверяем: водитель на месте pickup
    const trip = await this.getTrip(tripId);

    if (trip.driver_id !== driverId) {
      throw new Error('Unauthorized');
    }

    // Обновляем статус
    await this.db.query(
      `UPDATE trips
       SET status = 'in_progress', started_at = NOW()
       WHERE id = $1`,
      [tripId]
    );

    // Уведомляем rider
    await this.notifyRider(trip.rider_id, {
      type: 'trip_started',
      tripId
    });
  }

  async endTrip(tripId, driverId, endLat, endLon) {
    const trip = await this.getTrip(tripId);

    if (trip.driver_id !== driverId) {
      throw new Error('Unauthorized');
    }

    // Вычисляем финальную стоимость
    const fare = await this.calculateFinalFare(trip, endLat, endLon);

    // Обновляем trip
    await this.db.query(
      `UPDATE trips
       SET status = 'completed',
           ended_at = NOW(),
           end_lat = $1,
           end_lon = $2,
           fare = $3
       WHERE id = $4`,
      [endLat, endLon, fare, tripId]
    );

    // Обновляем статус водителя
    await this.redis.set(`driver:${driverId}:status`, 'available');
    await this.redis.del(`driver:${driverId}:active_trip`);

    // Обрабатываем платёж
    await this.processPayment(trip.rider_id, fare);

    // Уведомляем rider
    await this.notifyRider(trip.rider_id, {
      type: 'trip_completed',
      tripId,
      fare
    });

    return fare;
  }

  async calculateFinalFare(trip, endLat, endLon) {
    // Используем фактическое расстояние и время
    const distance = this.haversineDistance(
      trip.pickup_lat,
      trip.pickup_lon,
      endLat,
      endLon
    );

    const duration = (Date.now() - new Date(trip.started_at).getTime()) / 1000;

    const baseFare = 5.0;
    const perKm = 1.5;
    const perMinute = 0.3;

    let fare = baseFare +
               (distance * perKm) +
               (duration / 60 * perMinute);

    // Применяем surge (если был)
    const surgeMultiplier = await this.getTripSurgeMultiplier(trip.id);
    fare *= surgeMultiplier;

    return fare.toFixed(2);
  }

  async rateTrip(tripId, userId, rating, comment) {
    const trip = await this.getTrip(tripId);

    // Определяем: rider или driver оценивает
    const isRider = trip.rider_id === userId;
    const targetId = isRider ? trip.driver_id : trip.rider_id;

    // Сохраняем рейтинг
    await this.db.query(
      `INSERT INTO ratings
       (trip_id, from_user_id, to_user_id, rating, comment, created_at)
       VALUES ($1, $2, $3, $4, $5, NOW())`,
      [tripId, userId, targetId, rating, comment]
    );

    // Обновляем средний рейтинг
    await this.updateAverageRating(targetId);
  }

  async updateAverageRating(userId) {
    const result = await this.db.query(
      `SELECT AVG(rating) as avg_rating
       FROM ratings
       WHERE to_user_id = $1`,
      [userId]
    );

    const avgRating = parseFloat(result.rows[0].avg_rating);

    // Сохраняем в Redis для быстрого доступа
    await this.redis.set(`user:${userId}:rating`, avgRating.toFixed(2));
  }

  async getTrip(tripId) {
    const result = await this.db.query(
      'SELECT * FROM trips WHERE id = $1',
      [tripId]
    );

    if (result.rows.length === 0) {
      throw new Error('Trip not found');
    }

    return result.rows[0];
  }

  async processPayment(riderId, amount) {
    // Интеграция с payment gateway (Stripe)
    // Реализация в Payment Service
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

const service = new TripService();
```

### 4. Route Service

```javascript
const axios = require('axios');

class RouteService {
  async getRoute(origin, destination) {
    // Используем Google Maps Directions API или OSRM
    const response = await axios.get(
      'https://maps.googleapis.com/maps/api/directions/json',
      {
        params: {
          origin: `${origin.lat},${origin.lon}`,
          destination: `${destination.lat},${destination.lon}`,
          key: process.env.GOOGLE_MAPS_API_KEY
        }
      }
    );

    const route = response.data.routes[0];
    const leg = route.legs[0];

    return {
      distance: leg.distance.value / 1000, // km
      duration: leg.duration.value, // seconds
      polyline: route.overview_polyline.points,
      steps: leg.steps.map(step => ({
        instruction: step.html_instructions,
        distance: step.distance.value,
        duration: step.duration.value
      }))
    };
  }

  async getETA(origin, destination) {
    const route = await this.getRoute(origin, destination);
    return Math.ceil(route.duration / 60); // minutes
  }
}

module.exports = new RouteService();
```

## Оптимизации

### 1. Driver Location Sharding

```javascript
class ShardedLocationService {
  constructor() {
    // Шардируем по geohash prefix
    this.shards = {
      '9': new Redis({ host: 'redis-shard-9' }), // California
      'd': new Redis({ host: 'redis-shard-d' }), // New York
      'u': new Redis({ host: 'redis-shard-u' }), // Europe
      // ...
    };
  }

  getShard(lat, lon) {
    const geohash = Geohash.encode(lat, lon, 1);
    return this.shards[geohash[0]] || this.shards['9'];
  }

  async updateDriverLocation(driverId, lat, lon) {
    const shard = this.getShard(lat, lon);
    await shard.geoadd('drivers:online', lon, lat, driverId);
  }

  async getNearbyDrivers(lat, lon, radius) {
    const shard = this.getShard(lat, lon);
    return await shard.georadius('drivers:online', lon, lat, radius, 'km');
  }
}
```

### 2. Matching Optimization с QuadTree

```javascript
class QuadTreeMatcher {
  constructor() {
    this.quadtree = new QuadTree({
      minLat: -90,
      maxLat: 90,
      minLon: -180,
      maxLon: 180
    });
  }

  async findBestMatch(request) {
    // Находим водителей в QuadTree (быстрее чем Redis для in-memory)
    const candidates = this.quadtree.query({
      lat: request.pickup_lat,
      lon: request.pickup_lon,
      radius: 5 // km
    });

    // Оптимизация: batch geocoding для ETA расчёта
    const etas = await this.batchCalculateETA(candidates, request);

    // Выбираем водителя с минимальным ETA
    const best = candidates.reduce((min, driver, i) => {
      return etas[i] < etas[min] ? i : min;
    }, 0);

    return candidates[best];
  }
}
```

### 3. Demand/Supply Tracking

```javascript
class DemandSupplyTracker {
  async trackDemand(lat, lon) {
    const geohash = Geohash.encode(lat, lon, 5);
    await redis.incr(`demand:${geohash}`);
    await redis.expire(`demand:${geohash}`, 3600); // 1 hour window
  }

  async trackSupply(lat, lon) {
    const geohash = Geohash.encode(lat, lon, 5);
    await redis.incr(`supply:${geohash}`);
    await redis.expire(`supply:${geohash}`, 3600);
  }

  async getSurgeAreas() {
    // Находим области с высоким demand/supply ratio
    const allGeohashes = await redis.keys('demand:*');
    const surgeAreas = [];

    for (const key of allGeohashes) {
      const geohash = key.split(':')[1];
      const demand = parseInt(await redis.get(`demand:${geohash}`)) || 0;
      const supply = parseInt(await redis.get(`supply:${geohash}`)) || 1;

      const ratio = demand / supply;

      if (ratio > 2) {
        surgeAreas.push({
          geohash,
          ratio,
          surgeMultiplier: this.calculateSurge(ratio)
        });
      }
    }

    return surgeAreas;
  }
}
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const activeTrips = new prometheus.Gauge({
  name: 'rideshare_active_trips',
  help: 'Number of active trips'
});

const onlineDrivers = new prometheus.Gauge({
  name: 'rideshare_online_drivers',
  help: 'Number of online drivers'
});

const matchingDuration = new prometheus.Histogram({
  name: 'rideshare_matching_duration_seconds',
  help: 'Time to match driver',
  buckets: [1, 5, 10, 30, 60, 120]
});

const tripDuration = new prometheus.Histogram({
  name: 'rideshare_trip_duration_minutes',
  help: 'Trip duration',
  buckets: [5, 10, 20, 30, 60, 120]
});

const surgeMultiplier = new prometheus.Histogram({
  name: 'rideshare_surge_multiplier',
  help: 'Surge pricing multiplier',
  labelNames: ['geohash'],
  buckets: [1.0, 1.2, 1.5, 2.0, 3.0]
});
```

## Schema Design

```sql
-- Trips
CREATE TABLE trips (
  id BIGINT PRIMARY KEY,
  rider_id BIGINT NOT NULL,
  driver_id BIGINT NOT NULL,
  pickup_lat DECIMAL(10, 8),
  pickup_lon DECIMAL(11, 8),
  destination_lat DECIMAL(10, 8),
  destination_lon DECIMAL(11, 8),
  end_lat DECIMAL(10, 8),
  end_lon DECIMAL(11, 8),
  status VARCHAR(20),
  fare DECIMAL(10, 2),
  created_at TIMESTAMP,
  started_at TIMESTAMP,
  ended_at TIMESTAMP,
  INDEX idx_rider (rider_id, created_at DESC),
  INDEX idx_driver (driver_id, created_at DESC),
  INDEX idx_status (status)
);

-- Ride Requests
CREATE TABLE ride_requests (
  id UUID PRIMARY KEY,
  rider_id BIGINT NOT NULL,
  pickup_lat DECIMAL(10, 8),
  pickup_lon DECIMAL(11, 8),
  destination_lat DECIMAL(10, 8),
  destination_lon DECIMAL(11, 8),
  ride_type VARCHAR(20),
  status VARCHAR(20),
  matched_driver_id BIGINT,
  created_at TIMESTAMP,
  INDEX idx_rider_status (rider_id, status)
);

-- Ratings
CREATE TABLE ratings (
  id SERIAL PRIMARY KEY,
  trip_id BIGINT NOT NULL,
  from_user_id BIGINT NOT NULL,
  to_user_id BIGINT NOT NULL,
  rating INT CHECK (rating >= 1 AND rating <= 5),
  comment TEXT,
  created_at TIMESTAMP,
  INDEX idx_to_user (to_user_id)
);
```

## Что читать дальше

- **Урок 46**: Nearby Friends — geospatial search
- **Урок 45**: Search System — поиск поездок
- **Урок 41**: Notification System — уведомления водителей/пассажиров

## Проверь себя

1. Как обрабатывать ситуацию, когда нет доступных водителей?
2. Спроектируйте систему для 1M concurrent trips.
3. Как реализовать carpooling (несколько пассажиров в одной поездке)?
4. Реализуйте scheduled rides (заказ поездки заранее).
5. Как оптимизировать matching для минимизации total wait time всех riders?

---

[← Урок 49: Video Streaming](49-video-streaming.md) | [Урок 51: E-commerce Platform →](51-ecommerce.md)
