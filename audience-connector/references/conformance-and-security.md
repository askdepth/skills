# Conformance Suite, Security Guardrails & Onboarding

This reference details the 15-case conformance test suite, error code specification, special-category data protections, and the 11-step Askdepth platform onboarding flow.

---

## 1. Running the Conformance CLI

The `@askdepth/audience-connector` package ships with a built-in CLI (`audience-connector`) that runs all 15 conformance cases against any local or remote connector endpoint.

```bash
# Run full 15-case suite
npx @askdepth/audience-connector conformance \
  --url http://localhost:3000/askdepth/v1 \
  --secret "$ASKDEPTH_CONNECTOR_SECRET" \
  --unmapped-column password \
  --unmapped-column internal_notes \
  --filter-only-attribute country=US

# Run in CI with JSON output
npx @askdepth/audience-connector conformance \
  --url https://demo-shop.askdepth.com/askdepth/v1 \
  --secret "$ASKDEPTH_CONNECTOR_SECRET" \
  --unmapped-column password \
  --filter-only-attribute country=US \
  --json

# Re-run a single case (e.g. statistical sampling check N7)
npx @askdepth/audience-connector conformance \
  --url http://localhost:3000/askdepth/v1 \
  --secret "$ASKDEPTH_CONNECTOR_SECRET" \
  --case N7
```

### CLI Exit Codes
- `0` — All executed cases passed.
- `1` — Conformance failure (at least one positive or negative case failed).
- `2` — Runner or preflight error (missing arguments, unreachable URL, or invalid configuration).

---

## 2. The 15 Conformance Cases

Passing all 15 cases is a mandatory gate before a connector can be activated in Askdepth.

### Positive Cases (`P1–P7`)
| Case | ID | What It Verifies |
|---|---|---|
| **P1** | `health` | `GET /health` returns HTTP 200, `ok: true`, `contractVersion: "1.0.0"`, and valid `capabilities` array. |
| **P2** | `schema` | `GET /schema` returns HTTP 200 with non-empty `columns: Array<{ name, type }>`. |
| **P3** | `count` | `POST /candidates/count` returns HTTP 200 with a non-negative integer `count`. |
| **P4** | `search` | `POST /candidates/search` respects `limit`, returns valid `CandidateRow` items, and paginates via `nextCursor`. |
| **P5** | `externalIdIn` | Filtering with `{ field: 'externalId', op: 'in', values: [...] }` returns only matching candidate IDs. |
| **P6** | `attributeFilters` | Filtering by `attr.<name>` restricts results properly, while **filter-only** attributes (`--filter-only-attribute`) never appear in returned row `attributes`. |
| **P7** | `suppressExternalIds` | Candidate IDs passed in `suppressExternalIds` are excluded from both `/candidates/count` and `/candidates/search`. |

### Negative Security Cases (`N1–N8`)
| Case | ID | What It Verifies |
|---|---|---|
| **N1** | `unsigned-accepted` | Requests missing `x-askdepth-signature` or `x-askdepth-timestamp` are rejected with HTTP 401 `unauthorized`. |
| **N2** | `bad-signature-accepted` | Requests with corrupted signatures (`401 invalid_signature`) or timestamps outside ±300s (`401 expired_timestamp`) are rejected. |
| **N3** | `unmapped-columns-leaked` | Database columns passed via `--unmapped-column` (e.g., `password`, `internal_notes`) never appear in search results or cannot be queried. |
| **N4** | `error-leaks-data` | Error responses use strictly fixed wire messages and never leak SQL errors, connection strings, hostnames, bearer tokens, or row data. |
| **N5** | `cursor-nondeterministic` | Replaying the same `nextCursor` produces deterministic pages without duplicate or skipped candidate rows, and rejects cursors replayed across different queries (`400 invalid_cursor`). |
| **N6** | `row-cap-exceeded` | Requests with `limit > 1000` are rejected with HTTP 400 `limit_exceeded`, and no search response ever exceeds 1,000 rows. |
| **N7** | `sample-not-random` | `sample: { method: 'random', size }` produces an unbiased distribution across the dataset (mean rank percentile ~0.5) rather than returning a monotonic slice (such as oldest signups first). *Tip: Because live sampling is stochastic, allow 1 retry on `--case N7` in CI.* |
| **N8** | `write-path-exposed` | HTTP write verbs (`PUT`, `PATCH`, `DELETE`) return HTTP 405 `method_not_allowed`, unknown routes return 404 `not_found`, and SQL injection probes (`'; DROP TABLE ...`) in filter values are safely parameterized without altering database state. |

