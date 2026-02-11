# API Reference

Base URL: `https://api.docverify.dev`

All endpoints require authentication via Bearer token in the `Authorization` header.

!!! tip
    Interactive API docs are available at `/docs` (Swagger UI) and `/redoc` on any running DocVerify instance. Export the OpenAPI spec with `GET /v1/openapi.json` or `GET /v1/openapi.yaml`.

## Authentication

```bash
curl -H "Authorization: Bearer dv_your_api_key" https://api.docverify.dev/v1/verify
```

API keys start with `dv_` and are created via `POST /v1/keys`.

## Endpoints

### Verification

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/verify` | Verify AI output against source text |
| `POST` | `/v1/verify/file` | Verify AI output against an uploaded file |
| `POST` | `/v1/verify/stream` | Verify with real-time SSE streaming |
| `POST` | `/v1/verify/batch` | Verify multiple AI outputs against one source |
| `POST` | `/v1/verify/compare` | Compare outputs from different models |
| `GET` | `/v1/verifications` | List past verifications |
| `GET` | `/v1/verifications/{id}` | Get a specific verification result |

### API Keys

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/keys` | Create a new API key |
| `GET` | `/v1/keys` | List your API keys |
| `DELETE` | `/v1/keys/{id}` | Revoke an API key |
| `POST` | `/v1/keys/rotate` | Rotate a key (24h grace period) |
| `GET` | `/v1/keys/{id}/scopes` | Get key scopes |
| `PUT` | `/v1/keys/{id}/scopes` | Update key scopes |

### Templates

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/templates` | List verification templates |
| `POST` | `/v1/templates` | Create a custom template |
| `POST` | `/v1/templates/{id}/verify` | Verify using a template |
| `DELETE` | `/v1/templates/{id}` | Delete a template |

### Webhooks

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/webhooks/subscribe` | Subscribe to events |
| `GET` | `/v1/webhooks` | List webhook subscriptions |
| `DELETE` | `/v1/webhooks/{id}` | Delete a subscription |

