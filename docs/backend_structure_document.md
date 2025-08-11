# Backend Structure — Bookmark AI MVP (Convex + Qdrant)

> **Scope & Authority**: Mirrors the locked Source of Truth for Bookmark AI. Convex-first backend with Clerk auth, Polar billing, OpenAI summarization, and Qdrant AI search. **Do not** change providers, models, thresholds, or routes. All tunables live in `lib/config.ts`.

---

## 1) Architecture & Contracts (Locked)

- **Stack**: React Router v7 (SSR) frontend; **Convex** for database, functions, and HTTP routes; **Clerk** for auth; **Polar** for billing; **OpenAI** for summaries/embeddings; **Qdrant (managed)** for vector search; deploy on **Vercel**.
- **HTTP Endpoints** (Convex HTTP router):
  - `POST /payments/webhook` → `billing.handleWebhook` (Polar).
  - `/r/:id` → `redirect.handle` (tracked redirect increments `clickCount` then 302 to stored normalized URL).
- **Default sort in dashboard**: `clickCount desc → createdAt desc`.
- **Search**: Keyword (Convex) + AI Search (Qdrant). Merge + **MMR λ = 0.7**. Cutoff: **cosine ≥ 0.22** or **≥ 70% of top hit**.
- **Rate limit**: **100 summaries/user/day**, reset **midnight UTC**.
- **Normalization**: Lowercase host, strip `utm_*`, `fbclid`, remove fragments, collapse slashes, remove trailing slash (except root). Store only the normalized URL.

---

## 2) Environment Variables (Names are Locked)

Server-only secrets live in Convex env:

```
CONVEX_DEPLOYMENT
VITE_CONVEX_URL
NEXT_PUBLIC_CONVEX_URL
VITE_CLERK_PUBLISHABLE_KEY
CLERK_SECRET_KEY
FRONTEND_URL
OPENAI_API_KEY
POLAR_ACCESS_TOKEN
POLAR_ORGANIZATION_ID
POLAR_WEBHOOK_SECRET
QDRANT_URL
QDRANT_API_KEY
```

---

## 3) Convex Schema (with Indexes)

> **Rule**: Always filter by `userId`. The `url` stored is **already normalized**.

```ts
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  users: defineTable({
    tokenIdentifier: v.string(), // Clerk token identifier (for getUserId mapping)
    email: v.optional(v.string()),
    name: v.optional(v.string()),
    image: v.optional(v.string()),
    // Daily summary RL window (UTC)
    summaryCount: v.optional(v.number()),
    summaryResetAt: v.optional(v.number()),
    createdAt: v.optional(v.number()),
    updatedAt: v.optional(v.number()),
  })
    .index("by_token", ["tokenIdentifier"]),

  subscriptions: defineTable({
    userId: v.string(),
    customerId: v.optional(v.string()),
    status: v.optional(v.string()), // active, past_due, canceled, etc.
    amount: v.optional(v.number()), // minor units (e.g., cents)
    currency: v.optional(v.string()), // e.g., "usd"
    currentPeriodStart: v.optional(v.number()),
    currentPeriodEnd: v.optional(v.number()),
    cancelAtPeriodEnd: v.optional(v.boolean()),
    polarId: v.optional(v.string()),
    polarPriceId: v.optional(v.string()),
    interval: v.optional(v.string()),
    metadata: v.optional(v.any()),
  })
    .index("by_user", ["userId"])
    .index("by_polar", ["polarId"]),

  // **BOOKMARKS (Locked shape)**
  bookmarks: defineTable({
    userId: v.string(),
    url: v.string(), // normalized URL
    source: v.union(v.literal("instagram"), v.literal("other")),
    authorHandle: v.optional(v.string()),
    caption: v.optional(v.string()),
    description: v.optional(v.string()),
    hashtags: v.optional(v.array(v.string())),
    transcriptRaw: v.optional(v.string()), // not used in MVP
    title: v.string(), // "URL" if error
    summary: v.optional(v.string()), // ≤300 chars
    thumbnailUrl: v.optional(v.string()),
    tags: v.optional(v.array(v.string())), // keep; not editable in MVP
    status: v.union(v.literal("ok"), v.literal("error")),
    errorReason: v.optional(v.string()),
    createdAt: v.number(),
    updatedAt: v.number(),
    clickCount: v.number(),
    lastAccessedAt: v.optional(v.number()),
    embVersion: v.number(),
    primaryVectorId: v.optional(v.string()),
    chunkVectorIds: v.optional(v.array(v.string())),
  })
    // Fast per-user queries
    .index("by_user_createdAt", ["userId", "createdAt"])
    // Support default sort (then sort in code desc/desc)
    .index("by_user_click_created", ["userId", "clickCount", "createdAt"])
    // Dedupe guard (user + normalized URL)
    .index("by_user_url", ["userId", "url"]),

  // Optional; useful for auditing webhook deliveries
  webhookEvents: defineTable({
    type: v.string(),
    polarEventId: v.string(),
    createdAt: v.number(),
    modifiedAt: v.number(),
    data: v.any(),
  })
    .index("by_type", ["type"])
    .index("by_polarEventId", ["polarEventId"]),
});
```

