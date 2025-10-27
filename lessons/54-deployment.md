# Урок 54: Deployment Strategies

## Введение

Deployment Strategy (стратегия развёртывания) — это подход к обновлению приложений в production без простоев и с минимальным риском. Неправильное развёртывание может привести к downtime, потере данных и недовольству пользователей.

### Зачем нужны стратегии развёртывания?

1. **Zero-downtime deployments** — обновление без прерывания сервиса
2. **Risk mitigation** — снижение риска при внедрении изменений
3. **Rollback capability** — быстрый откат при проблемах
4. **Testing in production** — проверка на реальных пользователях (canary, A/B)
5. **Compliance** — соответствие требованиям (например, для financial services)

---

## Основные стратегии

### 1. Recreate (Пересоздание)

**Как работает:**
1. Останавливаем все старые экземпляры
2. Разворачиваем новые экземпляры
3. Запускаем трафик

**Схема:**
```
Old version:  [v1] [v1] [v1]
              ↓    ↓    ↓
              ✗    ✗    ✗  ← Stop all

              (downtime)

New version:  [v2] [v2] [v2] ← Start all
```

**Плюсы:**
- Простота реализации
- Нет проблем с версионностью (только одна версия работает)
- Подходит для stateful приложений

**Минусы:**
- Downtime (от секунд до минут)
- Риск: если новая версия не работает, downtime увеличивается

**Когда использовать:**
- Dev/staging окружения
- Legacy приложения без поддержки rolling updates
- Приложения с несовместимыми версиями БД

**Kubernetes пример:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  strategy:
    type: Recreate
  template:
    spec:
      containers:
        - name: app
          image: app:2.0.0
```

---

### 2. Rolling Update (Постепенное обновление)

**Как работает:**
1. Постепенно заменяем старые pod'ы на новые
2. Гарантируем минимальное количество available replicas
3. Трафик автоматически перенаправляется на новые pod'ы

**Схема:**
```
Step 1: [v1] [v1] [v1]
        ↓
Step 2: [v1] [v1] [v2]  ← One new pod
        ↓
Step 3: [v1] [v2] [v2]  ← Replace gradually
        ↓
Step 4: [v2] [v2] [v2]  ← All updated
```

**Плюсы:**
- Zero-downtime
- Постепенное внедрение изменений
- Автоматический rollback в Kubernetes

**Минусы:**
- Две версии работают одновременно (нужна обратная совместимость)
- Медленнее, чем recreate
- Сложнее отлаживать проблемы

**Kubernetes пример:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # Сколько pod'ов может быть недоступно
      maxSurge: 2            # Сколько дополнительных pod'ов можно создать
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
        - name: app
          image: app:2.0.0
          readinessProbe:      # ВАЖНО для rolling update
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

**Процесс rolling update в Kubernetes:**

```bash
# Start deployment
kubectl apply -f deployment.yaml

# Monitor rollout
kubectl rollout status deployment/app

# Pause rollout (если заметили проблемы)
kubectl rollout pause deployment/app

# Resume rollout
kubectl rollout resume deployment/app

# Rollback к предыдущей версии
kubectl rollout undo deployment/app

# Rollback к конкретной ревизии
kubectl rollout history deployment/app
kubectl rollout undo deployment/app --to-revision=3
```

---

### 3. Blue-Green Deployment

**Как работает:**
1. Разворачиваем новую версию (Green) параллельно старой (Blue)
2. Тестируем Green окружение
3. Переключаем весь трафик с Blue на Green
4. Оставляем Blue как резервное для быстрого rollback

**Схема:**
```
Blue (v1):  [v1] [v1] [v1]  ← 100% traffic
Green (v2): [v2] [v2] [v2]  ← 0% traffic (testing)

            ↓ Switch traffic

Blue (v1):  [v1] [v1] [v1]  ← 0% traffic (standby)
Green (v2): [v2] [v2] [v2]  ← 100% traffic
```

**Плюсы:**
- Instant rollback (просто переключаем обратно)
- Zero downtime
- Полное тестирование перед переключением
- Только одна версия получает трафик

**Минусы:**
- Требует 2x ресурсов (два полных окружения)
- Сложности с stateful компонентами (БД, кэш)
- Database migrations сложнее

**Реализация с Kubernetes Service:**

```yaml
# Blue Deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: app
          image: app:1.0.0

---
# Green Deployment (new)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: app
          image: app:2.0.0

---
# Service (switch by changing selector)
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
    version: blue    # ← Change to 'green' to switch
  ports:
    - port: 80
      targetPort: 8080
```

**Скрипт для переключения:**

```bash
#!/bin/bash