### Enterprise

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/teams` | Create a team |
| `GET` | `/v1/teams` | List teams |
| `POST` | `/v1/teams/{id}/members` | Add a team member |
| `DELETE` | `/v1/teams/{id}/members/{user}` | Remove a team member |
| `POST` | `/v1/policies` | Create a verification policy |
| `GET` | `/v1/policies` | List policies |
| `PUT` | `/v1/policies/{id}` | Update a policy |
| `DELETE` | `/v1/policies/{id}` | Delete a policy |
| `GET` | `/v1/audit-log` | Get audit log entries |
| `GET` | `/v1/analytics` | Get usage analytics |
| `GET` | `/v1/compliance/status` | Get compliance status |
| `GET` | `/v1/compliance/report` | Generate compliance report |
| `POST` | `/v1/gdpr/export` | Export user data (GDPR) |
| `POST` | `/v1/gdpr/forget` | Delete user data (GDPR) |

### System

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Health check |
| `GET` | `/v1/diagnostics` | Detailed system diagnostics |
| `GET` | `/metrics` | Prometheus metrics |

---

## POST /v1/verify

Verify AI-generated text against a source document.

### Request

```json
{
  "source": "The agreement terminates after 60 days written notice...",
  "source_type": "text",
  "ai_output": "The termination clause requires 90 days notice.",
  "task_type": "extraction",
  "domain": "legal",
  "options": {
    "granularity": "claim",
    "return_evidence": true,
    "severity_threshold": "low"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `source` | string | yes | Source document text |
| `ai_output` | string | yes | AI-generated text to verify |
| `source_type` | string | no | `text` (default), `pdf`, `docx` |
| `task_type` | string | no | `extraction`, `summary`, `analysis` |
| `domain` | string | no | `general` (default), `legal`, `financial`, `medical` |
| `options.granularity` | string | no | `claim` (default) or `sentence` |
| `options.return_evidence` | bool | no | Include source evidence (default: `true`) |
| `options.severity_threshold` | string | no | `low` (default), `medium`, `high` |

### Response

```json
{
  "verification_id": "ver_abc123",
  "overall_score": 0.42,
  "verdict": "MAJOR_ISSUES",
  "claims": [
    {
      "claim": "The termination clause requires 90 days notice",
      "verdict": "CONTRADICTED",
      "confidence": 0.94,
      "severity": "critical",
      "evidence": {
        "source_text": "terminates after 60 days written notice",
        "source_location": {"page": 3, "paragraph": 2},
        "reasoning": "Source specifies 60 days, not 90 days"
      }
    }
  ],
  "metadata": {
    "model_version": "docverify-nli-v1.0",
    "processing_time_ms": 340,
    "source_tokens": 4200,
    "claims_extracted": 2
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Unique ID for this verification |
| `overall_score` | float | 0.0 (all wrong) to 1.0 (all correct) |
| `verdict` | string | `ACCURATE`, `MINOR_ISSUES`, `MAJOR_ISSUES`, `UNRELIABLE` |
| `claims` | array | Per-claim verification results |
| `claims[].claim` | string | The extracted claim text |
| `claims[].verdict` | string | `SUPPORTED`, `CONTRADICTED`, `UNSUPPORTED`, `PARTIAL`, `AMBIGUOUS` |
| `claims[].confidence` | float | Model confidence (0-1) |
| `claims[].severity` | string | `critical`, `high`, `medium`, `low` |
| `claims[].evidence` | object | Source evidence (if `return_evidence` is true) |
| `metadata` | object | Processing metadata |

---

## POST /v1/verify/file

Verify AI output against an uploaded file (PDF, DOCX, TXT).

### Request

Multipart form data:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | file | yes | Source document file |
| `ai_output` | string | yes | AI-generated text to verify |
| `domain` | string | no | Domain adapter to use |

```bash
curl -X POST https://api.docverify.dev/v1/verify/file \
  -H "Authorization: Bearer dv_your_api_key" \
  -F "file=@contract.pdf" \
  -F "ai_output=The contract requires 90 days notice." \
  -F "domain=legal"
```

### Response

Same as `POST /v1/verify`.

---

## POST /v1/verify/stream

Verify with real-time Server-Sent Events. Returns claim results as they are processed.

### Request

Same body as `POST /v1/verify`.

### Response

`Content-Type: text/event-stream`

```
event: claim
data: {"claim": "The contract lasts 36 months", "verdict": "CONTRADICTED", "confidence": 0.94}

event: claim
data: {"claim": "Requires 90 days notice", "verdict": "CONTRADICTED", "confidence": 0.91}

event: done
data: {"verification_id": "ver_abc123", "overall_score": 0.12, "verdict": "MAJOR_ISSUES"}
```

See the [Streaming guide](guides/streaming.md) for details.

---

## POST /v1/verify/batch

Verify multiple AI outputs against one source document.

### Request

```json
{
  "source": "Source document text...",
  "ai_outputs": [
    "First AI output to verify",
    "Second AI output to verify"
  ],
  "domain": "legal"
}
```

### Response

```json
{
  "results": [
    {"verification_id": "ver_1", "overall_score": 0.95, "verdict": "ACCURATE", "claims": [...]},
    {"verification_id": "ver_2", "overall_score": 0.12, "verdict": "MAJOR_ISSUES", "claims": [...]}
  ]
}
```

See the [Batch & Compare guide](guides/batch-verify.md) for details.

---

## POST /v1/verify/compare

Compare outputs from different AI models against one source.

### Request

```json
{
  "source": "Source document text...",
  "ai_outputs": [
    {"id": "model_a", "text": "Output from model A"},
    {"id": "model_b", "text": "Output from model B"}
  ],
  "domain": "legal"
}
```

### Response

```json
{
  "results": [...],
  "comparison": {
    "claim_agreement": [
      {"claim": "contract lasts 24 months", "model_a": "SUPPORTED", "model_b": "CONTRADICTED"}
    ]
  },
  "summary": {
    "agreed": 5,
    "disagreed": 1,
    "agreement_rate": 0.83
  }
}
```

---

## Error Responses

All errors follow a consistent format:

```json
{
  "detail": "Human-readable error message",
  "error_id": "err_..."
}
```

| Status | Meaning |
|--------|---------|
| `400` | Bad request — invalid parameters |
| `401` | Unauthorized — missing or invalid API key |
| `403` | Forbidden — insufficient permissions |
| `404` | Not found |
| `409` | Conflict — duplicate resource |
| `422` | Validation error — invalid request body |
| `429` | Rate limit exceeded |
| `500` | Internal server error |
| `503` | Service unavailable |

The `error_id` can be used for log correlation when contacting support.

## Rate Limits

| Tier | Requests/day | Requests/minute |
|------|-------------|-----------------|
| Free | 100 | 10 |
| Pro | 10,000 | 100 |
| Enterprise | Unlimited | 1,000 |

Rate limit headers are included in every response:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 97
X-RateLimit-Reset: 1707667200
```
