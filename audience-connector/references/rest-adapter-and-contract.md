# REST Adapter & Wire Contract Reference

This reference covers building custom data source integrations using `restAdapter`, understanding the `@askdepth/audience-contract` wire specification, criteria DSL, keyset pagination cursors, and HMAC-SHA256 request signing.

---

## 1. Custom Data Sources with `restAdapter`

Use `restAdapter` when candidate records live behind an internal microservice, GraphQL/REST API, CRM, or non-PostgreSQL datastore.

### Signature & Configuration

```typescript
import { createConnector, restAdapter } from '@askdepth/audience-connector';

const connector = createConnector({
  secret: process.env.ASKDEPTH_CONNECTOR_SECRET!,
  fieldMapping: {
    externalId: 'id',
    email: 'email',
    segment: 'tier',
    signupAt: 'created_at',
    isActive: 'active',
  },
  attributes: {
    filterable: ['country', 'plan'],
    returnable: ['plan'],
  },
  adapter: restAdapter({
    // 1. Static column schema returned on GET /schema
    declaredSchema: {
      columns: [
        { name: 'id', type: 'string' },
        { name: 'email', type: 'string' },
        { name: 'tier', type: 'string' },
        { name: 'created_at', type: 'timestamp' },
        { name: 'active', type: 'boolean' },
        { name: 'country', type: 'string' },
        { name: 'plan', type: 'string' },
      ],
    },
    // 2. Declare which optional capabilities your backend honors
    // Defaults to ['declaredSchema']. Available flags:
    // 'externalIdIn' | 'attributeFilters' | 'dateRanges' | 'randomSample' | 'declaredSchema'
    capabilities: [
      'declaredSchema',
      'externalIdIn',
      'attributeFilters',
      'dateRanges',
      'randomSample',
    ],
    // 3. Candidate search implementation
    fetchCandidates: async (query, { signal }) => {
      const response = await fetch('https://internal-api.example.com/candidates/query', {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify(query),
        signal, // AbortSignal tied to interactive (5s) or background (30s) budget
      });
      if (!response.ok) {
        throw new Error(`Internal API error: ${response.status}`);
      }
      const data = await response.json();
      return {
        rows: data.items,           // Array of objects using your source column names
        nextCursor: data.nextToken, // Optional backend page token (wrapped by connector)
      };
    },
    // 4. Candidate count implementation (strongly recommended)
    fetchCount: async (query, { signal }) => {
      const response = await fetch('https://internal-api.example.com/candidates/count', {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify(query),
        signal,
      });
      const data = await response.json();
      return data.total;
    },
  }),
});
```

### `restAdapter` Safety Invariants
Even if your `fetchCandidates` callback returns extra fields or too many rows, `restAdapter` enforces:
1. **Strict Column Stripping**: Any property on returned row objects not present in `plan.select` (mapped canonical fields + `attributes.returnable`) is stripped before leaving the adapter.
2. **Hard Row Cap (`ROW_CAP = 1000`)**: Truncates any page exceeding 1,000 rows.
3. **Abort Over Partial (`CanonicalFieldSchema` Validation)**: Every row is validated against `CanonicalFieldSchema` (e.g., `externalId` must be a non-empty string, `email` must be a valid email). A single malformed row aborts the entire request with `adapter_error` (HTTP 502) rather than returning partial data.
4. **Derived Count Guard**: If you omit `fetchCount`, `restAdapter` falls back to calling `fetchCandidates` and counting returned rows. However, if that fallback hits `ROW_CAP` (1,000) or returns a `nextCursor`, `restAdapter` throws `adapter_error` rather than returning a silently truncated count. Always provide `fetchCount` for production datasets > 1,000 users.
5. **Opaque Cursor Wrapping**: Your backend's `nextCursor` string is wrapped inside `{ v: 1, after: { k: nextCursor }, qh: queryHash, seed }` so tampering or cross-query cursor replay is rejected with 400 `invalid_cursor`.

---

## 2. Wire Contract Specification (`@askdepth/audience-contract`)

