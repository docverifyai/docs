# Operations Runbook

Day-to-day operations guide for self-hosted DocVerify deployments.

## Deployment

### Standard Deploy

```bash
# Build and deploy (includes Docker build + migration)
make deploy

# Skip Docker build if only config changes
make deploy-skip-build

# Manual production stack management
make prod-up       # start all services
make prod-down     # stop all services
make prod-logs     # tail all logs
make prod-status   # show running services
```

### Rollback

1. Check the current running images: `docker compose -f docker/docker-compose.prod.yml ps`
2. Identify the previous image tag from your CI/CD history
3. Update the image tag in `docker-compose.prod.yml` or `.env`
4. Restart: `make prod-down && make prod-up`
5. Verify: `curl http://localhost:8000/health`

### Post-Deploy Checklist

After every deployment, verify:

- [ ] `GET /health` returns `{"status": "ok"}`
- [ ] `GET /v1/diagnostics` shows database: ok, pipeline: ready
- [ ] Run a test verification: `POST /v1/verify` with sample data
- [ ] Check Redis connectivity: diagnostics shows redis: ok
- [ ] Check Prometheus: `GET /metrics` returns metric data
- [ ] Check Grafana: dashboards load at `http://localhost:3000`
- [ ] Check logs for errors: `make prod-logs | grep ERROR`

## Database

### Migrations

```bash
# Apply all pending migrations
make db-migrate

# Create a new migration after model changes
make db-revision msg="description of change"

# Check current migration state
.venv/bin/alembic current
.venv/bin/alembic history
```

### Backups

Automated daily backups run at 3 AM UTC via the `backup` Docker service.

```bash
# Manual backup
bash scripts/backup_db.sh

# Check backup status via diagnostics
curl -H "Authorization: Bearer $API_KEY" \
  http://localhost:8000/v1/diagnostics | jq .checks.backup
```

**Configuration:**

- `BACKUP_DIR` — backup storage directory (default: `/backups`)
- `RETENTION_DAYS` — days to keep (default: 30)
- Backups are compressed with gzip (`docverify_YYYYMMDD_HHMMSS.sql.gz`)

### Restore

```bash
# 1. Stop the API to prevent new writes
make prod-down

# 2. Start only the database
docker compose -f docker/docker-compose.prod.yml up -d db

# 3. Drop and recreate the database
docker exec docverify-db-1 psql -U postgres -c "DROP DATABASE IF EXISTS docverify;"
docker exec docverify-db-1 psql -U postgres -c "CREATE DATABASE docverify;"

# 4. Restore from backup
gunzip -c /backups/docverify_20260211_030000.sql.gz | \
  docker exec -i docverify-db-1 pg_restore -U postgres -d docverify

# 5. Verify restoration
docker exec docverify-db-1 psql -U postgres -d docverify \
  -c "SELECT 1;  -- verify database responds"

# 6. Start all services
make prod-up
```

## Monitoring

### Prometheus

- URL: `http://localhost:9090` (internal only via Caddy)
- Scrapes `/metrics` on the API every 15 seconds
- Alerting rules in `monitoring/prometheus/alerts.yml`

**Alerts configured:**

| Alert | Condition | Severity |
|-------|-----------|----------|
| HighErrorRate | sustained high error rate | critical |
| HighLatency | sustained high latency | warning |
| TargetDown | API unreachable | critical |
| DiskSpaceHigh | high disk usage | warning |
| DbPoolExhausted | database connection failures | critical |

### Alertmanager

- URL: `http://localhost:9093`
- Email notifications configured via env vars: `ALERT_EMAIL_TO`, `SMTP_HOST`, `SMTP_FROM`
- PagerDuty for critical alerts, Slack for warnings

### Grafana

- URL: `http://localhost:3000`
- Default credentials: admin / `$GRAFANA_ADMIN_PASSWORD`
- Pre-provisioned dashboards: "DocVerify Overview" and "Business Metrics"
- Prometheus datasource auto-configured

