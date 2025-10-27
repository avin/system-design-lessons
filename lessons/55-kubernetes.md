# Урок 55: Kubernetes в Production

## Введение

Kubernetes — это де-факто стандарт для оркестрации контейнеров в production. Но запустить кластер — это только начало. Production-ready Kubernetes требует правильной настройки безопасности, масштабирования, мониторинга, и disaster recovery.

### Почему Kubernetes?

1. **Декларативная конфигурация** — описываем желаемое состояние, k8s обеспечивает его
2. **Self-healing** — автоматический перезапуск failed контейнеров
3. **Horizontal scaling** — автомасштабирование на основе метрик
4. **Service discovery & Load balancing** — встроенные
5. **Rolling updates & rollbacks** — zero-downtime deployments
6. **Портативность** — работает одинаково в AWS, GCP, Azure, on-premise

---

## Production-Ready Checklist

### 1. Cluster Setup

**Multi-Node, Multi-AZ:**
```
Control Plane (HA):
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │   Master 1   │  │   Master 2   │  │   Master 3   │
  │   (AZ-1a)    │  │   (AZ-1b)    │  │   (AZ-1c)    │
  └──────────────┘  └──────────────┘  └──────────────┘

Worker Nodes:
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │   Worker 1   │  │   Worker 2   │  │   Worker 3   │
  │   (AZ-1a)    │  │   (AZ-1b)    │  │   (AZ-1c)    │
  └──────────────┘  └──────────────┘  └──────────────┘
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │   Worker 4   │  │   Worker 5   │  │   Worker 6   │
  │   (AZ-1a)    │  │   (AZ-1b)    │  │   (AZ-1c)    │
  └──────────────┘  └──────────────┘  └──────────────┘
```

**Terraform для EKS (AWS):**

```hcl
# eks-cluster.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "production-cluster"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # Enable IRSA (IAM Roles for Service Accounts)
  enable_irsa = true

  # Cluster endpoint access
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = false  # Production: только private

  # Encryption at rest
  cluster_encryption_config = {
    resources        = ["secrets"]
    provider_key_arn = aws_kms_key.eks.arn
  }

  # Control plane logging
  cluster_enabled_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  eks_managed_node_groups = {
    # General purpose nodes
    general = {
      desired_size = 3
      min_size     = 3
      max_size     = 10

      instance_types = ["m5.xlarge"]
      capacity_type  = "ON_DEMAND"

      labels = {
        role = "general"
      }

      taints = []
    }

    # Spot instances для non-critical workloads
    spot = {
      desired_size = 2
      min_size     = 0
      max_size     = 5

      instance_types = ["m5.large", "m5.xlarge"]
      capacity_type  = "SPOT"

      labels = {
        role = "spot"
      }

      taints = [{
        key    = "spot"
        value  = "true"
        effect = "NoSchedule"
      }]
    }
  }

  # Security groups
  node_security_group_additional_rules = {
    ingress_self_all = {
      description = "Node to node all ports/protocols"
      protocol    = "-1"
      from_port   = 0
      to_port     = 0
      type        = "ingress"
      self        = true
    }
  }
}
```

---

### 2. Namespace Organization

```yaml
# namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production

---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging

---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    environment: shared
```

**Resource Quotas для каждого namespace:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    persistentvolumeclaims: "20"
    services.loadbalancers: "5"
```

**LimitRange (default limits для pod'ов):**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "100m"
        memory: "128Mi"
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "200m"
        memory: "256Mi"
      type: Container
```

---

### 3. Resource Requests and Limits

**ВСЕГДА** устанавливайте requests и limits:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: api
          image: api:1.0.0
          resources:
            requests:
              cpu: "500m"        # Гарантированно получит 0.5 CPU
              memory: "512Mi"    # Гарантированно получит 512MB
            limits:
              cpu: "2000m"       # Максимум 2 CPU
              memory: "2Gi"      # Максимум 2GB (при превышении — OOMKilled)

          # Anti-patterns:
          # ✗ Не указывать requests/limits вообще
          # ✗ limits без requests
          # ✗ Слишком большие limits (приводит к overcommit)
```

**QoS Classes:**
- **Guaranteed** (requests == limits) — наивысший приоритет, не будет evicted
- **Burstable** (requests < limits) — средний приоритет
- **BestEffort** (нет requests/limits) — первым будет evicted

---

### 4. Pod Disruption Budgets

Гарантирует минимальное количество доступных pod'ов во время disruptions (node drain, upgrade):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: production
spec:
  minAvailable: 2    # Минимум 2 pod'а всегда доступны
  selector:
    matchLabels:
      app: api

---
# Или используйте процент
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: worker-pdb
spec:
  maxUnavailable: 25%    # Максимум 25% может быть недоступно
  selector:
    matchLabels:
      app: worker
```

---

### 5. Horizontal Pod Autoscaler (HPA)

Автоматическое масштабирование на основе метрик:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
    # CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale when avg CPU > 70%

    # Memory utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

    # Custom metric (requests per second)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"      # Scale when avg > 1000 RPS

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5min before scaling down
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60              # Max 50% scale down per minute
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15              # Double capacity every 15s if needed
        - type: Pods
          value: 5
          periodSeconds: 15              # Or add max 5 pods every 15s
      selectPolicy: Max
```

**Custom Metrics с Prometheus Adapter:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-adapter
  namespace: monitoring
data:
  config.yaml: |
    rules:
      - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace: { resource: "namespace" }
            pod: { resource: "pod" }
        name:
          matches: "^(.*)_total$"
          as: "${1}_per_second"
        metricsQuery: 'rate(<<.Series>>{<<.LabelMatchers>>}[2m])'
```

---

### 6. Vertical Pod Autoscaler (VPA)