**Notes**

- Unique constraints are enforced at write-time (see mutations) by checking `by_user_url` before insert.
- Default dashboard sort uses `by_user_click_created` then in-memory desc ordering.

---

## 4) Server Functions (Convex) — Names & Responsibilities

### 4.1 Ingestion Pipeline (actions + mutations)

> **Stages (locked):** optimistic → fetch metadata → summarize → embed + Qdrant upsert → finalize. Backoff: 0.5s → 1s → 2s → 4s (max 4 tries).

```ts
// convex/ingest.ts
import { action, mutation, query } from "./_generated/server";
import { v } from "convex/values";
import { normalizeUrl } from "./lib/urlNormalize";
import { rateLimitCheckAndBump } from "./lib/rateLimit";
import { summarize300 } from "./lib/summarize";
import { makeEmbedding, qdrantUpsert } from "./lib/qdrant";
import { SEARCH, INGEST } from "../lib/config"; // server-side module

export const createOptimistic = mutation({
  args: { url: v.string(), source: v.union(v.literal("instagram"), v.literal("other")) },
  handler: async (ctx, args) => {
    const userId = (await ctx.auth.getUserIdentity())?.subject;
    if (!userId) throw new Error("unauthenticated");

    const url = normalizeUrl(args.url);
    const now = Date.now();

    // Dedupe: ensure no existing doc for (userId, url)
    const existing = await ctx.db
      .query("bookmarks")
      .withIndex("by_user_url", q => q.eq("userId", userId).eq("url", url))
      .first();
    if (existing) return existing._id; // no-op; return id to let UI open drawer

    const _id = await ctx.db.insert("bookmarks", {
      userId,
      url,
      source: args.source,
      title: "Fetching…",
      status: "ok",
      createdAt: now,
      updatedAt: now,
      clickCount: 0,
      embVersion: 1,
    });
    return _id;
  },
});

export const fetchMetadata = action({
  args: { bookmarkId: v.id("bookmarks") },
  handler: async (ctx, { bookmarkId }) => {
    // JSDOM/meta scraping (IG-specific selectors else OG/Twitter/meta).
    // Resolve: title, description, thumbnailUrl, authorHandle, caption.
    // On failure: set status="error", title=stored URL, errorReason.
    // (Implementation details omitted for brevity; see metadata.ts.)
  },
});

export const summarize = action({
  args: { bookmarkId: v.id("bookmarks"), title: v.string(), description: v.optional(v.string()) },
  handler: async (ctx, { bookmarkId, title, description }) => {
    await rateLimitCheckAndBump(ctx); // 100/day per user, UTC reset
    const summary = await summarize300({ title, description });
    return { bookmarkId, summary };
  },
});

export const embedUpsert = action({
  args: { bookmarkId: v.id("bookmarks"), title: v.string(), description: v.optional(v.string()) },
  handler: async (ctx, { bookmarkId, title, description }) => {
    if (INGEST.REQUIRE_TITLE_FOR_EMBED && !title?.trim()) throw new Error("title_required_for_embed");
    const text = [title, description].filter(Boolean).join("\n\n");
    const emb = await makeEmbedding(text);
    const userId = (await ctx.auth.getUserIdentity())?.subject!;
    const vectorId = await qdrantUpsert({ userId, bookmarkId: bookmarkId as unknown as string, emb });
    return { vectorId };
  },
});

export const finalize = mutation({
  args: {
    bookmarkId: v.id("bookmarks"),
    draft: v.object({
      title: v.string(),
      summary: v.optional(v.string()),
      thumbnailUrl: v.optional(v.string()),
      authorHandle: v.optional(v.string()),
      caption: v.optional(v.string()),
      description: v.optional(v.string()),
      status: v.union(v.literal("ok"), v.literal("error")),
      errorReason: v.optional(v.string()),
    }),
    primaryVectorId: v.optional(v.string()),
  },
  handler: async (ctx, { bookmarkId, draft, primaryVectorId }) => {
    const userId = (await ctx.auth.getUserIdentity())?.subject;
    if (!userId) throw new Error("unauthenticated");
    const now = Date.now();

    // Ownership check
    const doc = await ctx.db.get(bookmarkId);
    if (!doc || doc.userId !== userId) throw new Error("not_found");

    await ctx.db.patch(bookmarkId, {
      ...draft,
      updatedAt: now,
      primaryVectorId: primaryVectorId ?? doc.primaryVectorId,
    });
  },
});

export const retry = action({
  args: { bookmarkId: v.id("bookmarks"), stage: v.union(v.literal("metadata"), v.literal("summary")) },
  handler: async (ctx, { bookmarkId, stage }) => {
    // If metadata failed → re-run full pipeline
    // If summary failed → re-run summary then embed+upsert
  },
});
```

