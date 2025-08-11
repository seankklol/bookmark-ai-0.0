# Tech Stack & Dependencies — Bookmark AI (LOCKED)

> **Contract:** This document is enforceable. Items marked **LOCKED** must not be altered. *Template pins win.*

---

## Overview

**Diagram-level flow (textual):**

- **Client (React Router v7 + Vite + TS + Tailwind + shadcn/ui + Lucide)** → calls **Convex** queries/mutations/actions.
- **Convex (server)** →
  - Auth via **Clerk** (server-side entitlement checks).
  - Billing/entitlements via **Polar** (webhook → `POST /payments/webhook`).
  - Summaries & embeddings via **OpenAI (GPT-5 + text-embedding-3-small)**.
  - Vector search via **Qdrant** (HTTP API) called only from Convex.
- **Redirect service (Convex HTTP)** → `/r/:id` increments `clickCount` then **302** to normalized URL.
- **Deploy target:** **Vercel** with `@vercel/react-router` preset, SSR enabled.

**Scope guard (LOCKED):** No provider swaps; no extra services (e.g., Prisma/Supabase). No transcript pipeline in MVP.

---

## Dependencies (pinned to the SaaS template)

> **Rule:** Do **not** change versions. Inherit exactly from the template’s `package.json`.

**Core runtime & framework**

- `react`, `react-dom` (React 19)
- `react-router` **v7** (+ `@react-router/dev`, `@react-router/node`, `@react-router/serve`) — **SSR enabled**
- `vite` (build), `typescript`

**Styling & UI**

- `tailwindcss` **v4**, `@tailwindcss/vite`
- `shadcn/ui` (via generated components under `app/components/ui`)
- `lucide-react`, `@tabler/icons-react` (icons)
- `clsx`, `class-variance-authority`, `tailwind-merge`

**Backend & platform**

- `convex` (DB, server functions, HTTP routes)
- `@vercel/react-router` (Vercel preset)

**Auth & billing**

- `@clerk/react-router` (auth integration)
- `@polar-sh/sdk` (Polar API)
- `@convex-dev/polar` (Convex ↔ Polar helpers)

**AI**

- `@ai-sdk/openai`, `ai` (Vercel AI SDK) — **use for GPT-5 summaries & embeddings**

**Data viz & motion**

- `recharts`, `motion`

**Analytics**

- `@vercel/analytics` (**present**). **Sentry/PostHog**: referenced in project plan but **not installed by template** → **do not add** in MVP. Use Vercel Analytics only.

**Testing**

- Plan references **Playwright** and **Vitest**; **not present** in template → **do not add** until the template pins them. (Tests can be drafted; installation deferred.)

**Qdrant client**

- Preferred: `@qdrant/js-client-rest`. **If not in template, do not install.** Use a thin `fetch` wrapper against Qdrant HTTP until a pin is introduced.

---

## Build & Deploy

- **SSR** via React Router v7 (`react-router.config.ts`: `ssr: true`, `presets: [vercelPreset()]`).
- **Vercel** build outputs to `build/` with server entry `./build/server/index.js` served by `react-router-serve`.
- **Environment provisioning** done in Vercel + Convex project settings. **Never** bake secrets into client bundles.

---

## Environment Matrix

### Client-exposed (safe)

- `VITE_CONVEX_URL`
- `NEXT_PUBLIC_CONVEX_URL` (mirror of `VITE_CONVEX_URL`)
- `VITE_CLERK_PUBLISHABLE_KEY`

> Only expose minimal configuration strictly required by client SDKs.

### Server-only (secrets)

- `CONVEX_DEPLOYMENT`
- `CLERK_SECRET_KEY`
- `OPENAI_API_KEY`
- `POLAR_ACCESS_TOKEN`
- `POLAR_ORGANIZATION_ID`
- `POLAR_WEBHOOK_SECRET`
- `QDRANT_URL`
- `QDRANT_API_KEY`
- `FRONTEND_URL` (used by Convex HTTP to construct redirects and CORS)

**Secret handling policy (LOCKED):** Secrets live only in Convex/Vercel secret stores. All secret usage runs inside Convex actions/HTTP routes. **Never** read secrets in the client.

---

## Data & Search (LOCKED)

**Primary DB:** **Convex**

**Bookmarks schema (Convex)**

```ts
// Table: bookmarks
{
  _id: Id<"bookmarks">,
  userId: string,
  url: string,
  source: "instagram" | "other",
  authorHandle?: string,
  caption?: string,
  description?: string,
  transcriptRaw?: string, // not used in MVP
  title: string,              // "URL" if error
  summary?: string,           // ≤300 chars, English
  thumbnailUrl?: string,
  tags?: string[],            // kept; not editable in MVP
  status: "ok" | "error",
  errorReason?: string,
  createdAt: number,
  updatedAt: number,
  clickCount: number,
  lastAccessedAt?: number,
  embVersion: number,
  primaryVectorId?: string,
  chunkVectorIds?: string[],
}
```

