---
name: askdepth-sdk
description: >-
  Integrate, configure, and troubleshoot the Askdepth Browser & React Telemetry
  SDK (@askdepth/core, @askdepth/react, @askdepth/replay, @askdepth/contracts).
  Use this skill whenever the user asks to install or set up Askdepth analytics,
  wrap a Next.js or React app with AskdepthProvider, configure AST component
  mapping (withAskdepth, askdepthVitePlugin, askdepthWebpackPlugin,
  askdepthRspackPlugin), upload component maps (askdepth-upload-map), capture UI
  frustration anomalies (rage clicks, dead clicks, error clicks), add React
  error boundaries (AskdepthCatch), configure Zero-PII session replay masking
  (data-askdepth-mask, data-askdepth-block, data-askdepth-allow-rapid),
  propagate W3C traceparent headers, or link anonymous sessions to user IDs via
  askdepth.identify().
---

# Askdepth Browser & React SDK (`@askdepth/*`)

The **Askdepth SDK** is a zero-dependency Real User Monitoring (RUM), frustration detection, W3C distributed tracing, and Zero-PII session replay toolkit for modern web and React applications.

> **Brand Naming Rule**: The official product name is **Askdepth** (capital `A`, lowercase `d` — never write the camel-cased or hyphenated form). NPM package scopes are always lowercase `@askdepth/*`.

---

## 1. Package Ecosystem

| Package | Current Version | Purpose & When to Install | Bundle Budget |
| :--- | :--- | :--- | :--- |
| **`@askdepth/react`** | `^0.1.1` | React 18/19 & Next.js integration: `<AskdepthProvider>`, `useAskdepth()`, `<AskdepthCatch>`, Fiber tree inspection, bundler AST plugins (`@askdepth/react/plugin`), and the `askdepth-upload-map` CLI. | < 4.8 kB gzip |
| **`@askdepth/core`** | `^0.3.1` | Core browser engine: `Askdepth.init()`, consent lifecycle, rage/dead/error click detectors, W3C `traceparent` injection, memory batcher, and remote kill-switch. | < 10 kB gzip (~6.7 kB) |
| **`@askdepth/replay`** | `^0.3.1` | Opt-in 45-second rolling in-memory ring buffer for session replay, Zero-PII DOM/text redaction, and 45 KiB `keepalive` chunked uploads on anomaly trigger. | < 35 kB gzip (~31.5 kB) |
| **`@askdepth/contracts`** | `^0.2.1` | Shared wire protocol schemas (`Zod`), telemetry envelope types, W3C traceparent parsers, and component map manifest types. | N/A (types & schemas) |

---

## 2. Quick-Start Selection Guide

Choose the integration path that matches the target application:

1. **Next.js App Router (Next.js 14 / 15 / 16 with Webpack or Turbopack)**:
   - Install `@askdepth/react @askdepth/core @askdepth/replay`.
   - Wrap `next.config.mjs` with `withAskdepth` (and sanitize `turbopack.rules` on Next.js 16).
   - Mount `<AskdepthProvider>` inside a `'use client'` wrapper in `app/layout.tsx`.
   - Allowlist the Askdepth collector origin in Content Security Policy (`connect-src`).
   - Add `askdepth-upload-map` to your post-build step.
   - Read **[Next.js App Router Production Guide](./references/nextjs-app-router.md)** for complete copy-paste templates.
2. **React + Vite / Webpack / Rspack SPA**:
   - Install `@askdepth/react @askdepth/core @askdepth/replay`.
   - Add `askdepthVitePlugin`, `askdepthWebpackPlugin`, or `askdepthRspackPlugin` to the bundler config.
   - Wrap the root `<App />` in `<AskdepthProvider>`.
   - Read **[Vite, Bundlers & Vanilla JS Guide](./references/vite-and-vanilla.md)**.
3. **Vanilla JavaScript / TypeScript (No React)**:
   - Install `@askdepth/core` (and optionally `@askdepth/replay`).
   - Call `Askdepth.init({ writeKey, endpoint, consent, replay: true })`.
   - Read **[Vite, Bundlers & Vanilla JS Guide](./references/vite-and-vanilla.md)**.

---

## 3. Core Integration Workflow (5 Steps)

### Step 1: Install Packages

```bash
# React / Next.js applications
pnpm add @askdepth/react @askdepth/core @askdepth/replay

# Vanilla JS / TS applications
pnpm add @askdepth/core @askdepth/replay
```

