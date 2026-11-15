# PostgreSQL Adapter & Framework Integrations

This reference provides complete, production-tested integration templates for `@askdepth/audience-connector` with PostgreSQL (`postgresAdapter`) across Next.js App Router, Express, Fastify, AWS Lambda, and Cloudflare Workers.

---

## 1. PostgreSQL Adapter (`postgresAdapter`)

### Options & Signature

```typescript
import { Pool } from 'pg';
import { postgresAdapter } from '@askdepth/audience-connector';

const adapter = postgresAdapter({
  pool: new Pool({ connectionString: process.env.ASKDEPTH_CONNECTOR_DB_URL }),
  table: 'users',               // Target table name
  schema: 'public',             // Optional: defaults to 'public'
  externalIdColumn: 'id',       // Optional fallback column for externalId
  maxStatementTimeoutMs: 60000, // Optional ceiling on statement_timeout (default: 60,000 ms)
});
```

### Built-in PostgreSQL Security Invariants
- **Identifier Validation**: `schema`, `table`, and all mapped column names must match `/^[A-Za-z_][A-Za-z0-9_$]*$/` and are safely double-quoted in generated SQL.
- **Strict Parameterization**: All filter values are bound via `$1, $2, ...` parameters (`= ANY($n)` for arrays).
- **Read-Only Transaction Isolation**: Every query runs inside `BEGIN READ ONLY` and commits or rolls back cleanly.
- **Statement Timeouts**: Automatically executes `SELECT set_config('statement_timeout', $1, true)` inside each transaction (~100ms below the handler timeout budget: `5000ms` interactive / `30000ms` background). PostgreSQL error `57014` (`query_canceled`) translates cleanly to HTTP 504 `timeout`.
- **Deterministic Seeded Keyset Pagination**: Evaluates `(md5($seed || "external_id"::text), "external_id"::text) > ($h, $id)` ordered by `ORDER BY md5($seed || "external_id"::text) ASC, "external_id"::text ASC LIMIT $limit + 1`.

---

## 2. Next.js App Router Integration (Production Pattern)

This pattern mirrors the production Askdepth `demo-shop` setup using Next.js App Router, Prisma/PostgreSQL, and `proxy.ts` / `middleware.ts`.

### Step 1: Connector Singleton (`lib/askdepth-connector.ts`)

```typescript
import { Pool } from 'pg';
import { createConnector, postgresAdapter } from '@askdepth/audience-connector';

export const FIELD_MAPPING = {
  externalId: 'id',
  email: 'email',
  name: 'name',
  signupAt: 'createdAt',
  segment: 'segment',
  isActive: 'is_active',
  contactable: 'marketing_opt_in',
} as const;

export const ATTRIBUTES = {
  filterable: [
    'loyalty_tier',
    'preferred_category',
    'device_type',
    'has_abandoned_cart',
    'country', // Filter-only: present in filterable, omitted from returnable
  ],
  returnable: [
    'loyalty_tier',
    'preferred_category',
    'device_type',
    'has_abandoned_cart',
  ],
} as const;

declare global {
  // Prevent multiple pg Pools during Next.js HMR in development
  var __askdepthPgPool: Pool | undefined;
}

function getPgPool(): Pool {
  if (!globalThis.__askdepthPgPool) {
    globalThis.__askdepthPgPool = new Pool({
      connectionString: process.env.ASKDEPTH_CONNECTOR_DB_URL || process.env.DATABASE_URL,
      max: 5,
    });
  }
  return globalThis.__askdepthPgPool;
}

let connectorInstance: ReturnType<typeof createConnector> | null = null;

export function getAudienceConnector() {
  if (!connectorInstance) {
    connectorInstance = createConnector({
      secret: process.env.ASKDEPTH_CONNECTOR_SECRET!,
      previousSecret: process.env.ASKDEPTH_CONNECTOR_PREVIOUS_SECRET,
      adapter: postgresAdapter({
        pool: getPgPool(),
        table: 'users',
        schema: 'public',
      }),
      fieldMapping: { ...FIELD_MAPPING },
      attributes: {
        filterable: [...ATTRIBUTES.filterable],
        returnable: [...ATTRIBUTES.returnable],
      },
      basePath: '/askdepth/v1',
    });
  }
  return connectorInstance;
}
```

