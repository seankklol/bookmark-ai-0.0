# Implementation Plan Index
This file now serves as an index to the per-task implementation files in `docs/implementation-plan/`.

## Task Files
- [Task 01 — Decide product basics](implementation-plan/01-decide-product-basics.md)
- [Task 02 — Initialize repo & Vercel preview](implementation-plan/02-initialize-repo-vercel.md)
- [Task 03 — Provision Convex project/deployment](implementation-plan/03-provision-convex.md)
- [Task 04 — Verify Vercel preview & SSR baseline](implementation-plan/04-create-clerk-project.md)
- [Task 05 — Establish frontend design system baseline](implementation-plan/05-create-polar-product.md)
- [Task 06 — Provision Qdrant (managed)](implementation-plan/06-provision-qdrant.md)
- [Task 07 — Configure OpenAI & store secrets](implementation-plan/07-configure-openai-and-secrets.md)
- [Task 08 — Define Convex schema for bookmarks](implementation-plan/08-define-convex-schema.md)
- [Task 09 — Create Convex indexes](implementation-plan/09-create-convex-indexes.md)
- [Task 10 — Implement URL util](implementation-plan/10-implement-url-util.md)
- [Task 11 — Build createBookmark mutation](implementation-plan/11-build-create-mutation.md)
- [Task 12 — Implement metadata fetcher](implementation-plan/12-implement-metadata-fetcher.md)
- [Task 13 — Add exponential backoff helper](implementation-plan/13-add-exponential-backoff.md)
- [Task 14 — Wire ingestion orchestrator](implementation-plan/14-wire-ingestion-orchestrator.md)
- [Task 15 — OpenAI summarization action](implementation-plan/15-openai-summarization-action.md)
- [Task 16 — Enforce summarization rate limit](implementation-plan/16-enforce-rate-limit.md)
- [Task 17 — Implement embeddings action](implementation-plan/17-implement-embeddings-action.md)
- [Task 18 — Build Qdrant server wrapper](implementation-plan/18-build-qdrant-wrapper.md)
- [Task 19 — Initialize Qdrant collection](implementation-plan/19-initialize-qdrant-collection.md)
- [Task 20 — Extend orchestrator](implementation-plan/20-extend-orchestrator.md)
- [Task 21 — Implement retry semantics](implementation-plan/21-implement-retry-semantics.md)
- [Task 22 — Build dashboard grid](implementation-plan/22-build-dashboard-grid.md)
- [Task 23 — Implement side drawer](implementation-plan/23-implement-side-drawer.md)
- [Task 24 — Add optimistic states on card](implementation-plan/24-add-optimistic-states.md)
- [Task 25 — Create subscription-required route](implementation-plan/25-create-subscription-required-route.md)
- [Task 26 — Implement empty state](implementation-plan/26-implement-empty-state.md)
- [Task 27 — Implement global hotkeys](implementation-plan/27-implement-global-hotkeys.md)
- [Task 28 — Add keyword search](implementation-plan/28-add-keyword-search.md)
- [Task 29 — Add skeleton shimmer & tokens](implementation-plan/29-add-skeleton-shimmer.md)
- [Task 30 — Wire auth routes](implementation-plan/30-wire-auth-routes.md)
- [Task 31 — Implement AI search](implementation-plan/31-implement-ai-search.md)
- [Task 32 — Merge keyword & vector results](implementation-plan/32-merge-keyword-and-vector-results.md)
- [Task 33 — Apply final cutoff rule](implementation-plan/33-apply-final-cutoff-rule.md)
- [Task 34 — Add controller + UI toggle](implementation-plan/34-add-controller-ui-toggle.md)
- [Task 35 — Create central config module](implementation-plan/35-create-central-config-module.md)
- [Task 36 — Add tunables & health page](implementation-plan/36-add-tunables-and-health-page.md)
- [Task 37 — Create diagnostic script](implementation-plan/37-create-diagnostic-script.md)
- [Task 38 — Build pricing page](implementation-plan/38-build-pricing-page.md)
- [Task 39 — Implement Convex HTTP webhook](implementation-plan/39-implement-convex-http-route.md)
- [Task 40 — Enforce entitlement gating](implementation-plan/40-enforce-entitlement-gating.md)
- [Task 41 — Implement tracked redirect](implementation-plan/41-implement-tracked-redirect.md)
- [Task 42 — Add success page](implementation-plan/42-add-success-page.md)
- [Task 43 — Instrument PostHog events](implementation-plan/43-instrument-posthog-events.md)
- [Task 44 — Write unit tests](implementation-plan/44-write-unit-tests.md)
- [Task 45 — Write E2E tests](implementation-plan/45-write-e2e-tests.md)
- [Task 46 — Setup observability](implementation-plan/46-setup-observability.md)
- [Task 47 — Performance & A11y](implementation-plan/47-performance-and-a11y.md)
- [Task 48 — Release checklist](implementation-plan/48-release-checklist.md)
- [Task 49 — Production launch](implementation-plan/49-production-launch.md)
- [Task 50 — Post-launch guardrails](implementation-plan/50-post-launch-guardrails.md)

