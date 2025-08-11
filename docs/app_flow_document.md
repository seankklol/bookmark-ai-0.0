# App Flow — Bookmark AI MVP (Locked)

> **Non‑negotiable:** Mirrors the [LOCKED] context. No provider, route, or pipeline changes. Client is React Router v7 (SSR) + Tailwind + shadcn/ui; server is Convex with Clerk (auth), Polar (billing), OpenAI (GPT‑5 summaries), and Qdrant (vectors). Deploy: Vercel. English‑only summaries. No transcripts.

---

## Inputs Needed (Ask‑Only‑If‑Missing)
- **Brand voice:** default to “clean, playful, warm” if not specified.
- **Pricing labels:** **TODO →** confirm monthly price and currency for `/pricing` and CTA copy.

---

## Route Map (Purpose • Auth/Billing • Key Components)

### `/` — Landing
- **Purpose:** Explain value, show social proof/feature highlights, drive to sign‑in/subscribe.
- **Auth/Billing:** Public. If already subscribed, show **Continue to dashboard**.
- **Key components:** Navbar, Hero, Features, Footer. CTA to `/pricing`.

### `/pricing`
- **Purpose:** Show plan(s) fetched from Polar; start checkout.
- **Auth/Billing:** Public. Checkout requires sign‑in; after successful payment return to `/success`.
- **Key components:** Pricing cards (Polar), “Subscribe” buttons, FAQ blurb.

### `/success`
- **Purpose:** Post‑checkout confirmation and next‑step.
- **Auth/Billing:** Requires sign‑in. Server checks entitlement; if active, deep‑link to `/dashboard`. If not yet propagated, show light loader with retry.
- **Key components:** Success message, “Go to dashboard” button.

### `/dashboard`
- **Purpose:** Paid workspace. Add URLs; see cards; open drawer to edit.
- **Auth/Billing:** **Hard‑gated** (Clerk + Polar). If unauth → sign‑in; if unpaid → `/subscription-required?next=/dashboard`.
- **Key components:**
  - **Add URL** input (top). **Enter** submits.
  - **Search bar** (instant keyword). **Cmd/Ctrl+K** focuses.
  - **Cards grid** (16:9). Default sort: `clickCount desc → createdAt desc`.
  - **Side drawer** for title + summary edits. **Cmd/Ctrl+S** saves, **Esc** closes.
  - **Toasts** for save success/failure.

### `/dashboard/chat`
- **Purpose:** Template chat surface (non‑critical to MVP).
- **Auth/Billing:** Same hard‑gate as `/dashboard`.
- **Key components:** Streaming chat UI (from template) if retained.

### `/dashboard/settings`
- **Purpose:** Profile and subscription management.
- **Auth/Billing:** Hard‑gated.
- **Key components:** Account panel; **Manage subscription** (opens Polar Customer Portal); subscription status card.

### `/subscription-required`
- **Purpose:** Soft paywall with clear next step.
- **Auth/Billing:** Requires sign‑in. If user gets active plan mid‑session, auto‑redirect back to `next`.
- **Key components:** Plan summary, “View Plans” → `/pricing`.

### `/r/:id` (server redirect)
- **Purpose:** Reliable click tracking.
- **Auth/Billing:** Server‑side only; no UI. Always increments `clickCount` then **302** to stored URL.
- **Key components:** Convex HTTP route; open‑redirect protection.

---

## Navigation & Guards

**Top‑level nav**
- Landing: Home, Pricing, Sign in/Sign up.
- Authenticated dashboard shell: Sidebar (Dashboard, Chat, Settings) + user menu.

**Guards (SSR loaders/mutations)**
- Every `/dashboard*` route: `requireAuth()` → `hasActiveSubscription()`; otherwise redirect to `/subscription-required?next=<path>`.
- After successful webhook processing, client checks subscription state on page load and on settings change; if active, allow dashboard; if not, push to `/subscription-required`.
- Server reads/writes always scoped to `userId`.

**Redirects**
- Post‑checkout: `/success` → `/dashboard` once entitlement confirmed.
- Card URL clicks use `/r/:id` (never direct external hrefs in card primary link).

---

## State Charts (Textual, Precise)

