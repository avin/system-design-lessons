# Урок 56: Disaster Recovery

## Введение

Disaster Recovery (DR) — это набор политик, инструментов и процедур для восстановления критически важных технологических систем после катастрофического события: пожара в датацентре, сбоя всего региона AWS, ransomware атаки, случайного удаления production базы данных.

### Почему это важно?

1. **Business Continuity** — бизнес должен продолжать работать даже после катастрофы
2. **Data Loss Prevention** — минимизация потери данных
3. **Compliance** — требования регуляторов (GDPR, HIPAA, SOC 2)
4. **Customer Trust** — клиенты доверяют компаниям с надёжной инфраструктурой
5. **Financial Impact** — downtime стоит дорого ($100K-$1M+ в час для крупных компаний)

### Ключевые метрики

**RTO (Recovery Time Objective)** — максимально допустимое время простоя
- Пример: "Система должна быть восстановлена за 4 часа"

**RPO (Recovery Point Objective)** — максимально допустимая потеря данных
- Пример: "Мы можем потерять максимум 15 минут данных"

```
Data Loss (RPO)          Downtime (RTO)
     ↓                        ↓
     |                        |
─────┼────────────────────────┼──────────► time
  Disaster              Recovery
  occurs                complete
```

---

## Стратегии Disaster Recovery

### 1. Backup and Restore (холодный резерв)

**Как работает:**
- Регулярные backup'ы данных в другой регион
- При disaster'е разворачиваем инфраструктуру и восстанавливаем из backup

**Схема:**
```
Primary Region (us-east-1)          Backup Region (us-west-2)
┌────────────────────┐              ┌──────────────┐
│   Application      │              │  S3 Bucket   │
│   Database         │──backups────►│  (backups)   │
│   Files            │              └──────────────┘
└────────────────────┘

        ↓ Disaster

┌────────────────────┐              ┌──────────────────┐
│   UNAVAILABLE      │              │  Restore from    │
│                    │              │  backup & launch │
└────────────────────┘              └──────────────────┘
```

**RTO:** Часы
**RPO:** Зависит от частоты backup (обычно часы)
**Стоимость:** Низкая (только storage)

**Реализация:**

```bash
#!/bin/bash
# Automated backup script

# Database backup
pg_dump -h db.example.com -U postgres production_db | \
  gzip | \
  aws s3 cp - s3://backup-bucket/db/production_db_$(date +%Y%m%d_%H%M%S).sql.gz

# Application files backup
tar czf - /var/www/uploads | \
  aws s3 cp - s3://backup-bucket/files/uploads_$(date +%Y%m%d_%H%M%S).tar.gz

# Kubernetes manifests backup
kubectl get all --all-namespaces -o yaml | \
  gzip | \
  aws s3 cp - s3://backup-bucket/k8s/manifests_$(date +%Y%m%d_%H%M%S).yaml.gz

# Keep only last 30 days of backups
aws s3 ls s3://backup-bucket/db/ | \
  awk '{print $4}' | \
  head -n -30 | \
  xargs -I {} aws s3 rm s3://backup-bucket/db/{}
```

**Restore script:**

```bash
#!/bin/bash
# Disaster recovery restore

set -e

BACKUP_DATE=$1

echo "Restoring from backup: $BACKUP_DATE"

# 1. Restore database
echo "Restoring database..."
aws s3 cp s3://backup-bucket/db/production_db_${BACKUP_DATE}.sql.gz - | \
  gunzip | \
  psql -h new-db.example.com -U postgres production_db

# 2. Restore files
echo "Restoring application files..."
aws s3 cp s3://backup-bucket/files/uploads_${BACKUP_DATE}.tar.gz - | \
  tar xzf - -C /var/www/

# 3. Deploy application
echo "Deploying application..."
kubectl apply -f infrastructure/

# 4. Run smoke tests
echo "Running smoke tests..."
./scripts/smoke-tests.sh

echo "Recovery complete!"
```

**Автоматизация с AWS Backup:**