### Step 2: Configure Environment Variables

Never expose private deploy tokens (`ad_dep_...`) to the browser. Only the publishable browser write key (`ad_bw_...`) belongs in client-accessible variables:

```ini
# Public (Browser + SSR)
NEXT_PUBLIC_ASKDEPTH_WRITE_KEY="ad_bw_your_publishable_write_key"
NEXT_PUBLIC_ASKDEPTH_ENDPOINT="https://in.askdepth.com/v1/telemetry"
NEXT_PUBLIC_ASKDEPTH_ENVIRONMENT="production"
NEXT_PUBLIC_ASKDEPTH_BUILD_ID="demo-shop-v1"

# Private (Build / CI only — used to upload component maps)
ASKDEPTH_DEPLOY_TOKEN="ad_dep_your_private_deploy_token"
ASKDEPTH_INGEST_URL="https://in.askdepth.com"
ASKDEPTH_REPOSITORY="your-org/your-repo"
```

### Step 3: Instrument the Bundler for Component Attribution

In production builds (`NODE_ENV === 'production'`), the Askdepth AST compiler injects opaque `data-askdepth-id="cmp_xxxxxxxx"` attributes (never leaking source paths into client bundles) and emits `.askdepth/component-maps/<buildId>.manifest.json`. In development (`NODE_ENV !== 'production'`), it injects `data-askdepth-src="path/to/File.tsx:line:col"`.

```javascript
// next.config.mjs
import { withAskdepth } from '@askdepth/react/plugin';

const nextConfig = {
  transpilePackages: ['@askdepth/react', '@askdepth/core', '@askdepth/replay', '@askdepth/contracts'],
};

export default withAskdepth(nextConfig, {
  buildId: process.env.NEXT_PUBLIC_ASKDEPTH_BUILD_ID || 'local-dev',
  repository: process.env.ASKDEPTH_REPOSITORY || 'org/repo',
});
```

> **Next.js 16 Turbopack Note**: Next.js 16 moved `experimental.turbo.rules` to top-level `turbopack.rules` and requires stripping the `as` property on `*.tsx`/`*.jsx` rules to prevent `.tsx.tsx` double-extension errors. See **[Next.js App Router Production Guide](./references/nextjs-app-router.md)** for the exact Next.js 16 wrapper snippet.

### Step 4: Mount `<AskdepthProvider>`, Identity Sync & Error Boundaries

Wrap your client tree in `<AskdepthProvider>`, bridge authenticated users using Zero-PII `askdepth.identify(userId)` (passing only the database `externalId` that matches your Audience Connector, **never** email, name, or phone), and wrap interactive widgets in `<AskdepthCatch>`:

```tsx
'use client';

import { AskdepthProvider, AskdepthCatch, useAskdepth } from '@askdepth/react';

export function CheckoutSection() {
  const askdepth = useAskdepth();

  return (
    <AskdepthCatch
      fallback={(error, reset) => (
        <div role="alert">
          <p>Something went wrong: {error.message}</p>
          <button onClick={reset}>Retry</button>
        </div>
      )}
    >
      <button
        onClick={() => {
          askdepth.track('checkout_started', { item_count: 3 });
        }}
      >
        Proceed to Checkout
      </button>
    </AskdepthCatch>
  );
}
```

### Step 5: Apply Zero-PII DOM Attributes & Upload Component Maps

1. **Mask sensitive inputs & text**: Add `data-askdepth-mask=""` to containers with shipping addresses, names, or personal details (renders as `***` in replay).
2. **Block payment/secret subtrees**: Add `data-askdepth-block=""` to payment card forms, auth panels, or internal debug drawers (completely omitted from DOM serialization).
3. **Exempt rapid-click controls**: Add `data-askdepth-allow-rapid=""` to cart quantity `+` / `-` buttons or pagination controls so rapid clicking does not trigger false-positive `RAGE_CLICK` events.
4. **Upload component maps after build**:
   ```bash
   npx askdepth-upload-map \
     --endpoint "$ASKDEPTH_INGEST_URL" \
     --deploy-token "$ASKDEPTH_DEPLOY_TOKEN" \
     --build-id "$NEXT_PUBLIC_ASKDEPTH_BUILD_ID" \
     --repository "$ASKDEPTH_REPOSITORY"
   ```

---