### 4.2 Search (keyword + semantic + merge/MMR)

```ts
// convex/search.ts
import { action, query } from "./_generated/server";
import { v } from "convex/values";
import { mmr } from "./lib/mmr";
import { qdrantSearch } from "./lib/qdrant";
import { SEARCH } from "../lib/config";

export const keyword = query({
  args: { q: v.string() },
  handler: async (ctx, { q }) => {
    const userId = (await ctx.auth.getUserIdentity())?.subject;
    if (!userId) return [];
    const terms = q.toLowerCase();
    // Simple contains matching across fields; index-backed scan by user then filter in memory for relevance.
    const rows = await ctx.db
      .query("bookmarks")
      .withIndex("by_user_createdAt", x => x.eq("userId", userId))
      .collect();
    return rows.filter(r => [r.title, r.authorHandle, r.description, r.summary]
      .filter(Boolean).some(s => s!.toLowerCase().includes(terms)));
  },
});

export const semantic = action({
  args: { q: v.string() },
  handler: async (ctx, { q }) => {
    const userId = (await ctx.auth.getUserIdentity())?.subject!;
    return qdrantSearch({ userId, query: q });
  },
});

export const mergeAndMMR = action({
  args: { q: v.string() },
  handler: async (ctx, { q }) => {
    const [kw, sem] = await Promise.all([
      ctx.runQuery(keyword, { q }),
      ctx.runAction(semantic, { q }),
    ]);

    // Merge (union by bookmarkId)
    const map = new Map<string, { item: any; score: number }>();
    for (const it of kw) map.set(it._id, { item: it, score: 1 });
    for (const it of sem) map.set(it._id, { item: it, score: it._score ?? 1 });

    const pool = Array.from(map.values()).map(v => v.item);

    // MMR re-rank
    const ranked = mmr(pool, {
      lambda: SEARCH.MMR_LAMBDA,
      k: SEARCH.CANDIDATE_POOL, // pool size control pre-cutoff
      embed: async (x) => x.primaryVectorId ? x.primaryVectorId : x.title, // placeholder; not used to call model
    });

    // Cutoff rule
    const top = sem[0]?.score ?? 1;
    const cutoff = Math.max(SEARCH.FINAL_CUTOFF, top * SEARCH.RELATIVE_TOP_RATIO);
    return ranked.filter((r: any) => (r.score ?? 1) >= cutoff);
  },
});
```