```hcl
# terraform: AWS Backup plan
resource "aws_backup_plan" "production" {
  name = "production-backup-plan"

  rule {
    rule_name         = "daily_backup"
    target_vault_name = aws_backup_vault.primary.name
    schedule          = "cron(0 2 * * ? *)"  # 2 AM daily

    lifecycle {
      delete_after = 30
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr_region.arn

      lifecycle {
        delete_after = 90
      }
    }
  }
}

resource "aws_backup_selection" "production" {
  name         = "production-resources"
  plan_id      = aws_backup_plan.production.id
  iam_role_arn = aws_iam_role.backup.arn

  resources = [
    aws_db_instance.primary.arn,
    aws_efs_file_system.main.arn
  ]
}
```

---

### 2. Pilot Light (теплый резерв)

**Как работает:**
- Критические компоненты (БД) постоянно реплицируются в DR регион
- Остальная инфраструктура минимальна или отсутствует
- При disaster'е быстро разворачиваем application layer

**Схема:**
```
Primary Region                  DR Region
┌────────────────┐             ┌────────────────┐
│  Application   │             │  (minimal or   │
│  (active)      │             │   none)        │
└────────┬───────┘             └────────────────┘
         │
         ▼
┌────────────────┐  replica   ┌────────────────┐
│  Database      │────────────►│  Database      │
│  (active)      │             │  (standby)     │
└────────────────┘             └────────────────┘
```

**RTO:** Минуты до часов
**RPO:** Минуты (зависит от replication lag)
**Стоимость:** Средняя (постоянная БД + storage)

**Реализация с PostgreSQL:**

```yaml
# Primary database (us-east-1)
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-primary-config
data:
  postgresql.conf: |
    wal_level = replica
    max_wal_senders = 10
    wal_keep_size = 1GB
    hot_standby = on

  pg_hba.conf: |
    host replication replicator 0.0.0.0/0 md5

---
# Standby database (us-west-2)
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-standby-config
data:
  postgresql.conf: |
    hot_standby = on
    primary_conninfo = 'host=primary-db.us-east-1.example.com port=5432 user=replicator'
    restore_command = 'aws s3 cp s3://wal-archive/%f %p'
```

**Failover script:**

```bash
#!/bin/bash
# Promote standby to primary

set -e

echo "Starting failover to DR region..."

# 1. Promote standby database to primary
kubectl exec -it postgres-standby-0 -- \
  psql -U postgres -c "SELECT pg_promote();"

# 2. Update DNS to point to DR region
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "db.example.com",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "dr-db.us-west-2.example.com"}]
      }
    }]
  }'

# 3. Scale up application in DR region
kubectl scale deployment api-service --replicas=10 -n production

# 4. Update load balancer
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,TargetGroupArn=$DR_TARGET_GROUP_ARN

echo "Failover complete. Monitor closely!"
```

---

### 3. Warm Standby (активный резерв)

**Как работает:**
- Полная инфраструктура работает в DR регионе с уменьшенной capacity
- Данные постоянно реплицируются
- При disaster'е увеличиваем capacity и переключаем трафик

**Схема:**
```
Primary Region (100%)           DR Region (20-30%)
┌────────────────┐             ┌────────────────┐
│  Application   │             │  Application   │
│  [████████████]│             │  [███░░░░░░░░░]│
└────────┬───────┘             └────────┬───────┘
         │                              │
         ▼                              ▼
┌────────────────┐  replica   ┌────────────────┐
│  Database      │────────────►│  Database      │
│  (primary)     │             │  (replica)     │
└────────────────┘             └────────────────┘
```

**RTO:** Минуты
**RPO:** Секунды до минут
**Стоимость:** Средне-высокая (30-50% от full cost)

**Terraform для Multi-Region:**