# Deploy green
kubectl apply -f deployment-green.yaml

# Wait for green to be ready
kubectl wait --for=condition=available --timeout=300s deployment/app-green

# Run smoke tests on green
./run-smoke-tests.sh http://app-green-service

if [ $? -eq 0 ]; then
  echo "Green is healthy, switching traffic..."

  # Switch service to green
  kubectl patch service app-service -p '{"spec":{"selector":{"version":"green"}}}'

  echo "Traffic switched to green"

  # Monitor for 10 minutes
  sleep 600

  # If no issues, delete blue
  read -p "Delete blue deployment? (y/n) " -n 1 -r
  if [[ $REPLY =~ ^[Yy]$ ]]; then
    kubectl delete deployment app-blue
  fi
else
  echo "Green is unhealthy, keeping blue"
  exit 1
fi
```

---

### 4. Canary Deployment

**Как работает:**
1. Развёртываем новую версию на небольшом проценте серверов (5-10%)
2. Мониторим метрики (error rate, latency)
3. Постепенно увеличиваем трафик на новую версию
4. Rollback, если метрики ухудшились

**Схема:**
```
Stage 1: [v1] [v1] [v1] [v1] [v1]  95% traffic
         [v2]                        5% traffic (canary)

Stage 2: [v1] [v1] [v1]             70% traffic
         [v2] [v2] [v2]             30% traffic

Stage 3: [v2] [v2] [v2] [v2] [v2]  100% traffic
```

**Плюсы:**
- Минимальный риск (малый процент пользователей)
- Тестирование на реальном трафике
- Постепенное увеличение нагрузки
- Можно автоматизировать на основе метрик

**Минусы:**
- Сложнее настроить (нужен traffic splitting)
- Две версии работают долго
- Нужен robust monitoring

**Реализация с Istio:**

```yaml
# Destination Rule (определяем версии)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: app
spec:
  host: app-service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2

---
# Virtual Service (управление трафиком)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app
spec:
  hosts:
    - app-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: app-service
            subset: v2
    - route:
        - destination:
            host: app-service
            subset: v1
          weight: 95
        - destination:
            host: app-service
            subset: v2
          weight: 5    # ← Canary traffic
```

**Автоматический canary с Flagger:**

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5           # Количество failed checks для rollback
    maxWeight: 50          # Максимальный вес canary
    stepWeight: 10         # Шаг увеличения (10%, 20%, 30%, ...)
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99          # Минимум 99% success rate
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500         # Максимум 500ms latency (p99)
        interval: 1m
  webhooks:
    - name: load-test
      url: http://load-tester/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://app-service"
```

---

### 5. A/B Testing

**Как работает:**
- Разные версии показываются разным группам пользователей
- Цель — сравнить бизнес-метрики (conversion, engagement, revenue)

**Отличие от Canary:**
- Canary — технический эксперимент (stability, performance)
- A/B — бизнес-эксперимент (какая фича лучше?)

**Схема:**
```
Users:
  Group A (50%) → [v1] Feature X
  Group B (50%) → [v2] Feature Y

Measure: conversion rate, engagement, revenue
```

**Реализация:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app
spec:
  hosts:
    - app-service
  http:
    - match:
        - headers:
            user-group:
              exact: "group-a"
      route:
        - destination:
            host: app-service
            subset: v1
    - match:
        - headers:
            user-group:
              exact: "group-b"
      route:
        - destination:
            host: app-service
            subset: v2
    - route:  # Default: random split
        - destination:
            host: app-service
            subset: v1
          weight: 50
        - destination:
            host: app-service
            subset: v2
          weight: 50
```

**Application code для A/B группировки:**

```javascript
const express = require('express');
const app = express();

// Middleware для A/B группировки
app.use((req, res, next) => {
  let userGroup = req.cookies.user_group;

  if (!userGroup) {
    // Assign user to A or B based on user ID
    const userId = req.user?.id || req.ip;
    const hash = hashCode(userId);
    userGroup = hash % 2 === 0 ? 'group-a' : 'group-b';

    res.cookie('user_group', userGroup, { maxAge: 30 * 24 * 60 * 60 * 1000 }); // 30 days
  }

  req.userGroup = userGroup;

  // Add header for routing
  res.setHeader('user-group', userGroup);

  next();
});

app.get('/feature', (req, res) => {
  if (req.userGroup === 'group-a') {
    res.json({ feature: 'old_checkout', version: 'v1' });
  } else {
    res.json({ feature: 'new_checkout', version: 'v2' });
  }
});

function hashCode(str) {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    hash = ((hash << 5) - hash) + str.charCodeAt(i);
    hash = hash & hash;
  }
  return Math.abs(hash);
}

