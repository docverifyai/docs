# File Upload Verification

Verify AI-generated text against PDF, DOCX, or TXT source files using multipart upload.

## Supported Formats

| Format | Extension | Parser |
|--------|-----------|--------|
| PDF | `.pdf` | PyMuPDF |
| Word | `.docx` | python-docx |
| Plain text | `.txt` | built-in |

## Upload and Verify

=== "curl"

    ```bash
    curl -X POST https://api.docverify.dev/v1/verify/file \
      -H "Authorization: Bearer dv_your_api_key" \
      -F "file=@contract.pdf" \
      -F "ai_output=The contract requires 90 days notice and has a 36-month term." \
      -F "domain=legal"
    ```

=== "Python"

    ```python
    from docverify import DocVerifyClient

    client = DocVerifyClient(api_key="dv_your_api_key")

    result = client.verify_file(
        file_path="contract.pdf",
        ai_output="The contract requires 90 days notice and has a 36-month term.",
        domain="legal",
    )

    print(f"Verdict: {result.verdict}")
    for claim in result.claims:
        print(f"  {claim.verdict}: {claim.claim}")
        if claim.evidence:
            print(f"    Page {claim.evidence.source_location['page']}: {claim.evidence.source_text}")
    ```

=== "JavaScript"

    ```typescript
    import { DocVerifyClient } from "docverify";
    import fs from "fs";

    const client = new DocVerifyClient({ apiKey: "dv_your_api_key" });

    const fileBuffer = fs.readFileSync("contract.pdf");
    const result = await client.verifyFile(
      fileBuffer,
      "contract.pdf",
      "The contract requires 90 days notice and has a 36-month term.",
      { domain: "legal" },
    );

    console.log(result.verdict);
    ```

## Response

The response is identical to `POST /v1/verify`, with `source_location` in evidence including page and paragraph references from the original file.

```json
{
  "verification_id": "ver_abc123",
  "overall_score": 0.12,
  "verdict": "MAJOR_ISSUES",
  "claims": [
    {
      "claim": "The contract requires 90 days notice",
      "verdict": "CONTRADICTED",
      "confidence": 0.94,
      "severity": "critical",
      "evidence": {
        "source_text": "60-day termination notice period",
        "source_location": {"page": 3, "paragraph": 2},
        "reasoning": "Source says 60 days, not 90"
      }
    }
  ]
}
```

## Domain Adapters

Specify a `domain` to use a specialized LoRA adapter for improved accuracy:

| Domain | Best for |
|--------|----------|
| `general` | Default — works for any content |
| `legal` | Contracts, agreements, legal filings |
| `financial` | Financial reports, earnings, filings |
| `medical` | Clinical notes, medical records, research |

```bash
curl -X POST https://api.docverify.dev/v1/verify/file \
  -H "Authorization: Bearer dv_your_api_key" \
  -F "file=@patient_record.pdf" \
  -F "ai_output=Patient is 45 years old with type 1 diabetes." \
  -F "domain=medical"
```

## File Size Limits

- Maximum file size: 50 MB
- Maximum pages (PDF): 500
- Processing timeout applies for very large documents

For large documents, consider breaking them into sections and verifying individually, or use the [batch endpoint](batch-verify.md) to verify multiple outputs against the same source.
