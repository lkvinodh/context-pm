# Production Deployment Guide

> **Complete guide for deploying and maintaining AI agent systems in production environments**

This guide covers infrastructure setup, configuration management, security hardening, monitoring, cost optimization, and operational best practices for running production-grade AI agent systems.

---

## Table of Contents

1. [Overview](#overview)
2. [Infrastructure Requirements](#infrastructure-requirements)
3. [Deployment Architecture](#deployment-architecture)
4. [Installation & Setup](#installation--setup)
5. [Configuration Management](#configuration-management)
6. [Security & Compliance](#security--compliance)
7. [Monitoring & Observability](#monitoring--observability)
8. [Cost Management](#cost-management)
9. [Maintenance & Operations](#maintenance--operations)
10. [Troubleshooting](#troubleshooting)
11. [Disaster Recovery](#disaster-recovery)

---

## Overview

### Production Readiness Checklist

```markdown
## Production Readiness

### Core Functionality
- [ ] All data sources integrated and tested
- [ ] Incremental sync working reliably
- [ ] Retrieval system returning relevant results
- [ ] Agent responses meet quality standards
- [ ] Error handling covers all edge cases

### Performance
- [ ] Latency meets SLA (< 10s for 95th percentile)
- [ ] System handles expected query volume
- [ ] Cost per query within budget
- [ ] Memory usage stable over time

### Security
- [ ] Credentials stored securely (not in code)
- [ ] API tokens rotated regularly
- [ ] Audit logging enabled
- [ ] Access controls implemented
- [ ] Data encryption at rest and in transit

### Observability
- [ ] Structured logging configured
- [ ] Metrics exported (latency, tokens, errors)
- [ ] Alerting rules defined
- [ ] Dashboard created for monitoring

### Operations
- [ ] Deployment automated (CI/CD)
- [ ] Rollback procedure tested
- [ ] Backup strategy implemented
- [ ] Runbook documented
- [ ] On-call rotation defined
```

### Deployment Models

| Model | Use Case | Pros | Cons |
|-------|----------|------|------|
| **Local Desktop** | Personal productivity, prototyping | Simple, no infra cost | Single user, no HA |
| **VM/Server** | Small team (< 10 users) | Shared access, persistent | Manual scaling, single point of failure |
| **Container (Docker)** | Team deployment (10-50 users) | Portable, reproducible | Requires orchestration |
| **Kubernetes** | Enterprise (50+ users) | Auto-scaling, HA, load balancing | Complex setup |
| **Serverless** | Variable/bursty workload | Zero infra management, pay-per-use | Cold starts, vendor lock-in |

**Recommended for most teams**: Start with **Container (Docker)** → scale to **Kubernetes** if needed.

---

## Infrastructure Requirements

### Minimum Hardware Specs

**For Single User (Local Deployment)**:
```
CPU: 4 cores (Intel i5 or equivalent)
RAM: 8GB
Storage: 50GB SSD
Network: 10 Mbps stable connection
```

**For Team Deployment (10-50 users)**:
```
CPU: 8-16 cores
RAM: 32GB
Storage: 500GB SSD
Network: 100 Mbps
```

**For Enterprise Deployment (50+ users)**:
```
CPU: 32+ cores (distributed)
RAM: 128GB+ (distributed)
Storage: 2TB+ (NAS or distributed)
Network: 1 Gbps
Load Balancer: Required
```

### Cloud Provider Recommendations

**AWS**:
```yaml
Instance Type: t3.xlarge (4 vCPU, 16GB RAM)
Storage: EBS gp3 (500GB)
Estimated Cost: ~$150/month + data transfer

Kubernetes:
  EKS Cluster: $75/month
  Worker Nodes: 2x t3.xlarge = $300/month
  Load Balancer: $20/month
  Total: ~$400/month
```

**Azure**:
```yaml
Instance Type: Standard_D4s_v3 (4 vCPU, 16GB RAM)
Storage: Premium SSD (500GB)
Estimated Cost: ~$160/month + bandwidth

AKS Cluster: Free control plane
  Worker Nodes: 2x Standard_D4s_v3 = $320/month
  Total: ~$350/month
```

**GCP**:
```yaml
Instance Type: n2-standard-4 (4 vCPU, 16GB RAM)
Storage: SSD Persistent Disk (500GB)
Estimated Cost: ~$140/month + egress

GKE Cluster: $75/month
  Worker Nodes: 2x n2-standard-4 = $280/month
  Total: ~$360/month
```

### On-Premise Requirements

```yaml
Hardware:
  Server: Dell PowerEdge R640 or equivalent
  CPU: Intel Xeon (16+ cores)
  RAM: 64GB ECC
  Storage: 1TB NVMe SSD RAID
  Network: Dual 10GbE NICs

Software:
  OS: Ubuntu 22.04 LTS
  Container Runtime: Docker 24.0+
  Orchestration: Kubernetes 1.28+ (optional)
  
Backup:
  NAS: Synology or QNAP (4TB+)
  Backup Schedule: Daily incremental, weekly full
  
Networking:
  Firewall: Enterprise-grade (Palo Alto, Fortinet)
  VPN: Required for remote access
  Internal DNS: Required
```

---

## Deployment Architecture

### Container-Based Deployment (Recommended)

```dockerfile
# Dockerfile

FROM python:3.10-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY src/ ./src/
COPY config/ ./config/

# Create data directories
RUN mkdir -p /data/documents /data/embeddings /data/state /logs

# Non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app /data /logs
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Expose port
EXPOSE 8000

# Run application
CMD ["python", "-m", "uvicorn", "src.api.app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose (Multi-Container)

```yaml
# docker-compose.yml

version: '3.8'

services:
  agent-api:
    build: .
    container_name: ai-agent-api
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - GITHUB_TOKEN=${GITHUB_TOKEN}
      - CONFLUENCE_TOKEN=${CONFLUENCE_TOKEN}
      - LOG_LEVEL=INFO
    volumes:
      - ./data:/data
      - ./logs:/logs
      - ./config:/app/config:ro
    networks:
      - agent-network
    depends_on:
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    container_name: ai-agent-cache
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - agent-network
    command: redis-server --appendonly yes

  nginx:
    image: nginx:alpine
    container_name: ai-agent-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    networks:
      - agent-network
    depends_on:
      - agent-api

networks:
  agent-network:
    driver: bridge

volumes:
  redis-data:
```

### Kubernetes Deployment

```yaml
# k8s/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-agent
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: ai-agent
  template:
    metadata:
      labels:
        app: ai-agent
        version: v1.0
    spec:
      containers:
      - name: agent-api
        image: your-registry/ai-agent:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: ai-agent-secrets
              key: anthropic-api-key
        - name: LOG_LEVEL
          value: "INFO"
        resources:
          requests:
            memory: "4Gi"
            cpu: "1000m"
          limits:
            memory: "8Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
        volumeMounts:
        - name: data
          mountPath: /data
        - name: config
          mountPath: /app/config
          readOnly: true
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: ai-agent-data
      - name: config
        configMap:
          name: ai-agent-config

---
apiVersion: v1
kind: Service
metadata:
  name: ai-agent-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: ai-agent
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-agent-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - ai-agent.yourcompany.com
    secretName: ai-agent-tls
  rules:
  - host: ai-agent.yourcompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ai-agent-service
            port:
              number: 80
```

---

## Installation & Setup

### Step-by-Step Deployment

#### 1. Prepare Environment

```bash
# Clone repository
git clone https://github.com/yourorg/production-ai-agent.git
cd production-ai-agent

# Create production configuration
cp .env.example .env.production
nano .env.production  # Edit with production values
```

#### 2. Build Container

```bash
# Build Docker image
docker build -t ai-agent:latest .

# Tag for registry
docker tag ai-agent:latest your-registry.com/ai-agent:v1.0

# Push to registry
docker push your-registry.com/ai-agent:v1.0
```

#### 3. Deploy with Docker Compose

```bash
# Load environment
export $(cat .env.production | xargs)

# Deploy
docker-compose up -d

# Check status
docker-compose ps
docker-compose logs -f agent-api
```

#### 4. Initial Sync

```bash
# Run initial document sync
docker-compose exec agent-api python -m src.main sync

# Verify sync
docker-compose exec agent-api python -m src.main status
```

#### 5. Test Deployment

```bash
# Health check
curl http://localhost:8000/health

# Test query
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What is X?"}'
```

---

## Configuration Management

### Environment-Specific Configs

```python
# config/environments.py

from dataclasses import dataclass
from typing import Optional

@dataclass
class Environment:
    """Environment configuration"""
    name: str
    debug: bool
    log_level: str
    api_host: str
    api_port: int
    
    # External services
    anthropic_api_url: str
    github_url: str
    confluence_url: str
    
    # Performance
    max_workers: int
    request_timeout: int
    
    # Caching
    enable_cache: bool
    cache_ttl: int

# Development
DEV = Environment(
    name="development",
    debug=True,
    log_level="DEBUG",
    api_host="localhost",
    api_port=8000,
    anthropic_api_url="https://api.anthropic.com",
    github_url="https://github.com",
    confluence_url="https://yourcompany.atlassian.net",
    max_workers=2,
    request_timeout=60,
    enable_cache=False,
    cache_ttl=300
)

# Production
PROD = Environment(
    name="production",
    debug=False,
    log_level="INFO",
    api_host="0.0.0.0",
    api_port=8000,
    anthropic_api_url="https://api.anthropic.com",
    github_url="https://github.yourcompany.com",
    confluence_url="https://confluence.yourcompany.com",
    max_workers=8,
    request_timeout=120,
    enable_cache=True,
    cache_ttl=3600
)

def get_environment(env_name: str = "production") -> Environment:
    """Get environment configuration"""
    envs = {
        "development": DEV,
        "staging": STAGING,
        "production": PROD
    }
    return envs.get(env_name, PROD)
```

### Secret Management

**Using Kubernetes Secrets**:

```bash
# Create secrets
kubectl create secret generic ai-agent-secrets \
  --from-literal=anthropic-api-key=$ANTHROPIC_API_KEY \
  --from-literal=github-token=$GITHUB_TOKEN \
  --from-literal=confluence-token=$CONFLUENCE_TOKEN \
  -n production

# Verify
kubectl get secrets -n production
```

**Using HashiCorp Vault**:

```python
# config/secrets.py

import hvac
import os

class SecretManager:
    """Manage secrets with Vault"""
    
    def __init__(self):
        self.client = hvac.Client(
            url=os.getenv("VAULT_ADDR"),
            token=os.getenv("VAULT_TOKEN")
        )
    
    def get_secret(self, path: str, key: str) -> str:
        """Retrieve secret from Vault"""
        secret = self.client.secrets.kv.v2.read_secret_version(path=path)
        return secret['data']['data'][key]
    
    def rotate_secret(self, path: str, key: str, new_value: str):
        """Rotate secret"""
        self.client.secrets.kv.v2.create_or_update_secret(
            path=path,
            secret={key: new_value}
        )

# Usage
secrets = SecretManager()
api_key = secrets.get_secret("production/ai-agent", "anthropic_api_key")
```

---

## Security & Compliance

### 1. API Authentication

```python
# src/api/auth.py

from fastapi import Security, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from datetime import datetime, timedelta

security = HTTPBearer()

class AuthManager:
    """API authentication"""
    
    SECRET_KEY = os.getenv("JWT_SECRET_KEY")
    ALGORITHM = "HS256"
    
    @staticmethod
    def create_token(user_id: str) -> str:
        """Generate JWT token"""
        payload = {
            "user_id": user_id,
            "exp": datetime.utcnow() + timedelta(hours=24)
        }
        return jwt.encode(payload, AuthManager.SECRET_KEY, algorithm=AuthManager.ALGORITHM)
    
    @staticmethod
    def verify_token(credentials: HTTPAuthorizationCredentials = Security(security)) -> str:
        """Verify JWT token"""
        try:
            payload = jwt.decode(
                credentials.credentials,
                AuthManager.SECRET_KEY,
                algorithms=[AuthManager.ALGORITHM]
            )
            return payload["user_id"]
        except jwt.ExpiredSignatureError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Token expired"
            )
        except jwt.JWTError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token"
            )

# Protect endpoints
@app.post("/query")
async def query(
    request: QueryRequest,
    user_id: str = Depends(AuthManager.verify_token)
):
    # Process query...
    pass
```

### 2. Rate Limiting

```python
# src/api/rate_limit.py

from fastapi import Request, HTTPException
from collections import defaultdict
from datetime import datetime, timedelta
import asyncio

class RateLimiter:
    """Rate limiting per user"""
    
    def __init__(self, requests_per_minute: int = 60):
        self.requests_per_minute = requests_per_minute
        self.requests = defaultdict(list)
        self.lock = asyncio.Lock()
    
    async def check_limit(self, user_id: str):
        """Check if user is within rate limit"""
        async with self.lock:
            now = datetime.now()
            
            # Remove old requests (> 1 minute ago)
            self.requests[user_id] = [
                req_time for req_time in self.requests[user_id]
                if now - req_time < timedelta(minutes=1)
            ]
            
            # Check limit
            if len(self.requests[user_id]) >= self.requests_per_minute:
                raise HTTPException(
                    status_code=429,
                    detail=f"Rate limit exceeded: {self.requests_per_minute} requests per minute"
                )
            
            # Add current request
            self.requests[user_id].append(now)

# Middleware
rate_limiter = RateLimiter(requests_per_minute=60)

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    if request.url.path.startswith("/query"):
        user_id = request.state.user_id  # Set by auth middleware
        await rate_limiter.check_limit(user_id)
    
    response = await call_next(request)
    return response
```

### 3. Audit Logging

```python
# src/utils/audit.py

import json
import logging
from datetime import datetime
from pathlib import Path

class AuditLogger:
    """Audit logging for compliance"""
    
    def __init__(self, audit_file: str = "/logs/audit.jsonl"):
        self.audit_file = Path(audit_file)
        self.audit_file.parent.mkdir(parents=True, exist_ok=True)
        self.logger = logging.getLogger("audit")
    
    def log_query(
        self,
        user_id: str,
        query: str,
        documents_accessed: list,
        response_length: int,
        duration_ms: float
    ):
        """Log query for audit"""
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": "QUERY",
            "user_id": user_id,
            "query_hash": hashlib.sha256(query.encode()).hexdigest(),
            "documents_accessed": documents_accessed,
            "response_length": response_length,
            "duration_ms": duration_ms
        }
        
        self._write_entry(entry)
    
    def log_access(self, user_id: str, resource: str, action: str):
        """Log resource access"""
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": "ACCESS",
            "user_id": user_id,
            "resource": resource,
            "action": action
        }
        
        self._write_entry(entry)
    
    def _write_entry(self, entry: dict):
        """Append entry to audit log"""
        with open(self.audit_file, 'a') as f:
            f.write(json.dumps(entry) + '\n')
        
        self.logger.info(f"Audit: {entry['event_type']} by {entry['user_id']}")

# Usage
audit_logger = AuditLogger()

@app.post("/query")
async def query(request: QueryRequest, user_id: str = Depends(auth)):
    start = time.time()
    
    result = agent.query(request.query)
    
    duration_ms = (time.time() - start) * 1000
    
    audit_logger.log_query(
        user_id=user_id,
        query=request.query,
        documents_accessed=[doc.id for doc in result.documents],
        response_length=len(result.response),
        duration_ms=duration_ms
    )
    
    return result
```

---

## Monitoring & Observability

### Prometheus Metrics

```python
# src/utils/metrics.py

from prometheus_client import Counter, Histogram, Gauge, generate_latest
import time

# Define metrics
query_counter = Counter(
    'agent_queries_total',
    'Total number of queries',
    ['architecture', 'category', 'status']
)

query_duration = Histogram(
    'agent_query_duration_seconds',
    'Query processing time',
    ['architecture', 'category'],
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0, 60.0]
)

token_usage = Histogram(
    'agent_tokens_used',
    'Token usage per query',
    ['architecture', 'model'],
    buckets=[100, 500, 1000, 5000, 10000, 50000, 100000]
)

cost_per_query = Histogram(
    'agent_cost_usd',
    'Cost per query in USD',
    ['architecture', 'model'],
    buckets=[0.001, 0.01, 0.1, 0.5, 1.0, 5.0]
)

active_queries = Gauge(
    'agent_active_queries',
    'Number of currently processing queries'
)

error_counter = Counter(
    'agent_errors_total',
    'Total number of errors',
    ['error_type']
)

# Instrumentation
class MetricsCollector:
    """Collect and export metrics"""
    
    @staticmethod
    def record_query(
        architecture: str,
        category: str,
        duration: float,
        tokens: int,
        cost: float,
        model: str,
        status: str = "success"
    ):
        """Record query metrics"""
        query_counter.labels(
            architecture=architecture,
            category=category,
            status=status
        ).inc()
        
        query_duration.labels(
            architecture=architecture,
            category=category
        ).observe(duration)
        
        token_usage.labels(
            architecture=architecture,
            model=model
        ).observe(tokens)
        
        cost_per_query.labels(
            architecture=architecture,
            model=model
        ).observe(cost)
    
    @staticmethod
    def record_error(error_type: str):
        """Record error"""
        error_counter.labels(error_type=error_type).inc()

# FastAPI endpoint
@app.get("/metrics")
async def metrics():
    """Prometheus metrics endpoint"""
    return Response(
        content=generate_latest(),
        media_type="text/plain"
    )
```

### Grafana Dashboard

```json
{
  "dashboard": {
    "title": "AI Agent System",
    "panels": [
      {
        "title": "Query Rate",
        "targets": [
          {
            "expr": "rate(agent_queries_total[5m])"
          }
        ]
      },
      {
        "title": "P95 Latency",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(agent_query_duration_seconds_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Cost per Hour",
        "targets": [
          {
            "expr": "sum(rate(agent_cost_usd_sum[1h]))"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(agent_errors_total[5m])"
          }
        ]
      }
    ]
  }
}
```

### Alerting Rules

```yaml
# prometheus/alerts.yml

groups:
  - name: ai-agent-alerts
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(agent_errors_total[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors/sec"
      
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(agent_query_duration_seconds_bucket[5m])) > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High query latency"
          description: "P95 latency is {{ $value }}s"
      
      - alert: HighCost
        expr: sum(rate(agent_cost_usd_sum[1h])) > 10
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High API costs"
          description: "Cost is ${{ $value }}/hour"
```

---

## Cost Management

### Cost Tracking

```python
# src/utils/cost_tracker.py

from dataclasses import dataclass
from datetime import datetime, timedelta
import json

@dataclass
class CostReport:
    """Cost report"""
    period_start: datetime
    period_end: datetime
    total_cost: float
    total_queries: int
    cost_per_query: float
    breakdown_by_model: dict
    breakdown_by_user: dict

class CostTracker:
    """Track and report API costs"""
    
    def __init__(self, cost_file: str = "/data/costs.jsonl"):
        self.cost_file = cost_file
    
    def record_cost(
        self,
        user_id: str,
        model: str,
        input_tokens: int,
        output_tokens: int,
        cost: float
    ):
        """Record cost entry"""
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "user_id": user_id,
            "model": model,
            "input_tokens": input_tokens,
            "output_tokens": output_tokens,
            "cost_usd": cost
        }
        
        with open(self.cost_file, 'a') as f:
            f.write(json.dumps(entry) + '\n')
    
    def generate_report(self, days: int = 30) -> CostReport:
        """Generate cost report"""
        end = datetime.utcnow()
        start = end - timedelta(days=days)
        
        entries = self._load_entries(start, end)
        
        total_cost = sum(e["cost_usd"] for e in entries)
        total_queries = len(entries)
        
        # Breakdown by model
        by_model = {}
        for entry in entries:
            model = entry["model"]
            by_model[model] = by_model.get(model, 0) + entry["cost_usd"]
        
        # Breakdown by user
        by_user = {}
        for entry in entries:
            user = entry["user_id"]
            by_user[user] = by_user.get(user, 0) + entry["cost_usd"]
        
        return CostReport(
            period_start=start,
            period_end=end,
            total_cost=total_cost,
            total_queries=total_queries,
            cost_per_query=total_cost / total_queries if total_queries > 0 else 0,
            breakdown_by_model=by_model,
            breakdown_by_user=by_user
        )
    
    def _load_entries(self, start: datetime, end: datetime) -> list:
        """Load entries within time range"""
        entries = []
        
        with open(self.cost_file, 'r') as f:
            for line in f:
                entry = json.loads(line)
                timestamp = datetime.fromisoformat(entry["timestamp"])
                
                if start <= timestamp <= end:
                    entries.append(entry)
        
        return entries

# Usage
cost_tracker = CostTracker()

# Generate monthly report
report = cost_tracker.generate_report(days=30)
print(f"Total cost: ${report.total_cost:.2f}")
print(f"Queries: {report.total_queries}")
print(f"Cost per query: ${report.cost_per_query:.4f}")
```

### Cost Optimization Strategies

1. **Prompt Caching** (Anthropic feature):
```python
# Enable prompt caching for repeated contexts
response = client.messages.create(
    model="claude-sonnet-4.6",
    system=[
        {
            "type": "text",
            "text": "Long context that doesn't change...",
            "cache_control": {"type": "ephemeral"}  # Cache this
        }
    ],
    messages=[...]
)
```

2. **Model Selection**:
```python
# Use smaller models for simple tasks
def select_model(query_complexity: str) -> str:
    if query_complexity == "simple":
        return "claude-haiku-4-5"  # $0.25/MTok
    elif query_complexity == "medium":
        return "claude-sonnet-4.6"  # $3/MTok
    else:
        return "claude-opus-4.7"  # $15/MTok
```

3. **Context Optimization**:
```python
# Limit context to essential documents
def optimize_context(documents: List[Document], max_tokens: int = 10000):
    total_tokens = 0
    selected = []
    
    for doc in documents:
        doc_tokens = count_tokens(doc.content)
        if total_tokens + doc_tokens <= max_tokens:
            selected.append(doc)
            total_tokens += doc_tokens
        else:
            break
    
    return selected
```

---

## Maintenance & Operations

### Backup Strategy

```bash
#!/bin/bash
# scripts/backup.sh

BACKUP_DIR="/backups/ai-agent"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup documents
tar -czf $BACKUP_DIR/documents_$TIMESTAMP.tar.gz /data/documents/

# Backup state
cp /data/state/sync_state.json $BACKUP_DIR/sync_state_$TIMESTAMP.json

# Backup embeddings
tar -czf $BACKUP_DIR/embeddings_$TIMESTAMP.tar.gz /data/embeddings/

# Backup logs
tar -czf $BACKUP_DIR/logs_$TIMESTAMP.tar.gz /logs/

# Cleanup old backups (keep last 30 days)
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete
find $BACKUP_DIR -name "*.json" -mtime +30 -delete

echo "Backup completed: $TIMESTAMP"
```

### Scheduled Tasks

```bash
# Crontab configuration

# Daily sync at 2 AM
0 2 * * * /app/scripts/sync.sh >> /logs/cron.log 2>&1

# Hourly backup
0 * * * * /app/scripts/backup.sh >> /logs/backup.log 2>&1

# Weekly cost report
0 9 * * 1 /app/scripts/cost_report.sh | mail -s "Weekly Cost Report" ops@company.com

# Daily health check
30 8 * * * /app/scripts/health_check.sh >> /logs/health.log 2>&1
```

### Runbook

```markdown
## Operations Runbook

### Daily Tasks
- [ ] Check system health dashboard
- [ ] Review error logs for anomalies
- [ ] Verify backup completion
- [ ] Monitor cost trends

### Weekly Tasks
- [ ] Review cost report
- [ ] Check for API token expiration
- [ ] Update dependencies (security patches)
- [ ] Review slow query log

### Monthly Tasks
- [ ] Rotate API credentials
- [ ] Review and optimize expensive queries
- [ ] Capacity planning review
- [ ] Disaster recovery drill

### Incident Response
1. **High Error Rate**
   - Check agent logs: `docker-compose logs -f agent-api`
   - Verify API connectivity: `curl https://api.anthropic.com/v1/health`
   - Check rate limits: Review Prometheus metrics
   - Rollback if recent deployment: `./scripts/rollback.sh`

2. **High Latency**
   - Identify slow queries: Check P95 latency metric
   - Review retrieval performance: Check document count
   - Check for resource exhaustion: `docker stats`
   - Scale horizontally: `kubectl scale deployment ai-agent --replicas=5`

3. **Data Source Failures**
   - Check adapter health: `/api/health/adapters`
   - Verify credentials: Test authentication manually
   - Check network connectivity: `ping github.com`
   - Enable fallback mode: Set `FALLBACK_MODE=true`
```

---

## Troubleshooting

### Common Issues

**Issue 1: Slow Query Performance**

```bash
# Diagnose
docker-compose exec agent-api python -m src.debug.profile_query "test query"

# Check retrieval time
# Check LLM API latency
# Check network latency

# Solutions:
# 1. Reduce context size
# 2. Enable caching
# 3. Optimize embeddings
```

**Issue 2: Authentication Failures**

```bash
# Test API tokens
curl -H "Authorization: Bearer $ANTHROPIC_API_KEY" https://api.anthropic.com/v1/models

# Rotate tokens
./scripts/rotate_credentials.sh

# Update secrets
kubectl create secret generic ai-agent-secrets \
  --from-literal=anthropic-api-key=$NEW_KEY \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Issue 3: Out of Memory**

```bash
# Check memory usage
docker stats ai-agent-api

# Increase memory limit
docker-compose up -d --scale agent-api=1 --memory=8g

# Kubernetes
kubectl set resources deployment ai-agent --limits=memory=16Gi
```

---

## Disaster Recovery

### Recovery Plan

```markdown
## Disaster Recovery Plan

### RTO (Recovery Time Objective): 4 hours
### RPO (Recovery Point Objective): 24 hours

### Backup Locations
- Primary: /backups/ai-agent (daily)
- Secondary: S3 bucket (weekly)
- Tertiary: Offsite tape (monthly)

### Recovery Procedures

#### Scenario 1: Data Corruption
1. Stop service: `docker-compose down`
2. Restore from backup:
   ```bash
   tar -xzf /backups/documents_latest.tar.gz -C /data/
   cp /backups/sync_state_latest.json /data/state/sync_state.json
   ```
3. Restart service: `docker-compose up -d`
4. Verify: Run health check

#### Scenario 2: Complete System Failure
1. Provision new infrastructure (2 hours)
2. Deploy application from container registry (30 min)
3. Restore data from S3 (1 hour)
4. Run full sync (30 min)
5. Verify all integrations

#### Scenario 3: Data Center Outage
1. Failover to secondary region
2. Update DNS to point to backup
3. Restore from offsite backup
4. Resume operations in new region
```

---

*Last Updated: April 2026*
