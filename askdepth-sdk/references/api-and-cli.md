# Askdepth SDK Complete API & CLI Reference

This reference documents all public exports, TypeScript interfaces, payload bounds, and CLI options across `@askdepth/core`, `@askdepth/react`, `@askdepth/replay`, and `@askdepth/contracts`.

---

## 1. `@askdepth/core` API

### `Askdepth` Singleton Methods

```typescript
import { Askdepth, type AskdepthInitOptions, type ConsentState } from '@askdepth/core';
```

| Method | Signature | Description |
| :--- | :--- | :--- |
| `Askdepth.init` | `(options: AskdepthInitOptions) => typeof Askdepth` | Initializes the browser runtime. Idempotent if called with identical options; tears down and re-initializes if options change. Safe no-op during SSR. |
| `Askdepth.setConsent` | `(consent: 'granted' \| 'denied' \| 'unknown') => void` | Transitions consent state. `'granted'` starts DOM listeners, allocates session ID, enables `traceparent` injection, and resumes transport. `'denied'` / `'unknown'` stops listeners, clears session ID, and pauses transport. |
| `Askdepth.revokeConsent` | `() => void` | Shorthand for `Askdepth.setConsent('denied')`. |
| `Askdepth.onConsentChange` | `(listener: (consent: ConsentState) => void) => () => void` | Subscribes to consent state changes. Returns an unsubscribe function. |
| `Askdepth.track` | `(name: string, properties?: Record<string, unknown>) => void` | Queues a custom telemetry event. Subject to bounded property limits (below). |
| `Askdepth.identify` | `(userId: string, traits?: Record<string, unknown>) => void` | Links the current session to a database `externalId` ($\le 200$ chars). Pass Zero-PII traits only. |
| `Askdepth.getTraceparent` | `() => string \| null` | Returns active W3C `traceparent` string (`00-<traceId>-<spanId>-01`) or `null` if uninitialized or consent not granted. |
| `Askdepth.getSessionId` | `() => string \| null` | Returns active session UUID or `null` if consent is not `'granted'`. |
| `Askdepth.isInitialized` | `() => boolean` | Returns `true` when running in a browser with active listeners and not terminated by kill-switch. |
| `Askdepth.registerComponentResolver` | `(resolver: ComponentResolver) => () => void` | Registers a custom callback `(target: Element) => ComponentLocation \| null` to map DOM nodes to UI components. |
| `Askdepth.reportCaughtError` | `(error: Error, target?: Element \| null, componentStack?: string) => void` | Reports a handled error (`ERROR_CLICK` with `error_code: 'REACT_COMPONENT_ERROR'`) and triggers a 45s replay flash. |

### `AskdepthInitOptions`

```typescript
export interface AskdepthInitOptions {
  /** Required publishable browser write key (e.g. "ad_bw_...") */
  writeKey: string;
  /** Collector endpoint URL (default: "https://in.askdepth.com") */
  endpoint?: string;
  /** Initial consent state (default: "unknown") */
  consent?: 'granted' | 'denied' | 'unknown';
  /** Session sampling ratio between 0.0 and 1.0 (default: 1.0) */
  sampleRate?: number;
  /** Deployment environment tag (default: "production") */
  environment?: 'production' | 'staging' | 'development';
  /** Release or commit correlation ID matching uploaded component maps */
  buildId?: string;
  /** Enable 45s rolling session replay buffer */
  replay?: boolean | {
    endpoint?: string;         // Defaults to `${endpoint}/v1/replays/upload`
    checkoutEveryNms?: number; // FullSnapshot interval in ms (default: 45000)
  };
  /** Origins allowed to receive automatic W3C `traceparent` headers on fetch/XHR */
  allowedTracingOrigins?: string[];
  /** Additional CSS selectors exempt from Rage Click detection */
  allowRapidClickSelectors?: string[];
}
```

### Bounded Property Limits (`track` and `identify`)
To protect payload budgets and prevent accidental memory bloat, `@askdepth/core` enforces strict bounds on `properties` and `traits`:
- **Max keys per object**: `32`
- **Max key length**: `64` characters
- **Max string value length**: `512` characters (truncated if longer)
- **Max array length**: `50` elements
- **Max nesting depth**: `3` levels

---

## 2. `@askdepth/react` API

### Runtime Exports (`@askdepth/react`)