**Vector store:** **Qdrant** (managed)

- Single collection: `bookmarks_primary`
- Vector size **1536**, distance **cosine**
- **HNSW**: `m=32`, `ef_construct=256`; query-time `search_ef=96`
- Payload: `{ userId: string, bookmarkId: string, embVersion: number, createdAt?: number }`
- **Isolation**: **Every** upsert/search is filtered by `userId`. No unfiltered queries.

**Search behavior (LOCKED)**

- **Keyword** (Convex) on `title`, `authorHandle?`, `description?`, `summary`
- **AI Search** button → Embedding query against Qdrant → take **top 64** candidates → merge with keyword hits → **MMR λ=0.7** → return items with **cosine ≥ 0.22** *or* **≥ 70% of top hit**, whichever is higher. No fixed cap.

---

## Service Responsibilities (LOCKED)

- **Clerk** → Authentication; server-side entitlement checks
- **Polar** → Billing & entitlements; webhook into `` (Convex HTTP)
- **OpenAI (GPT-5)** → Summaries (≤300 chars, English-only)
- **OpenAI (text-embedding-3-small)** → 1536‑d embeddings
- **Qdrant** → Vector search (via Convex-only HTTP)
- **Convex HTTP routes** → `POST /payments/webhook` and redirect ``

---

## Locked Constraints (copy exactly)

- **Architecture**: React+Vite+TS+Tailwind+shadcn+Lucide; Convex; Clerk; Polar; OpenAI; Qdrant; Vercel. **Template pins win.**
- **Env names**: as listed in **Environment Matrix** above (including both `VITE_CONVEX_URL` and `NEXT_PUBLIC_CONVEX_URL`).
- **Webhook path**: `POST /payments/webhook` (Convex HTTP). Polar must target this exact path.
- **Redirect rule**: `/r/:id` increments `clickCount` server-side then 302 to stored normalized URL.
- **Normalization & Ingestion pipeline**:
  - Normalize URL (lowercase host, strip trackers `utm_*`, `fbclid`, remove fragments, collapse slashes, drop trailing slash except root).
  - Optimistic card → Metadata (JSDOM) → **Summary** (prompt below) → **Embed** (1536-d) → Qdrant upsert → finalize.
  - **Summary prompt (LOCKED):**
    > “Summarize the page neutrally in ≤300 characters for quick recall. Avoid hype, emojis, and commands. Write in clear English. If content is a video, capture the main idea, not timestamps. Return only the summary text, no quotes.”
  - Enforce Unicode‑grapheme ≤300; daily summary cap **100/user**, reset **midnight UTC**; backoff **0.5s→1s→2s→4s**.
- **Search behavior**: top‑64 candidate pool, **MMR λ=0.7**, cutoffs **cosine ≥ 0.22** or **≥70% of top hit**.
- **UI behaviors**: Grid cards 16:9, side‑drawer for edits (Cmd/Ctrl+S saves; Esc closes), optimistic states, error chip + Retry.
- **A11y/Hotkeys**: focus-visible rings; **Enter** submits Add URL; **Cmd/Ctrl+K** focuses search; **Cmd/Ctrl+S** saves; **Esc** closes drawer.
- **Design tokens**: Nunito 400/600/700; warm/playful palette; radius 12px; focus ring 2px primary; skeleton shimmer \~1.2s.

---

## Risks & Mitigations

- **Rate limits / timeouts (OpenAI & Qdrant)** → Centralized backoff + retry; cap summaries 100/day; surface degradation toast on bursts of failures.
- **Summary length creep** → Enforce grapheme-safe truncation before persist; validate server-side.
- **Vector upsert consistency** → Upsert only after successful summary; store `embVersion`; on title edit, re-embed and upsert.
- **Webhook spoofing** → Verify Polar signature with `POLAR_WEBHOOK_SECRET`; accept only expected event types; 2xx only on success.
- **Cross-tenant leakage** → Mandatory `userId` filter in **every** Convex read & Qdrant query. Typed wrappers disallow missing filter.
- **Open redirect risk** → `/r/:id` resolves only stored normalized URL; reject external input.
- **Secret exposure** → No secrets in client; only server-side Convex code can read secrets.

---

## Minimal Setup Stubs (TypeScript)

### 1) Env typing guards (server-side)

```ts
// convex/env.ts
const req = (k: string) => {
  const v = process.env[k];
  if (!v) throw new Error(`Missing required env: ${k}`);
  return v;
};
export const ENV = {
  OPENAI_API_KEY: req('OPENAI_API_KEY'),
  QDRANT_URL: req('QDRANT_URL'),
  QDRANT_API_KEY: req('QDRANT_API_KEY'),
  POLAR_WEBHOOK_SECRET: req('POLAR_WEBHOOK_SECRET'),
  FRONTEND_URL: req('FRONTEND_URL'),
};
```

### 2) OpenAI summary action (Convex)

