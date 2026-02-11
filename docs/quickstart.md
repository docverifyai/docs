# Quickstart

Get started verifying AI outputs in under 5 minutes.

## 1. Get an API Key

```bash
curl -X POST https://api.docverify.dev/v1/keys \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "name": "My App"}'
```

Response:
```json
{
  "api_key": "dv_abc123...",
  "key_id": "key_...",
  "tier": "free",
  "daily_limit": 100
}
```

!!! warning
    Save your `api_key` — it won't be shown again.

## 2. Verify Your First Document

=== "Python"

    ```bash
    pip install docverify
    ```

    ```python
    from docverify import DocVerifyClient

    client = DocVerifyClient(api_key="dv_your_api_key")

    result = client.verify(
        source="The agreement has a 24-month term with a 60-day termination notice period.",
        ai_output="The contract lasts 36 months and can be terminated with 90 days notice.",
        domain="legal",
    )

    print(f"Verdict: {result.verdict}")       # MAJOR_ISSUES
    print(f"Score: {result.overall_score}")    # 0.12
    for claim in result.claims:
        print(f"  {claim.verdict}: {claim.claim}")
        if claim.evidence:
            print(f"    Evidence: {claim.evidence.source_text}")
    ```

=== "JavaScript"

    ```bash
    npm install docverify
    ```

    ```typescript
    import { DocVerifyClient } from "docverify";

    const client = new DocVerifyClient({
      apiKey: "dv_your_api_key",
    });

    const result = await client.verify({
      source: "The agreement has a 24-month term with a 60-day termination notice period.",
      ai_output: "The contract lasts 36 months and can be terminated with 90 days notice.",
      domain: "legal",
    });

    console.log(`Verdict: ${result.verdict}`);
    for (const claim of result.claims) {
      console.log(`  ${claim.verdict}: ${claim.claim}`);
    }
    ```

=== "curl"

    ```bash
    curl -X POST https://api.docverify.dev/v1/verify \
      -H "Authorization: Bearer dv_your_api_key" \
      -H "Content-Type: application/json" \
      -d '{
        "source": "The agreement has a 24-month term with a 60-day termination notice period.",
        "ai_output": "The contract lasts 36 months and can be terminated with 90 days notice.",
        "domain": "legal"
      }'
    ```

=== "CLI"

    ```bash
    pip install docverify
    export DOCVERIFY_API_KEY=dv_your_api_key

    docverify verify \
      --source "The agreement has a 24-month term." \
      --ai-output "The contract lasts 36 months."
    ```

## 3. Understanding the Response

```json
{
  "verification_id": "ver_abc123",
  "overall_score": 0.12,
  "verdict": "MAJOR_ISSUES",
  "claims": [
    {
      "claim": "The contract lasts 36 months",
      "verdict": "CONTRADICTED",
      "confidence": 0.94,
      "severity": "critical",
      "evidence": {
        "source_text": "24-month term",
        "source_location": {"page": 1, "paragraph": 1},
        "reasoning": "Source says 24 months, not 36"
      }
    },
    {
      "claim": "Can be terminated with 90 days notice",
      "verdict": "CONTRADICTED",
      "confidence": 0.91,
      "severity": "critical",
      "evidence": {
        "source_text": "60-day termination notice period",
        "source_location": {"page": 1, "paragraph": 1},
        "reasoning": "Source says 60 days, not 90"
      }
    }
  ],
  "metadata": {
    "model_version": "docverify-nli-v1.0",
    "processing_time_ms": 340,
    "claims_extracted": 2
  }
}
```

### Claim Verdicts

| Verdict | Meaning |
|---------|---------|
| `SUPPORTED` | Claim matches the source |
| `CONTRADICTED` | Claim conflicts with the source |
| `UNSUPPORTED` | No evidence found in source |
| `PARTIAL` | Partially supported |
| `AMBIGUOUS` | Evidence is unclear |

### Overall Verdicts

| Verdict | Meaning |
|---------|---------|
| `ACCURATE` | All claims supported |
| `MINOR_ISSUES` | Some partial or unsupported claims |
| `MAJOR_ISSUES` | Contradictions found |
| `UNRELIABLE` | Mostly contradicted or unsupported |

## 4. Next Steps

- **[File Upload](guides/file-upload.md)** — verify PDFs, DOCX, and TXT files directly
- **[Batch & Compare](guides/batch-verify.md)** — verify multiple outputs or compare models
- **[Streaming](guides/streaming.md)** — get real-time results via SSE
- **[API Reference](api-reference.md)** — all endpoints, parameters, and responses
- **[Python SDK](sdks/python.md)** — full SDK documentation
- **[JavaScript SDK](sdks/javascript.md)** — full SDK documentation