---


# Phase 0 — Repo & Environment Setup

1. Decide product basics: set **Product Name = “Bookmark AI”**, brand voice = warm, playful, plain English; **Pricing = USD \$19.99/month** (no trial). AC: Copy appears on landing/pricing; Polar plan created reflecting USD \$19.99.
2. Initialize repo from template (RSK) and rename to `bookmark-ai`; connect GitHub → Vercel with `@vercel/react-router` preset and SSR enabled. AC: PR to `main` triggers Vercel Preview successfully.
3. Provision **Convex** project/deployment; record `CONVEX_DEPLOYMENT`, `VITE_CONVEX_URL`, and `NEXT_PUBLIC_CONVEX_URL` per template. AC: `npx convex dev` works locally; health check ok.
4. Verify **Vercel preview + SSR** baseline. AC: Preview URL renders the `dashboard` route without runtime errors; SSR is active (initial HTML includes dashboard shell); no secrets in client bundle (build grep shows none). Refs: `docs/app_flow_document.md`, `docs/frontend_guidelines_document.md`, `docs/backend_structure_document.md`.
5. Establish **frontend design system baseline**. AC: Tokens/typography applied (Nunito 400/600/700, radius 12px, focus-visible rings); dashboard shell meets AA contrast; no console style warnings; hot reload works. Refs: `docs/frontend_guidelines_document.md`, `docs/app_flow_document.md`.
6. Provision **Qdrant (managed)**; obtain `QDRANT_URL`, `QDRANT_API_KEY` for server-only use. AC: Server can auth to Qdrant via wrapper smoke test.
7. Configure **OpenAI** `OPENAI_API_KEY`; set **FRONTEND_URL** for local (ngrok if needed). Store all secrets in Convex/Vercel envs (server-only where applicable). AC: No secret referenced in client bundles (build grep shows none).

# Phase 1 — Schema, Normalization, Ingest (Optimistic) & Metadata

8. Define **Convex schema** for `bookmarks` exactly per SoT (fields + types incl. `embVersion`, `primaryVectorId`, `clickCount`, `status`, `errorReason`). AC: Typegen passes; migrations apply.
9. Create **Convex indexes**: `(userId, createdAt desc)`, `(userId, clickCount desc)`, and unique per-user on `normalizedUrl`. AC: Duplicate URL for same user is rejected with deterministic error.
10. Implement ``** util** (lowercase host, strip default ports/fragments, remove `utm_*`/`fbclid`, collapse slashes, trim trailing slash). AC: Golden tests for tricky URLs pass.
11. Build ``** mutation**: validate URL/protocol → normalize → dedupe → insert optimistic doc (`status="ok"`, title `"Fetching…"`, empty summary). AC: Returns doc id immediately; card appears in UI.
12. Implement **metadata fetcher (Convex action)** via JSDOM: IG-specific selectors else OG/Twitter/meta; thumbnail fallback: `og:image`→favicon→server screenshot→placeholder. AC: 3 real domains validated; missing meta sets `status="error"`, `title=url`.
13. Add **exponential backoff helper** (0.5s→1s→2s→4s; max 4 tries) used by metadata/summary/embed. AC: Unit test proves schedule & max retries.
14. Wire **ingestion orchestrator** (server): after optimistic create, run metadata; update doc fields; on fatal fetch failure set `status="error"` + `errorReason`. AC: State machine transitions logged.

