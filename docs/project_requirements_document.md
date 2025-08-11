# Product Requirements Document — Bookmark AI MVP

## Overview

**Problem** Users paste links to articles and social posts, then cannot easily recall the gist or find them later. Traditional bookmarks lack summaries, keyword/semantic search, and reliable click tracking, creating friction between saving and re‑finding.

**Target users / JTBD** People who routinely collect links (students, knowledge workers, creators) and need **fast save → summarize → search → open** with minimal setup. Primary jobs: (1) capture a URL, (2) get a short neutral summary in English for quick recall, (3) find items via keyword or semantic search, (4) track which items they actually open.

**Positioning** Bookmark AI is a paid, recall‑first bookmarking app: paste a link, get an English summary in seconds, and find it later via smart search—no clutter, just signal.

## Goals & Non‑goals

**Top goals (MVP)**

1. Ship a stable paid‑only dashboard that ingests a URL → summary in ≤ **P50 8s / P90 15s** end‑to‑end.
2. Achieve ≥ **90% metadata fetch success** (excluding remote site blocks) and clear, retriable error paths.
3. Prove search value: **≥25% AI‑Search CTR** on search result clicks and track via `/r/:id`.

**Non‑goals**

- No transcripts pipeline, bulk import, browser extension, advanced filters, or multilingual summaries.
- No provider swaps; template pins win for all packages.

## User Stories (with acceptance notes)

1. **Add URL & auto summarize** — As a user, I paste a URL and press Enter.

   - Accept: Creates an optimistic card (“Fetching… → Summarizing…”), extracts metadata (title/description/thumbnail), runs the locked summary prompt (≤300 chars, English), finalizes the card with title, date, and thumbnail. If core metadata missing: card shows **Error** chip + **Retry**.

2. **Retry failures** — As a user, I can retry when something fails.

   - Accept: If metadata failed → rerun full pipeline; if summary failed → rerun summary + embed only. Exponential backoff 0.5s→1s→2s→4s up to 4 tries.

3. **Keyword search** — As a user, I can instantly search by text.

   - Accept: Queries Convex‑indexed fields (title, authorHandle?, description?, summary, transcript?) and returns results instantly for my account only.

4. **AI Search (semantic)** — As a user, I can click **AI Search** for semantic recall.

   - Accept: Server embeds query, queries Qdrant (`bookmarks_primary`) with **userId** filter, takes top‑64 candidates, merges with keyword hits, runs **MMR λ=0.7**, returns items with **cosine ≥0.22 OR ≥70% of the top hit**. No fixed cap.

5. **Edit details** — As a user, I can edit title and summary in a side drawer.

   - Accept: Drawer opens from card click; Title + Summary editable. **Cmd/Ctrl+S** saves; **Esc** closes. Success/failure toasts.

6. **Click tracking** — As a user, when I open a bookmarked link it’s counted.

   - Accept: Link opens via server redirect `/r/:id` that increments `clickCount` then 302s to the normalized URL. Dashboard default sort: `clickCount desc → createdAt desc`.

7. **Billing gate** — As a user without payment, I’m blocked from dashboard features.

   - Accept: Auth via Clerk; entitlements from Polar. Paid‑only dashboard; webhook updates entitlement. Unpaid users see pricing and are redirected to subscribe.

8. **Rate limit** — As a heavy user, I cannot exceed daily summary quota.

   - Accept: Hard fails at **100 summaries/user/day** resetting at midnight UTC with `rate_limited` errorReason; UI shows Error chip + Retry semantics.

## Scope & Constraints (must copy as‑is)