```ts
// convex/summarize.ts
import { action } from "./_generated/server";
import { createOpenAI } from "@ai-sdk/openai";
import { generateText } from "ai";
import { v } from "convex/values";

const openai = createOpenAI({ apiKey: process.env.OPENAI_API_KEY! });

export const summarize = action({
  args: { text: v.string() },
  handler: async (_ctx, { text }) => {
    const prompt =
      "Summarize the page neutrally in ≤300 characters for quick recall. " +
      "Avoid hype, emojis, and commands. Write in clear English. " +
      "If content is a video, capture the main idea, not timestamps. " +
      "Return only the summary text, no quotes.";

    const { text: out } = await generateText({
      model: openai("gpt-5"),
      prompt: `${prompt}\n\nCONTENT:\n${text}`,
    });

    // Unicode-grapheme-safe trim to 300 chars (fallback: slice)
    return out.length > 300 ? [...out].slice(0, 300).join("") : out;
  },
});
```

### 3) Qdrant HTTP wrapper (no client lib; respects template pins)

```ts
// convex/qdrant.ts
interface Point {
  id: string;
  vector: number[]; // 1536-d
  payload: { userId: string; bookmarkId: string; embVersion: number; createdAt?: number };
}

const Q = {
  base: process.env.QDRANT_URL!,
  key: process.env.QDRANT_API_KEY!,
  coll: 'bookmarks_primary',
};

export async function upsertPoints(points: Point[]) {
  const res = await fetch(`${Q.base}/collections/${Q.coll}/points`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json', 'api-key': Q.key },
    body: JSON.stringify({ points }),
  });
  if (!res.ok) throw new Error(`Qdrant upsert failed: ${res.status}`);
}

export async function search(userId: string, vector: number[], topK = 64) {
  const res = await fetch(`${Q.base}/collections/${Q.coll}/points/search`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'api-key': Q.key },
    body: JSON.stringify({
      vector,
      limit: topK,
      filter: { must: [{ key: 'userId', match: { value: userId } }] },
      params: { hnsw_ef: 96 },
      with_payload: true,
    }),
  });
  if (!res.ok) throw new Error(`Qdrant search failed: ${res.status}`);
  return res.json();
}
```

### 4) Convex HTTP — Polar webhook & tracked redirect

```ts
// convex/http.ts (sketch)
import { httpRouter } from "convex/server";
import { DatabaseReader, DatabaseWriter } from "./_generated/server";

const http = httpRouter();

http.route({
  path: "/payments/webhook",
  method: "POST",
  handler: async (req, ctx) => {
    const sig = req.headers.get('polar-webhook-signature') || '';
    const secret = process.env.POLAR_WEBHOOK_SECRET!;
    const body = await req.text();
    // verifySignature(body, sig, secret) — implement/borrow from @convex-dev/polar
    // On success, update entitlements/subscription rows.
    return new Response(null, { status: 204 });
  },
});

http.route({
  path: "/r/:id",
  method: "GET",
  handler: async (req, ctx) => {
    const id = ctx.params.id;
    const db = ctx.db as DatabaseWriter;
    const b = await db.get(id as any);
    if (!b) return new Response("Not found", { status: 404 });
    await db.patch(b._id, { clickCount: (b.clickCount ?? 0) + 1, lastAccessedAt: Date.now() });
    return new Response(null, { status: 302, headers: { Location: b.url } });
  },
});

export default http;
```

### 5) Central search config (constants only; no magic numbers)

```ts
// app/lib/config.ts (server-side twin as well)
export const SEARCH = {
  VECTOR_DIM: 1536,
  HNSW_M: 32,
  HNSW_EF_CONSTRUCT: 256,
  SEARCH_EF: 96,
  MMR_LAMBDA: 0.7,
  CANDIDATE_POOL: 64,
  FINAL_CUTOFF: 0.22,
  RELATIVE_TOP_RATIO: 0.70,
};

export const INGEST = {
  EMB_MODEL: 'text-embedding-3-small',
  SUMMARY_MAX_CHARS: 300,
  REQUIRE_TITLE_FOR_EMBED: true,
  BACKOFF_STEPS_MS: [500, 1000, 2000, 4000],
  DAILY_SUMMARY_LIMIT: 100,
  DAILY_RESET_TZ: 'UTC',
};

export const QDRANT = { COLLECTION_PRIMARY: 'bookmarks_primary' };
```

---

## Operational Notes

- **Always** filter by `userId` in Convex reads and Qdrant queries.
- Client never touches Qdrant/OpenAI keys; only Convex reads secrets.
- Keep `/webhook/polar` references out of code/docs; **canonical path is **``.

---

## Open Questions (provide once; does not block MVP wiring)

1. **Currency + monthly price** for pricing examples (e.g., `USD $/mo`).
2. **Product voice**: defaulting to *clean, warm, and matter‑of‑fact* (consistent with design tokens). Update if a different tone is required.