app.listen(3000);
```

---

### 6. Shadow Deployment (Dark Launch)

**Как работает:**
1. Новая версия получает копию production трафика
2. Ответы новой версии не возвращаются пользователям
3. Сравниваем поведение старой и новой версии

**Схема:**
```
Request
  ↓
[v1] → Response to user
  ↓ (duplicate)
[v2] → Response discarded (only for testing)
```

**Плюсы:**
- Zero risk для пользователей
- Тестирование на реальном трафике
- Обнаружение багов до release

**Минусы:**
- Требует 2x ресурсов
- Сложности с side effects (запись в БД, отправка email)

**Реализация с Istio:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app
spec:
  hosts:
    - app-service
  http:
    - route:
        - destination:
            host: app-service
            subset: v1
          weight: 100
      mirror:
        host: app-service
        subset: v2       # ← Shadow traffic
      mirrorPercentage:
        value: 100       # Mirror 100% of traffic
```

---

## Database Migrations в Deployment

### Проблема

При обновлении приложения часто нужны изменения в схеме БД. Как это сделать без downtime?

### Стратегия: Expand-Contract

**Phase 1: Expand (добавляем новое)**
1. Добавляем новый столбец/таблицу (не удаляем старое)
2. Разворачиваем новую версию приложения
3. Приложение пишет в оба места (старое и новое)

**Phase 2: Migrate Data**
4. Мигрируем данные из старого в новое (background job)

**Phase 3: Contract (удаляем старое)**
5. Переключаем приложение только на новое
6. Удаляем старые столбцы/таблицы

**Пример:**

```sql
-- Phase 1: Expand (deploy with v1)
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

-- Application v2 writes to both first_name, last_name AND full_name
UPDATE users SET full_name = first_name || ' ' || last_name WHERE full_name IS NULL;

-- Phase 2: Data migration (background job)
-- Migrate remaining data

-- Phase 3: Contract (deploy v3, then drop old columns)
-- Application v3 uses only full_name
ALTER TABLE users DROP COLUMN first_name, DROP COLUMN last_name;
```

### Blue-Green с Database

**Проблема:** два окружения используют одну БД — схема должна быть совместима с обеими версиями.

**Решение:**
1. Используйте backward-compatible migrations
2. Или используйте две отдельные БД с репликацией
3. Или используйте expand-contract подход

---

## Feature Flags (для постепенного rollout)

**Идея:** новый код уже в production, но выключен за feature flag.

```javascript
const featureFlags = require('./feature-flags');

app.get('/checkout', async (req, res) => {
  const useNewCheckout = await featureFlags.isEnabled('new_checkout', {
    userId: req.user.id,
    environment: 'production'
  });

  if (useNewCheckout) {
    return res.json(await newCheckoutFlow(req.user));
  } else {
    return res.json(await oldCheckoutFlow(req.user));
  }
});
```

**Feature Flag Service:**

```javascript
class FeatureFlagService {
  constructor(redis) {
    this.redis = redis;
  }

  async isEnabled(flagName, context = {}) {
    // Get flag configuration
    const config = await this.redis.hgetall(`flag:${flagName}`);

    if (!config || config.enabled === 'false') {
      return false;
    }

    // Percentage rollout
    if (config.percentage) {
      const percentage = parseInt(config.percentage);
      const hash = this.hashUserId(context.userId);
      return (hash % 100) < percentage;
    }

    // Whitelist
    if (config.whitelist) {
      const whitelist = JSON.parse(config.whitelist);
      return whitelist.includes(context.userId);
    }

    return config.enabled === 'true';
  }

  async setFlag(flagName, config) {
    await this.redis.hmset(`flag:${flagName}`, config);
  }

  hashUserId(userId) {
    let hash = 0;
    const str = String(userId);
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
    }
    return Math.abs(hash);
  }
}

// Usage
const flags = new FeatureFlagService(redis);

// Enable for 10% of users
await flags.setFlag('new_checkout', {
  enabled: 'true',
  percentage: '10'
});

// Enable for specific users
await flags.setFlag('beta_feature', {
  enabled: 'true',
  whitelist: JSON.stringify(['user123', 'user456'])
});
```

---

## Rollback Strategies

### Автоматический Rollback

```yaml
# Kubernetes with Flagger (automatic rollback)
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: app
spec:
  analysis:
    threshold: 3           # Rollback after 3 failed checks
    metrics:
      - name: error-rate
        thresholdRange:
          max: 5           # Rollback if error rate > 5%
```

### Manual Rollback