**Scope (MVP):** Web app with landing + auth/billing gating. Paid-only dashboard lets users paste URLs → auto-fetch metadata → auto-summarize (≤300 chars, English only) → search (keyword + optional semantic). Grid cards with a side-drawer for edits. Click tracking. **No transcripts pipeline** in MVP. **Absolute rule:** Use the SaaS template’s package versions exactly as pinned in its `package.json`. If any conflict arises, template pins win. **Architecture:** Frontend = React + Vite, React Router v7 (SSR), TS, Tailwind, shadcn/ui, Lucide. Backend = **Convex** (db + server functions + http routes), **Clerk** (auth), **Polar** (billing/entitlements), **OpenAI (GPT-5)** for summaries via Convex actions, **Qdrant (managed)** for vector search. Deploy on **Vercel**. **Env & secrets (names locked):** `CONVEX_DEPLOYMENT`, `VITE_CONVEX_URL`, `NEXT_PUBLIC_CONVEX_URL`, `VITE_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, `FRONTEND_URL`, `OPENAI_API_KEY`, `POLAR_*`, `QDRANT_URL`, `QDRANT_API_KEY`. **Billing webhook route (Convex HTTP):** `POST /payments/webhook` (Polar must call this). **Click tracking:** Open links via server redirect `/r/:id` that increments `clickCount` then 302s to the normalized URL. Default sort: `clickCount desc → createdAt desc`. **Convex data model — bookmarks (final):**

```ts
{
  _id: Id<"bookmarks">,
  userId: string,
  url: string,
  source: "instagram" | "other",
  authorHandle?: string,
  caption?: string,
  description?: string,
  hashtags?: string[],
  transcriptRaw?: string, // not used in MVP
  title: string,              // "URL" if error
  summary?: string,           // ≤300 chars
  thumbnailUrl?: string,
  tags?: string[],            // keep; not editable in MVP
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

**URL normalization before dedupe:** lowercase host; remove default ports; strip `utm_*`, `fbclid`; remove fragments; collapse multi‑slashes; drop trailing slash (except root); keep non‑tracker query params. **Ingestion pipeline:**

1. Create optimistic bookmark (`status:"ok"`, title “Fetching…”, empty summary).
2. Fetch metadata (JSDOM). Prefer `og:title` → `twitter:title` → `<title>`. Description: `og:description` → `twitter:description` → meta description. Thumbnail: `og:image`/`twitter:image` → high‑res favicon → server screenshot → placeholder.
3. If core metadata missing: set `status:"error"`, `title=URL`, show Error chip + **Retry**.
4. Summarize with OpenAI (locked prompt):
   > “Summarize the page neutrally in ≤300 characters for quick recall. Avoid hype, emojis, and commands. Write in clear English. If content is a video, capture the main idea, not timestamps. Return only the summary text, no quotes.” Enforce **≤300 Unicode graphemes**, English‑only.
5. Embedding (1536‑d) only if **title exists**. Embed composed text (title + IG caption or meta description; fallback: first \~300 words) and **upsert to Qdrant** `bookmarks_primary`.
6. Persist updates (including `embVersion`, `primaryVectorId`).
7. Finalize optimistic card (Fetching→Summarizing→Final). **Retry:** if metadata failed → full pipeline; if summary failed → summary+embed only.\
   **Limits:** 100 summaries/user/day, resets midnight UTC.\
   **Errors:** `fetch_failed`, `no_metadata`, `model_error`, `timeout`, `rate_limited`, `policy_blocked`, `unknown`.\
   **Backoff:** 0.5s → 1s → 2s → 4s (max 4 tries). **Search:**

- Instant keyword search (Convex) over: `title`, `authorHandle?`, `description?`, `summary`, `transcript?` (tolerate missing).
- **AI Search button:** query‑embed → Qdrant `bookmarks_primary` (cosine, size=1536), filter by `userId`, take top‑64 candidates, merge with keyword hits, **MMR λ=0.7**, then return items with **cosine ≥0.22 OR ≥70% of top hit**. No fixed cap.
- Centralize tunables in `lib/config.ts` (server) + client‑safe subset. **Qdrant:** single global collection `bookmarks_primary`; HNSW: `m=32`, `ef_construct=256`; query `search_ef=96`. Payload includes `{ userId, bookmarkId, embVersion }`. Server‑only access; HTTPS + API key; strict per‑user filter. **UI behaviors:** Card grid 16:9; side‑drawer for edits; title & summary editable; **Cmd/Ctrl+S** save; **Esc** closes; toasts; optimistic states. **A11y/Hotkeys:** Focus‑visible rings; semantic landmarks; labeled inputs; **Enter** submits Add URL; **Cmd/Ctrl+K** focuses search; **Cmd/Ctrl+S** saves; **Esc** closes drawer. **Design tokens:** Typography Nunito 400/600/700; primary `#3B82F6`, accent `#F43F5E`, bg `#F9FAFB`, surface `#FFFFFF`, muted `#E5E7EB`, text `#111827`, subtext `#4B5563`, success `#22C55E`, warning `#F59E0B`, danger `#EF4444`. Radius 12px.

## Functional Details

### Auth & Billing

- Auth via Clerk (template). Paid entitlement via Polar (single monthly plan, no trial). Polar calls `POST /payments/webhook` (Convex) to update entitlements. Server verifies signatures and updates subscription state used for gating. Paid‑only dashboard.

### Data & Security

- Every Convex read/write and every Qdrant op filters by `userId`. Qdrant is server‑only; clients never receive Qdrant credentials. `/r/:id` redirects **only** to the stored normalized URL (no open redirects).

### Observability & Analytics

- Sentry (server exceptions: fetch, summarize, embed, Qdrant). PostHog: screen views, add‑URL attempts, success/failure, AI Search clicks, drawer open/save, redirect clicks. Respect `userId` scoping.

## Success Signals (MVP)

- Activation rate to paid dashboard.
- % metadata success and daily summary limit adherence.
- Ingest P50/P90 latency; AI‑Search CTR; click‑through via `/r/:id`.
- Week‑over‑week retained searchers (proxy for recall utility).

## Acceptance Criteria (execution‑ready)

- **Ingest flow**: Optimistic card, metadata extraction order, summary prompt (≤300 graphemes), embed title‑gated, Qdrant upsert, finalize animation. Error chip + Retry with backoff.
- **Search behavior**: Keyword fields as listed; AI Search uses 1536‑d embeddings, `userId` filter, top‑64 candidate pool, **MMR λ=0.7**, threshold rule (**≥0.22** or **≥70% of top**), no fixed cap. Tunables live in `lib/config.ts`.
- **Redirect tracking**: `/r/:id` increments `clickCount` server‑side and 302s to normalized URL. Default sort is `clickCount desc → createdAt desc`.
- **Auth/Billing**: Unpaid users gated from dashboard; Polar webhook routed at `POST /payments/webhook`, signature verified; entitlement enforced server‑side.
- **Rate limits**: 100 summaries/user/day resetting at midnight UTC with clear error messaging.
- **A11y/Hotkeys**: Focus‑visible rings; **Enter/Cmd+K/Cmd+S/Esc** behaviors implemented.
- **Config centralization**: All constants (search/ingest/Qdrant) centralized in `lib/config.ts` + client‑safe subset.

## Technical Snippets (to de‑risk)

**Convex redirect handler**

```ts
// convex/http.ts (sketch)
export const http = httpRouter();
http.route({ path: "/r/:id", method: "GET" }, async (req, ctx) => {
  const id = req.param("id");
  const b = await ctx.db.get("bookmarks", id);
  if (!b) return new Response("Not found", { status: 404 });
  await ctx.db.patch("bookmarks", id, { clickCount: (b.clickCount ?? 0) + 1, lastAccessedAt: Date.now() });
  return Response.redirect(b.url, 302);
});
```

**Qdrant query wrapper**

```ts
// server/search.ts
const RESULTS = await qdrant.search({
  collection: "bookmarks_primary",
  vector: queryEmbedding,
  filter: { must: [{ key: "userId", match: { value: userId } }] },
  limit: 64, // candidate pool
  params: { ef: 96 },
});
```

**Threshold rule & MMR**

```ts
const top = RESULTS[0]?.score ?? 1;
const cutoff = Math.max(0.22, 0.70 * top);
const pool = RESULTS.filter(r => r.score >= cutoff);
const ranked = mmr(pool, queryEmbedding, { lambda: 0.7 });
```

**Config constants**

```ts
// lib/config.ts
export const SEARCH = { VECTOR_DIM: 1536, HNSW_M: 32, HNSW_EF_CONSTRUCT: 256, SEARCH_EF: 96, MMR_LAMBDA: 0.7, CANDIDATE_POOL: 64, FINAL_CUTOFF: 0.22, RELATIVE_TOP_RATIO: 0.70 };
export const INGEST = { SUMMARY_MAX_CHARS: 300, REQUIRE_TITLE_FOR_EMBED: true, BACKOFF_STEPS_MS: [500,1000,2000,4000], DAILY_SUMMARY_LIMIT: 100, DAILY_RESET_TZ: 'UTC' };
export const QDRANT = { COLLECTION_PRIMARY: 'bookmarks_primary' };
```

**URL normalization**

```ts
export function normalizeUrl(raw: string): string {
  const u = new URL(raw);
  u.host = u.host.toLowerCase();
  if ((u.protocol === 'http:' && u.port === '80') || (u.protocol === 'https:' && u.port === '443')) u.port = '';
  // strip trackers
  const params = new URLSearchParams(u.search);
  [...params.keys()].forEach(k => { if (k.startsWith('utm_') || k === 'fbclid') params.delete(k); });
  u.search = params.toString() ? `?${params.toString()}` : '';
  // remove fragment
  u.hash = '';
  // collapse slashes & drop trailing slash (except root)
  u.pathname = u.pathname.replace(/\\/+/, '/');
  if (u.pathname !== '/' && u.pathname.endsWith('/')) u.pathname = u.pathname.slice(0, -1);
  return u.toString();
}
```

---

**This PRD mirrors the LOCKS exactly and is implementation‑ready for the MVP.**