### Structured Logs

API logs are JSON-formatted. Key fields:

- `event` — action name (e.g., `verify_request`, `auth_success`)
- `level` — info, warning, error
- `api_key_id` — authenticated key ID
- `processing_time_ms` — request duration
- `request_id` — X-Request-ID for tracing

```bash
# Filter for errors
make prod-logs | grep '"level": "error"'

# Filter for slow requests (>5s)
make prod-logs | jq 'select(.processing_time_ms > 5000)'
```

## Troubleshooting

### Common Issues

**"Connection pool exhausted" errors:**

- Increase `DB_POOL_SIZE` and `DB_MAX_OVERFLOW` in `.env`
- Check `/v1/diagnostics` for pool stats
- Look for long-running queries holding connections

**"Redis connection refused":**

- Check Redis container: `docker compose ps redis`
- Verify `REDIS_URL` in `.env`
- Rate limiting and caching will fail open (requests still served)

**"Pipeline not loaded" in diagnostics:**

- Model download may have failed — check logs for model loading errors
- Verify `MODEL_PATH` directory exists and contains model weights
- Try: `make download-models`

**Slow verification (>10s):**

- Check GPU availability: `GET /v1/diagnostics` shows gpu section
- CPU inference is ~10x slower than GPU
- Consider enabling response caching and semantic caching
- For batch operations, claims are batched to amortize overhead

**High memory usage:**

- NLI model requires ~2GB RAM
- Each LoRA adapter adds ~2MB
- Custom adapters are cached in memory
- Evidence embeddings are per-request (not cached across requests)

### Log Locations

| Component | Location |
|-----------|----------|
| API | `make prod-logs` or `docker logs docverify-api-1` |
| PostgreSQL | `docker logs docverify-db-1` |
| Redis | `docker logs docverify-redis-1` |
| Celery | `docker logs docverify-celery-worker-1` |
| Backup | `docker logs docverify-backup-1` |

### Restart Procedures

```bash
# Restart single service
docker compose -f docker/docker-compose.prod.yml restart api

# Full restart (preserves data volumes)
make prod-down && make prod-up
```

!!! danger
    `docker compose down -v` removes volumes and causes **permanent data loss**. Only use this for dev/test environments.

## Scaling

### Horizontal Scaling

- **API workers:** Run multiple `api` containers behind a load balancer. Each worker loads the NLI model independently (~2GB RAM each).
- **Celery workers:** Add more Celery worker containers for async adapter training.
- **Redis:** Single instance is sufficient for most workloads. Consider Redis Sentinel for HA.

### Vertical Scaling

- **GPU:** Fastest path to lower latency. T4 -> A10 -> A100.
- **RAM:** 8GB minimum (4GB API + model, 2GB PostgreSQL, 1GB Redis).
- **CPU:** More cores help with concurrent request handling.

### NLI Inference Bottleneck

The NLI model is the primary bottleneck:

| Hardware | Latency |
|----------|---------|
| CPU | ~2-5s per verification |
| NVIDIA T4 | ~200-500ms |
| NVIDIA A10 | ~100-200ms |

Strategies to reduce NLI load:

1. **Response caching** — identical inputs skip NLI entirely
2. **Semantic caching** — similar inputs return cached results
3. **Batch verify** — builds evidence index once for multiple outputs
4. **ONNX backend** — faster CPU inference

## Key Rotation

### Rotating the secret key

!!! warning
    Changing `SECRET_KEY` invalidates **all** existing API keys. Users will need to regenerate their keys.

1. Generate new salt: `make generate-salt`
2. Announce maintenance window to users
3. Update `SECRET_KEY` in `.env`
4. Restart API: `docker compose restart api`
5. All existing keys now fail authentication
6. Users must create new keys via `POST /v1/keys`

### Per-Key Rotation

Individual keys can be rotated without affecting others:

- `POST /v1/keys/rotate` — generates a new key, revokes the old one after 24h grace period
- Users should update their integrations during the grace period
