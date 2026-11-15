# Next.js App Router Production Integration Guide

This reference provides battle-tested, production-grade patterns for integrating `@askdepth/react`, `@askdepth/core`, and `@askdepth/replay` into a Next.js App Router application (Next.js 14, 15, and 16, including Turbopack and standalone Docker/Firebase App Hosting builds).

---

## 1. Next.js Configuration (`next.config.mjs`)

Wrap your Next.js configuration with `withAskdepth` from `@askdepth/react/plugin`.

### Handling Next.js 16 Turbopack Rules

In Next.js 16, `experimental.turbo.rules` was promoted to top-level `turbopack.rules`, and passing `as: '*.tsx'` on rules matching `*.tsx` causes Turbopack to append `.tsx` twice (`.tsx.tsx`). Use the pattern below to support both Webpack and Next.js 16 Turbopack cleanly:

```javascript
// next.config.mjs
import { execSync } from 'node:child_process';
import { withAskdepth } from '@askdepth/react/plugin';

function resolveGitSha() {
  if (process.env.ASKDEPTH_COMMIT_SHA) return process.env.ASKDEPTH_COMMIT_SHA;
  if (process.env.COMMIT_SHA) return process.env.COMMIT_SHA;
  try {
    return execSync('git rev-parse HEAD', { stdio: ['ignore', 'pipe', 'ignore'] })
      .toString()
      .trim();
  } catch {
    return undefined;
  }
}

const commitSha = resolveGitSha();
const buildId =
  process.env.NEXT_PUBLIC_ASKDEPTH_BUILD_ID ||
  process.env.ASKDEPTH_BUILD_ID ||
  (commitSha ? commitSha.slice(0, 12) : 'app-v1');
const repository = process.env.ASKDEPTH_REPOSITORY || 'your-org/your-repo';

/** @type {import('next').NextConfig} */
const baseConfig = {
  output: 'standalone',
  transpilePackages: [
    '@askdepth/react',
    '@askdepth/core',
    '@askdepth/replay',
    '@askdepth/contracts',
  ],
  env: {
    NEXT_PUBLIC_ASKDEPTH_BUILD_ID: buildId,
  },
};

const wrapped = withAskdepth(baseConfig, {
  buildId,
  repository,
  ...(commitSha && /^[a-f0-9]{40}$/i.test(commitSha) ? { commitSha } : {}),
});

// Next.js 16 Turbopack compatibility: promote `experimental.turbo.rules` to top-level
// `turbopack.rules` and strip `as` to prevent `.tsx.tsx` double-extension build errors.
const rawTurboRules = wrapped.experimental?.turbo?.rules;
if (wrapped.experimental && 'turbo' in wrapped.experimental) {
  delete wrapped.experimental.turbo;
  if (Object.keys(wrapped.experimental).length === 0) {
    delete wrapped.experimental;
  }
}

const cleanedRules = {};
if (rawTurboRules) {
  for (const [pattern, rule] of Object.entries(rawTurboRules)) {
    if (rule && typeof rule === 'object') {
      const { as: _as, ...rest } = rule;
      cleanedRules[pattern] = rest;
    }
  }
}

/** @type {import('next').NextConfig} */
const nextConfig = {
  ...wrapped,
  turbopack: {
    rules: cleanedRules,
  },
};

export default nextConfig;
```

---

## 2. Server-to-Client Config Hydration (`app/layout.tsx` & `components/providers.tsx`)

In containerized deployments (Docker, Cloud Run, Firebase App Hosting), runtime secrets or environment variables may be injected at container startup rather than baked at `next build` time. Reading them in the Root Server Layout (`app/layout.tsx`) and passing them down as `initialAskdepthConfig` guarantees the client provider always receives live configuration.

### Helper: Normalize Endpoint & Environment (`lib/askdepth-config.ts`)

```typescript
// lib/askdepth-config.ts
export type AskdepthEnvironment = 'production' | 'staging' | 'development';

export interface AskdepthRuntimeConfig {
  writeKey: string;
  endpoint: string;
  buildId: string;
  environment: AskdepthEnvironment;
}

export function normalizeTelemetryEndpoint(raw?: string): string {
  const base = (raw || 'https://in.askdepth.com').trim().replace(/\/+$/, '');
  return base.endsWith('/v1/telemetry') ? base : `${base}/v1/telemetry`;
}

export function normalizeEnvironment(raw?: string): AskdepthEnvironment {
  if (raw === 'staging' || raw === 'development' || raw === 'production') {
    return raw;
  }
  return process.env.NODE_ENV === 'production' ? 'production' : 'development';
}
```

### Root Layout (`app/layout.tsx`)