Автоматическая рекомендация и установка requests/limits:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  updatePolicy:
    updateMode: "Auto"    # Auto, Initial, Recreate, Off
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "4"
          memory: "8Gi"
        controlledResources: ["cpu", "memory"]
```

**⚠️ Warning:** VPA и HPA могут конфликтовать — обычно используют что-то одно, или HPA на CPU + VPA на memory.

---

### 7. Cluster Autoscaler

Автоматическое добавление/удаление worker nodes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
          command:
            - ./cluster-autoscaler
            - --cloud-provider=aws
            - --namespace=kube-system
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/production-cluster
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
            - --scale-down-delay-after-add=10m
            - --scale-down-unneeded-time=10m
            - --scale-down-utilization-threshold=0.5
```

---

### 8. Pod Priority and Preemption

```yaml
# High priority for critical services
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "High priority for critical services"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Low priority for batch jobs"

---
# Use in deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  template:
    spec:
      priorityClassName: high-priority
      containers:
        - name: api
          image: api:1.0.0
```

---

### 9. Pod Affinity/Anti-Affinity

**Anti-Affinity (распределить по разным нодам):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  template:
    spec:
      affinity:
        # Pod anti-affinity (не размещать на одной ноде)
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - api
              topologyKey: "kubernetes.io/hostname"

        # Node affinity (размещать только на определённых нодах)
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-role
                    operator: In
                    values:
                      - general
                      - compute

          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: instance-type
                    operator: In
                    values:
                      - m5.xlarge
```

**Affinity (co-locate pod'ы):**

```yaml
# Example: размещать cache близко к API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
spec:
  template:
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - api
              topologyKey: "kubernetes.io/hostname"
```

---

### 10. Network Policies

**Default deny all:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**Allow specific traffic:**

```yaml
# API can receive traffic from ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080

---
# API can call database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

### 11. Security Best Practices

#### SecurityContext

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true        # Не запускать от root
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: api
          image: api:1.0.0
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL              # Drop все capabilities
              add:
                - NET_BIND_SERVICE # Добавить только нужные

          volumeMounts:
            - name: tmp
              mountPath: /tmp      # Writable volume для temp files
            - name: cache
              mountPath: /app/cache

      volumes:
        - name: tmp
          emptyDir: {}
        - name: cache
          emptyDir: {}
```

#### Pod Security Standards

```yaml
# Enforce Pod Security Standards на namespace level
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

#### Secrets Management

**External Secrets Operator (интеграция с AWS Secrets Manager):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: production/database
        property: username
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

---

### 12. Ingress and Load Balancing

**Nginx Ingress Controller:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"      # 100 req/s per IP
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

**Cert Manager для автоматических SSL сертификатов:**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

---

### 13. Persistent Storage

**StorageClass:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

**StatefulSet с PVC:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:15
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

---

### 14. ConfigMaps and Secrets

```yaml
# ConfigMap для configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: production
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s

---
# Secret для sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: production
type: Opaque
stringData:
  DB_PASSWORD: "secret_password"
  API_KEY: "secret_api_key"

---
# Use in deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  template:
    spec:
      containers:
        - name: api
          image: api:1.0.0
          envFrom:
            - configMapRef:
                name: api-config
            - secretRef:
                name: api-secrets
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: api-config
```

---

### 15. Monitoring and Logging

**Prometheus ServiceMonitor:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-service
  namespace: production
spec:
  selector:
    matchLabels:
      app: api
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

**Fluentd DaemonSet для логов:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    spec:
      serviceAccountName: fluentd
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch.monitoring.svc.cluster.local"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

---

### 16. Backup and Disaster Recovery

**Velero для backup:**

```bash
# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero

# Create backup
velero backup create production-backup \
  --include-namespaces production \
  --ttl 720h

# Schedule automatic backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production

# Restore from backup
velero restore create --from-backup production-backup
```

---

### 17. GitOps с ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-service
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/example/k8s-manifests
    targetRevision: main
    path: production/api
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Delete resources не в Git
      selfHeal: true     # Auto-sync при drift
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## Best Practices Summary

### 1. Always Set Resource Requests/Limits
### 2. Use Pod Disruption Budgets
### 3. Enable HPA для stateless services
### 4. Implement Health Checks (liveness + readiness)
### 5. Use NetworkPolicies (least privilege)
### 6. Never run as root (SecurityContext)
### 7. Use External Secrets (не храните secrets в Git)
### 8. Enable audit logging
### 9. Multi-AZ deployment для HA
### 10. Regular backups с Velero

---

## Trade-offs

| Аспект | Вариант A | Вариант B |
|--------|-----------|-----------|
| **Node Size** | Много маленьких нод | Мало больших нод |
| Fault tolerance | Лучше | Хуже |
| Bin packing | Хуже | Лучше |
| Cost | Выше | Ниже |
| **Autoscaling** | HPA only | HPA + VPA |
| Complexity | Низкая | Высокая |
| Optimization | Хуже | Лучше |
| Conflicts | Нет | Возможны |

---

## Проверь себя

1. Что такое QoS Classes в Kubernetes (Guaranteed, Burstable, BestEffort)?
2. Зачем нужен Pod Disruption Budget?
3. В чём разница между HPA и VPA?
4. Как настроить pod anti-affinity для HA?
5. Зачем нужен readOnlyRootFilesystem в securityContext?
6. Как работает External Secrets Operator?
7. Что такое GitOps и как его реализовать с ArgoCD?
8. Как делать backup Kubernetes кластера с Velero?

---
**Предыдущий урок**: [Урок 54: Deployment Strategies](54-deployment.md)
**Следующий урок**: [Урок 56: Disaster Recovery](56-disaster-recovery.md)
