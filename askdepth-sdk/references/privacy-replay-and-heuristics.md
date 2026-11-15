# Privacy, Session Replay & Frustration Heuristics Reference

This reference details how `@askdepth/core` and `@askdepth/replay` detect user frustration, correlate backend/JS errors, and enforce strict Zero-PII guarantees on the client before any byte leaves the browser.

---

## 1. Frustration Heuristic Detectors (`@askdepth/core`)

When `consent` is `'granted'`, `@askdepth/core` activates three lightweight DOM and network heuristics:

### 1.1 Rage Click (`RAGE_CLICK`)
- **Trigger Rule**: Fires when a user performs **$\ge 3$ clicks within $1500\text{ ms}$** where all pairwise click coordinates fall within a **$30\text{ px}$ radius** (evaluated over a trailing window of 5 clicks).
- **Automatic Exemptions**:
  - Elements matching `[data-askdepth-allow-rapid]` (or inside a parent with `data-askdepth-allow-rapid`).
  - Native `<input type="number">` and `[role="spinbutton"]` elements, plus adjacent stepper sibling buttons.
  - Any CSS selector configured in `allowRapidClickSelectors` passed to `Askdepth.init` / `<AskdepthProvider>`.
- **Best Practice**: Always add `data-askdepth-allow-rapid=""` to cart quantity `+` / `-` buttons, pagination next/prev buttons, and image gallery arrows:
  ```tsx
  <button type="button" data-askdepth-allow-rapid="" onClick={incrementQty}>
    +
  </button>
  ```

### 1.2 Dead Click (`DEAD_CLICK`)
- **Target Elements**: Interactive elements (`a[href]`, `button`, `input`, `select`, `textarea`, `[role="button"]`, `[tabindex]`, or elements with computed `cursor: pointer`), excluding focused text inputs (`isOpenTextInput`).
- **Trigger Rule**: Arms an **$800\text{ ms}$ observation window** after a click. If **zero** DOM mutations occur, **zero** network requests (`fetch`/`XHR`) are initiated, **zero** route transitions occur, and **zero** text selections are made within $800\text{ ms}$, emits `DEAD_CLICK` with `observed_duration_ms: 800`.

### 1.3 Error Click (`ERROR_CLICK`)
Correlates user clicks with client-side exceptions or failing backend calls:
1. **JavaScript Exception Correlation ($500\text{ ms}$ window)**:
   - Listens to `window.onerror`, `unhandledrejection`, and `<AskdepthCatch>` (`Askdepth.reportCaughtError`).
   - If an error occurs within **$500\text{ ms}$** of a user click, emits `ERROR_CLICK` (`error_type: 'js_exception'`).
   - **Wire Sanitization**: Raw exception messages are sanitized on the wire to prevent PII leakage in error strings; call stacks are trimmed to at most **8 frames** ($\le 1200$ chars) with URL query parameters stripped, accompanied by a deterministic SHA-256 stack fingerprint.
2. **Network Failure Correlation ($10,000\text{ ms}$ window)**:
   - Intercepts `fetch` and `XMLHttpRequest`.
   - If a request initiated after a click completes with **HTTP status `0` (network failure), `4xx`, or `5xx`** within **$10,000\text{ ms}$**, emits `ERROR_CLICK` (`error_type: 'network_error'`), attaching the HTTP status code, stripped URL path, and correlated W3C `traceparent`.

---

## 2. W3C Distributed Tracing (`traceparent`)

When `allowedTracingOrigins` is configured on `Askdepth.init` or `<AskdepthProvider>`:

```tsx
<AskdepthProvider
  writeKey={writeKey}
  allowedTracingOrigins={[
    typeof window !== 'undefined' ? window.location.origin : '',
    'https://api.yourdomain.com',
  ].filter(Boolean)}
>
```

- Every outbound `fetch` or `XMLHttpRequest` matching an allowed origin automatically receives a W3C `traceparent` header (`00-<traceId>-<spanId>-01`).
- You can also read the active header manually (e.g., for WebSocket handshakes or custom GraphQL clients) via `Askdepth.getTraceparent()` or `const { getTraceparent } = useAskdepth()`.

---

## 3. Zero-PII Session Replay & DOM Sanitization (`@askdepth/replay`)