```tsx
// app/layout.tsx
import React from 'react';
import { Providers } from '@/components/providers';
import {
  normalizeEnvironment,
  normalizeTelemetryEndpoint,
} from '@/lib/askdepth-config';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  const initialAskdepthConfig = {
    writeKey: process.env.NEXT_PUBLIC_ASKDEPTH_WRITE_KEY || '',
    endpoint: normalizeTelemetryEndpoint(
      process.env.NEXT_PUBLIC_ASKDEPTH_ENDPOINT || process.env.ASKDEPTH_INGEST_URL
    ),
    buildId: process.env.NEXT_PUBLIC_ASKDEPTH_BUILD_ID || 'app-v1',
    environment: normalizeEnvironment(process.env.NEXT_PUBLIC_ASKDEPTH_ENVIRONMENT),
  };

  return (
    <html lang="en">
      <body>
        <Providers initialAskdepthConfig={initialAskdepthConfig}>
          {children}
        </Providers>
      </body>
    </html>
  );
}
```

### Client Providers (`components/providers.tsx`)

```tsx
// components/providers.tsx
'use client';

import React from 'react';
import { AskdepthProvider } from '@askdepth/react';
import type { AskdepthRuntimeConfig } from '@/lib/askdepth-config';
import { AskdepthIdentitySync } from '@/components/askdepth-identity-sync';

interface ProvidersProps {
  children: React.ReactNode;
  initialAskdepthConfig: AskdepthRuntimeConfig;
}

export function Providers({ children, initialAskdepthConfig }: ProvidersProps) {
  const appTree = (
    <>
      <AskdepthIdentitySync />
      {children}
    </>
  );

  if (!initialAskdepthConfig.writeKey) {
    return appTree;
  }

  return (
    <AskdepthProvider
      writeKey={initialAskdepthConfig.writeKey}
      endpoint={initialAskdepthConfig.endpoint}
      buildId={initialAskdepthConfig.buildId}
      environment={initialAskdepthConfig.environment}
      consent="granted" // Or 'unknown' until cookie consent is accepted
      sampleRate={1}
      replay={true}
      allowedTracingOrigins={
        typeof window !== 'undefined' ? [window.location.origin] : []
      }
    >
      {appTree}
    </AskdepthProvider>
  );
}
```

---

## 3. Zero-PII Identity Bridge (`components/askdepth-identity-sync.tsx`)

To link browser sessions and frustration anomalies with your customer records in the Askdepth Audience Connector, call `askdepth.identify(userId)` once the user is authenticated.

**Critical Privacy Invariants**:
1. Pass **only** the immutable database `id` (which maps to `externalId` in `@askdepth/audience-connector`).
2. **Never** pass email addresses, names, or phone numbers in `traits`.
3. Exclude internal staff or admin accounts (`role === 'admin'`) so internal testing never triggers customer outreach.
4. Deduplicate calls per `(userId, sessionId)` pair.

```tsx
// components/askdepth-identity-sync.tsx
'use client';

import { useEffect, useRef } from 'react';
import { useSession } from 'next-auth/react'; // Or your auth provider hook
import { useAskdepth } from '@askdepth/react';

export function AskdepthIdentitySync() {
  const { data: session, status } = useSession();
  const askdepth = useAskdepth();
  const lastIdentifiedRef = useRef<string | null>(null);

  const userId = session?.user?.id ?? null;
  const userRole = (session?.user as { role?: string } | undefined)?.role ?? null;
  const sdkReady = askdepth.isReady();

  useEffect(() => {
    if (status !== 'authenticated' || !userId || !sdkReady) {
      if (status === 'unauthenticated') {
        lastIdentifiedRef.current = null;
      }
      return;
    }

    // Never link internal admin users as customer outreach targets
    if (userRole?.toLowerCase() === 'admin') {
      return;
    }

    const sessionId = askdepth.getSessionId() ?? 'pending';
    const dedupeKey = `${userId}:${sessionId}`;
    if (lastIdentifiedRef.current === dedupeKey) {
      return;
    }

    // Zero-PII: only pass the database ID matching Audience Connector `externalId`
    askdepth.identify(userId, {
      page: typeof window !== 'undefined' ? window.location.href : undefined,
    });
    lastIdentifiedRef.current = dedupeKey;
  }, [status, userId, userRole, sdkReady, askdepth]);

  return null;
}
```

---

## 4. Middleware / Request Proxy & Content Security Policy (`proxy.ts` or `middleware.ts`)

If your Next.js app defines a `proxy.ts` (Next.js 16) or `middleware.ts` (Next.js 14/15) with a Content Security Policy (CSP):

1. Bypass middleware processing on `/askdepth/v1` if you also host the Askdepth Audience Connector in the same Next.js app.
2. Add Askdepth ingest domains (`https://*.askdepth.com`, `https://*.askdepth.io`, and any custom collector URL) to `connect-src`.

