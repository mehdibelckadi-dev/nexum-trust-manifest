# Nexum Trust Manifest

**A machine-readable risk profile for APIs consumed by AI agents.**

---

## What is a Trust Manifest?

A Trust Manifest is a structured JSON document that describes the security risk profile of an API — specifically in the context of AI agent integration. It answers three questions that every AI deployment team needs before wiring an agent to an external API:

1. **What can go wrong?** — A scored list of security findings (credential leakage, destructive ambiguity, unbounded scope, etc.)
2. **What is the blast radius?** — A risk tier (0–2) that quantifies how dangerous unrestricted agent access would be.
3. **What guardrails are needed?** — Auto-detected invariants (immutable fields, numeric limits, required headers) and a list of fields that require human review before the manifest can be considered complete.

Trust Manifests are generated automatically from MCP or OpenAPI specs by the [Nexum Scanner](https://github.com/your-org/nexum). They are designed to be:

- **Read by humans** — security engineers and AI platform teams reviewing integrations before deployment.
- **Checked by orchestration layers** — API gateways, agent supervisors, or CI pipelines that enforce a minimum trust tier.
- **Completed by hand** — the scanner produces a *draft*. Fields marked `REQUIRES_HUMAN_REVIEW` must be filled in by someone who understands the deployment context.

---

## Why it exists

AI agents that call APIs operate without the implicit caution a human developer applies when reading documentation. They will call `DELETE /users` if it is the most direct path to completing a task. They will pass credentials in a query string if the spec allows it. They will retry a non-idempotent payment endpoint.

Existing API security tooling (OWASP scanners, API gateways, WAFs) is built to protect APIs *from* external attackers. It is not designed to constrain *an authorized agent* that has valid credentials but no understanding of consequence.

The Trust Manifest fills that gap. It is not a replacement for authentication or authorization — it is a pre-deployment contract that makes the risk of an integration legible before an agent is allowed to call it autonomously.

---

## Manifest structure

```json
{
  "manifest_version": "1.0-draft",
  "scanned_at": "2026-05-10T11:54:19.617650+00:00",
  "source_file": "github-rest-api.json",
  "inferred_risk_tier": 2,
  "nexum_risk_score": 100,
  "model_compatibility_range": "REQUIRES_HUMAN_REVIEW",
  "auto_detected_invariants": {
    "immutable_fields": ["id", "created_at"],
    "numeric_limits": {
      "Repository.watchers_count": { "minimum": 0 }
    },
    "required_headers": ["Idempotency-Key"]
  },
  "findings_summary": [...],
  "manual_review_required": [...]
}
```

### Field reference

| Field | Type | Description |
|---|---|---|
| `manifest_version` | string | Schema version. Current: `"1.0-draft"`. |
| `scanned_at` | string (ISO 8601) | UTC timestamp of the scan. |
| `source_file` | string | Filename of the scanned spec. |
| `inferred_risk_tier` | integer (0–2) | Risk tier derived from the score. See tiers below. |
| `nexum_risk_score` | integer (0–100) | Aggregate risk score. See scoring below. |
| `model_compatibility_range` | string | Which model capability levels are safe for this integration. Always `REQUIRES_HUMAN_REVIEW` in drafts. |
| `auto_detected_invariants.immutable_fields` | string[] | Fields with `readOnly: true` in component schemas. Must not be modified by agents. |
| `auto_detected_invariants.numeric_limits` | object | Numeric and length constraints extracted from schemas. Agents must stay within these bounds. |
| `auto_detected_invariants.required_headers` | string[] | Required headers detected in operation parameters (e.g. `Idempotency-Key`). |
| `findings_summary` | object[] | One entry per finding. See finding fields below. |
| `manual_review_required` | object[] | Fields the scanner could not fill automatically, with the reason why. |

### Finding fields

| Field | Description |
|---|---|
| `rule_id` | The rule that fired (e.g. `NEXUM-001`). |
| `rule_name` | Human-readable rule name (e.g. `AuthLeakageRisk`). |
| `severity` | `CRITICAL` \| `HIGH` \| `MEDIUM` \| `LOW` |
| `path` | The API endpoint where the finding was detected. |
| `method` | HTTP method (`GET`, `POST`, `DELETE`, `PATCH`, …). |

---

## Risk scoring

Each finding contributes points to the aggregate score based on severity:

| Severity | Points |
|---|---|
| `CRITICAL` | 25 |
| `HIGH` | 10 |
| `MEDIUM` | 5 |
| `LOW` | 1 |

The score is capped at **100**. The risk tier is derived from the score:

| Tier | Score range | Label |
|---|---|---|
| 0 | 0 – 30 | Low Risk |
| 1 | 31 – 60 | Moderate Risk |
| 2 | 61 – 100 | High Risk |

**Tier 0** APIs can generally be used by agents with standard guardrails.
**Tier 1** APIs require additional constraints: scoped permissions, human approval for destructive operations, or operation-level rate limits.
**Tier 2** APIs should not be called autonomously without a human in the loop for every consequential action.

---

## Detection rules

The scanner applies five deterministic rules. No LLM is involved in any detection.

| Rule ID | Name | What it detects | Severity |
|---|---|---|---|
| `NEXUM-001` | `AuthLeakageRisk` | Credentials transmitted via query string parameters | `CRITICAL` |
| `NEXUM-002` | `DestructiveAmbiguity` | Destructive operations (`DELETE`) without a specific resource ID in the path | `CRITICAL` |
| `NEXUM-003` | `UnboundedScope` | Wildcard parameters or `DELETE`/`PATCH` operations without required filter parameters | `HIGH` |
| `NEXUM-004` | `IdempotencyMissing` | Mutating operations (`POST`, `PUT`, `PATCH`) without an `Idempotency-Key` header | `HIGH` |
| `NEXUM-005` | `SchemaVolatility` | `additionalProperties: true` in request body schemas of mutating operations | `MEDIUM` |

---

## How to generate a manifest

Install the [Nexum Scanner](https://github.com/your-org/nexum) and run:

```bash
nexum scan your-api-spec.json
```

This prints the Trust Manifest draft as JSON. To also generate a PDF report:

```bash
nexum report your-api-spec.json --output report.pdf
```

Or use the hosted scanner at [https://getnexum.dev](https://getnexum.dev)

---

## How to use a manifest

**Before wiring an agent to an API:**

1. Run the scanner and review the draft.
2. Fill in all fields marked `REQUIRES_HUMAN_REVIEW`.
3. Confirm or override the `inferred_risk_tier` based on your deployment context.
4. Store the completed manifest alongside the integration config.

**At deployment time:**

- Reject integrations with `inferred_risk_tier: 2` that do not have explicit human-approval workflows defined.
- Use `auto_detected_invariants.required_headers` to enforce headers at the gateway level.
- Use `auto_detected_invariants.immutable_fields` to build agent-side write guards.

**In CI:**

```bash
nexum scan api-spec.json | jq '.inferred_risk_tier' | grep -q '^[01]$' || exit 1
```

Fails the pipeline if the API is Tier 2 — enforcing a human review gate before deployment.

---

## Manifest status

| Version | Status |
|---|---|
| `1.0-draft` | Current. All machine-generated manifests are drafts until manually reviewed and signed off. |

The `model_compatibility_range` field is reserved for a future version that will specify which model capability tiers (e.g. tool-use, multi-step reasoning) are safe for each integration. It cannot be reliably inferred from an OpenAPI/MCP spec and requires human judgment.

---

## Examples

- [`examples/tier0-example.json`](examples/tier0-example.json) — Read-only API, no findings, Tier 0.
- [`examples/tier1-example.json`](examples/tier1-example.json) — API with controlled mutations, Tier 1.
- [`examples/tier2-example.json`](examples/tier2-example.json) — Destructive API (based on a real GitHub REST API scan), Tier 2.

---

## License

MIT