# Phase 2 — Summarization, Limits, Embeddings & Qdrant Init

15. Implement **OpenAI summarization action** with the **locked prompt**; enforce **Unicode grapheme ≤ 300** with safe truncation. AC: Strings with emojis counted correctly.
16. Enforce **rate limit**: max **100 summaries/user/day** (UTC reset); on exceed, return `rate_limited`. AC: Counter persists across restarts; resets at midnight UTC.
17. Implement **embeddings action** (`text-embedding-3-small`, 1536-d). Gate: ``; compose text: always title + (IG caption or meta description). AC: Empty title rejects embed with clear error.
18. Build **Qdrant server wrapper** (upsert/query/delete) that **always includes **``** in payload & filters queries by **``. AC: Unit test fails any unfiltered query attempt.
19. **Initialize Qdrant collection** `bookmarks_primary` with `{size:1536, distance:cosine}`, HNSW `{m:32, ef_construct:256}`; set query-time `search_ef=96`. AC: Collection exists; schema inspected via API.
20. Extend orchestrator: **summary → embed → Qdrant upsert**; update Convex (`embVersion`, `primaryVectorId`). AC: Fresh save shows summary text and becomes searchable.
21. Implement **Retry semantics**: metadata failure → full pipeline retry; summary failure → summary+embed only. AC: Manual toggle of injected failure proves correct retry path.

# Phase 3 — Dashboard UI, Drawer, Hotkeys, Keyword Search

22. Build **dashboard grid**: fixed 16:9 cards (desktop \~240×135 / mobile \~160×90), fields: thumbnail/icon, title, clickable URL (later via `/r/:id`), date. Default sort: `clickCount desc → createdAt desc`. AC: Sorting verified with seeded data.
23. Implement **side drawer** with editable Title + Summary; **Cmd/Ctrl+S** saves; **Esc** closes; success/failure toasts. AC: Keyboard tests pass; fields persist.
24. Add **optimistic states** on card: “Fetching… → Summarizing…” then morph animation on success; Error chip + **Retry** button on failure. AC: State transitions visible within one flow.
25. Create **subscription-required route** & guard dashboard server-side via Clerk+Convex entitlement check (UI shows message). AC: Unpaid user blocked from dashboard.
26. Implement **empty state** with centered Add-URL input; placeholder microcopy in product voice. AC: Snapshot approved.
27. Implement global **hotkeys**: **Enter** submits Add URL, **Cmd/Ctrl+K** focuses search bar, **Cmd/Ctrl+S** saves drawer edits, **Esc** closes drawer. AC: Playwright verifies.
28. Add **keyword search** (as-you-type) over Convex-indexed fields: `title`, `authorHandle?`, `description?`, `summary` (tolerant to missing transcript). AC: Returns expected relevance order.
29. Add **skeleton shimmer** and hover elevation (subtle) per tokens; Nunito 400/600/700; radius **12px**; focus-visible rings. AC: A11y contrast AA met.
30. Wire **auth routes** (sign-in/up) with Clerk; redirect unauthorized users appropriately. AC: Logout flow returns to landing.

# Phase 4 — AI Search, MMR, Thresholds & Central Config

