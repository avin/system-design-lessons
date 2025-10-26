# Урок 49: Video Streaming Platform

## Введение

Video Streaming Platform — это одна из самых требовательных систем из-за огромных объёмов данных, необходимости поддержки различных устройств и форматов, и требований к низкой латентности. Системы типа YouTube, Netflix, Twitch обрабатывают петабайты видео и миллиарды просмотров ежедневно.

В этом уроке мы спроектируем масштабируемую платформу потокового видео, способную обслуживать миллионы пользователей с адаптивным качеством.

**Примеры:**
- **YouTube**: user-generated content, 500 hours uploaded/minute
- **Netflix**: professional content, 200M+ subscribers
- **Twitch**: live streaming
- **TikTok**: short-form video

## Требования к системе

### Функциональные требования

1. **Video Upload**:
   - Загрузка видео (до 10 GB)
   - Метаданные (title, description, tags)
   - Thumbnails

2. **Video Processing**:
   - Transcoding (различные разрешения: 360p, 720p, 1080p, 4K)
   - Adaptive bitrate streaming (HLS/DASH)
   - Thumbnail generation

3. **Video Playback**:
   - Адаптивное качество
   - Seek (перемотка)
   - Subtitles

4. **Recommendation**:
   - Персонализированные рекомендации
   - Trending videos

5. **Engagement**:
   - Likes, comments
   - Subscribe to channels
   - Watch history

### Нефункциональные требования

1. **Scalability**:
   - 1B users
   - 500M DAU
   - 1M concurrent streams

2. **Performance**:
   - Video start latency: < 1 sec
   - Buffering: < 2%
   - Upload processing: < 5 min for 1 hour video

3. **Availability**: 99.9%

4. **Storage**:
   - Petabytes of video
   - Cost-efficient

5. **Bandwidth**:
   - CDN delivery
   - Global reach

### Back-of-the-envelope

```
Users: 1B total, 500M DAU
Average watch time: 30 min/day
Average video bitrate: 5 Mbps

Daily bandwidth:
500M users × 30 min × 5 Mbps = 500M × 1800 sec × 5 Mb/sec
= 4.5 × 10^15 bits/day = 4.5 Pbps/day ≈ 562 TB/day

Storage:
New uploads: 500 hours/minute (YouTube scale)
500 × 60 min × 60 sec × 5 Mbps = 9 × 10^9 Mbps = 9 PB/day
With transcoding (5 resolutions): 9 PB × 5 = 45 PB/day
1 year: 45 PB × 365 = 16 EB (Exabytes)

Cost (S3):
$0.023/GB × 16M GB = $368K/day
```

## Архитектура высокого уровня

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     ↓
┌─────────────────┐
│   CDN (Video)   │
│ CloudFront/Akamai│
└────┬────────────┘
     │
     ↓
┌─────────────────┐
│  API Gateway    │
└────┬────────────┘
     │
     ├─────────┬─────────┬─────────┐
     ↓         ↓         ↓         ↓
┌────────┐┌────────┐┌────────┐┌────────┐
│ Upload ││ Video  ││Recommend││ User   │
│Service ││Service ││Service  ││Service │
└───┬────┘└───┬────┘└────┬───┘└────────┘
    │         │          │
    ↓         ↓          ↓
┌────────────────────────────┐
│    Message Queue (Kafka)   │
└──────┬──────────┬──────────┘
       │          │
       ↓          ↓
┌──────────┐┌────────────┐
│Transcoding││ Metadata   │
│  Worker  ││  Database  │
└─────┬────┘└────────────┘
      │
      ↓
┌──────────────────┐
│  Object Storage  │
│  (S3 / GCS)      │
└──────────────────┘
```

## Ключевые компоненты

### 1. Upload Service

```javascript
const express = require('express');
const multer = require('multer');
const AWS = require('aws-sdk');
const s3 = new AWS.S3();