### 4.3 Redirect & Billing (HTTP)

```ts
// convex/http.ts
import { httpRouter, httpAction } from "convex/server";
import { api } from "./_generated/api";

const http = httpRouter();

// Tracked redirect: /r/:id
http.route({
  path: "/r/:id",
  method: "GET",
  handler: httpAction(async (ctx, req) => {
    const id = req.params?.id as string;
    const bookmark = await ctx.db.get(id as any);
    if (!bookmark) return new Response("Not found", { status: 404 });

    // Increment reliably server-side
    await ctx.db.patch(id as any, { clickCount: (bookmark.clickCount ?? 0) + 1, lastAccessedAt: Date.now() });

    // 302 to normalized URL
    return Response.redirect(bookmark.url, 302);
  }),
});

// Polar webhook: POST /payments/webhook
http.route({
  path: "/payments/webhook",
  method: "POST",
  handler: api.subscriptions.billing.handleWebhook,
});

export default http;
```

---

## 5) Security & Validation (Non‑Negotiable)

- **Per-user isolation**: Every query/mutation/action filters by `userId`. Semantic search **always** filters Qdrant by `userId` payload.
- **Webhook verification**: Verify Polar signature; only ack 2xx on success. Store event rows to `webhookEvents` for idempotency.
- **Input validation**: Use Convex `v.*` validators for all args. URLs must be valid `http(s)` and normalized before storage/use.
- **Server-only secrets**: Never expose OpenAI/Qdrant/Polar secrets to clients.
- **Open redirect audit**: `/r/:id` only uses stored **normalized** URL—never user-provided target.
- **Rate limits**: enforce daily summary cap in server logic; optional per-IP gates via platform (non-blocking in MVP).

---

## 6) Qdrant Wrapper (Server-Only)

```ts
// convex/lib/qdrant.ts
import { QDRANT } from "../../lib/config";

// Lightweight HTTP client (no client-side exposure)
const base = process.env.QDRANT_URL!;
const apiKey = process.env.QDRANT_API_KEY!;
const headers = { "api-key": apiKey, "content-type": "application/json" };

export async function qdrantUpsert({ userId, bookmarkId, emb }: { userId: string; bookmarkId: string; emb: number[] }) {
  const points = [{ id: `${userId}:${bookmarkId}`, vector: emb, payload: { userId, bookmarkId, embVersion: 1 } }];
  await fetch(`${base}/collections/${QDRANT.COLLECTION_PRIMARY}/points`, { method: "PUT", headers, body: JSON.stringify({ points }) });
  return `${userId}:${bookmarkId}`; // primaryVectorId
}

export async function qdrantSearch({ userId, query }: { userId: string; query: string }) {
  const emb = await makeEmbedding(query);
  const body = {
    vector: emb,
    limit: 64,
    params: { hnsw_ef: 96 },
    filter: { must: [{ key: "userId", match: { value: userId } }] },
    with_payload: true,
    score_threshold: undefined, // cutoff applied app-side per locked rules
  };
  const res = await fetch(`${base}/collections/${QDRANT.COLLECTION_PRIMARY}/points/search`, { method: "POST", headers, body: JSON.stringify(body) });
  const json = await res.json();
  return (json?.result ?? []).map((r: any) => ({ _id: r.payload.bookmarkId, score: r.score }));
}

export async function makeEmbedding(text: string): Promise<number[]> {
  // Implement with OpenAI embeddings (text-embedding-3-small, size 1536)
  // Using server-side SDK or HTTPS; return numeric vector.
  return new Array(1536).fill(0); // placeholder in doc
}
```

**Collection & Indexing (provisioning)**

