# JavaScript SDK

Official JavaScript/TypeScript SDK for the DocVerify document verification API.

[![npm](https://img.shields.io/npm/v/docverify)](https://www.npmjs.com/package/docverify)

## Installation

```bash
npm install docverify
```

## Quick Start

```typescript
import { DocVerifyClient } from "docverify";

const client = new DocVerifyClient({
  apiKey: "dv_your_api_key",
});

const result = await client.verify({
  source: "The contract term is 24 months.",
  ai_output: "The contract term is 36 months.",
  domain: "legal",
});

console.log(result.verdict);  // "MAJOR_ISSUES"
for (const claim of result.claims) {
  console.log(`  ${claim.verdict}: ${claim.claim}`);
}
```

## Configuration

```typescript
const client = new DocVerifyClient({
  apiKey: "dv_your_key",       // required — must start with "dv_"
  baseUrl: "https://api.docverify.dev",  // default
  timeout: 30000,              // default: 30000ms
});
```

## Core Methods

### Verify

```typescript
const result = await client.verify({
  source: "source document text",
  ai_output: "AI-generated text to verify",
  domain: "legal",           // optional: legal, medical, financial, general
  task_type: "summarization", // optional: extraction, summarization
});
```

### Verify File (Multipart Upload)

```typescript
const fileBuffer = fs.readFileSync("contract.pdf");
const result = await client.verifyFile(fileBuffer, "contract.pdf", "Check this summary", {
  domain: "legal",
});
```

### Batch Verify

```typescript
const result = await client.batchVerify({
  source: "source document text",
  ai_outputs: ["summary A", "summary B", "summary C"],
});
// result.results — array of VerificationResponse
```

### Compare

```typescript
const result = await client.compare({
  source: "source document text",
  ai_outputs: ["model A output", "model B output"],
});
// result.comparison — cross-output claim agreement analysis
// result.summary — agreement rate and totals
```

## Response Objects

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | `string` | Unique ID for the verification |
| `overall_score` | `number` | 0-1 accuracy score |
| `verdict` | `string` | `ACCURATE`, `MINOR_ISSUES`, `MAJOR_ISSUES`, `UNRELIABLE` |
| `claims` | `ClaimVerification[]` | Per-claim verdicts with evidence |
| `metadata` | `VerificationMetadata` | Model version, timing, token counts |

## Error Handling

```typescript
import {
  DocVerifyError,
  AuthenticationError,
  RateLimitError,
  ValidationError,
} from "docverify";

try {
  await client.verify({ source: "s", ai_output: "o" });
} catch (error) {
  if (error instanceof AuthenticationError) {
    // 401/403 — invalid or missing API key
  } else if (error instanceof RateLimitError) {
    // 429 — rate limit exceeded
  } else if (error instanceof ValidationError) {
    // 422 — invalid request payload
  } else if (error instanceof DocVerifyError) {
    // Other API errors (statusCode available)
    console.error(error.statusCode, error.message);
  }
}
```

## Enterprise Features

The SDK provides full coverage of DocVerify enterprise features:

- **Teams**: `createTeam()`, `listTeams()`, `addTeamMember()`, `removeTeamMember()`
- **Policies**: `createPolicy()`, `listPolicies()`, `updatePolicy()`, `deletePolicy()`
- **Alerts**: `createAlert()`, `listAlerts()`, `deleteAlert()`, `getAlertHistory()`
- **Webhooks**: `subscribeWebhook()`, `listWebhooks()`, `deleteWebhook()`
- **Audit**: `getAuditLogs()`, `getReport()`
- **Analytics**: `getAnalytics()`, `getUsageForecast()`, `getUsageTrends()`
- **Templates**: `listTemplates()`, `createTemplate()`, `verifyWithTemplate()`
- **Namespaces**: `listNamespaces()`, `createNamespace()`, `deleteNamespace()`
- **Key Management**: `rotateKey()`, `getScopes()`, `updateScopes()`
- **IP Allowlist**: `getIpAllowlist()`, `setIpAllowlist()`, `clearIpAllowlist()`
- **Compliance**: `getComplianceStatus()`, `getComplianceReport()`, `gdprExport()`, `gdprForget()`

## TypeScript

Full TypeScript support with exported types for all request/response objects. Type declarations are bundled in the package.

## Source

[github.com/docverifyai/js-sdk](https://github.com/docverifyai/js-sdk)