### Constants
- `CONTRACT_VERSION = '1.0.0'` (wire protocol version, independent of npm package version)
- `ROW_CAP = 1_000` (maximum rows per `/candidates/search` response)
- `DEFAULT_BASE_PATH = '/askdepth/v1'`
- `ENDPOINTS = { health: '/health', schema: '/schema', count: '/candidates/count', search: '/candidates/search' }`
- `SIGNATURE_HEADER = 'x-askdepth-signature'`
- `TIMESTAMP_HEADER = 'x-askdepth-timestamp'`
- `REPLAY_WINDOW_SECONDS = 300` (±5 minutes clock skew tolerance)

### Canonical Row Schema (`CanonicalFieldSchema`)
```typescript
interface CandidateRow {
  externalId: string;                   // Required non-empty string
  email: string;                        // Required valid email address
  name?: string;                        // Optional display name
  segment?: string;                     // Optional lifecycle segment
  signupAt?: string;                    // Optional ISO 8601 UTC timestamp
  isActive?: boolean;                   // Optional activity flag
  contactable?: boolean;                // Optional opt-in flag
  attributes?: Record<string, unknown>; // Only attributes listed in attributes.returnable
}
```

### Criteria DSL (`QuerySchema`)
Queries are strictly **AND-only** (`criteria.all`), with no `OR`, `NOT`, or nested boolean groups:

```typescript
interface Query {
  all: Array<
    | { field: 'segment'; op: 'in'; values: [string, ...string[]] }
    | { field: 'signupAt'; op: 'between'; from: string; to: string } // ISO 8601
    | { field: 'isActive'; op: 'eq'; value: boolean }
    | { field: 'externalId'; op: 'in'; values: [string, ...string[]] }
    | { field: `attr.${string}`; op: 'eq'; value: string | number | boolean }
    | { field: `attr.${string}`; op: 'in'; values: Array<string | number | boolean> }
  >;
  suppressExternalIds?: string[]; // Candidate IDs to exclude (e.g. recently interviewed)
  sample?: {
    method: 'random';
    size: number;                 // Positive integer
  };
}
```

> **Note on `sample` Precedence**: In `POST /candidates/search`, `sample` may appear at the top level of the request body (`{ criteria, sample }`) or inside `criteria` (`{ criteria: { sample } }`). The connector evaluates `request.sample ?? request.criteria.sample`.

---

## 3. HMAC-SHA256 Signing & Secret Rotation

### How Signing Works
Every request from Askdepth includes two headers:
- `x-askdepth-timestamp`: Unix epoch timestamp in seconds (e.g., `1763208000`).
- `x-askdepth-signature`: `'v1=' + hmacSha256Hex(secret, `${timestamp}.${signedBody}`)`.

> [!WARNING]
> **GET / HEAD Empty-Body Rule**: For `GET` and `HEAD` requests (`/health`, `/schema`), `signedBodyFor(method, rawBody)` always returns `""` (empty string). The signed payload is `${timestamp}.` (note the trailing dot). Signing a non-empty body on a `GET` request will fail with 401 `invalid_signature`.

### Generating Signed Requests (For Custom Tests or Platform Clients)

```typescript
import {
  sign,
  signedBodyFor,
  SIGNATURE_HEADER,
  TIMESTAMP_HEADER,
} from '@askdepth/audience-contract';

export function buildSignedRequestHeaders(
  method: string,
  rawBody: string,
  secretHexOrString: string,
): Record<string, string> {
  const timestamp = Math.floor(Date.now() / 1000);
  const bodyToSign = signedBodyFor(method, rawBody);
  const signature = sign(bodyToSign, timestamp, Buffer.from(secretHexOrString, 'utf8'));

  return {
    [SIGNATURE_HEADER]: signature, // "v1=<64-hex-chars>"
    [TIMESTAMP_HEADER]: String(timestamp),
    ...(method !== 'GET' && method !== 'HEAD'
      ? { 'content-type': 'application/json' }
      : {}),
  };
}
```

### Zero-Downtime Secret Rotation (`previousSecret`)
When rotating `ASKDEPTH_CONNECTOR_SECRET`:
1. Move the current secret into `previousSecret` (`ASKDEPTH_CONNECTOR_PREVIOUS_SECRET`) and set the newly issued secret as `secret` (`ASKDEPTH_CONNECTOR_SECRET`).
2. `verifyRequest` checks `secret` first and transparently falls back to `previousSecret` if signature verification against `secret` returns `mismatch`.
3. Once the new secret is active in Askdepth, remove `previousSecret`.