- **Collection**: `bookmarks_primary`
- **Vector size**: 1536; **distance**: cosine
- **HNSW**: `{ m: 32, ef_construct: 256 }`; **query** uses `search_ef = 96`
- **Payload**: `{ userId, bookmarkId, embVersion }`
- **Access**: Only from server; API key via Convex env

---

## 7) OpenAI Summary Action (Locked Prompt)

```ts
// convex/lib/summarize.ts
export async function summarize300({ title, description }: { title: string; description?: string }) {
  const prompt = `Summarize the page neutrally in ≤300 characters for quick recall. Avoid hype, emojis, and commands. Write in clear English. If content is a video, capture the main idea, not timestamps. Return only the summary text, no quotes.`;

  const content = [title, description].filter(Boolean).join("\n\n");
  const text = await callOpenAIChat({ system: prompt, user: content });

  // Clamp to ≤300 **Unicode graphemes** using Intl.Segmenter
  const seg = new (Intl as any).Segmenter("en", { granularity: "grapheme" });
  const graphemes = Array.from(seg.segment(text), (s: any) => s.segment);
  const clamped = graphemes.slice(0, 300).join("");
  return clamped;
}
```

> **Gate**: Only embed when `title` exists (REQUIRE\_TITLE\_FOR\_EMBED).

---

## 8) HTTP Routes (Convex)

- `POST /payments/webhook` → `billing.handleWebhook` (verify signature with Polar secret, update `subscriptions`, optionally record `webhookEvents`).
- `GET /r/:id` → `redirect.handle` (server-side increment + 302 to normalized URL).

---

## 9) Billing Webhook Handler (Polar)

```ts
// convex/subscriptions.ts (excerpt)
import { action, httpAction, mutation, query } from "./_generated/server";
import { Webhook, WebhookVerificationError } from "standardwebhooks";
import { v } from "convex/values";

export const handleWebhook = httpAction(async (ctx, req) => {
  try {
    const rawBody = await req.text();
    const headers: Record<string, string> = {};
    req.headers.forEach((v, k) => (headers[k] = v));

    const secret = process.env.POLAR_WEBHOOK_SECRET!;
    const webhook = new Webhook(btoa(secret)); // Convex runtime lacks Buffer
    webhook.verify(rawBody, headers); // throws on invalid

    const evt = JSON.parse(rawBody);
    // Idempotency (optional):
    if (evt?.id) {
      const dup = await ctx.db
        .query("webhookEvents")
        .withIndex("by_polarEventId", q => q.eq("polarEventId", evt.id))
        .first();
      if (!dup) await ctx.db.insert("webhookEvents", { type: evt.type, polarEventId: evt.id, createdAt: Date.now(), modifiedAt: Date.now(), data: evt });
    }

    // Map event → subscription record changes (active/canceled/past_due, period ends, amount/currency, customerId, etc.)
    await ctx.runMutation(updateSubscriptionFromEvent, { body: evt });

    return new Response(JSON.stringify({ ok: true }), { status: 200, headers: { "content-type": "application/json" } });
  } catch (e) {
    const status = e instanceof WebhookVerificationError ? 403 : 400;
    return new Response(JSON.stringify({ ok: false }), { status, headers: { "content-type": "application/json" } });
  }
});

export const updateSubscriptionFromEvent = mutation({
  args: { body: v.any() },
  handler: async (ctx, { body }) => {
    // Parse event types and upsert subscription row keyed by userId/customerId.
  },
});
```

---

## 10) URL Normalization Utility (for Dedupe & Security)

```ts
// convex/lib/urlNormalize.ts
export function normalizeUrl(input: string): string {
  const u = new URL(input);
  u.hash = ""; // remove fragments
  u.host = u.host.toLowerCase();
  if ((u.port === "80" && u.protocol === "http:") || (u.port === "443" && u.protocol === "https:")) u.port = "";

  // Strip tracking params
  const params = u.searchParams;
  const kill = [/^utm_/i, /^fbclid$/i];
  [...params.keys()].forEach(k => { if (kill.some(rx => rx.test(k))) params.delete(k); });

  // Collapse multiple slashes
  u.pathname = u.pathname.replace(/\/+/g, "/");
  // Remove trailing slash except root
  if (u.pathname.length > 1 && u.pathname.endsWith("/")) u.pathname = u.pathname.slice(0, -1);

  return u.toString();
}
```

