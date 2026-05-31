# Deployment & Infrastructure Guide

## Production Architecture Overview

### Multi-Tier Deployment Stack

```
┌─────────────────────────────────────────────────────────────┐
│                     CDN (Cloudflare)                         │
│                 Static Assets, Images, Maps                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   NGINX Load Balancer                        │
│              (Round-robin, sticky sessions)                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              Kubernetes Cluster (16+ nodes)                  │
├─────────────────────────────────────────────────────────────┤
│ Backend API Pods (16 replicas)                              │
│ Game Engine Pods (4 dedicated workers)                      │
│ WebSocket Service (8 replicas)                             │
│ Background Job Workers (4 replicas)                        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              Data Layer (High Availability)                  │
├─────────────────────────────────────────────────────────────┤
│ PostgreSQL Primary (RDS Multi-AZ)                           │
│ PostgreSQL Read Replica 1                                   │
│ PostgreSQL Read Replica 2                                   │
│ PostgreSQL Standby (automatic failover)                     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              Cache Layer (Redis Cluster)                    │
├─────────────────────────────────────────────────────────────┤
│ Redis Shard 1 (Primary + Replica)                           │
│ Redis Shard 2 (Primary + Replica)                           │
│ Redis Shard 3 (Primary + Replica)                           │
│ Redis Shard 4 (Primary + Replica)                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Storage (S3/MinIO)                        │
│          Backups, Logs, User Uploads, Game Maps            │
└─────────────────────────────────────────────────────────────┘
```

## Docker Configuration

### Backend Dockerfile

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source code
COPY . .

# Build TypeScript
RUN npm run build

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

# Start server
CMD ["npm", "start"]
```

### Frontend Dockerfile

```dockerfile
FROM node:18-alpine as build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

ARG REACT_APP_API_URL=http://localhost:3000
ARG REACT_APP_WS_URL=ws://localhost:3000

ENV REACT_APP_API_URL=$REACT_APP_API_URL
ENV REACT_APP_WS_URL=$REACT_APP_WS_URL

RUN npm run build

FROM nginx:alpine

COPY --from=build /app/build /usr/share/nginx/html

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

## Kubernetes Deployment

### Backend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: call-of-war-backend
  namespace: production
spec:
  replicas: 16
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 3
      maxUnavailable: 1
  selector:
    matchLabels:
      app: call-of-war-backend
  template:
    metadata:
      labels:
        app: call-of-war-backend
    spec:
      containers:
      - name: backend
        image: call-of-war-backend:latest
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        resources:
          requests:
            cpu: 1000m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 2Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Database Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  namespace: production
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgres
    role: primary
  type: ClusterIP

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Gi
  storageClassName: fast-ssd
```

### Redis Cluster ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-config
  namespace: production
data:
  redis.conf: |
    cluster-enabled yes
    cluster-config-file /var/lib/redis/nodes.conf
    cluster-node-timeout 5000
    appendonly yes
    maxmemory 2gb
    maxmemory-policy allkeys-lru
    save 900 1
    save 300 10
    save 60 10000
```

## Monitoring & Observability

### Prometheus Configuration

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
- job_name: 'backend'
  static_configs:
  - targets: ['localhost:9090']
  
- job_name: 'postgres'
  static_configs:
  - targets: ['localhost:9187']
  
- job_name: 'redis'
  static_configs:
  - targets: ['localhost:9121']

alerting:
  alertmanagers:
  - static_configs:
    - targets: ['localhost:9093']
```

### Key Metrics to Monitor

```
game_tick_duration_ms          # Game loop tick time
game_active_players            # Current player count
game_armies_count              # Total armies
game_units_count               # Total units
game_combats_active            # Active battles

database_query_duration_ms     # Query latency
database_connections_active    # Active DB connections
database_connections_idle      # Idle connections
database_connection_errors     # Connection failures

redis_cache_hits               # Cache hit ratio
redis_memory_used_bytes        # Cache memory usage
redis_evictions_total          # Cache evictions

websocket_connections_active   # Active WS connections
websocket_messages_per_second  # Message throughput
websocket_connection_errors    # Connection failures

http_request_duration_seconds  # API latency
http_requests_total            # Request count
http_errors_total              # Error count
```

### Grafana Dashboards

**Dashboard 1: Game Health**
- Active players (gauge)
- Tick latency (graph)
- Combat count (gauge)
- Server CPU/Memory (gauge)

**Dashboard 2: Database Performance**
- Query time (percentiles)
- Connection pool status
- Replication lag
- Backup status

**Dashboard 3: Infrastructure**
- Pod count by service
- Network throughput
- Disk usage
- Error rates

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
      redis:
        image: redis:7
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
        node-version: '18'
    - run: npm ci
    - run: npm run lint
    - run: npm run test
    - run: npm run build

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: docker/setup-buildx-action@v2
    - uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    - uses: docker/build-push-action@v4
      with:
        context: ./backend
        push: true
        tags: myregistry/call-of-war-backend:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: azure/setup-kubectl@v3
    - run: kubectl apply -f k8s/production.yaml
    - run: kubectl rollout status deployment/call-of-war-backend -n production
```

