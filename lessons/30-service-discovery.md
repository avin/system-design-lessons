# Урок 30: Service Discovery

[← Предыдущий урок: Backend for Frontend (BFF)](29-backend-for-frontend.md) | [Следующий урок: Consensus алгоритмы →](31-consensus-algorithms.md)

## Введение

Service Discovery — это механизм автоматического обнаружения сетевых адресов сервисов в распределённой системе. В микросервисной архитектуре сервисы динамически создаются, масштабируются и удаляются, поэтому hardcoded IP-адреса не работают.

**Проблема:**

```javascript
// ❌ Hardcoded адреса
const orderService = axios.create({
  baseURL: 'http://192.168.1.10:3000'  // Что если instance упал?
});                                     // Что при scaling?
```

**Решение: Service Discovery**

```javascript
// ✅ Dynamic discovery
const serviceUrl = await serviceDiscovery.getServiceUrl('orders-service');
const orderService = axios.create({ baseURL: serviceUrl });
```

## Паттерны Service Discovery

### 1. Client-Side Discovery

Клиент ответственен за определение адресов доступных instances.

```
Client → Service Registry → получает список instances
       → выбирает instance (load balancing)
       → делает запрос напрямую к instance
```

**Примеры:** Netflix Eureka, Consul

**Преимущества:**
- Клиент контролирует load balancing
- Меньше network hops

**Недостатки:**
- Логика discovery в каждом клиенте
- Coupling с Service Registry

### 2. Server-Side Discovery

Load Balancer ответственен за routing к нужному instance.

```
Client → Load Balancer → Service Registry (для получения instances)
                       → routing к здоровому instance
```

**Примеры:** AWS ELB, Kubernetes Service, NGINX

**Преимущества:**
- Клиент не знает о discovery
- Централизованное управление

**Недостатки:**
- Дополнительный network hop
- Load Balancer — potential bottleneck

## Consul (Client-Side Discovery)

### Установка Consul

```bash
# Docker
docker run -d --name=consul -p 8500:8500 consul agent -server -ui -bootstrap-expect=1 -client=0.0.0.0
```

### Регистрация сервиса

```javascript
const Consul = require('consul');
const express = require('express');

const consul = new Consul({ host: 'localhost', port: 8500 });

const app = express();
const PORT = process.env.PORT || 3000;
const SERVICE_NAME = 'orders-service';
const SERVICE_ID = `${SERVICE_NAME}-${PORT}`;

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.listen(PORT, async () => {
  console.log(`Server running on port ${PORT}`);

  // Регистрация в Consul
  await consul.agent.service.register({
    id: SERVICE_ID,
    name: SERVICE_NAME,
    address: process.env.HOST || 'localhost',
    port: PORT,
    tags: ['v1', 'orders'],
    check: {
      http: `http://localhost:${PORT}/health`,
      interval: '10s',
      timeout: '5s',
      deregistertimeout: '1m',
    },
  });

  console.log(`Registered with Consul as ${SERVICE_ID}`);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  await consul.agent.service.deregister(SERVICE_ID);
  console.log('Deregistered from Consul');
  process.exit(0);
});
```

### Discovery клиент

```javascript
class ConsulServiceDiscovery {
  constructor(consul) {
    this.consul = consul;
    this.cache = new Map();
  }

  async getServiceUrl(serviceName) {
    // Получить все healthy instances
    const services = await this.consul.health.service({
      service: serviceName,
      passing: true,  // Только passing health checks
    });

    if (services.length === 0) {
      throw new Error(`No healthy instances of ${serviceName}`);
    }

    // Load balancing: round-robin
    const service = this.selectInstance(serviceName, services);

    return `http://${service.Service.Address}:${service.Service.Port}`;
  }

  selectInstance(serviceName, services) {
    if (!this.cache.has(serviceName)) {
      this.cache.set(serviceName, 0);
    }

    const index = this.cache.get(serviceName);
    const service = services[index % services.length];

    this.cache.set(serviceName, index + 1);

    return service;
  }

  async watchService(serviceName, callback) {
    const watch = this.consul.watch({
      method: this.consul.health.service,
      options: {
        service: serviceName,
        passing: true,
      },
    });

    watch.on('change', (data) => {
      console.log(`Service ${serviceName} changed:`, data.length, 'instances');
      callback(data);
    });

    watch.on('error', (err) => {
      console.error('Watch error:', err);
    });

    return watch;
  }
}

// Использование
const discovery = new ConsulServiceDiscovery(consul);