---

## 11) Central Config (Single Source for Tunables)

```ts
// lib/config.ts (server-side; export a client-safe subset if needed)
export const SEARCH = {
  VECTOR_DIM: 1536,
  HNSW_M: 32,
  HNSW_EF_CONSTRUCT: 256,
  SEARCH_EF: 96,
  MMR_LAMBDA: 0.7,
  CANDIDATE_POOL: 64, // MMR pool size; results trimmed by cutoff
  FINAL_CUTOFF: 0.22, // cosine
  RELATIVE_TOP_RATIO: 0.70, // 70% of top hit
};

export const INGEST = {
  EMB_MODEL: "text-embedding-3-small",
  SUMMARY_MAX_CHARS: 300,
  REQUIRE_TITLE_FOR_EMBED: true,
  BACKOFF_STEPS_MS: [500, 1000, 2000, 4000],
  DAILY_SUMMARY_LIMIT: 100,
  DAILY_RESET_TZ: "UTC",
};

export const QDRANT = {
  COLLECTION_PRIMARY: "bookmarks_primary",
};
```

---

## 12) Permissions Model & Access Patterns

- **AuthN** via Clerk in Convex actions (`ctx.auth.getUserIdentity()` → `subject` as `userId`).
- **AuthZ**: All reads/writes scoped by `userId`. No cross-user queries or Qdrant searches.
- **Client ↔ Server**: UI calls Convex functions only; **no client calls to Qdrant/OpenAI**.
- **Indexes**: Use `by_user_*` indexes to avoid full scans; paginate for large corpora.

---

## 13) Error Taxonomy & Backoff (Locked)

- `fetch_failed`, `no_metadata`, `model_error`, `timeout`, `rate_limited`, `policy_blocked` (reserved), `unknown`.
- Exponential backoff per stage: 0.5s → 1s → 2s → 4s (max 4 tries).
- On metadata failure: **Retry** runs full pipeline. On summary failure: **Retry** runs summary → embed → upsert.

---

## 14) Observability Hooks (Recommended for MVP)

- **Sentry**: capture exceptions in metadata fetch, summarize, embed, Qdrant ops (mask PII).
- **PostHog**: events for add-URL attempts, success/failure, AI Search clicks, drawer open/save, redirect clicks.

---

## 15) Example Client Usages (UI → Convex)

```ts
// Add URL flow (client)
const id = await api.ingest.createOptimistic({ url, source });
await api.ingest.fetchMetadata({ bookmarkId: id });
const { summary } = await api.ingest.summarize({ bookmarkId: id, title, description });
const { vectorId } = await api.ingest.embedUpsert({ bookmarkId: id, title, description });
await api.ingest.finalize({ bookmarkId: id, draft: { title, summary, thumbnailUrl, status: "ok" }, primaryVectorId: vectorId });
```

---

## 16) Open Questions / TODOs for Billing Examples

- **Product name**: **Bookmark AI** (locked).
- **Currency + Monthly price**: **TBD** (provide actual currency code and amount for webhook mapping examples, e.g., `USD` + `999` cents). Update `subscriptions.amount`, `subscriptions.currency` and any price-display utilities accordingly.

---

## 17) Acceptance Checks (Backend)

- Adding a URL produces an optimistic card, then a finalized bookmark with normalized URL, summary ≤300 chars, and a `primaryVectorId` set.
- Clicking a card link hits `/r/:id`, increments `clickCount`, responds 302 to the stored URL.
- AI Search returns results with MMR λ=0.7 and applies cutoff `(cosine ≥ 0.22) OR (≥ 70% of top hit)`; all Qdrant ops filtered by `userId`.
- Polar webhook verified; `subscriptions` row updates; events recorded idempotently in `webhookEvents`.
- Rate limit (100/day) enforced; `rate_limited` errorReason surfaced on exceeding.