### 1) **Add URL**
```
Idle
  on SUBMIT(url)
    → Submitting
Submitting
  action: normalize(url); createOptimisticBookmark({title:"Fetching…", status:"ok"})
  → Fetching
Fetching
  action: fetchMetadata(url) with backoff
  on META_OK(meta) → Summarizing
  on META_ERR(reason) → Final(error:{status:"error", title:url, reason})
Summarizing
  action: summarize(meta) with backoff; enforce ≤300 chars
  on SUM_OK(summary) → Embedding
  on SUM_ERR(reason) → Final(error:{status:"error", reason})
Embedding
  guard: title exists
  action: embed(text) → upsert Qdrant; persist Convex
  on EMB_OK → Final(success)
  on EMB_ERR(reason) → Final(error:{status:"error", reason})
Final(success)
  action: replace optimistic card; morph animation
Final(error)
  UI: Error chip + Retry button
```

### 2) **Retry** (from error card)
```
ErrorCard.Retry
  if reason in {fetch_failed,no_metadata,timeout,unknown}
    → FullPipeline (metadata → summarize → embed)
  else if reason in {model_error}
    → SummaryOnly (summarize → embed)
Backoff: 0.5s → 1s → 2s → 4s (max 4 tries)
```

### 3) **Edit Drawer**
```
Closed
  on OPEN(bookmarkId) → Opening
Opening → Editing (drawer rendered with form)
Editing on SAVE (Cmd/Ctrl+S)
  → Saving(optimistic)
Saving(optimistic)
  action: mutate Convex; if title changed -> re-embed + upsert
  on OK → Saved → Closed
  on SaveError(reason) → SaveError (toast) → Editing
Esc at any time → Closed (discard if not saving)
```

### 4) **AI Search**
```
Input(query)
  on RUN_AI_SEARCH → Embed(query) → Qdrant(top64, filter by userId)
  → MergeWithKeywordHits → MMR(λ=0.7) → Threshold(cos≥0.22 or ≥70% of top)
  → Results
```

---

## Empty • Loading • Optimistic States
- **Dashboard empty:** Centered Add‑URL input with friendly helper text.
- **Grid skeletons:** 16:9 shimmering cards; standard cycle ≈ 1.2s.
- **Optimistic add:** Placeholder card shows **Fetching… → Summarizing…** then morphs into final card.
- **Error state:** Red “Error” chip with tooltip; **Retry** affordance inline.

---

## Hotkeys & Accessibility
- **Hotkeys:**
  - **Enter** → submit Add URL.
  - **Cmd/Ctrl+K** → focus Search.
  - **Cmd/Ctrl+S** → save edits when drawer focused.
  - **Esc** → close drawer.
- **A11y:**
  - Focus‑visible rings (2px, primary color).
  - Semantic landmarks (`<header>`, `<nav>`, `<main>`, `<aside>`).
  - Every input has an explicit `<label>` or `aria-label`.
  - Contrast ≥ WCAG AA.

---

## Event & Error Handling
- **Toasts:**
  - `success` (save complete, subscription active)
  - `warn` (rate‑limit nearing, degraded AI)
  - `error` (save failed, pipeline error; short, clear copy)
- **Per‑card errors:** Error chip + tooltip from `errorReason`; Retry logic per state chart.
- **Network timeouts:** Each stage (metadata/summarize/embed) obeys the exponential backoff policy (0.5s → 4s, ×4).
- **Global degradation:** If multiple AI errors within a minute, show one‑time “summarization is degraded” toast; queue retries.

---

## Implementation Notes & Snippets (TS/JS)

### Router guard (loader‑level)
```ts
// Example for /dashboard and children
export async function loader({ request }: LoaderArgs) {
  const user = await requireAuth(request); // Clerk
  const sub = await convex.query("subscriptions.checkUserSubscriptionStatus");
  if (!sub?.hasActiveSubscription) {
    const next = new URL(request.url).pathname;
    throw redirect(`/subscription-required?next=${encodeURIComponent(next)}`);
  }
  return null;
}
```

