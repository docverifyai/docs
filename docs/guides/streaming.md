# Streaming Verification

Get verification results in real time using Server-Sent Events (SSE). Each claim is streamed as it's verified, so you can show progress to users instead of waiting for the full response.

## How It Works

1. Send a verification request to `/v1/verify/stream`
2. Receive `claim` events as each claim is verified
3. Receive a `done` event with the final summary

## Usage

=== "curl"

    ```bash
    curl -N -X POST https://api.docverify.dev/v1/verify/stream \
      -H "Authorization: Bearer dv_your_api_key" \
      -H "Content-Type: application/json" \
      -d '{
        "source": "The agreement has a 24-month term with 60 days termination notice.",
        "ai_output": "The contract lasts 36 months and requires 90 days notice to terminate.",
        "domain": "legal"
      }'
    ```

    Output:

    ```
    event: claim
    data: {"claim": "The contract lasts 36 months", "verdict": "CONTRADICTED", "confidence": 0.94, "severity": "critical", "evidence": {"source_text": "24-month term", "reasoning": "Source says 24 months, not 36"}}

    event: claim
    data: {"claim": "Requires 90 days notice to terminate", "verdict": "CONTRADICTED", "confidence": 0.91, "severity": "critical", "evidence": {"source_text": "60 days termination notice", "reasoning": "Source says 60 days, not 90"}}

    event: done
    data: {"verification_id": "ver_abc123", "overall_score": 0.12, "verdict": "MAJOR_ISSUES", "claims_count": 2}
    ```

=== "Python"

    ```python
    import httpx

    with httpx.stream(
        "POST",
        "https://api.docverify.dev/v1/verify/stream",
        headers={"Authorization": "Bearer dv_your_api_key"},
        json={
            "source": "The agreement has a 24-month term with 60 days termination notice.",
            "ai_output": "The contract lasts 36 months and requires 90 days notice.",
            "domain": "legal",
        },
    ) as response:
        for line in response.iter_lines():
            if line.startswith("data: "):
                print(line[6:])
    ```

=== "JavaScript"

    ```typescript
    const response = await fetch("https://api.docverify.dev/v1/verify/stream", {
      method: "POST",
      headers: {
        "Authorization": "Bearer dv_your_api_key",
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        source: "The agreement has a 24-month term with 60 days termination notice.",
        ai_output: "The contract lasts 36 months and requires 90 days notice.",
        domain: "legal",
      }),
    });

    const reader = response.body!.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const text = decoder.decode(value);
      for (const line of text.split("\n")) {
        if (line.startsWith("data: ")) {
          const data = JSON.parse(line.slice(6));
          console.log(data);
        }
      }
    }
    ```

## Event Types

| Event | Description |
|-------|-------------|
| `claim` | A single claim verification result |
| `done` | Final summary with overall score and verdict |
| `error` | An error occurred during processing |

### `claim` event payload

```json
{
  "claim": "The contract lasts 36 months",
  "verdict": "CONTRADICTED",
  "confidence": 0.94,
  "severity": "critical",
  "evidence": {
    "source_text": "24-month term",
    "reasoning": "Source says 24 months, not 36"
  }
}
```

### `done` event payload

```json
{
  "verification_id": "ver_abc123",
  "overall_score": 0.12,
  "verdict": "MAJOR_ISSUES",
  "claims_count": 2
}
```

### `error` event payload

```json
{
  "error": "Pipeline timeout",
  "error_id": "err_xyz789"
}
```

## When to Use Streaming

- **User-facing UIs** — show claim results as they appear instead of a loading spinner
- **Long documents** — get partial results while the full document is still processing
- **Progress tracking** — count claims as they arrive to show a progress bar

For non-interactive use cases (batch processing, CI/CD), the standard `/v1/verify` endpoint is simpler.