### Step 2: Catch-All Route Handler (`app/askdepth/v1/[[...route]]/route.ts`)

Pass all HTTP verbs (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`) directly to `connector.fetch(request)` so negative conformance case **N8** (`write-path-exposed`) receives the proper 405 `method_not_allowed` wire response from the connector router:

```typescript
import { getAudienceConnector } from '@/lib/askdepth-connector';

export const runtime = 'nodejs';
export const dynamic = 'force-dynamic';

async function handle(request: Request): Promise<Response> {
  return getAudienceConnector().fetch(request);
}

export const GET = handle;
export const POST = handle;
export const PUT = handle;
export const PATCH = handle;
export const DELETE = handle;
export const OPTIONS = handle;
```

### Step 3: Bypass in `proxy.ts` / `middleware.ts`

Ensure Next.js middleware/proxy passes `/askdepth/v1` requests straight through without mutating headers, cookies, or body streams:

```typescript
import { NextResponse, type NextRequest } from 'next/server';

export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Pass Askdepth Audience Connector HMAC-authenticated wire calls straight through
  if (pathname.startsWith('/askdepth/v1')) {
    return NextResponse.next();
  }

  // ... rest of your application middleware
  return NextResponse.next();
}
```

### Step 4: Prisma Schema Mapping Example (`prisma/schema.prisma`)

When using Prisma to manage the `users` table queried by `postgresAdapter`, note that `postgresAdapter` queries actual **PostgreSQL column names** (the names in `@map(...)` if present):

```prisma
model User {
  id                String   @id @default(cuid())
  email             String   @unique
  name              String?
  password          String?  // Unmapped sensitive column (excluded from connector)
  segment           String   @default("new_customer")
  isActive          Boolean  @default(true) @map("is_active")
  marketingOptIn    Boolean  @default(true) @map("marketing_opt_in")
  loyaltyTier       String   @default("standard") @map("loyalty_tier")
  preferredCategory String   @default("electronics") @map("preferred_category")
  deviceType        String   @default("desktop") @map("device_type")
  hasAbandonedCart  Boolean  @default(false) @map("has_abandoned_cart")
  country           String   @default("US")
  internalNotes     String?  @map("internal_notes") // Unmapped sensitive column
  createdAt         DateTime @default(now()) @db.Timestamptz(6)
  updatedAt         DateTime @updatedAt @db.Timestamptz(6)

  @@index([segment])
  @@index([createdAt])
  @@map("users")
}
```

### Step 5: `next.config.mjs` External Packages

Keep `pg` external on the server and transpile `@askdepth/*` packages if needed:

```javascript
const nextConfig = {
  serverExternalPackages: ['@prisma/client', 'pg'],
  transpilePackages: [
    '@askdepth/audience-connector',
    '@askdepth/audience-contract',
  ],
};
```

---

## 3. Express Integration (`@askdepth/audience-connector/express`)

> [!IMPORTANT]
> **The Express Raw Body Rule**: Never use `express.json()` on routes handled by `expressHandler`. `express.json()` consumes and parses the body stream, destroying the exact raw byte representation required for HMAC-SHA256 verification. If `expressHandler` detects a pre-parsed object on `req.body`, it immediately responds with HTTP 500 (`shim_misconfigured`).

```typescript
import express from 'express';
import { Pool } from 'pg';
import { createConnector, postgresAdapter } from '@askdepth/audience-connector';
import { expressHandler } from '@askdepth/audience-connector/express';

const connector = createConnector({
  secret: process.env.ASKDEPTH_CONNECTOR_SECRET!,
  basePath: '/askdepth/v1',
  adapter: postgresAdapter({
    pool: new Pool({ connectionString: process.env.READONLY_DATABASE_URL }),
    table: 'users',
  }),
  fieldMapping: {
    externalId: 'id',
    email: 'email',
    segment: 'plan',
    signupAt: 'created_at',
    isActive: 'is_active',
  },
  attributes: {
    filterable: ['country', 'plan'],
    returnable: ['plan'],
  },
});

const app = express();

// Mount raw body parser strictly before expressHandler
app.use('/askdepth/v1', express.raw({ type: 'application/json' }), expressHandler(connector));

// Other app routes can still use express.json()
app.use(express.json());

app.listen(8787, () => {
  console.log('Askdepth Audience Connector listening on :8787');
});
```

---

## 4. Fastify Integration (`@askdepth/audience-connector/fastify`)

`fastifyPlugin` registers a buffer content-type parser (`{ parseAs: 'buffer' }`) for `application/json` within its encapsulated scope and routes requests to `connector.fetch`:

```typescript
import Fastify from 'fastify';
import { Pool } from 'pg';
import { createConnector, postgresAdapter } from '@askdepth/audience-connector';
import { fastifyPlugin } from '@askdepth/audience-connector/fastify';

const fastify = Fastify({ logger: true });

const connector = createConnector({
  secret: process.env.ASKDEPTH_CONNECTOR_SECRET!,
  basePath: '/askdepth/v1',
  adapter: postgresAdapter({
    pool: new Pool({ connectionString: process.env.READONLY_DATABASE_URL }),
    table: 'users',
  }),
  fieldMapping: {
    externalId: 'id',
    email: 'email',
  },
});

fastify.register(fastifyPlugin(connector));

await fastify.listen({ port: 8787, host: '0.0.0.0' });
```

---

## 5. AWS Lambda (API Gateway HTTP API v2) (`@askdepth/audience-connector/lambda`)

`lambdaHandler` accepts `APIGatewayProxyEventV2`, automatically decodes base64-encoded request bodies when `isBase64Encoded` is true, and returns a structured `LambdaResultV2`:

```typescript
import { Pool } from 'pg';
import { createConnector, postgresAdapter } from '@askdepth/audience-connector';
import { lambdaHandler } from '@askdepth/audience-connector/lambda';

const pool = new Pool({
  connectionString: process.env.READONLY_DATABASE_URL,
  max: 2,
});

const connector = createConnector({
  secret: process.env.ASKDEPTH_CONNECTOR_SECRET!,
  basePath: '/askdepth/v1',
  adapter: postgresAdapter({ pool, table: 'users' }),
  fieldMapping: {
    externalId: 'id',
    email: 'email',
  },
});

export const handler = lambdaHandler(connector);
```

---

## 6. Cloudflare Workers Integration

Because `connector.fetch` uses the standard Web `Request` / `Response` API, it works directly as a Cloudflare Worker fetch handler. You **must** enable `nodejs_compat` in `wrangler.toml` so `node:crypto` is available for HMAC verification.

`wrangler.toml`:
```toml
name = "askdepth-audience-connector"
main = "src/index.ts"
compatibility_date = "2024-09-23"
compatibility_flags = ["nodejs_compat"]
```

`src/index.ts`:
```typescript
import { createConnector, restAdapter } from '@askdepth/audience-connector';

export interface Env {
  ASKDEPTH_CONNECTOR_SECRET: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const connector = createConnector({
      secret: env.ASKDEPTH_CONNECTOR_SECRET,
      fieldMapping: { externalId: 'id', email: 'email' },
      adapter: restAdapter({
        declaredSchema: {
          columns: [
            { name: 'id', type: 'string' },
            { name: 'email', type: 'string' },
          ],
        },
        fetchCandidates: async () => ({ rows: [] }),
        fetchCount: async () => 0,
      }),
    });

    return connector.fetch(request);
  },
};
```