```hcl
# modules/app-region/main.tf
variable "region" {}
variable "capacity_percentage" {}

resource "aws_autoscaling_group" "app" {
  name                = "app-asg-${var.region}"
  vpc_zone_identifier = var.subnet_ids
  min_size            = var.min_size * var.capacity_percentage / 100
  max_size            = var.max_size
  desired_capacity    = var.desired_size * var.capacity_percentage / 100

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  target_group_arns = [aws_lb_target_group.app.arn]
}

resource "aws_db_instance" "replica" {
  count                 = var.region == var.primary_region ? 0 : 1
  replicate_source_db   = aws_db_instance.primary.arn
  instance_class        = "db.r5.large"
  publicly_accessible   = false
  backup_retention_period = 7
}

# Primary region (100% capacity)
module "primary_region" {
  source              = "./modules/app-region"
  region              = "us-east-1"
  capacity_percentage = 100
}

# DR region (30% capacity)
module "dr_region" {
  source              = "./modules/app-region"
  region              = "us-west-2"
  capacity_percentage = 30
}
```

**AWS Route53 Health Check and Failover:**

```hcl
resource "aws_route53_health_check" "primary" {
  fqdn              = "api-us-east-1.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_record" "primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  set_identifier = "primary"
  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }

  health_check_id = aws_route53_health_check.primary.id
}

resource "aws_route53_record" "dr" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  set_identifier = "dr"
  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = aws_lb.dr.dns_name
    zone_id                = aws_lb.dr.zone_id
    evaluate_target_health = true
  }
}
```

---

### 4. Hot Site / Active-Active (горячий резерв)

**Как работает:**
- Два полноценных региона работают одновременно
- Трафик распределяется между ними (50/50 или по геолокации)
- При disaster'е один регион принимает весь трафик

**Схема:**
```
Region A (50%)                  Region B (50%)
┌────────────────┐             ┌────────────────┐
│  Application   │             │  Application   │
│  [████████████]│◄───traffic─►│  [████████████]│
└────────┬───────┘             └────────┬───────┘
         │                              │
         ▼                              ▼
┌────────────────┐  bi-directional  ┌────────────────┐
│  Database      │◄────replication──►│  Database      │
│  (multi-master)│                   │  (multi-master)│
└────────────────┘                   └────────────────┘
```

**RTO:** Секунды
**RPO:** ~0 (синхронная репликация или eventual consistency)
**Стоимость:** Высокая (2x full cost)

**Multi-Region Database с CockroachDB:**

```yaml
# CockroachDB cluster across 3 regions
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cockroachdb
spec:
  serviceName: cockroachdb
  replicas: 9  # 3 per region
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - topologyKey: topology.kubernetes.io/zone
              labelSelector:
                matchLabels:
                  app: cockroachdb
      containers:
        - name: cockroachdb
          image: cockroachdb/cockroach:v23.1.0
          command:
            - /cockroach/cockroach
            - start
            - --logtostderr
            - --insecure
            - --advertise-host=$(POD_NAME).cockroachdb
            - --http-addr=0.0.0.0
            - --join=cockroachdb-0.cockroachdb,cockroachdb-1.cockroachdb,cockroachdb-2.cockroachdb
            - --locality=region=us-east-1,zone=$(POD_NAMESPACE)
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
```

**Global Load Balancing с AWS Global Accelerator:**

```hcl
resource "aws_globalaccelerator_accelerator" "main" {
  name            = "production-accelerator"
  ip_address_type = "IPV4"
  enabled         = true

  attributes {
    flow_logs_enabled   = true
    flow_logs_s3_bucket = aws_s3_bucket.flow_logs.bucket
  }
}

resource "aws_globalaccelerator_listener" "main" {
  accelerator_arn = aws_globalaccelerator_accelerator.main.id
  protocol        = "TCP"
  port_range {
    from_port = 443
    to_port   = 443
  }
}

resource "aws_globalaccelerator_endpoint_group" "us_east" {
  listener_arn = aws_globalaccelerator_listener.main.id
  endpoint_group_region = "us-east-1"
  traffic_dial_percentage = 50

  endpoint_configuration {
    endpoint_id = aws_lb.us_east.arn
    weight      = 100
  }

  health_check_interval_seconds = 30
  health_check_path            = "/health"
  health_check_protocol        = "HTTPS"
  threshold_count              = 3
}

resource "aws_globalaccelerator_endpoint_group" "us_west" {
  listener_arn = aws_globalaccelerator_listener.main.id
  endpoint_group_region = "us-west-2"
  traffic_dial_percentage = 50

  endpoint_configuration {
    endpoint_id = aws_lb.us_west.arn
    weight      = 100
  }

  health_check_interval_seconds = 30
  health_check_path            = "/health"
  health_check_protocol        = "HTTPS"
  threshold_count              = 3
}
```