```bash
# Kubernetes rollback
kubectl rollout undo deployment/app

# Rollback to specific revision
kubectl rollout undo deployment/app --to-revision=5

# Blue-Green rollback (just switch back)
kubectl patch service app-service -p '{"spec":{"selector":{"version":"blue"}}}'

# Istio rollback (change traffic weights)
kubectl patch virtualservice app --type merge -p '
spec:
  http:
    - route:
        - destination:
            subset: v1
          weight: 100
        - destination:
            subset: v2
          weight: 0
'
```

---

## CI/CD Pipeline для Automated Deployments

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      # Build and push Docker image
      - name: Build image
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker tag myapp:${{ github.sha }} registry.example.com/myapp:${{ github.sha }}
          docker push registry.example.com/myapp:${{ github.sha }}

      # Update Kubernetes deployment
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/app app=registry.example.com/myapp:${{ github.sha }}

      # Wait for rollout
      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/app --timeout=5m

      # Run smoke tests
      - name: Smoke tests
        run: |
          ./scripts/smoke-tests.sh

      # Rollback if tests failed
      - name: Rollback on failure
        if: failure()
        run: |
          kubectl rollout undo deployment/app
```

**Canary deployment с Flagger (progressive delivery):**

```yaml
# Flagger canary (automatic progressive delivery)
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  progressDeadlineSeconds: 600
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    iterations: 10         # 10 iterations of 1m = 10 minutes total
    stepWeight: 10         # Increase by 10% each iteration
    maxWeight: 50
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
      - name: request-duration
        thresholdRange:
          max: 500
  webhooks:
    - name: pre-rollout
      type: pre-rollout
      url: http://flagger-loadtester/
      metadata:
        type: cmd
        cmd: "echo starting canary deployment"
    - name: load-test
      url: http://flagger-loadtester/
      timeout: 5s
      metadata:
        type: cmd
        cmd: "hey -z 1m -q 10 -c 2 http://app.default/"
```

---

## Best Practices

### 1. Всегда используйте Health Checks

Readiness probe критически важен для rolling updates — новый pod не получит трафик, пока не будет ready.

### 2. Graceful Shutdown

```javascript
// Handle SIGTERM (Kubernetes sends this before killing pod)
process.on('SIGTERM', () => {
  console.log('SIGTERM received, starting graceful shutdown');

  server.close(() => {
    console.log('HTTP server closed');

    // Close DB connections
    db.end(() => {
      console.log('Database connections closed');
      process.exit(0);
    });
  });

  // Force shutdown after 30 seconds
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
});
```

### 3. Database Migrations отдельно от Deployment

```bash
# Run migrations before deployment (в CI/CD)
kubectl run migration --image=app:$VERSION --command -- npm run migrate

# Wait for migration to complete
kubectl wait --for=condition=complete job/migration --timeout=5m

# Then deploy application
kubectl apply -f deployment.yaml
```

### 4. Мониторинг после Deployment

- Следите за error rate, latency, CPU/memory
- Настройте автоматические алерты
- Держите Blue/старую версию ready для rollback

### 5. Feature Flags вместо Long-Lived Branches

Лучше merge в main с выключенным feature flag, чем держать долгоживущие ветки.

---

## Trade-offs

| Стратегия | Zero Downtime | Rollback Speed | Resource Cost | Complexity |
|-----------|---------------|----------------|---------------|------------|
| **Recreate** | ✗ | Slow | Low | Low |
| **Rolling Update** | ✓ | Medium | Low | Medium |
| **Blue-Green** | ✓ | Instant | High (2x) | Medium |
| **Canary** | ✓ | Fast | Low-Medium | High |
| **A/B Testing** | ✓ | Fast | Medium | High |
| **Shadow** | ✓ | N/A (testing) | High (2x) | High |

---

## Что почитать дальше?

1. **Урок 53: Monitoring и Observability** — мониторинг deployments
2. **Урок 55: Kubernetes в Production** — production-ready k8s
3. **Урок 26: Circuit Breaker** — resilience при deployments

---

## Проверь себя

1. В чём разница между Rolling Update и Blue-Green deployment?
2. Когда использовать Canary deployment вместо Rolling Update?
3. Как делать database migrations без downtime (expand-contract)?
4. Что такое Feature Flags и зачем они нужны?
5. Как автоматизировать rollback на основе метрик?
6. В чём разница между Canary и A/B testing?
7. Что такое Shadow deployment и когда его использовать?
8. Зачем нужен graceful shutdown?

---
**Предыдущий урок**: [Урок 53: Monitoring и Observability](53-monitoring.md)
**Следующий урок**: [Урок 55: Kubernetes в Production](55-kubernetes.md)
