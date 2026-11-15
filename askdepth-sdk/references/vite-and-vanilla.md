# Vite, Webpack, Rspack & Vanilla JS Integration Guide

This reference covers integrating Askdepth into React Single-Page Applications (Vite, Webpack, Rspack) as well as framework-free Vanilla JavaScript/TypeScript applications.

---

## 1. React + Vite Setup

### Step 1: Register `askdepthVitePlugin` (`vite.config.ts`)

`@askdepth/react/plugin` exports `askdepthVitePlugin` (built on `unplugin`). During build, it instruments JSX elements with `data-askdepth-id="cmp_xxxxxxxx"` (production) or `data-askdepth-src="file:line:col"` (development) and writes `.askdepth/component-maps/<buildId>.manifest.json`.

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { askdepthVitePlugin } from '@askdepth/react/plugin';

export default defineConfig({
  plugins: [
    react(),
    askdepthVitePlugin({
      buildId: process.env.VITE_ASKDEPTH_BUILD_ID || process.env.GITHUB_SHA?.slice(0, 12),
      repository: process.env.ASKDEPTH_REPOSITORY || 'your-org/your-repo',
      environment: process.env.NODE_ENV,
    }),
  ],
});
```

### Step 2: Mount `<AskdepthProvider>` at Entry (`src/main.tsx`)

```tsx
// src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { AskdepthProvider } from '@askdepth/react';
import App from './App';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <AskdepthProvider
      writeKey={import.meta.env.VITE_ASKDEPTH_WRITE_KEY}
      endpoint={import.meta.env.VITE_ASKDEPTH_ENDPOINT || 'https://in.askdepth.com'}
      buildId={import.meta.env.VITE_ASKDEPTH_BUILD_ID}
      environment={import.meta.env.PROD ? 'production' : 'development'}
      consent="unknown" // Call Askdepth.setConsent('granted') when user accepts cookies
      replay={true}
      allowedTracingOrigins={[window.location.origin, 'https://api.yourdomain.com']}
    >
      <App />
    </AskdepthProvider>
  </React.StrictMode>
);
```

---

## 2. Webpack & Rspack Setup

`@askdepth/react/plugin` also provides native plugins for Webpack 5 and Rspack:

```javascript
// webpack.config.js (or rspack.config.js)
import { askdepthWebpackPlugin, askdepthRspackPlugin } from '@askdepth/react/plugin';

export default {
  // ...
  plugins: [
    // Use askdepthWebpackPlugin for Webpack, or askdepthRspackPlugin for Rspack:
    askdepthWebpackPlugin({
      buildId: process.env.ASKDEPTH_BUILD_ID,
      repository: 'your-org/your-repo',
      commitSha: process.env.GITHUB_SHA,
    }),
  ],
};
```

---

## 3. Vanilla JavaScript / TypeScript (Without React)

For applications using Vue, Svelte, Angular, Astro, or plain HTML/JS, install `@askdepth/core` and `@askdepth/replay` directly.

```typescript
import { Askdepth } from '@askdepth/core';

// 1. Initialize the runtime
Askdepth.init({
  writeKey: 'ad_bw_your_publishable_write_key',
  endpoint: 'https://in.askdepth.com',
  environment: 'production',
  buildId: 'release-2026-11-15',
  consent: 'unknown', // Starts paused until consent is granted
  sampleRate: 1.0,
  replay: {
    checkoutEveryNms: 45_000, // Full DOM snapshot every 45s
  },
  allowedTracingOrigins: ['https://api.example.com'],
  allowRapidClickSelectors: ['.qty-stepper', '.carousel-next'],
});

// 2. Subscribe to consent changes (optional)
const unsubscribe = Askdepth.onConsentChange((state) => {
  console.log('Askdepth consent state changed to:', state);
});

// 3. Connect to your Cookie / CMP Banner
document.querySelector('#cookie-accept')?.addEventListener('click', () => {
  Askdepth.setConsent('granted');
});

document.querySelector('#cookie-decline')?.addEventListener('click', () => {
  Askdepth.revokeConsent(); // Alias for Askdepth.setConsent('denied')
});

// 4. Identify authenticated user (Zero-PII: database externalId only)
Askdepth.identify('usr_948102', {
  plan: 'enterprise',
});

// 5. Track custom product/funnel events
Askdepth.track('subscription_upgraded', {
  from_plan: 'starter',
  to_plan: 'enterprise',
  billing_interval: 'annual',
});
```

---

## 4. Custom Component Resolver (Non-React Frameworks)

If you use `@askdepth/core` without `@askdepth/react`, you can still map DOM elements to your framework's components (e.g., Vue or Svelte component instances) on frustration events by registering a custom component resolver:

```typescript
import { Askdepth } from '@askdepth/core';

const unregister = Askdepth.registerComponentResolver((target: Element) => {
  const customEl = target.closest('[data-component-name]');
  if (!customEl) return null;

  return {
    componentName: customEl.getAttribute('data-component-name') || 'Unknown',
    componentId: customEl.getAttribute('data-askdepth-id') || undefined,
    sourceLocation: customEl.getAttribute('data-askdepth-src') || undefined,
    componentStack: [],
  };
});
```

---

## 5. Server-Side Rendering (SSR) Safety Invariants

All `@askdepth/*` packages are strictly verified for Node.js, Deno, Bun, and Edge SSR environments:

- Importing `@askdepth/core`, `@askdepth/replay`, or `@askdepth/react` on the server never touches `window`, `document`, or `navigator` at module load time.
- Calling `Askdepth.init()`, `Askdepth.track()`, `Askdepth.identify()`, or `Askdepth.setConsent()` when `typeof window === 'undefined'` is a safe, synchronous no-op.
- `Askdepth.getTraceparent()` and `Askdepth.getSessionId()` return `null` during SSR.
- `Askdepth.isInitialized()` returns `false` during SSR.