---

## Database Disaster Recovery

### PostgreSQL Streaming Replication

```sql
-- On Primary
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret';

-- postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
archive_mode = on
archive_command = 'aws s3 cp %p s3://wal-archive/%f'
```

```bash
# Setup Standby
pg_basebackup -h primary.example.com -U replicator -D /var/lib/postgresql/data -P -Xs -R

# standby.signal file is created automatically by -R flag

# Start standby
pg_ctl start
```

### MySQL Multi-Source Replication

```sql
-- On Master 1
CREATE USER 'replicator'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;

-- On Replica
CHANGE MASTER TO
  MASTER_HOST='master1.example.com',
  MASTER_USER='replicator',
  MASTER_PASSWORD='password',
  MASTER_AUTO_POSITION=1
FOR CHANNEL 'master1';

START SLAVE FOR CHANNEL 'master1';

CHANGE MASTER TO
  MASTER_HOST='master2.example.com',
  MASTER_USER='replicator',
  MASTER_PASSWORD='password',
  MASTER_AUTO_POSITION=1
FOR CHANNEL 'master2';

START SLAVE FOR CHANNEL 'master2';
```

### Point-in-Time Recovery (PITR)

```bash
# PostgreSQL PITR

# 1. Take base backup
pg_basebackup -h localhost -U postgres -D /backup/base -Ft -z -P

# 2. Archive WAL files continuously (postgresql.conf)
# archive_command = 'cp %p /backup/wal/%f'

# 3. Restore to specific point in time
tar xzf /backup/base/base.tar.gz -C /var/lib/postgresql/data

# 4. Create recovery.conf
cat > /var/lib/postgresql/data/recovery.conf <<EOF
restore_command = 'cp /backup/wal/%f %p'
recovery_target_time = '2025-10-27 10:30:00'
recovery_target_action = 'promote'
EOF

# 5. Start PostgreSQL
pg_ctl start
```

---

## Application-Level Disaster Recovery

### Multi-Region Deployment с Kubernetes

```yaml
# kustomization.yaml
bases:
  - ../base

nameSuffix: -us-east-1
commonLabels:
  region: us-east-1

configMapGenerator:
  - name: app-config
    literals:
      - REGION=us-east-1
      - DB_HOST=db-us-east-1.example.com
      - REDIS_HOST=redis-us-east-1.example.com

---
# Different kustomization for us-west-2
nameSuffix: -us-west-2
commonLabels:
  region: us-west-2

configMapGenerator:
  - name: app-config
    literals:
      - REGION=us-west-2
      - DB_HOST=db-us-west-2.example.com
      - REDIS_HOST=redis-us-west-2.example.com
```

### Circuit Breaker для Graceful Degradation

```javascript
const CircuitBreaker = require('opossum');

// Если primary БД недоступна, переключаемся на replica
class DatabaseService {
  constructor() {
    this.primaryCircuit = new CircuitBreaker(this.queryPrimary.bind(this), {
      timeout: 3000,
      errorThresholdPercentage: 50,
      resetTimeout: 30000
    });

    this.primaryCircuit.fallback(() => this.queryReplica());
  }

  async queryPrimary(sql) {
    return await this.primaryDb.query(sql);
  }

  async queryReplica(sql) {
    console.log('Primary DB unavailable, using replica');
    return await this.replicaDb.query(sql);
  }

  async query(sql) {
    return await this.primaryCircuit.fire(sql);
  }
}
```

---

## Disaster Recovery Testing

### DR Drill Plan