app.get('/api/orders/:id', async (req, res) => {
  try {
    const serviceUrl = await discovery.getServiceUrl('orders-service');

    const response = await axios.get(`${serviceUrl}/orders/${req.params.id}`);

    res.json(response.data);

  } catch (error) {
    console.error('Service discovery error:', error);
    res.status(503).json({ error: 'Service unavailable' });
  }
});

// Watch для обновления кэша
discovery.watchService('orders-service', (instances) => {
  console.log('Orders service instances updated:', instances.length);
});
```

### DNS Interface Consul

```bash
# Consul предоставляет DNS интерфейс
dig @127.0.0.1 -p 8600 orders-service.service.consul

# Ответ:
# orders-service.service.consul. 0 IN A 192.168.1.10
# orders-service.service.consul. 0 IN A 192.168.1.11
```

```javascript
// Использование DNS
const response = await axios.get('http://orders-service.service.consul:3000/orders/123');
```

## Kubernetes Service Discovery

### Kubernetes Service (Server-Side)

```yaml
# orders-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: orders
          image: orders-service:v1
          ports:
            - containerPort: 3000
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: orders-service
spec:
  selector:
    app: orders
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

**Как это работает:**

```
Клиент запрос к "orders-service"
       ↓
DNS резолвит в ClusterIP (виртуальный IP)
       ↓
kube-proxy (iptables/ipvs) распределяет запрос
       ↓
Один из pod'ов orders-service
```

### Использование в коде

```javascript
// В Kubernetes просто используйте service name
const ORDERS_SERVICE_URL = process.env.ORDERS_SERVICE_URL || 'http://orders-service';

app.get('/api/orders/:id', async (req, res) => {
  const response = await axios.get(`${ORDERS_SERVICE_URL}/orders/${req.params.id}`);
  res.json(response.data);
});
```

### Headless Service (Client-Side в K8s)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders-service-headless
spec:
  selector:
    app: orders
  clusterIP: None  # Headless
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```

```javascript
const dns = require('dns').promises;

async function getOrderServiceInstances() {
  // DNS вернёт IP всех pod'ов
  const addresses = await dns.resolve4('orders-service-headless.default.svc.cluster.local');

  return addresses.map(ip => `http://${ip}:3000`);
}

// Client-side load balancing
const instances = await getOrderServiceInstances();
const randomInstance = instances[Math.floor(Math.random() * instances.length)];
const response = await axios.get(`${randomInstance}/orders/123`);
```

## Eureka (Netflix OSS)

### Eureka Server

```yaml
# eureka-server/application.yml
server:
  port: 8761

eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
  server:
    enableSelfPreservation: false
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### Eureka Client

```yaml
# orders-service/application.yml
spring:
  application:
    name: orders-service

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    preferIpAddress: true
    leaseRenewalIntervalInSeconds: 10
    leaseExpirationDurationInSeconds: 30
```

```java
@SpringBootApplication
@EnableEurekaClient
public class OrdersServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrdersServiceApplication.class, args);
    }
}

@RestController
public class OrdersController {
    @Autowired
    private DiscoveryClient discoveryClient;

    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id) {
        // Service discovery
        List<ServiceInstance> instances =
            discoveryClient.getInstances("payment-service");

        if (instances.isEmpty()) {
            throw new ServiceUnavailableException("Payment service unavailable");
        }

        ServiceInstance instance = instances.get(0);
        String url = instance.getUri() + "/payments/" + id;

        // Make request
        RestTemplate restTemplate = new RestTemplate();
        return restTemplate.getForObject(url, Payment.class);
    }
}
```

## Service Mesh (Istio) Discovery

С Service Mesh discovery управляется автоматически через sidecar proxy.

```yaml
# Просто деплой сервиса
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: orders
          image: orders-service:v1
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: orders-service
spec:
  selector:
    app: orders
  ports:
    - port: 80
      targetPort: 3000
```

```javascript
// Код не меняется — Envoy sidecar автоматически делает discovery
const response = await axios.get('http://orders-service/orders/123');
```

Istio автоматически:
- Обнаруживает instances
- Load balancing
- Health checking
- Circuit breaking
- Retry logic

## Health Checks

### Типы Health Checks

**1. Liveness Probe**: жив ли сервис?

```javascript
app.get('/health/live', (req, res) => {
  // Простая проверка: процесс запущен
  res.json({ status: 'alive' });
});
```

**2. Readiness Probe**: готов ли принимать трафик?

```javascript
app.get('/health/ready', async (req, res) => {
  try {
    // Проверить зависимости
    await db.ping();
    await redis.ping();

    res.json({ status: 'ready' });

  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});
```

**3. Startup Probe**: завершилась ли инициализация?