class UploadService {
  constructor() {
    this.db = require('./database');
    this.kafka = require('./kafka');

    // Multer для multipart uploads
    this.upload = multer({
      limits: { fileSize: 10 * 1024 * 1024 * 1024 }, // 10 GB
      storage: multer.memoryStorage()
    });

    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Initiate upload (multipart)
    app.post('/upload/init', async (req, res) => {
      try {
        const { userId, filename, filesize, contentType } = req.body;

        const uploadId = await this.initiateMultipartUpload(
          userId,
          filename,
          filesize,
          contentType
        );

        res.json({ success: true, uploadId });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Upload part
    app.post('/upload/part', this.upload.single('file'), async (req, res) => {
      try {
        const { uploadId, partNumber } = req.body;
        const file = req.file;

        const etag = await this.uploadPart(uploadId, partNumber, file);

        res.json({ success: true, etag });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Complete upload
    app.post('/upload/complete', async (req, res) => {
      try {
        const { uploadId, parts, metadata } = req.body;

        const videoId = await this.completeUpload(uploadId, parts, metadata);

        res.json({ success: true, videoId });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Direct upload (small files)
    app.post('/upload/direct', this.upload.single('video'), async (req, res) => {
      try {
        const { userId } = req.body;
        const file = req.file;
        const metadata = JSON.parse(req.body.metadata);

        const videoId = await this.directUpload(userId, file, metadata);

        res.json({ success: true, videoId });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3001, () => {
      console.log('Upload Service listening on port 3001');
    });
  }

  async initiateMultipartUpload(userId, filename, filesize, contentType) {
    const uploadId = this.generateUploadId();
    const s3Key = `uploads/${userId}/${uploadId}/${filename}`;

    // Создаём multipart upload в S3
    const multipart = await s3.createMultipartUpload({
      Bucket: process.env.S3_BUCKET,
      Key: s3Key,
      ContentType: contentType
    }).promise();

    // Сохраняем метаданные в БД
    await this.db.query(
      `INSERT INTO uploads
       (id, user_id, filename, filesize, s3_key, s3_upload_id, status, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, 'in_progress', NOW())`,
      [uploadId, userId, filename, filesize, s3Key, multipart.UploadId]
    );

    return uploadId;
  }

  async uploadPart(uploadId, partNumber, file) {
    // Получаем upload metadata
    const upload = await this.getUpload(uploadId);

    // Загружаем part в S3
    const result = await s3.uploadPart({
      Bucket: process.env.S3_BUCKET,
      Key: upload.s3_key,
      PartNumber: parseInt(partNumber),
      UploadId: upload.s3_upload_id,
      Body: file.buffer
    }).promise();

    return result.ETag;
  }

  async completeUpload(uploadId, parts, metadata) {
    const upload = await this.getUpload(uploadId);

    // Завершаем multipart upload
    await s3.completeMultipartUpload({
      Bucket: process.env.S3_BUCKET,
      Key: upload.s3_key,
      UploadId: upload.s3_upload_id,
      MultipartUpload: {
        Parts: parts.map(p => ({
          ETag: p.etag,
          PartNumber: p.partNumber
        }))
      }
    }).promise();

    // Создаём video record
    const videoId = this.generateVideoId();

    await this.db.query(
      `INSERT INTO videos
       (id, user_id, title, description, filename, s3_key, status, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, 'processing', NOW())`,
      [
        videoId,
        upload.user_id,
        metadata.title,
        metadata.description,
        upload.filename,
        upload.s3_key
      ]
    );

    // Удаляем upload record
    await this.db.query('DELETE FROM uploads WHERE id = $1', [uploadId]);

    // Отправляем в очередь на обработку
    await this.publishForProcessing(videoId);

    return videoId;
  }

  async directUpload(userId, file, metadata) {
    const videoId = this.generateVideoId();
    const s3Key = `videos/${userId}/${videoId}/original.${this.getExtension(file.originalname)}`;

    // Загружаем в S3
    await s3.putObject({
      Bucket: process.env.S3_BUCKET,
      Key: s3Key,
      Body: file.buffer,
      ContentType: file.mimetype
    }).promise();

    // Создаём video record
    await this.db.query(
      `INSERT INTO videos
       (id, user_id, title, description, filename, s3_key, status, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, 'processing', NOW())`,
      [videoId, userId, metadata.title, metadata.description, file.originalname, s3Key]
    );

    // Отправляем на обработку
    await this.publishForProcessing(videoId);

    return videoId;
  }

  async publishForProcessing(videoId) {
    await this.kafka.send({
      topic: 'video-processing',
      messages: [{
        key: videoId,
        value: JSON.stringify({
          videoId,
          timestamp: Date.now()
        })
      }]
    });
  }

  async getUpload(uploadId) {
    const result = await this.db.query(
      'SELECT * FROM uploads WHERE id = $1',
      [uploadId]
    );

    if (result.rows.length === 0) {
      throw new Error('Upload not found');
    }

    return result.rows[0];
  }

  getExtension(filename) {
    return filename.split('.').pop();
  }

  generateUploadId() {
    return require('crypto').randomUUID();
  }

  generateVideoId() {
    const Snowflake = require('./snowflake');
    return new Snowflake(1, 1).generate().toString();
  }
}

const service = new UploadService();
```

### 2. Transcoding Worker

```javascript
const { Kafka } = require('kafkajs');
const ffmpeg = require('fluent-ffmpeg');
const fs = require('fs');
const path = require('path');

class TranscodingWorker {
  constructor() {
    this.kafka = new Kafka({
      clientId: 'transcoding-worker',
      brokers: ['kafka:9092']
    });

    this.consumer = this.kafka.consumer({ groupId: 'transcoders' });
    this.s3 = new AWS.S3();
    this.db = require('./database');

    // Разрешения для transcoding
    this.resolutions = [
      { name: '360p', width: 640, height: 360, bitrate: '800k' },
      { name: '480p', width: 854, height: 480, bitrate: '1400k' },
      { name: '720p', width: 1280, height: 720, bitrate: '2800k' },
      { name: '1080p', width: 1920, height: 1080, bitrate: '5000k' },
      { name: '1440p', width: 2560, height: 1440, bitrate: '8000k' }
    ];
  }

  async start() {
    await this.consumer.connect();
    await this.consumer.subscribe({ topic: 'video-processing' });

    console.log('Transcoding worker started');

    await this.consumer.run({
      eachMessage: async ({ message }) => {
        try {
          const { videoId } = JSON.parse(message.value.toString());
          await this.processVideo(videoId);
        } catch (error) {
          console.error('Transcoding error:', error);
        }
      }
    });
  }

  async processVideo(videoId) {
    console.log(`Processing video ${videoId}`);

    const startTime = Date.now();

    // Получаем video metadata
    const video = await this.getVideo(videoId);

    // Скачиваем original video из S3
    const inputPath = await this.downloadVideo(video.s3_key);

    // Получаем информацию о видео
    const videoInfo = await this.getVideoInfo(inputPath);

    // Transcode в различные разрешения
    const transcoded = await this.transcodeVideo(videoId, inputPath, videoInfo);

    // Генерируем thumbnails
    const thumbnails = await this.generateThumbnails(videoId, inputPath);

    // Создаём HLS manifest
    await this.createHLSManifest(videoId, transcoded);

    // Обновляем статус
    await this.db.query(
      `UPDATE videos
       SET status = 'ready',
           duration = $1,
           resolutions = $2,
           thumbnail_url = $3,
           processed_at = NOW()
       WHERE id = $4`,
      [videoInfo.duration, JSON.stringify(transcoded.map(t => t.name)), thumbnails[0], videoId]
    );

    // Очищаем временные файлы
    await this.cleanup(inputPath);

    console.log(`Video ${videoId} processed in ${Date.now() - startTime}ms`);
  }

  async downloadVideo(s3Key) {
    const tempPath = path.join('/tmp', `${Date.now()}_${path.basename(s3Key)}`);

    const fileStream = fs.createWriteStream(tempPath);
    const s3Stream = this.s3.getObject({
      Bucket: process.env.S3_BUCKET,
      Key: s3Key
    }).createReadStream();

    await new Promise((resolve, reject) => {
      s3Stream.pipe(fileStream);
      fileStream.on('finish', resolve);
      fileStream.on('error', reject);
    });

    return tempPath;
  }

  async getVideoInfo(inputPath) {
    return new Promise((resolve, reject) => {
      ffmpeg.ffprobe(inputPath, (err, metadata) => {
        if (err) return reject(err);

        const videoStream = metadata.streams.find(s => s.codec_type === 'video');

        resolve({
          duration: metadata.format.duration,
          width: videoStream.width,
          height: videoStream.height,
          bitrate: metadata.format.bit_rate
        });
      });
    });
  }

  async transcodeVideo(videoId, inputPath, videoInfo) {
    const results = [];

    // Фильтруем resolutions (не upscale)
    const targetResolutions = this.resolutions.filter(r =>
      r.height <= videoInfo.height
    );

    for (const resolution of targetResolutions) {
      console.log(`Transcoding to ${resolution.name}...`);

      const outputPath = `/tmp/${videoId}_${resolution.name}.mp4`;

      await this.transcode(inputPath, outputPath, resolution);

      // Загружаем в S3
      const s3Key = `videos/${videoId}/${resolution.name}.mp4`;
      await this.uploadToS3(outputPath, s3Key);

      results.push({
        name: resolution.name,
        s3Key,
        url: `https://cdn.example.com/${s3Key}`
      });

      // Удаляем временный файл
      fs.unlinkSync(outputPath);
    }

    return results;
  }

  async transcode(inputPath, outputPath, resolution) {
    return new Promise((resolve, reject) => {
      ffmpeg(inputPath)
        .videoCodec('libx264')
        .audioCodec('aac')
        .size(`${resolution.width}x${resolution.height}`)
        .videoBitrate(resolution.bitrate)
        .audioBitrate('128k')
        .outputOptions([
          '-preset fast',
          '-movflags +faststart' // для progressive download
        ])
        .output(outputPath)
        .on('end', resolve)
        .on('error', reject)
        .run();
    });
  }

  async generateThumbnails(videoId, inputPath) {
    const thumbnails = [];

    // Генерируем 5 thumbnails на разных временных метках
    const timestamps = ['10%', '25%', '50%', '75%', '90%'];

    for (let i = 0; i < timestamps.length; i++) {
      const outputPath = `/tmp/${videoId}_thumb_${i}.jpg`;

      await new Promise((resolve, reject) => {
        ffmpeg(inputPath)
          .screenshots({
            timestamps: [timestamps[i]],
            filename: path.basename(outputPath),
            folder: path.dirname(outputPath),
            size: '320x180'
          })
          .on('end', resolve)
          .on('error', reject);
      });

      // Загружаем в S3
      const s3Key = `videos/${videoId}/thumbnails/thumb_${i}.jpg`;
      await this.uploadToS3(outputPath, s3Key);

      thumbnails.push(`https://cdn.example.com/${s3Key}`);

      fs.unlinkSync(outputPath);
    }

    return thumbnails;
  }

  async createHLSManifest(videoId, transcoded) {
    // Создаём master playlist (m3u8)
    let manifest = '#EXTM3U\n';
    manifest += '#EXT-X-VERSION:3\n';

    for (const video of transcoded) {
      const bandwidth = this.getBandwidth(video.name);
      const resolution = this.getResolutionDimensions(video.name);

      manifest += `#EXT-X-STREAM-INF:BANDWIDTH=${bandwidth},RESOLUTION=${resolution}\n`;
      manifest += `${video.name}.m3u8\n`;
    }

    // Загружаем manifest в S3
    const s3Key = `videos/${videoId}/master.m3u8`;

    await this.s3.putObject({
      Bucket: process.env.S3_BUCKET,
      Key: s3Key,
      Body: manifest,
      ContentType: 'application/vnd.apple.mpegurl'
    }).promise();
  }

  async uploadToS3(filePath, s3Key) {
    const fileStream = fs.createReadStream(filePath);

    await this.s3.upload({
      Bucket: process.env.S3_BUCKET,
      Key: s3Key,
      Body: fileStream
    }).promise();
  }

  async getVideo(videoId) {
    const result = await this.db.query(
      'SELECT * FROM videos WHERE id = $1',
      [videoId]
    );

    return result.rows[0];
  }

  getBandwidth(resolution) {
    const map = {
      '360p': 800000,
      '480p': 1400000,
      '720p': 2800000,
      '1080p': 5000000,
      '1440p': 8000000
    };

    return map[resolution] || 5000000;
  }

  getResolutionDimensions(resolution) {
    const map = {
      '360p': '640x360',
      '480p': '854x480',
      '720p': '1280x720',
      '1080p': '1920x1080',
      '1440p': '2560x1440'
    };

    return map[resolution] || '1920x1080';
  }

  async cleanup(inputPath) {
    try {
      fs.unlinkSync(inputPath);
    } catch (error) {
      console.error('Cleanup error:', error);
    }
  }
}

const worker = new TranscodingWorker();
worker.start();
```

### 3. Video Service (Playback)

```javascript
class VideoService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Get video metadata
    app.get('/videos/:videoId', async (req, res) => {
      try {
        const { videoId } = req.params;

        const video = await this.getVideo(videoId);

        res.json(video);
      } catch (error) {
        res.status(404).json({ error: 'Video not found' });
      }
    });

    // Get playback URL (HLS manifest)
    app.get('/videos/:videoId/play', async (req, res) => {
      try {
        const { videoId } = req.params;
        const { userId } = req.query;

        const playbackUrl = await this.getPlaybackUrl(videoId, userId);

        res.json({ playbackUrl });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Record view
    app.post('/videos/:videoId/view', async (req, res) => {
      try {
        const { videoId } = req.params;
        const { userId, watchTime } = req.body;

        await this.recordView(videoId, userId, watchTime);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Like video
    app.post('/videos/:videoId/like', async (req, res) => {
      try {
        const { videoId } = req.params;
        const { userId } = req.body;

        await this.likeVideo(videoId, userId);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Search videos
    app.get('/videos/search', async (req, res) => {
      try {
        const { query, page = 1, limit = 20 } = req.query;

        const results = await this.searchVideos(query, page, limit);

        res.json(results);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3002, () => {
      console.log('Video Service listening on port 3002');
    });
  }

  async getVideo(videoId) {
    // Проверяем cache
    const cacheKey = `video:${videoId}`;
    const cached = await this.redis.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // Загружаем из БД
    const result = await this.db.query(
      `SELECT v.*, u.username as channel_name
       FROM videos v
       JOIN users u ON v.user_id = u.id
       WHERE v.id = $1 AND v.status = 'ready'`,
      [videoId]
    );

    if (result.rows.length === 0) {
      throw new Error('Video not found');
    }

    const video = result.rows[0];

    // Кэшируем (1 hour)
    await this.redis.setex(cacheKey, 3600, JSON.stringify(video));

    return video;
  }

  async getPlaybackUrl(videoId, userId) {
    // Генерируем signed URL для HLS manifest
    const video = await this.getVideo(videoId);

    // CloudFront signed URL
    const cloudFront = new AWS.CloudFront.Signer(
      process.env.CLOUDFRONT_KEY_PAIR_ID,
      process.env.CLOUDFRONT_PRIVATE_KEY
    );

    const url = `https://${process.env.CLOUDFRONT_DOMAIN}/videos/${videoId}/master.m3u8`;

    const signedUrl = cloudFront.getSignedUrl({
      url,
      expires: Math.floor(Date.now() / 1000) + 3600 // 1 hour
    });

    // Логируем playback start (для analytics)
    await this.logPlaybackStart(videoId, userId);

    return signedUrl;
  }

  async recordView(videoId, userId, watchTime) {
    // Записываем view (только если посмотрено > 30 секунд)
    if (watchTime < 30) return;

    // Проверяем: уже считали view?
    const viewKey = `view:${videoId}:${userId}`;
    const alreadyViewed = await this.redis.exists(viewKey);

    if (alreadyViewed) return;

    // Инкрементируем счётчик
    await this.db.query(
      'UPDATE videos SET views_count = views_count + 1 WHERE id = $1',
      [videoId]
    );

    // Сохраняем в watch history
    await this.db.query(
      `INSERT INTO watch_history (user_id, video_id, watch_time, watched_at)
       VALUES ($1, $2, $3, NOW())`,
      [userId, videoId, watchTime]
    );

    // Помечаем как просмотренное (TTL 24 hours)
    await this.redis.setex(viewKey, 86400, '1');
  }

  async likeVideo(videoId, userId) {
    // Проверяем: уже лайкнуто?
    const likeKey = `like:${videoId}:${userId}`;
    const alreadyLiked = await this.redis.sismember(`likes:${videoId}`, userId);

    if (alreadyLiked) return;

    // Добавляем like
    await this.redis.sadd(`likes:${videoId}`, userId);

    // Инкрементируем счётчик
    await this.db.query(
      'UPDATE videos SET likes_count = likes_count + 1 WHERE id = $1',
      [videoId]
    );
  }

  async searchVideos(query, page, limit) {
    // Используем Elasticsearch для поиска
    const esClient = require('./elasticsearch');

    const result = await esClient.search({
      index: 'videos',
      body: {
        query: {
          multi_match: {
            query,
            fields: ['title^3', 'description', 'tags'],
            fuzziness: 'AUTO'
          }
        },
        from: (page - 1) * limit,
        size: limit,
        sort: [
          { views_count: 'desc' },
          { created_at: 'desc' }
        ]
      }
    });

    return {
      videos: result.hits.hits.map(hit => hit._source),
      total: result.hits.total.value,
      page,
      limit
    };
  }

  async logPlaybackStart(videoId, userId) {
    // Логируем в analytics system
    const kafka = require('./kafka');

    await kafka.send({
      topic: 'video-analytics',
      messages: [{
        value: JSON.stringify({
          event: 'playback_start',
          videoId,
          userId,
          timestamp: Date.now()
        })
      }]
    });
  }
}

const service = new VideoService();
```

### 4. Recommendation Service

```javascript
class RecommendationService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
  }

  async getRecommendations(userId, limit = 20) {
    // Стратегия рекомендаций:
    // 1. Collaborative filtering (похожие пользователи)
    // 2. Content-based (похожие видео)
    // 3. Trending videos

    const recommendations = [];

    // Получаем watch history пользователя
    const watchHistory = await this.getWatchHistory(userId);

    // Collaborative filtering
    const collaborative = await this.getCollaborativeRecommendations(
      userId,
      watchHistory,
      limit / 3
    );

    recommendations.push(...collaborative);

    // Content-based (на основе последних просмотров)
    if (watchHistory.length > 0) {
      const lastWatched = watchHistory[0];
      const contentBased = await this.getContentBasedRecommendations(
        lastWatched.video_id,
        limit / 3
      );

      recommendations.push(...contentBased);
    }

    // Trending videos
    const trending = await this.getTrendingVideos(limit / 3);
    recommendations.push(...trending);

    // Удаляем дубликаты и уже просмотренные
    const seen = new Set(watchHistory.map(w => w.video_id));
    const unique = recommendations.filter(v =>
      !seen.has(v.id) && recommendations.indexOf(v) === recommendations.findIndex(r => r.id === v.id)
    );

    return unique.slice(0, limit);
  }

  async getWatchHistory(userId, limit = 50) {
    const result = await this.db.query(
      `SELECT video_id, watch_time, watched_at
       FROM watch_history
       WHERE user_id = $1
       ORDER BY watched_at DESC
       LIMIT $2`,
      [userId, limit]
    );

    return result.rows;
  }

  async getCollaborativeRecommendations(userId, watchHistory, limit) {
    // Находим похожих пользователей (cosine similarity на watch history)
    const videoIds = watchHistory.map(w => w.video_id);

    const result = await this.db.query(
      `SELECT wh.video_id, COUNT(*) as score
       FROM watch_history wh
       WHERE wh.user_id != $1
         AND wh.video_id IN (
           SELECT video_id FROM watch_history WHERE user_id = $1
         )
       GROUP BY wh.video_id
       ORDER BY score DESC
       LIMIT $2`,
      [userId, limit]
    );

    // Загружаем детали видео
    return this.loadVideos(result.rows.map(r => r.video_id));
  }

  async getContentBasedRecommendations(videoId, limit) {
    // Находим видео с похожими тегами/категорией
    const video = await this.getVideo(videoId);

    const result = await this.db.query(
      `SELECT id, title, views_count
       FROM videos
       WHERE id != $1
         AND (category = $2 OR tags && $3)
         AND status = 'ready'
       ORDER BY views_count DESC
       LIMIT $4`,
      [videoId, video.category, video.tags, limit]
    );

    return result.rows;
  }

  async getTrendingVideos(limit) {
    // Кэшируем trending videos (обновляем каждые 10 минут)
    const cacheKey = 'trending_videos';
    const cached = await this.redis.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // Trending = много просмотров за последние 24 часа
    const result = await this.db.query(
      `SELECT v.id, v.title, v.thumbnail_url, v.views_count,
              COUNT(wh.id) as recent_views
       FROM videos v
       LEFT JOIN watch_history wh ON v.id = wh.video_id
         AND wh.watched_at > NOW() - INTERVAL '24 hours'
       WHERE v.status = 'ready'
       GROUP BY v.id
       ORDER BY recent_views DESC, v.views_count DESC
       LIMIT $1`,
      [limit]
    );

    const trending = result.rows;

    // Кэшируем (10 minutes)
    await this.redis.setex(cacheKey, 600, JSON.stringify(trending));

    return trending;
  }

  async loadVideos(videoIds) {
    if (videoIds.length === 0) return [];

    const result = await this.db.query(
      'SELECT * FROM videos WHERE id = ANY($1)',
      [videoIds]
    );

    return result.rows;
  }

  async getVideo(videoId) {
    const result = await this.db.query(
      'SELECT * FROM videos WHERE id = $1',
      [videoId]
    );

    return result.rows[0];
  }
}

module.exports = RecommendationService;
```

## Оптимизации

### 1. Adaptive Bitrate Streaming (ABR)

```javascript
// Client-side (Video.js с HLS plugin)
const player = videojs('video-player', {
  html5: {
    hls: {
      enableLowInitialPlaylist: true,
      smoothQualityChange: true,
      overrideNative: true
    }
  }
});

player.src({
  src: 'https://cdn.example.com/videos/123/master.m3u8',
  type: 'application/x-mpegURL'
});

// ABR автоматически выберет качество на основе bandwidth
```

### 2. Video Preloading

```javascript
async function preloadNextVideo(currentVideoId, userId) {
  // Предсказываем следующее видео
  const nextVideo = await recommendationService.getNext(currentVideoId, userId);

  // Preload первых 5 секунд
  const preloadUrl = `${nextVideo.playbackUrl}?preload=true`;

  // Используем link preload hint
  const link = document.createElement('link');
  link.rel = 'preload';
  link.as = 'video';
  link.href = preloadUrl;
  document.head.appendChild(link);
}
```

### 3. CDN Optimization

```javascript
// CloudFront distribution configuration
const distribution = {
  Origins: [{
    Id: 'S3-videos',
    DomainName: 'videos-bucket.s3.amazonaws.com',
    OriginPath: '/videos'
  }],
  DefaultCacheBehavior: {
    TargetOriginId: 'S3-videos',
    ViewerProtocolPolicy: 'redirect-to-https',
    AllowedMethods: ['GET', 'HEAD', 'OPTIONS'],
    CachedMethods: ['GET', 'HEAD'],
    Compress: true,
    DefaultTTL: 86400, // 1 day
    MaxTTL: 31536000 // 1 year
  },
  CacheBehaviors: [{
    PathPattern: '*.m3u8',
    TargetOriginId: 'S3-videos',
    DefaultTTL: 10, // 10 seconds для manifests
    MinTTL: 0
  }]
};
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const videoUploads = new prometheus.Counter({
  name: 'video_uploads_total',
  help: 'Total video uploads',
  labelNames: ['status']
});

const transcodingDuration = new prometheus.Histogram({
  name: 'video_transcoding_duration_seconds',
  help: 'Time to transcode video',
  labelNames: ['resolution'],
  buckets: [60, 300, 600, 1800, 3600]
});

const videoViews = new prometheus.Counter({
  name: 'video_views_total',
  help: 'Total video views'
});

const bufferingEvents = new prometheus.Counter({
  name: 'video_buffering_events_total',
  help: 'Total buffering events'
});

const startupLatency = new prometheus.Histogram({
  name: 'video_startup_latency_seconds',
  help: 'Time from play to first frame',
  buckets: [0.5, 1, 2, 5, 10]
});
```

## Что читать дальше

- **Урок 18**: CDN — доставка видео пользователям
- **Урок 52**: Recommendation System — персонализированные рекомендации
- **Урок 21**: Message Queues — обработка видео через очереди

## Проверь себя

1. Почему используется адаптивный битрейт (ABR) вместо фиксированного качества?
2. Спроектируйте систему для live streaming (< 3 sec latency).
3. Как оптимизировать storage cost для petabytes видео?
4. Реализуйте resume playback (возобновление с места остановки).
5. Как обрабатывать copyright detection (Content ID)?

---

[← Урок 48: Ticketing System](48-ticketing-system.md) | [Урок 50: Ride-sharing System →](50-ride-sharing.md)