### 3.1 Rolling 45-Second Ring Buffer & Anomaly Flash
- `@askdepth/replay` does **not** continuously stream every user session to the server.
- Instead, it maintains an in-memory rolling **45-second FIFO ring buffer** with periodic `FullSnapshot` checkpoints (`checkoutEveryNms: 45000`).
- When a high-priority anomaly occurs (`RAGE_CLICK`, `DEAD_CLICK`, `ERROR_CLICK`, or `<AskdepthCatch>`), the engine calls `flash(anomalyId, triggerTimestamp)`, compressing the trailing 45-second slice with Gzip/Deflate (`fflate`), splitting it into **$\le 45\text{ KiB}$ binary chunks** (staying safely beneath the browser's 64 KiB `keepalive: true` quota after Base64 + JSON envelope wrapping), and uploading it to `${endpoint}/v1/replays/upload`.

### 3.2 DOM Privacy Attributes

| Attribute / CSS Class | Effect on Recorded DOM & Replay | Typical Use Cases |
| :--- | :--- | :--- |
| `data-askdepth-mask=""`<br>`.askdepth-mask`<br>`data-rr-mask=""` | Replaces all descendant text characters with `*` preserving length. | Customer shipping/billing address forms, user profile headers, order history details. |
| `data-askdepth-block=""`<br>`.askdepth-block`<br>`data-rr-block=""` | **Complete Subtree Exclusion**: Drops all child nodes (`childNodes = []`) and strips all attributes except `data-askdepth-block`, `width`, `height`, `rr_width`, `rr_height` (purges `id`, `class`, `style`, `src`, `href`, `data-*`, `aria-*`). | Payment card forms, 1-click login credentials, admin/debug drawers, experiment cohort badges. |
| `<input>`, `<textarea>`, `<select>` | Masked with asterisks (`*`) by default. Password and payment card inputs are unconditionally blocked. | All standard HTML form controls. |
| `[contenteditable]` | Tracked statefully by `EventSanitizer`; incremental text mutations are masked to `*`. | Rich-text editors, comment boxes, support chat inputs. |

### 3.3 Automatic In-Stream Regex Scrubbing (`maskText`)
Even outside `data-askdepth-mask` containers, `@askdepth/replay` scrubs text nodes and URLs in real time before inserting events into the ring buffer:
- **JWTs** (`eyJ...`) $\to$ `[REDACTED_JWT]`
- **Bearer Tokens** (`Bearer ...`) $\to$ `Bearer [REDACTED_TOKEN]`
- **API Keys / Secrets** (`api_key=...`, `client_secret=...`) $\to$ `[REDACTED_SECRET]`
- **Payment Cards** (13–19 digits passing Luhn check or prefixed with `card:`/`cc:`/`pan:`) $\to$ `[REDACTED_CARD]`
- **Emails / Phones / Passports / SSNs** $\to$ `[REDACTED_EMAIL]`, `[REDACTED_PHONE]`, `[REDACTED_DOCUMENT]`
- **Sensitive URL Query Params** (`token`, `secret`, `email`, `password`, `key`, `cvv`, `auth`) $\to$ stripped from URLs and CSS `url(...)` rules.

---

## 4. Browser Resource Limits & Remote Kill-Switch

### 4.1 Safari / WebKit Low-Memory Protection
- Safari/iOS aggressively terminates tabs (jetsam) under memory pressure.
- `@askdepth/replay` automatically clamps the ring buffer memory budget to **1.5 MB** on WebKit/iOS/Private Browsing (compared to **3.0 MB** on desktop Chromium).
- If memory pressure exceeds thresholds, high-frequency pointer (`mousemove`, `touchmove`) and scroll samples are shed first while preserving structural DOM mutations.

### 4.2 Transport Batching & Unload Beaconing
- Telemetry batches are capped at **50 events** or **64 KiB**, flushing every **2000 ms**, when queue length hits **10 events**, or immediately on high-priority anomalies.
- On `pagehide` / `visibilitychange: hidden`, the SDK flushes via `fetch(..., { keepalive: true })` rather than `navigator.sendBeacon` because `sendBeacon` cannot attach the required `x-askdepth-write-key` authentication header.

### 4.3 Remote Kill-Switch
If the Askdepth collector responds with:
- HTTP status **`410 Gone`**,
- Response header **`x-askdepth-kill: true`**, or
- JSON body **`{ "kill": true }`**,

the SDK immediately executes an emergency client-side shutdown:
1. Removes all global DOM listeners (`pointer`, `click`, `scroll`, `error`, `unhandledrejection`, `popstate`, `hashchange`).
2. Disconnects mutation observers and frees the replay ring buffer and transport queues.
3. Turns all subsequent `Askdepth.*` calls into zero-overhead no-ops for the lifetime of the page.