## Scaling Strategy

### Horizontal Scaling

**Scale Out Backend:**
```bash
kubectl scale deployment call-of-war-backend --replicas=32 -n production
```

**Scale Redis:**
```bash
# Add new Redis shard
redis-cli --cluster add-node 10.0.0.100:6379 10.0.0.1:6379 --cluster-slave
```

**Scale Database:**
```bash
# Create read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier call-of-war-replica-2 \
  --source-db-identifier call-of-war-primary
```

### Auto-Scaling Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-autoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: call-of-war-backend
  minReplicas: 16
  maxReplicas: 64
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## Backup & Disaster Recovery

### Database Backup Strategy

```
Primary Backup:
- Frequency: Every 6 hours
- Retention: 30 days
- Storage: S3 (redundant regions)

WAL Archiving:
- Continuous archiving to S3
- Enables point-in-time recovery
- Retention: 7 days

Replication:
- Synchronous to standby
- Automatic failover
- RPO: <1 second
```

### Disaster Recovery Procedures

**RTO: 5 minutes** (Recovery Time Objective)
**RPO: <1 minute** (Recovery Point Objective)

```bash
# 1. Detect failure
kubectl logs pod-name -n production

# 2. Failover database
aws rds promote-read-replica --db-instance-identifier call-of-war-replica-1

# 3. Update connection strings
kubectl set env deployment/call-of-war-backend \
  DATABASE_URL=postgresql://new-primary:5432/call_of_war

# 4. Verify health
kubectl rollout status deployment/call-of-war-backend

# 5. Restore cache (warm-up)
redis-cli --cluster call-of-war-backup.rdb RESTORE
```

## Security Hardening

### Network Security

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: call-of-war-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: call-of-war-backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: nginx-ingress
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
```

### SSL/TLS Configuration

```
Certificates: Let's Encrypt (auto-renewal)
Protocol: TLS 1.3
Ciphers: ECDHE + AES-256-GCM
HSTS: Enabled (1 year)
Certificate Pinning: Enabled for API

NGINX Configuration:
ssl_certificate /etc/letsencrypt/live/call-of-war.io/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/call-of-war.io/privkey.pem;
ssl_protocols TLSv1.3 TLSv1.2;
ssl_ciphers HIGH:!aNULL:!MD5;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

### Secrets Management

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  jwt-secret: BASE64_ENCODED_SECRET
  db-password: BASE64_ENCODED_PASSWORD
  redis-password: BASE64_ENCODED_PASSWORD
  api-key: BASE64_ENCODED_API_KEY
```

## Performance Tuning

### PostgreSQL Configuration

```ini
# postgresql.conf
max_connections = 300
shared_buffers = 25GB
effective_cache_size = 75GB
work_mem = 50MB
maintenance_work_mem = 2GB
random_page_cost = 1.1
effective_io_concurrency = 200
wal_buffers = 16MB
default_statistics_target = 100
```

### Redis Configuration

```conf
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru
tcp-keepalive 60
timeout 300
tcp-backlog 511
```

### NGINX Tuning

```nginx
upstream backend {
  least_conn;
  server backend-1:3000 max_fails=3 fail_timeout=30s;
  server backend-2:3000 max_fails=3 fail_timeout=30s;
  server backend-3:3000 max_fails=3 fail_timeout=30s;
  keepalive 32;
}

server {
  listen 80;
  client_max_body_size 10M;
  
  location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_buffering off;
    proxy_request_buffering off;
  }
  
  location /socket.io {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

## Cost Optimization

### Resource Allocation

```
Compute (Kubernetes nodes):
- 16 nodes × m5.2xlarge = $4,000/month
- Auto-scaling adds 25% during peak = additional $1,000/month

Database (PostgreSQL RDS):
- db.r6i.4xlarge primary = $3,500/month
- db.r6i.2xlarge replicas (2×) = $3,500/month
- Storage (500GB SSD) = $500/month

Cache (Redis):
- cache.r6g.xlarge (4 shards) = $1,200/month

Data Transfer (CDN):
- CloudFront = $500/month average

Monitoring (Prometheus/Grafana):
- Self-hosted = minimal ($200/month)

Total Monthly: ~$14,400/month for 10,000 players
Cost per player: ~$1.44/month
```

---

## Monitoring Checklist

- [ ] Prometheus scraping all targets
- [ ] Grafana dashboards displaying metrics
- [ ] PagerDuty alerts configured
- [ ] Backup jobs running successfully
- [ ] SSL certificates auto-renewing
- [ ] Log aggregation working (ELK/Loki)
- [ ] Database replication healthy
- [ ] Redis cluster balanced
- [ ] Load balancer health checks passing
- [ ] Disaster recovery tested monthly