```typescript
import {
  AskdepthProvider,
  useAskdepth,
  AskdepthCatch,
  resolveComponentLocation,
  type AskdepthProviderProps,
  type AskdepthContextValue,
} from '@askdepth/react';
```

- **`<AskdepthProvider {...props}>`**:
  Accepts all `AskdepthInitOptions` (`writeKey` or alias `apiKey`, `endpoint`, `consent`, `environment`, `buildId`, `replay`, `sampleRate`, `allowedTracingOrigins`, `allowRapidClickSelectors`) plus `children`. Automatically registers the React Fiber component resolver and SPA navigation listeners.
- **`useAskdepth(): AskdepthContextValue`**:
  Returns:
  ```typescript
  interface AskdepthContextValue {
    track: (name: string, properties?: Record<string, unknown>) => void;
    identify: (userId: string, traits?: Record<string, unknown>) => void;
    getTraceparent: () => string | null;
    isReady: () => boolean;
    getSessionId: () => string | null;
  }
  ```
  Safe to call outside `<AskdepthProvider>` (returns no-op methods with `isReady() === false`).
- **`<AskdepthCatch fallback={...}>`**:
  React Error Boundary (`'use client'`). `fallback` can be a `ReactNode` or a render function `(error: Error, reset: () => void) => ReactNode`.
- **`resolveComponentLocation(target: Element)`**:
  Inspects `data-askdepth-id`, `data-askdepth-src`, and walks React 18/19 Fiber nodes (`__reactFiber$*`) up to depth 64 to extract component name and hierarchy stack.

### Compiler Plugin Exports (`@askdepth/react/plugin`)

```typescript
import {
  withAskdepth,
  askdepthVitePlugin,
  askdepthWebpackPlugin,
  askdepthRspackPlugin,
} from '@askdepth/react/plugin';
```

Plugin options:
```typescript
interface AskdepthPluginOptions {
  buildId?: string;
  repository?: string;
  commitSha?: string;
  environment?: string;
  outputDir?: string; // Defaults to ".askdepth/component-maps"
}
```

### Programmatic Upload Helpers (`@askdepth/react/upload-map`)

```typescript
import { findManifestFile, uploadComponentMap } from '@askdepth/react/upload-map';
```
- `findManifestFile(dir: string, buildId?: string): string | null`: Consolidates per-chunk JSON maps in `.askdepth/component-maps/<buildId>/` into `<buildId>.manifest.json` and returns its absolute path.
- `uploadComponentMap(options): Promise<{ success: boolean; message: string }>`: Uploads the consolidated manifest with SHA-256 integrity checksum to Askdepth Ingest.

---

## 3. CLI Reference: `askdepth-upload-map`

Installed as a binary by `@askdepth/react`:

```bash
npx askdepth-upload-map [options]
```

| CLI Flag | Shorthand | Environment Variable Fallback | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `--endpoint <url>` | `-e` | `ASKDEPTH_INGEST_URL` | `https://in.askdepth.com` | Base URL of the Askdepth Ingest API (strip `/v1/telemetry` suffix if sharing env var). |
| `--deploy-token <token>` | `-t` | `ASKDEPTH_DEPLOY_TOKEN` | *(Required)* | Private project Deploy Token (`ad_dep_...`). |
| `--api-key <key>` | `-k`, `--write-key` | `ASKDEPTH_API_KEY`, `ASKDEPTH_WRITE_KEY` | *(Fallback)* | Alias for `--deploy-token`. |
| `--build-id <id>` | `-b` | `ASKDEPTH_BUILD_ID`, `NEXT_PUBLIC_ASKDEPTH_BUILD_ID`, `VERCEL_GIT_COMMIT_SHA`, `GITHUB_SHA`, `BUILD_ID` | Auto-detected | Build identifier matching the runtime `buildId`. |
| `--repository <repo>` | `-r` | `ASKDEPTH_REPOSITORY`, `GITHUB_REPOSITORY` | Auto-detected | Repository slug in `owner/repo` format. |
| `--commit-sha <sha>` | `-c` | `ASKDEPTH_COMMIT_SHA`, `GITHUB_SHA` | `git rev-parse HEAD` | Full 40-character hexadecimal Git commit SHA. |
| `--dir <path>` | `-d` | `ASKDEPTH_COMPONENT_MAP_DIR` | `./.askdepth/component-maps` | Directory containing `<buildId>.manifest.json`. |
| `--help` | `-h` | — | — | Print CLI usage information. |