31. Implement **AI search**: embed query → Qdrant search in `bookmarks_primary` with **payload filter **``; take top **64** by vector score. AC: Wrapper returns candidates.
32. **Merge** keyword hits with vector candidates; implement **MMR re-ranking** with **λ = 0.7**. AC: Unit test shows diversity improvement vs. pure similarity.
33. Apply **final cutoff** rule: include items with **cosine ≥ 0.22 OR ≥ 70% of top hit**; no hard cap. AC: Synthetic test verifies threshold behavior.
34. Add **controller + UI toggle** (“AI Search”) to fetch merged results and render; instrument PostHog event. AC: Clicks recorded with userId.
35. Create **central config module** `lib/config.ts` (server) and **client-safe subset**; ensure all controllers import from it:

```ts
export const SEARCH = { VECTOR_DIM: 1536, HNSW_M: 32, HNSW_EF_CONSTRUCT: 256, SEARCH_EF: 96, MMR_LAMBDA: 0.7, CANDIDATE_POOL: 64, FINAL_CUTOFF: 0.22, RELATIVE_TOP_RATIO: 0.70 };
export const INGEST = { EMB_MODEL: 'text-embedding-3-small', SUMMARY_MAX_CHARS: 300, REQUIRE_TITLE_FOR_EMBED: true, BACKOFF_STEPS_MS: [500,1000,2000,4000], DAILY_SUMMARY_LIMIT: 100, DAILY_RESET_TZ: 'UTC' };
export const QDRANT = { COLLECTION_PRIMARY: 'bookmarks_primary' };

// client-safe subset (no secrets)
export const clientConfig = {
  SEARCH: {
    MMR_LAMBDA: SEARCH.MMR_LAMBDA,
    FINAL_CUTOFF: SEARCH.FINAL_CUTOFF,
    RELATIVE_TOP_RATIO: SEARCH.RELATIVE_TOP_RATIO,
  },
};
```

AC: Static analysis shows no magic numbers elsewhere; imports compile. 36. Add **tunables** (e.g., `SEARCH_EF`) read from config; expose a server-only health page that echoes current constants. AC: `/admin/health` behind auth displays config values. 37. Create **diagnostic script** to validate Qdrant connectivity and payload schema for a test user. AC: CI step logs green.

# Phase 5 — Redirect Tracking & Events

41. Implement **tracked redirect** `` (Convex HTTP action): increment `clickCount` server-side then **302** to stored normalized URL; prevent open-redirect. AC: Only known bookmark URLs are used.
43. Instrument **PostHog** events: views, add URL attempts, success/failure, AI search clicks, drawer saves, redirect clicks. AC: Dashboard shows events by user.

# Phase 6 — Tests, Observability, Polish & Launch

44. **Unit tests**: `urlNormalization`; grapheme-count limit; retry/backoff; MMR edge cases (ties/diversity); **Qdrant user filter enforcement** (reject unfiltered). AC: All green in CI.
45. **E2E (Playwright)**: happy ingest; error + retry; keyword + AI search with thresholding; drawer edit + hotkeys; `` increments & redirects. AC: All specs pass in CI against Preview.
46. **Observability**: integrate **Sentry** for server exceptions (fetch, summarize, embed, Qdrant ops) with PII scrubbing; **PostHog** dashboards for core funnels. AC: Sentry receives test error; dashboards populated.
47. **Performance & A11y**: run Lighthouse/Axe; ensure focus-visible, labels, landmarks, AA contrast; confirm perf budgets (**keyword P95 ≤120ms**, **ingest P50 ≤8s/P90 ≤15s**). AC: Reports stored in CI artifacts.
48. **Release checklist**: verify all secrets present; Qdrant collection live; **rate limits enforced**; default sort correct; hotkeys functional. AC: Checklist PR approved.
49. **Production launch**: set custom domain; promote Vercel Production; rotate temporary keys; switch Polar to live mode; final smoke test. AC: 200 OK on landing and gated dashboard works for paid user.
50. **Post-launch guardrails**: set alert for ≥5 summarization errors/min → show banner toast; schedule monthly Qdrant key rotation; weekly metadata/emb failure review; backup Convex ids & Qdrant payload pointers. AC: Alerts firing on synthetic test; runbook documented.
