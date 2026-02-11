# DocVerify

**Catch AI hallucinations before your customers do.**

DocVerify is a verification API that compares AI-generated outputs against source documents and returns a structured accuracy report — claim by claim, with evidence citations and confidence scores.

When a legal tech company uses AI to summarize a contract, DocVerify catches "60 days" turning into "90 days" before clients see it.

## How It Works

```
Source Document ──> Document Parser ──> Evidence Index
                                              |
AI Output ─────> Claim Extractor ──> NLI Engine ──> Scorer ──> Verdict
```

1. **Parse** the source document (PDF, DOCX, or text) into searchable chunks
2. **Extract** atomic claims from the AI-generated output
3. **Verify** each claim against the source using natural language inference
4. **Score** the overall accuracy and return a structured report

## Quick Example

=== "Python"

    ```python
    from docverify import DocVerifyClient

    client = DocVerifyClient(api_key="dv_your_key")

    result = client.verify(
        source="The agreement has a 24-month term with 60 days termination notice.",
        ai_output="The contract lasts 36 months and requires 90 days notice.",
        domain="legal",
    )

    print(result.verdict)  # "MAJOR_ISSUES"
    for claim in result.claims:
        print(f"  {claim.verdict}: {claim.claim}")
    ```

=== "JavaScript"

    ```typescript
    import { DocVerifyClient } from "docverify";

    const client = new DocVerifyClient({ apiKey: "dv_your_key" });
    const result = await client.verify({
      source: "The agreement has a 24-month term with 60 days termination notice.",
      ai_output: "The contract lasts 36 months and requires 90 days notice.",
      domain: "legal",
    });
    console.log(result.verdict); // "MAJOR_ISSUES"
    ```

=== "curl"

    ```bash
    curl -X POST https://api.docverify.dev/v1/verify \
      -H "Authorization: Bearer dv_your_key" \
      -H "Content-Type: application/json" \
      -d '{
        "source": "The agreement has a 24-month term with 60 days termination notice.",
        "ai_output": "The contract lasts 36 months and requires 90 days notice.",
        "domain": "legal"
      }'
    ```

## Features

- **Multi-format input** — PDF, DOCX, and plain text via file upload or text
- **Domain adapters** — specialized models for legal, medical, financial documents
- **Batch verification** — verify multiple AI outputs against one source
- **Comparison mode** — cross-compare outputs from different models
- **Streaming** — real-time results via Server-Sent Events
- **Enterprise** — teams, policies, audit logging, RBAC, IP allowlists, GDPR compliance
- **Integrations** — LangChain, LlamaIndex, Slack, Zapier, webhooks
- **SDKs** — [Python](sdks/python.md) and [JavaScript/TypeScript](sdks/javascript.md) with full type coverage
- **Dashboard** — browser-based verification history and API playground
- **On-premise** — [deploy in your own infrastructure](self-hosting/docker.md) with Docker

## Next Steps

<div class="grid cards" markdown>

- :material-rocket-launch: **[Quickstart](quickstart.md)** — verify your first document in 5 minutes
- :material-api: **[API Reference](api-reference.md)** — every endpoint, parameter, and response
- :material-language-python: **[Python SDK](sdks/python.md)** — pip install docverify
- :material-language-javascript: **[JavaScript SDK](sdks/javascript.md)** — npm install docverify
- :material-server: **[Self-Hosting](self-hosting/docker.md)** — run DocVerify on your own infrastructure

</div>
