# Askdepth Agent Skills (`askdepth/skills`)

Official AI coding agent skills for the **Askdepth** ecosystem. These skills teach coding agents (Claude Code, Cursor, Codex, Antigravity, Windsurf, and `npx skills`-compatible agents) how to install, configure, instrument, and verify Askdepth Browser/React Telemetry SDKs and the Askdepth Audience Connector.

---

## Quick Install

Install all Askdepth skills into your project with a single command:

```bash
npx skills add askdepth/skills
```

Or install a specific skill:

```bash
# Browser, React & Next.js Telemetry + Session Replay SDK
npx skills add askdepth/skills --skill askdepth-sdk

# Zero-Data-Egress Audience Connector (PostgreSQL, REST, Express, Next.js, Fastify, Lambda)
npx skills add askdepth/skills --skill audience-connector
```

You can also install directly from individual package repositories:

```bash
npx skills add askdepth/sdk
npx skills add askdepth/audience-connector
```

---

## Included Skills

| Skill | Directory | Packages Covered | When It Activates |
| :--- | :--- | :--- | :--- |
| **[`askdepth-sdk`](./askdepth-sdk/SKILL.md)** | `askdepth-sdk/` | `@askdepth/react`<br>`@askdepth/core`<br>`@askdepth/replay`<br>`@askdepth/contracts` | Installing or configuring Askdepth browser telemetry, `<AskdepthProvider>`, `useAskdepth()`, `<AskdepthCatch>`, Next.js `withAskdepth` (Webpack & Turbopack), Vite/Webpack/Rspack AST plugins, `askdepth-upload-map` CLI, Zero-PII session replay masking (`data-askdepth-mask`, `data-askdepth-block`, `data-askdepth-allow-rapid`), rage/dead/error click detection, W3C `traceparent` propagation, and `askdepth.identify()` Zero-PII user bridging. |
| **[`audience-connector`](./audience-connector/SKILL.md)** | `audience-connector/` | `@askdepth/audience-connector`<br>`@askdepth/audience-contract` | Setting up or debugging `/askdepth/v1` endpoints, `createConnector`, `postgresAdapter`, `restAdapter`, framework shims (`expressHandler`, `fastifyPlugin`, `lambdaHandler`, Next.js App Router, Cloudflare Workers), HMAC-SHA256 request verification (`x-askdepth-signature`, `x-askdepth-timestamp`), canonical field & filterable/returnable attribute mapping, and the 15-case `audience-connector conformance` CLI suite. |

---

## Repository Structure

Each skill uses **progressive disclosure** so agents keep a lean context window while loading deep framework templates and API references on demand:

```text
askdepth/skills
├── README.md
├── askdepth-sdk/
│   ├── SKILL.md
│   └── references/
│       ├── nextjs-app-router.md
│       ├── vite-and-vanilla.md
│       ├── privacy-replay-and-heuristics.md
│       └── api-and-cli.md
└── audience-connector/
    ├── SKILL.md
    └── references/
        ├── postgres-and-frameworks.md
        ├── rest-adapter-and-contract.md
        └── conformance-and-security.md
```

---

## Supported Agents

- **Claude Code** (`.claude/skills/`)
- **Cursor** (`.cursor/rules/` / `.agents/skills/`)
- **OpenAI Codex** (`.codex/skills/` / `.agents/skills/`)
- **Google Antigravity** (`.agents/skills/`)
- Any agent supporting the [`skills.sh`](https://skills.sh) standard