```markdown
# Disaster Recovery Drill - 2025 Q4

## Objectives
- Test RTO/RPO targets
- Validate runbooks
- Train team on DR procedures

## Scenario
Simulate complete failure of us-east-1 region

## Steps

### 1. Pre-Drill (9:00 AM)
- [ ] Notify all stakeholders
- [ ] Take snapshot of current state
- [ ] Enable extra monitoring
- [ ] Prepare rollback plan

### 2. Simulate Disaster (9:30 AM)
- [ ] Terminate all EC2 instances in us-east-1
- [ ] Shutdown primary database
- [ ] Update Route53 to simulate DNS failure

### 3. Execute DR Procedures (9:35 AM)
- [ ] Declare disaster
- [ ] Execute failover runbook
- [ ] Promote standby database
- [ ] Scale up DR region
- [ ] Update DNS to DR region
- [ ] Run smoke tests

### 4. Validation (10:00 AM)
- [ ] Verify all services operational
- [ ] Check data integrity
- [ ] Test critical user flows
- [ ] Measure actual RTO/RPO

### 5. Failback (11:00 AM)
- [ ] Restore primary region
- [ ] Sync data from DR to primary
- [ ] Failback traffic to primary
- [ ] Verify replication resumed

### 6. Post-Mortem (2:00 PM)
- [ ] Document what went well
- [ ] Document issues found
- [ ] Update runbooks
- [ ] Create action items
```

**Automated DR Testing:**

```bash
#!/bin/bash
# Chaos Engineering for DR

set -e

echo "Starting DR drill..."

# 1. Take snapshot for rollback
echo "Creating snapshot..."
SNAPSHOT_ID=$(aws rds create-db-snapshot \
  --db-instance-identifier production-db \
  --db-snapshot-identifier dr-drill-$(date +%Y%m%d) \
  --query 'DBSnapshot.DBSnapshotIdentifier' \
  --output text)

# 2. Simulate primary region failure
echo "Simulating primary region failure..."
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name primary-asg \
  --desired-capacity 0

# 3. Monitor DR region takeover
echo "Waiting for DR region to take over..."
START_TIME=$(date +%s)

while true; do
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health)

  if [ "$HTTP_CODE" = "200" ]; then
    END_TIME=$(date +%s)
    RTO=$((END_TIME - START_TIME))
    echo "DR takeover successful! RTO: ${RTO}s"
    break
  fi

  sleep 5
done

# 4. Run smoke tests
echo "Running smoke tests..."
./scripts/smoke-tests.sh

# 5. Restore primary region
echo "Restoring primary region..."
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name primary-asg \
  --desired-capacity 3

echo "DR drill complete!"
```

---

## Monitoring and Alerting для DR

```yaml
# Prometheus alerts для DR scenarios
groups:
  - name: disaster_recovery
    interval: 30s
    rules:
      # Database replication lag
      - alert: HighReplicationLag
        expr: pg_replication_lag_seconds > 60
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High replication lag to DR region"
          description: "Replication lag is {{ $value }}s (RPO at risk)"

      # Cross-region health check failure
      - alert: DRRegionUnhealthy
        expr: up{job="dr-region-health"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "DR region is unhealthy"
          description: "Cannot reach DR region for {{ $value }} minutes"

      # Backup failure
      - alert: BackupFailed
        expr: time() - backup_last_success_timestamp > 86400
        labels:
          severity: critical
        annotations:
          summary: "Backup has not succeeded in 24 hours"
          description: "Last successful backup: {{ $value | humanizeDuration }}"

      # DR capacity insufficient
      - alert: DRCapacityInsufficient
        expr: |
          (
            kube_deployment_status_replicas_available{namespace="production", region="dr"}
            / kube_deployment_spec_replicas{namespace="production", region="primary"}
          ) < 0.25
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "DR region has insufficient capacity"
          description: "DR region has only {{ $value | humanizePercentage }} of primary capacity"
```

---

## Runbook Template

```markdown
# Disaster Recovery Runbook

## Trigger Conditions
- Primary region is completely unavailable for > 5 minutes
- Database corruption detected
- Security incident requiring isolation

## Prerequisites
- Access to AWS/GCP console
- Access to DR region infrastructure
- PagerDuty incident created
- War room established

## Failover Procedure

### Step 1: Assess Situation (5 min)
**Goal:** Determine if DR failover is necessary

- [ ] Check primary region status dashboard
- [ ] Verify database availability
- [ ] Check cloud provider status page
- [ ] Estimate recovery time if staying in primary

**Decision:** Proceed with DR if recovery time > RTO (4 hours)

### Step 2: Initiate Failover (10 min)
**Goal:** Begin traffic shift to DR region

```bash
# Promote standby database
kubectl exec -it postgres-standby-0 -n production -- \
  psql -c "SELECT pg_promote();"