## 4. Critical Rules & Gotchas

1. **Zero-PII Identity Rule**: When calling `askdepth.identify(userId, traits)`, pass only the opaque database ID (e.g., `usr_123` or `cust_hero_sarah`) that corresponds to `externalId` in `@askdepth/audience-connector`. Do **not** pass emails, full names, or phone numbers into telemetry traits, and skip identifying internal admin accounts (`role === 'admin'`).
2. **Consent Lifecycle**: `consent` defaults to `'unknown'`. While `'unknown'` or `'denied'`, DOM listeners, session IDs, W3C tracing, and network flushes remain paused. Call `Askdepth.setConsent('granted')` (or pass `consent="granted"` to `<AskdepthProvider>`) to activate collection; call `Askdepth.revokeConsent()` to immediately tear down listeners and purge buffers.
3. **W3C Distributed Tracing (`traceparent`)**: Pass `allowedTracingOrigins: [window.location.origin]` (or your backend API origins) so `@askdepth/core` automatically injects `traceparent` headers into matching `fetch` and `XMLHttpRequest` calls. When a backend route fails with HTTP 4xx/5xx within 10 seconds of a click, Askdepth correlates the click (`ERROR_CLICK`) with the exact backend trace.
4. **CSP `connect-src` Allowlist**: If your app enforces a Content Security Policy, allowlist `https://*.askdepth.com`, `https://*.askdepth.io`, and your custom collector origin in `connect-src`.
5. **Audience Connector Route Coexistence**: If your Next.js app also mounts `@askdepth/audience-connector` at `/askdepth/v1`, bypass middleware/proxy processing for `pathname.startsWith('/askdepth/v1')` so HMAC request bytes are never mutated.

---

## 5. Verification Checklist

Run these checks to confirm a working integration:

- [ ] **Build & Component Maps**: Run `npm run build` (or `pnpm build`) and verify `.askdepth/component-maps/<buildId>.manifest.json` is generated on disk.
- [ ] **Map Upload**: Run `npx askdepth-upload-map` (or `node scripts/upload-component-map.mjs`) with `ASKDEPTH_DEPLOY_TOKEN` and confirm a success message.
- [ ] **SSR Smoke Test**: Verify importing `@askdepth/core` and `@askdepth/react` during server rendering produces zero `window`/`document` errors (`Askdepth.isInitialized()` returns `false` on the server and `true` in the browser once consent is `'granted'`).
- [ ] **Telemetry Headers**: In Browser DevTools Network tab, trigger an event or rage click and verify `POST /v1/telemetry` sends headers `x-askdepth-write-key`, `x-askdepth-protocol-version`, `x-askdepth-sdk-name`, `x-askdepth-sdk-version`, and `x-askdepth-build-id`.
- [ ] **Privacy Attributes**: Confirm all PII forms carry `data-askdepth-mask=""`, payment/credential widgets carry `data-askdepth-block=""`, and stepper buttons carry `data-askdepth-allow-rapid=""`.

---

## 6. Detailed Reference Guides

Load these references on demand depending on your task:

- **[Next.js App Router Production Guide](./references/nextjs-app-router.md)** — Complete Next.js 14/15/16 setup including Turbopack rule sanitization, server-to-client runtime env hydration, `<AskdepthProvider>`, `AskdepthIdentitySync`, `<AskdepthCatch>`, CSP `proxy.ts`, and `scripts/upload-component-map.mjs`.
- **[Vite, Bundlers & Vanilla JS Guide](./references/vite-and-vanilla.md)** — Setup for Vite (`askdepthVitePlugin`), Webpack (`askdepthWebpackPlugin`), Rspack (`askdepthRspackPlugin`), React SPAs, Vanilla JS/TS (`Askdepth.init`), consent banner integration, and SSR safety invariants.
- **[Privacy, Session Replay & Frustration Heuristics](./references/privacy-replay-and-heuristics.md)** — Deep dive into `RAGE_CLICK`, `DEAD_CLICK`, and `ERROR_CLICK` detection thresholds, Zero-PII DOM & regex scrubbing, 45s ring buffer & 45 KiB `keepalive` chunking, Safari low-memory clamping, and remote kill-switch behavior.
- **[Complete API & CLI Reference](./references/api-and-cli.md)** — Full TypeScript signatures, bounded property limits, provider props, and `askdepth-upload-map` CLI flags and environment variables.
