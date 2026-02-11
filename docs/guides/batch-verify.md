# Batch & Compare

Verify multiple AI outputs against a single source, or compare outputs from different models side by side.

## Batch Verification

Send multiple AI-generated texts to verify against one source document. The source is parsed once and reused for all outputs, making batch requests more efficient than individual calls.

=== "Python"

    ```python
    from docverify import DocVerifyClient

    client = DocVerifyClient(api_key="dv_your_api_key")

    result = client.batch_verify(
        source="The patient is 45 years old with type 2 diabetes diagnosed in 2019.",
        ai_outputs=[
            {"id": "model_a", "text": "A 45-year-old patient with type 2 diabetes."},
            {"id": "model_b", "text": "A 55-year-old patient with type 1 diabetes."},
        ],
        domain="medical",
    )

    for r in result.results:
        print(f"  {r.verdict}: score={r.overall_score}")
    ```

=== "JavaScript"

    ```typescript
    import { DocVerifyClient } from "docverify";

    const client = new DocVerifyClient({ apiKey: "dv_your_api_key" });

    const result = await client.batchVerify({
      source: "The patient is 45 years old with type 2 diabetes diagnosed in 2019.",
      ai_outputs: ["A 45-year-old with type 2 diabetes.", "A 55-year-old with type 1 diabetes."],
    });

    for (const r of result.results) {
      console.log(`${r.verdict}: score=${r.overall_score}`);
    }
    ```

=== "curl"

    ```bash
    curl -X POST https://api.docverify.dev/v1/verify/batch \
      -H "Authorization: Bearer dv_your_api_key" \
      -H "Content-Type: application/json" \
      -d '{
        "source": "The patient is 45 years old with type 2 diabetes diagnosed in 2019.",
        "ai_outputs": [
          "A 45-year-old patient with type 2 diabetes.",
          "A 55-year-old patient with type 1 diabetes."
        ],
        "domain": "medical"
      }'
    ```

### Batch Response

```json
{
  "results": [
    {
      "verification_id": "ver_1",
      "overall_score": 0.95,
      "verdict": "ACCURATE",
      "claims": [...]
    },
    {
      "verification_id": "ver_2",
      "overall_score": 0.12,
      "verdict": "MAJOR_ISSUES",
      "claims": [...]
    }
  ]
}
```

### Batch Limits

| Tier | Max outputs per batch |
|------|----------------------|
| Free | 5 |
| Pro | 50 |
| Enterprise | 200 |

## Compare Mode

Compare outputs from different AI models to see where they agree and disagree on specific claims.

=== "Python"

    ```python
    comparison = client.compare(
        source="The merger was valued at $4.2 billion and approved by the board on March 15.",
        ai_outputs=[
            {"id": "gpt4", "text": "The $4.2B merger was board-approved on March 15."},
            {"id": "claude", "text": "The $4.5B merger received board approval in March."},
        ],
        domain="financial",
    )

    print(f"Agreement rate: {comparison['summary']['agreement_rate']}")
    for item in comparison["comparison"]["claim_agreement"]:
        print(f"  {item['claim']}: gpt4={item['gpt4']}, claude={item['claude']}")
    ```

=== "JavaScript"

    ```typescript
    const comparison = await client.compare({
      source: "The merger was valued at $4.2 billion and approved by the board on March 15.",
      ai_outputs: [
        { id: "gpt4", text: "The $4.2B merger was board-approved on March 15." },
        { id: "claude", text: "The $4.5B merger received board approval in March." },
      ],
      domain: "financial",
    });

    console.log(`Agreement rate: ${comparison.summary.agreement_rate}`);
    ```

=== "curl"

    ```bash
    curl -X POST https://api.docverify.dev/v1/verify/compare \
      -H "Authorization: Bearer dv_your_api_key" \
      -H "Content-Type: application/json" \
      -d '{
        "source": "The merger was valued at $4.2 billion and approved by the board on March 15.",
        "ai_outputs": [
          {"id": "gpt4", "text": "The $4.2B merger was board-approved on March 15."},
          {"id": "claude", "text": "The $4.5B merger received board approval in March."}
        ],
        "domain": "financial"
      }'
    ```

### Compare Response

```json
{
  "results": [
    {"verification_id": "ver_1", "overall_score": 0.95, "verdict": "ACCURATE", "claims": [...]},
    {"verification_id": "ver_2", "overall_score": 0.65, "verdict": "MINOR_ISSUES", "claims": [...]}
  ],
  "comparison": {
    "claim_agreement": [
      {"claim": "merger valued at $4.2B", "gpt4": "SUPPORTED", "claude": "CONTRADICTED"},
      {"claim": "approved on March 15", "gpt4": "SUPPORTED", "claude": "PARTIAL"}
    ]
  },
  "summary": {
    "agreed": 0,
    "disagreed": 2,
    "agreement_rate": 0.0
  }
}
```

## Use Cases

- **Model evaluation** — compare accuracy across different LLMs before choosing a provider
- **Regression testing** — verify a new model version hasn't introduced hallucinations
- **A/B testing** — measure accuracy improvements between prompt versions
- **Quality assurance** — batch-verify all AI-generated content before publishing