### Add URL action (client)
```ts
async function onSubmit(e: FormEvent) {
  e.preventDefault();
  const url = input.value.trim();
  if (!url) return;
  addOptimisticCard(url); // title: "Fetching…"
  try {
    await convex.action("pipeline.ingestAndSummarize", { url });
  } catch (e) {
    // Card will render Error chip via server state
  }
}
```

### Tracked redirect (Convex HTTP route)
```ts
http.route({
  path: "/r/:id",
  method: "GET",
  handler: httpAction(async (ctx, req) => {
    const { id } = req.params; // bookmark id
    const b = await ctx.runQuery("bookmarks.get", { id, userId: undefined /* server check */ });
    if (!b) return new Response("Not found", { status: 404 });
    await ctx.runMutation("bookmarks.incrementClick", { id: b._id });
    // Only redirect to the stored, normalized URL
    return new Response(null, { status: 302, headers: { Location: b.url } });
  }),
});
```

### Hotkeys (drawer)
```ts
useEffect(() => {
  function onKey(e: KeyboardEvent) {
    if (e.key === "s" && (e.metaKey || e.ctrlKey)) { e.preventDefault(); save(); }
    if (e.key === "Escape") close();
  }
  window.addEventListener("keydown", onKey);
  return () => window.removeEventListener("keydown", onKey);
}, [save, close]);
```

### AI search (server action)
```ts
export const semantic = action(async ({ userId }, { query }) => {
  const qEmb = await embed(query); // 1536‑d
  const top = await qdrant.search({
    collection: COLLECTION_PRIMARY,
    vector: qEmb,
    filter: { must: [{ key: "userId", match: { value: userId } }] },
    limit: 64,
    params: { ef: 96 },
  });
  const merged = mergeWithKeywordHits(top, await keyword(query));
  const reranked = mmr(merged, { lambda: 0.7 });
  return threshold(reranked, { absolute: 0.22, relativeTopRatio: 0.7 });
});
```

### Default sort (server)
```ts
export const list = query(async ({ userId }) => {
  return db.table("bookmarks")
    .filter({ userId })
    .order("clickCount", "desc")
    .order("createdAt", "desc")
    .collect();
});
```

---

## Design Tokens & UI A11y Locks (Mirror)
- **Typography:** Nunito 400/600/700.
- **Colors:** Primary #3B82F6, Accent #F43F5E, Background #F9FAFB, Surface #FFFFFF, Muted #E5E7EB, Text #111827, Subtext #4B5563, Success #22C55E, Warning #F59E0B, Danger #EF4444.
- **Spacing:** 4px grid. **Radius:** 12px.
- **Components:** Drawer 440–480px. Cards 16:9 with subtle hover elevation. Skeleton shimmer ≈ 1.2s.
- **Hotkeys & A11y:** as listed above.

---

## Data & Security Locks (Mirror)
- **Schema (bookmarks)** includes: `userId`, `url`, `title`, `summary(≤300)`, `thumbnailUrl`, `status`, `errorReason`, `clickCount`, `embVersion`, `primaryVectorId`, etc.
- **Normalization:** lowercase host, strip trackers (`utm_*`, `fbclid`), remove fragments, collapse slashes, trim trailing slash.
- **Entitlements:** server‑verified via webhook and subscription checks.
- **Vectors/Qdrant:** size=1536, distance=cosine; HNSW m=32, ef_construct=256; query search_ef=96; global collection with mandatory `userId` filter.

---

## Copy Tone (PROJECT_DETAILS)
- **Default:** concise, helpful, lightly playful. Avoid hype.
- **CTA examples:** “Add a URL”, “Run AI Search”, “Manage subscription”.
- **TODO:** finalize pricing copy once currency/amount confirmed.

---

## QA Hooks
- Playwright flows: login→dashboard, add URL success/failure + Retry, keyword search, AI Search (thresholded), drawer edit/save hotkeys, tracked redirect increments.

---

## Appendix: Limits & Backoff (Mirror)
- **Daily summaries:** 100/user/day, reset midnight UTC.
- **Errors:** `fetch_failed`, `no_metadata`, `model_error`, `timeout`, `rate_limited`, `unknown` (reserved: `policy_blocked`).
- **Backoff:** 0.5s → 1s → 2s → 4s; max 4 tries per stage.

