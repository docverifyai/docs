# Enterprise Features

DocVerify provides enterprise-grade features for teams that need access control, compliance, and organizational management.

## Teams

Organize API keys and usage under teams. Each team has its own usage limits, billing, and audit trail.

```python
# Create a team
team = client.create_team(name="Legal Department")

# Add members
client.add_team_member(team_id=team["id"], email="alice@company.com", role="admin")
client.add_team_member(team_id=team["id"], email="bob@company.com", role="member")

# List teams
teams = client.list_teams()
```

```typescript
const team = await client.createTeam({ name: "Legal Department" });
await client.addTeamMember(team.id, { email: "alice@company.com", role: "admin" });
```

## Verification Policies

Define rules that are applied to every verification within a team or namespace. Policies enforce minimum thresholds and can block results that don't meet quality standards.

```python
client.create_policy(
    name="Legal minimum",
    rules={
        "min_confidence": 0.8,
        "fail_on_contradicted": True,
        "required_domain": "legal",
        "max_unsupported_ratio": 0.2,
    },
)
```

## Alerts

Get notified when verifications fail or match certain conditions.

```python
client.create_alert(
    name="High contradiction alert",
    condition={"verdict": "MAJOR_ISSUES", "min_contradicted": 3},
    channels=["email", "slack"],
)
```

## RBAC (Role-Based Access Control)

| Role | Permissions |
|------|-------------|
| `owner` | Full access, billing, team deletion |
| `admin` | Manage members, keys, policies, alerts |
| `member` | Verify, view history, use dashboard |
| `viewer` | Read-only access to verification results |

Scopes can also be set per API key:

```python
client.update_scopes(key_id="key_123", scopes=["verify", "verify:file", "history:read"])
```

## IP Allowlist

Restrict API access to specific IP addresses or CIDR ranges.

```python
client.set_ip_allowlist(["203.0.113.0/24", "198.51.100.42"])

# Check current allowlist
ips = client.get_ip_allowlist()

# Clear (allow all)
client.clear_ip_allowlist()
```

## Audit Logging

Every API action is recorded in the audit log with the authenticated key, timestamp, and request details.

```python
logs = client.get_audit_logs(
    start_date="2026-01-01",
    end_date="2026-02-01",
    action="verify",
)
for entry in logs:
    print(f"{entry['timestamp']} {entry['action']} by {entry['key_id']}")
```

## Compliance

### GDPR

Export or delete all data associated with an API key or email:

```python
# Export all user data
export = client.gdpr_export(email="user@example.com")

# Delete all user data
client.gdpr_forget(email="user@example.com")
```

### Compliance Status

```python
status = client.get_compliance_status()
# {"gdpr": "compliant", "data_retention": "30 days", "encryption": "at-rest and in-transit"}

report = client.get_compliance_report(format="pdf")
```

## Namespaces

Isolate verifications into separate namespaces for multi-tenant applications or environment separation.

```python
# Create namespace
client.create_namespace(name="production")
client.create_namespace(name="staging")

# Verify within a namespace
result = client.verify(
    source="...",
    ai_output="...",
    namespace="production",
)
```

## Analytics

```python
analytics = client.get_analytics(
    start_date="2026-01-01",
    end_date="2026-02-01",
    group_by="day",
)
# Returns: daily verification counts, average scores, verdict breakdown

forecast = client.get_usage_forecast(days=30)
# Returns: predicted usage for the next 30 days
```

## Webhooks

Subscribe to events and get notified when verifications complete or fail.

```python
client.subscribe_webhook(
    url="https://your-app.com/webhooks/docverify",
    events=["verification.completed", "verification.failed"],
    secret="whsec_your_secret",
)
```

Webhook payloads include a signature header (`X-DocVerify-Signature`) for verification. See the [API Reference](../api-reference.md) for the full list of webhook events.