```typescript
// proxy.ts (or middleware.ts)
import { NextResponse, type NextRequest } from 'next/server';

export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // 1. Pass Askdepth Audience Connector HMAC wire calls straight through
  if (pathname.startsWith('/askdepth/v1')) {
    return NextResponse.next();
  }

  // 2. Allowlist Askdepth telemetry & replay endpoints in CSP connect-src
  const extraConnectOrigins = new Set<string>([
    'https://*.askdepth.com',
    'https://*.askdepth.io',
  ]);
  for (const rawUrl of [
    process.env.NEXT_PUBLIC_ASKDEPTH_ENDPOINT,
    process.env.ASKDEPTH_INGEST_URL,
  ]) {
    if (rawUrl) {
      try {
        extraConnectOrigins.add(new URL(rawUrl).origin);
      } catch {
        // Ignore invalid URL env values
      }
    }
  }

  const response = NextResponse.next();
  const connectSrc = ["'self'", ...Array.from(extraConnectOrigins)].join(' ');
  response.headers.set(
    'Content-Security-Policy',
    `default-src 'self'; connect-src ${connectSrc};`
  );
  return response;
}
```

---

## 5. Component Map Consolidation & Upload Script (`scripts/upload-component-map.mjs`)

When Next.js builds with Turbopack or SWC across multiple workers, chunk manifests are written into `.askdepth/component-maps/<buildId>/`. Calling `findManifestFile(mapDir, buildId)` consolidates them into `.askdepth/component-maps/<buildId>.manifest.json`, and `uploadComponentMap(...)` uploads the manifest to Askdepth Ingest.

Add this script at `scripts/upload-component-map.mjs` and wire `"build": "next build && node scripts/upload-component-map.mjs"` in `package.json`:

```javascript
#!/usr/bin/env node
// scripts/upload-component-map.mjs
import { execSync } from 'node:child_process';
import { existsSync } from 'node:fs';
import { resolve } from 'node:path';
import { findManifestFile, uploadComponentMap } from '@askdepth/react/upload-map';

function resolveGitSha() {
  if (process.env.ASKDEPTH_COMMIT_SHA) return process.env.ASKDEPTH_COMMIT_SHA;
  if (process.env.COMMIT_SHA && /^[a-f0-9]{40}$/i.test(process.env.COMMIT_SHA)) {
    return process.env.COMMIT_SHA;
  }
  try {
    const sha = execSync('git rev-parse HEAD', {
      stdio: ['ignore', 'pipe', 'ignore'],
    })
      .toString()
      .trim();
    return /^[a-f0-9]{40}$/i.test(sha)
      ? sha
      : '0000000000000000000000000000000000000000';
  } catch {
    return '0000000000000000000000000000000000000000';
  }
}

async function main() {
  const mapDir = resolve(process.cwd(), '.askdepth/component-maps');
  if (!existsSync(mapDir)) {
    console.log(
      '[Askdepth] No .askdepth/component-maps directory found; skipping upload.'
    );
    return;
  }

  const commitSha = resolveGitSha();
  const buildId =
    process.env.NEXT_PUBLIC_ASKDEPTH_BUILD_ID ||
    process.env.ASKDEPTH_BUILD_ID ||
    commitSha.slice(0, 12);

  // Consolidate Turbopack/SWC chunk maps into `<buildId>.manifest.json`
  const manifestPath = findManifestFile(mapDir, buildId);
  if (manifestPath) {
    console.log(`[Askdepth] Component map manifest ready: ${manifestPath}`);
  }

  const deployToken = (
    process.env.ASKDEPTH_DEPLOY_TOKEN ||
    process.env.ASKDEPTH_API_KEY ||
    ''
  ).trim();

  if (!deployToken) {
    console.log(
      '[Askdepth] Set ASKDEPTH_DEPLOY_TOKEN (ad_dep_...) to auto-upload component maps to Askdepth Ingest.'
    );
    return;
  }

  const rawEndpoint =
    process.env.ASKDEPTH_INGEST_URL ||
    process.env.NEXT_PUBLIC_ASKDEPTH_ENDPOINT ||
    'https://in.askdepth.com';
  const ingestBaseUrl = rawEndpoint.replace(/\/v1\/telemetry\/?$/, '');
  const repository = process.env.ASKDEPTH_REPOSITORY || 'your-org/your-repo';

  const result = await uploadComponentMap({
    endpoint: ingestBaseUrl,
    deployToken,
    buildId,
    repository,
    commitSha,
    dir: mapDir,
  });

  if (result.success) {
    console.log(`[Askdepth] ${result.message}`);
  } else {
    console.warn(`[Askdepth] Component map upload skipped/failed: ${result.message}`);
  }
}

main().catch((err) => {
  console.warn(
    '[Askdepth] Warning: component map upload did not complete:',
    err instanceof Error ? err.message : String(err)
  );
});
```

> **Tip**: Add `.askdepth/` to `.gitignore` so generated local component maps are not committed to git.
