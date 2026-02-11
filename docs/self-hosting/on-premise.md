# On-Premise Deployment

Deploy DocVerify in air-gapped or restricted network environments using pre-packaged Docker images.

## Initial Setup

```bash
# 1. Unpack the distribution
tar xf docverify-onprem-v1.0.0.tar
cd docverify-onprem/

# 2. Configure environment
cp .env.template .env
# Edit .env: set SECRET_KEY, POSTGRES_PASSWORD, etc.

# 3. Load Docker images
bash install.sh

# 4. Start the stack
docker compose -f docker-compose.onprem.yml up -d

# 5. Run database migrations
docker exec docverify-api alembic upgrade head

# 6. Verify
curl http://localhost:8000/health
```

## GPU Setup

For GPU inference in on-premise environments:

1. Install [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
2. Set `DEVICE=cuda` in `.env`
3. Add GPU resource reservations to the api service in `docker-compose.onprem.yml`:

```yaml
services:
  api:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | local postgres | PostgreSQL connection string |
| `REDIS_URL` | local redis | Redis connection string |
| `SECRET_KEY` | **must change** | Secret for key hashing |
| `DEVICE` | `cpu` | `cpu` or `cuda` |
| `MODEL_PATH` | `./model_weights` | Path to model files |
| `DB_POOL_SIZE` | — | Database connection pool size |
| `DB_MAX_OVERFLOW` | — | Max overflow connections |
| `REDIS_MAX_CONNECTIONS` | — | Redis connection pool size |
| `BACKUP_DIR` | `/backups` | Backup storage directory |
| `RETENTION_DAYS` | 30 | Days to keep backups |

!!! warning
    You **must** change `SECRET_KEY` from the default before creating any API keys. Changing it later invalidates all existing API keys.

## Updating

```bash
# 1. Pull new images or load new tar
docker load < docverify-api-v1.1.0.tar

# 2. Run migrations
docker exec docverify-api alembic upgrade head

# 3. Restart
docker compose -f docker-compose.onprem.yml restart api
```

## Network Requirements

DocVerify does not require outbound internet access after initial setup. All ML models are bundled in the Docker image.

Inbound access needed:

| Port | Service | Required |
|------|---------|----------|
| 8000 | API | Yes |
| 3000 | Grafana | Optional |
| 9090 | Prometheus | Optional |

## Data Residency

All data stays within your infrastructure:

- **Database** — PostgreSQL stores API keys, verification history, and audit logs
- **Cache** — Redis stores rate limits and response cache (ephemeral)
- **Models** — ML model weights stored on disk at `MODEL_PATH`
- **Backups** — stored at `BACKUP_DIR` on the host

No telemetry or usage data is sent externally.

## Scaling

See the [Operations Runbook](operations.md#scaling) for horizontal and vertical scaling guidance.