# Scale up DR region
kubectl scale deployment --all --replicas=10 -n production

# Update DNS
./scripts/failover-dns.sh --to=dr-region
```

**Expected:** DNS propagation starts (TTL: 60s)

### Step 3: Verify DR Region (15 min)
**Goal:** Confirm DR region is serving traffic correctly

- [ ] Check application logs for errors
- [ ] Run smoke tests: `./scripts/smoke-tests.sh`
- [ ] Verify database is accepting writes
- [ ] Check monitoring dashboards
- [ ] Test critical user flows manually

### Step 4: Monitor (ongoing)
**Goal:** Ensure stability in DR region

- [ ] Watch error rates in Grafana
- [ ] Monitor database replication (if multi-master)
- [ ] Check capacity/autoscaling
- [ ] Communicate status to stakeholders

### Step 5: Failback (when ready)
**Goal:** Return to primary region

- [ ] Verify primary region is healthy
- [ ] Sync data from DR to primary
- [ ] Test primary region in shadow mode
- [ ] Gradual traffic shift (10% → 50% → 100%)
- [ ] Monitor for issues

## Rollback Plan
If DR failover fails:
1. Revert DNS changes: `./scripts/failover-dns.sh --to=primary`
2. Investigate root cause
3. Attempt manual recovery

## Communication Template
```
Subject: [INCIDENT] Disaster Recovery Failover Initiated

We have initiated a disaster recovery failover due to [reason].

Status: IN PROGRESS
RTO: 4 hours
Current Impact: [describe]
ETA: [time]

Updates will be posted every 30 minutes.
```

## Post-Incident
- [ ] Schedule post-mortem
- [ ] Update runbook with lessons learned
- [ ] Fix identified issues
- [ ] Re-test DR procedures
```

---

## Best Practices

### 1. Automate Everything
Ручной DR failover медленный и подвержен ошибкам.

### 2. Test Regularly
DR plan, который не тестируется, не работает. Минимум 2 раза в год.

### 3. Multi-Region by Default
Для critical систем используйте хотя бы Warm Standby.

### 4. Monitor Replication Lag
RPO зависит от replication lag — мониторьте его 24/7.

### 5. Document Everything
Runbook должен быть настолько детальным, чтобы junior engineer мог выполнить failover.

### 6. Immutable Backups
Защитите backups от ransomware (immutable storage, air gap).

### 7. Practice Failback
Failover — это только половина. Failback обратно в primary тоже нужно тренировать.

---

## Trade-offs

| Стратегия | RTO | RPO | Cost | Complexity |
|-----------|-----|-----|------|------------|
| **Backup & Restore** | Hours | Hours | Low | Low |
| **Pilot Light** | 10-60 min | Minutes | Medium | Medium |
| **Warm Standby** | Minutes | Seconds | Medium-High | Medium |
| **Active-Active** | Seconds | Near-zero | High (2x) | High |

---

## Что почитать дальше?

1. **Урок 55: Kubernetes в Production** — HA setup
2. **Урок 57: Security Best Practices** — security для DR
3. **Урок 19: Replication** — database replication patterns

---

## Проверь себя

1. В чём разница между RTO и RPO?
2. Какие 4 основные стратегии DR существуют (Backup/Restore, Pilot Light, Warm Standby, Active-Active)?
3. Как реализовать PostgreSQL streaming replication для DR?
4. Зачем нужен Point-in-Time Recovery (PITR)?
5. Как тестировать DR plan (chaos engineering)?
6. Что должно быть в DR runbook?
7. Как сделать automatic failover с Route53 health checks?
8. Почему важно тестировать не только failover, но и failback?

---
**Предыдущий урок**: [Урок 55: Kubernetes в Production](55-kubernetes.md)
**Следующий урок**: [Урок 57: Security Best Practices](57-security.md)