```javascript
let initialized = false;

async function initialize() {
  // Долгая инициализация
  await loadConfigFromRemote();
  await connectToDatabase();
  await warmupCache();

  initialized = true;
}

app.get('/health/startup', (req, res) => {
  if (initialized) {
    res.json({ status: 'started' });
  } else {
    res.status(503).json({ status: 'starting' });
  }
});

// Запустить инициализацию асинхронно
initialize().then(() => {
  console.log('Service initialized');
});
```

### Kubernetes Health Checks

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: orders-pod
spec:
  containers:
    - name: orders
      image: orders-service:v1
      ports:
        - containerPort: 3000

      # Startup: 30 попыток × 10s = до 5 минут на старт
      startupProbe:
        httpGet:
          path: /health/startup
          port: 3000
        periodSeconds: 10
        failureThreshold: 30

      # Liveness: убить под если не отвечает
      livenessProbe:
        httpGet:
          path: /health/live
          port: 3000
        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3

      # Readiness: убрать из Service Endpoints
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 3000
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 2
```

## Load Balancing Strategies

### 1. Round Robin

```javascript
class RoundRobinLoadBalancer {
  constructor() {
    this.currentIndex = 0;
  }

  selectInstance(instances) {
    if (instances.length === 0) {
      throw new Error('No instances available');
    }

    const instance = instances[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % instances.length;

    return instance;
  }
}
```

### 2. Least Connections

```javascript
class LeastConnectionsLoadBalancer {
  constructor() {
    this.connections = new Map();
  }

  selectInstance(instances) {
    let minConnections = Infinity;
    let selectedInstance = null;

    for (const instance of instances) {
      const connections = this.connections.get(instance.id) || 0;

      if (connections < minConnections) {
        minConnections = connections;
        selectedInstance = instance;
      }
    }

    return selectedInstance;
  }

  incrementConnections(instanceId) {
    const current = this.connections.get(instanceId) || 0;
    this.connections.set(instanceId, current + 1);
  }

  decrementConnections(instanceId) {
    const current = this.connections.get(instanceId) || 0;
    this.connections.set(instanceId, Math.max(0, current - 1));
  }
}
```

### 3. Weighted Round Robin

```javascript
class WeightedRoundRobinLoadBalancer {
  constructor() {
    this.currentIndex = 0;
    this.currentWeight = 0;
  }

  selectInstance(instances) {
    // instances = [{ url: '...', weight: 5 }, { url: '...', weight: 3 }]

    const totalWeight = instances.reduce((sum, i) => sum + i.weight, 0);

    while (true) {
      this.currentIndex = (this.currentIndex + 1) % instances.length;

      if (this.currentIndex === 0) {
        this.currentWeight = this.currentWeight - 1;

        if (this.currentWeight <= 0) {
          this.currentWeight = Math.max(...instances.map(i => i.weight));
        }
      }

      if (instances[this.currentIndex].weight >= this.currentWeight) {
        return instances[this.currentIndex];
      }
    }
  }
}
```

### 4. Locality-Aware (зональный)

```javascript
class LocalityAwareLoadBalancer {
  constructor(currentZone) {
    this.currentZone = currentZone;
  }

  selectInstance(instances) {
    // Приоритет: same zone > same region > any
    const sameZone = instances.filter(i => i.zone === this.currentZone);

    if (sameZone.length > 0) {
      return sameZone[Math.floor(Math.random() * sameZone.length)];
    }

    const sameRegion = instances.filter(i => i.region === this.currentRegion);

    if (sameRegion.length > 0) {
      return sameRegion[Math.floor(Math.random() * sameRegion.length)];
    }

    return instances[Math.floor(Math.random() * instances.length)];
  }
}
```

## Caching и TTL

```javascript
class ServiceDiscoveryWithCache {
  constructor(consul, ttl = 30000) {
    this.consul = consul;
    this.ttl = ttl;
    this.cache = new Map();
  }

  async getServiceInstances(serviceName) {
    const cached = this.cache.get(serviceName);

    if (cached && Date.now() - cached.timestamp < this.ttl) {
      console.log('Cache hit:', serviceName);
      return cached.instances;
    }

    console.log('Cache miss:', serviceName);

    const services = await this.consul.health.service({
      service: serviceName,
      passing: true,
    });

    const instances = services.map(s => ({
      id: s.Service.ID,
      address: s.Service.Address,
      port: s.Service.Port,
      tags: s.Service.Tags,
    }));

    this.cache.set(serviceName, {
      instances,
      timestamp: Date.now(),
    });

    return instances;
  }

  invalidateCache(serviceName) {
    this.cache.delete(serviceName);
  }