---

## 3. Closed Wire Error Code Table (`CONNECTOR_ERROR_CODES`)

All error responses match `{ error: { code: ConnectorErrorCode, message: string } }` with fixed messages so internal stack traces or DB errors never leak:

| Error Code | HTTP Status | When Emitted |
|---|---|---|
| `unauthorized` | `401` | Missing `x-askdepth-signature` or `x-askdepth-timestamp` header |
| `invalid_signature` | `401` | HMAC-SHA256 digest mismatch or malformed signature header |
| `expired_timestamp` | `401` | Timestamp skew exceeds `±300` seconds (`REPLAY_WINDOW_SECONDS`) |
| `malformed_request` | `400` | Invalid JSON body, schema validation failure, or unmapped field/attribute |
| `unsupported_capability` | `400` | Query uses a capability flag not advertised by the connector/adapter |
| `limit_exceeded` | `400` | Requested `limit > 1000` (`ROW_CAP`) |
| `invalid_cursor` | `400` | Malformed cursor or cursor `queryHash` does not match current query |
| `not_found` | `404` | Path does not match `<basePath>/health`, `/schema`, `/candidates/count`, or `/candidates/search` |
| `method_not_allowed` | `405` | Non-`GET` on `/health` or `/schema`, or non-`POST` on `/candidates/*` |
| `adapter_error` | `502` | Database/upstream failure or malformed row aborted pull |
| `timeout` | `504` | Query exceeded budget (`5000ms` interactive / `30000ms` background) |
| `internal` | `500` | Unexpected unhandled exception (details scrubbed) |

---

## 4. Special-Category Data Refusal (GDPR Article 9 Guardrail)

At initialization time, `createConnector` validates all column names in `fieldMapping` and `attributes` against the `SPECIAL_CATEGORY` regex:

```regexp
/(health|medical|diagnos|\bnhs\b|religio|\bfaith\b|political|ethnic|\brace\b|racial|sexual.?orientation|gender.?identity|biometr|genetic|trade.?union|disabilit)/i
```

If any mapped column or attribute name matches, `createConnector` throws immediately and refuses to boot. Never map medical, biometric, religious, political, or racial data into an Audience Connector.

---

## 5. The 11-Step Askdepth Platform Onboarding Wizard

When connecting your deployed endpoint in the Askdepth UI (`/workspace/[slug]/settings/integrations/connectors/[connectorId]/[step]`), the platform walks through 11 verification gates:

1. **`subjects` (Data Subjects)**: Confirm target audience (`B2B`, `B2C`, or `Employees`). Selecting `Employees` pauses automated onboarding for manual compliance approval.
2. **`connection` (Endpoint Probe)**: Platform sends a signed `GET /health` and `GET /schema` to verify connectivity and discover columns.
3. **`mapping` (Field Mapping)**: Map canonical fields (`externalId`, `email`, `name`, `signupAt`, `segment`, `isActive`, `contactable`) and up to 10 filterable/returnable attributes.
4. **`lawful-basis` (GDPR Basis)**: Declare lawful basis (`consent`, `legitimate_interests`, or `contract`).
5. **`age` (18+ Confirmation)**: Confirm all queryable candidates are adults (18+).
6. **`analytics` (Telemetry Identity Resolution)**: Link `externalId` to your Askdepth Analytics project (`Askdepth.identify(externalId)`).
7. **`dpa` (Data Processing Agreement)**: Organization-level acceptance of the Askdepth DPA.
8. **`sending-domain` (Email Outreach DNS)**: Verify SPF/DKIM DNS records for participant outreach emails.
9. **`conformance` (Automated 15-Case Run)**: Askdepth executes the 15-case conformance runner (`P1–P7`, `N1–N8`) against your live URL.
10. **`limits` (Guardrails & Cooldowns)**: Configure maximum invitation volume, respondent cooldown periods, and data retention windows.
11. **`review` (Activation)**: Activate the connector (only one connector may be in `ACTIVE` state per organization at any time).
