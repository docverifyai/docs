# Docker Deployment

Run DocVerify on your own infrastructure using Docker.

## Quick Start

```bash
git clone https://github.com/docverifyai/docverify && cd docverify

# Start infrastructure (PostgreSQL + Redis)
make docker-up

# Install and configure
make dev
cp .env.example .env  # Edit with your settings
make db-migrate

# Download ML models (~2GB first run)
make download-models

# Run
make run
# API: http://localhost:8000
# Docs: http://localhost:8000/docs
# Dashboard: http://localhost:8000/dashboard
```

## Production Deployment

Use the production Docker Compose file for a complete stack with Caddy reverse proxy, Prometheus, Grafana, and automated backups.

```bash
# Configure production environment
cp .env.example .env
```

Edit `.env` with production values:

```bash
# Required — change these
SECRET_KEY=your-random-secret-here     # run: make generate-secret
POSTGRES_PASSWORD=strong-password
GRAFANA_ADMIN_PASSWORD=strong-password

# ML model
DEVICE=cpu                             # or cuda for GPU
MODEL_PATH=./model_weights

# Database
DATABASE_URL=postgresql://docverify:CHANGE_ME@db:5432/docverify
DB_POOL_SIZE=
DB_MAX_OVERFLOW=

# Redis
REDIS_URL=redis://redis:6379/0
REDIS_MAX_CONNECTIONS=

# Optional
SENTRY_DSN=                            # Error tracking
ALERT_EMAIL_TO=ops@yourcompany.com     # Alert notifications
SMTP_HOST=smtp.yourcompany.com
SMTP_FROM=docverify@yourcompany.com
```

Start the production stack:

```bash
make prod-up
```

This starts:

| Service | Port | Description |
|---------|------|-------------|
| API | 8000 | DocVerify API |
| Caddy | 80, 443 | Reverse proxy with auto-TLS |
| PostgreSQL | 5432 | Database |
| Redis | 6379 | Cache and rate limiting |
| Prometheus | 9090 | Metrics |
| Grafana | 3000 | Dashboards |
| Alertmanager | 9093 | Alert routing |
| Celery Worker | — | Async task processing |
| Backup | — | Daily DB backups at 3 AM UTC |

## GPU Support

For GPU inference (recommended for production):

1. Install [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
2. Set `DEVICE=cuda` in `.env`
3. The production compose file includes GPU resource reservations

Performance comparison:

| Hardware | Latency per verification |
|----------|--------------------------|
| CPU | ~2-5 seconds |
| NVIDIA T4 | ~200-500ms |
| NVIDIA A10 | ~100-200ms |

## Health Checks

```bash
# Basic health
curl http://localhost:8000/health
# {"status": "ok"}

# Detailed diagnostics (requires auth)
curl -H "Authorization: Bearer $API_KEY" http://localhost:8000/v1/diagnostics
# {"database": "ok", "redis": "ok", "pipeline": "ready", "gpu": "available"}
```

## Updating

```bash
# Pull latest code
git pull

# Rebuild and restart
make prod-down
make deploy

# Verify
curl http://localhost:8000/health
```

## Architecture

```
                    ┌──────────┐
    Internet ──────>│  Caddy   │──────> :8000 DocVerify API
                    │  :80/443 │
                    └──────────┘
                         │
            ┌────────────┼────────────┐
            │            │            │
       ┌────┴────┐ ┌─────┴────┐ ┌────┴─────┐
       │ Postgres │ │  Redis   │ │  Celery  │
       │  :5432   │ │  :6379   │ │  Worker  │
       └─────────┘ └──────────┘ └──────────┘
```

For on-premise deployments with air-gapped networks, see [On-Premise](on-premise.md). For day-to-day operations, see the [Operations Runbook](operations.md).