  clearCache() {
    this.cache.clear();
  }
}
```

## Monitoring Service Discovery

### Метрики

```javascript
const promClient = require('prom-client');

const serviceDiscoveryRequests = new promClient.Counter({
  name: 'service_discovery_requests_total',
  help: 'Total service discovery requests',
  labelNames: ['service', 'status'],
});

const serviceDiscoveryDuration = new promClient.Histogram({
  name: 'service_discovery_duration_seconds',
  help: 'Service discovery duration',
  labelNames: ['service'],
});

const availableInstances = new promClient.Gauge({
  name: 'service_available_instances',
  help: 'Number of available service instances',
  labelNames: ['service'],
});

async function getServiceInstances(serviceName) {
  const end = serviceDiscoveryDuration.startTimer({ service: serviceName });

  try {
    const instances = await consul.health.service({
      service: serviceName,
      passing: true,
    });

    availableInstances.set({ service: serviceName }, instances.length);
    serviceDiscoveryRequests.inc({ service: serviceName, status: 'success' });

    return instances;

  } catch (error) {
    serviceDiscoveryRequests.inc({ service: serviceName, status: 'error' });
    throw error;

  } finally {
    end();
  }
}
```

## Best Practices

### 1. Graceful Shutdown

```javascript
let isShuttingDown = false;

// Health check учитывает shutdown
app.get('/health/ready', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'shutting down' });
  }

  res.json({ status: 'ready' });
});

process.on('SIGTERM', async () => {
  console.log('SIGTERM received, starting graceful shutdown');

  isShuttingDown = true;

  // Дать Load Balancer время убрать нас из rotation (10s)
  await new Promise(resolve => setTimeout(resolve, 10000));

  // Deregister from Consul
  await consul.agent.service.deregister(SERVICE_ID);

  // Закрыть сервер
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });

  // Force shutdown после 30s
  setTimeout(() => {
    console.error('Forced shutdown');
    process.exit(1);
  }, 30000);
});
```

### 2. Retry с Backoff

```javascript
async function getServiceUrlWithRetry(serviceName, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await discovery.getServiceUrl(serviceName);

    } catch (error) {
      if (attempt === maxRetries) {
        throw error;
      }

      const delay = Math.pow(2, attempt) * 100;  // 200ms, 400ms, 800ms
      console.log(`Retry ${attempt}/${maxRetries} after ${delay}ms`);

      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

### 3. Circuit Breaker для Discovery

```javascript
const CircuitBreaker = require('opossum');

const discoveryBreaker = new CircuitBreaker(
  async (serviceName) => {
    return await consul.health.service({
      service: serviceName,
      passing: true,
    });
  },
  {
    timeout: 5000,
    errorThresholdPercentage: 50,
    resetTimeout: 30000,
  }
);

// Fallback: использовать кэш
discoveryBreaker.fallback((serviceName) => {
  console.log('Circuit open, using cache');
  const cached = cache.get(serviceName);

  if (!cached) {
    throw new Error('No cached instances and discovery unavailable');
  }

  return cached.instances;
});
```

## Выводы

Service Discovery — критический компонент микросервисной архитектуры:

**Паттерны:**
- **Client-Side**: клиент управляет discovery (Consul, Eureka)
- **Server-Side**: load balancer управляет (K8s Service, AWS ELB)

**Ключевые концепции:**
- Service Registry
- Health Checks (liveness, readiness, startup)
- Load Balancing
- Caching

**Best Practices:**
- Graceful shutdown
- Health checks на всех уровнях
- Caching с TTL
- Circuit breaker
- Мониторинг доступности instances

Service Discovery упрощает управление динамической инфраструктурой и делает систему более resilient.

## Что читать дальше?

- [Урок 31: Consensus алгоритмы](31-consensus-algorithms.md)
- [Урок 26: Service Mesh](26-service-mesh.md)
- [Урок 28: API Gateway паттерны](28-api-gateway-patterns.md)

## Проверь себя

1. В чём разница между Client-Side и Server-Side discovery?
2. Что такое Service Registry?
3. Какие типы health checks существуют и зачем каждый нужен?
4. Как работает Service Discovery в Kubernetes?
5. Что такое Headless Service в K8s?
6. Какие стратегии load balancing существуют?
7. Зачем нужен graceful shutdown при service discovery?
8. Как кэширование помогает с performance discovery?
9. Что произойдёт если Service Registry упадёт?
10. Как Service Mesh упрощает service discovery?

---

[← Предыдущий урок: Backend for Frontend (BFF)](29-backend-for-frontend.md) | [Следующий урок: Consensus алгоритмы →](31-consensus-algorithms.md)
