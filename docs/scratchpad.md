# Splitting Implementation Plan into Individual Task Files

## Purpose
Break down the monolithic `docs/implementation_plan.md` into one implementation-detail file per task, as required by the global project management rules.

## Naming Convention
- **Directory:** `docs/implementation-plan/`
- **Filename:** `{NN}-{slug}.md`
  - `NN` – two-digit task number (01-50)
  - `slug` – concise kebab-case summary of the task name (≤ 6 words)
- **Branch Name** inside each file: `task/{NN}-{slug}`

## File Template (to be initialised for every task)
```md
# Task NN — <Full Task Title>

Branch Name: task/NN-slug

## Background and Motivation
<original description>

## Key Challenges and Analysis
- _TBD by Planner/Executor_

## High-level Task Breakdown
1. Create branch `task/NN-slug` from `main`.
2. Implement the work described.
3. Verify acceptance criteria.

### Acceptance Criteria
- <copied from original "AC:" clause>

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
```

## Task Mapping
| # | File | Original Task Title |
| --- | --- | --- |
| 01 | 01-decide-product-basics.md | Decide product basics |
| 02 | 02-initialize-repo-vercel.md | Initialize repo & Vercel preview |
| 03 | 03-provision-convex.md | Provision Convex project/deployment |
| 04 | 04-create-clerk-project.md | Create Clerk project |
| 05 | 05-create-polar-product.md | Create Polar org + product/price |
| 06 | 06-provision-qdrant.md | Provision Qdrant (managed) |
| 07 | 07-configure-openai-and-secrets.md | Configure OpenAI & store secrets |
| 08 | 08-define-convex-schema.md | Define Convex schema for bookmarks |
| 09 | 09-create-convex-indexes.md | Create Convex indexes |
| 10 | 10-implement-url-util.md | Implement URL util |
| 11 | 11-build-create-mutation.md | Build createBookmark mutation |
| 12 | 12-implement-metadata-fetcher.md | Implement metadata fetcher |
| 13 | 13-add-exponential-backoff.md | Add exponential backoff helper |
| 14 | 14-wire-ingestion-orchestrator.md | Wire ingestion orchestrator |
| 15 | 15-openai-summarization-action.md | Implement OpenAI summarization action |
| 16 | 16-enforce-rate-limit.md | Enforce summarization rate limit |
| 17 | 17-implement-embeddings-action.md | Implement embeddings action |
| 18 | 18-build-qdrant-wrapper.md | Build Qdrant server wrapper |
| 19 | 19-initialize-qdrant-collection.md | Initialize Qdrant collection |
| 20 | 20-extend-orchestrator.md | Extend orchestrator (summary→embed) |
| 21 | 21-implement-retry-semantics.md | Implement retry semantics |
| 22 | 22-build-dashboard-grid.md | Build dashboard grid |
| 23 | 23-implement-side-drawer.md | Implement side drawer |
| 24 | 24-add-optimistic-states.md | Add optimistic states on card |
| 25 | 25-create-subscription-required-route.md | Create subscription-required route |
| 26 | 26-implement-empty-state.md | Implement empty state |
| 27 | 27-implement-global-hotkeys.md | Implement global hotkeys |
| 28 | 28-add-keyword-search.md | Add keyword search |
| 29 | 29-add-skeleton-shimmer.md | Add skeleton shimmer & tokens |
| 30 | 30-wire-auth-routes.md | Wire auth routes |
| 31 | 31-implement-ai-search.md | Implement AI search |
| 32 | 32-merge-keyword-and-vector-results.md | Merge keyword hits with vector candidates |
| 33 | 33-apply-final-cutoff-rule.md | Apply final cutoff rule |
| 34 | 34-add-controller-ui-toggle.md | Add controller + UI toggle (AI Search) |
| 35 | 35-create-central-config-module.md | Create central config module |
| 36 | 36-add-tunables-and-health-page.md | Add tunables & health page |
| 37 | 37-create-diagnostic-script.md | Create diagnostic script |
| 38 | 38-build-pricing-page.md | Build pricing page |
| 39 | 39-implement-convex-http-route.md | Implement Convex HTTP webhook route |
| 40 | 40-enforce-entitlement-gating.md | Enforce entitlement gating |
| 41 | 41-implement-tracked-redirect.md | Implement tracked redirect action |
| 42 | 42-add-success-page.md | Add success page |
| 43 | 43-instrument-posthog-events.md | Instrument PostHog events |
| 44 | 44-write-unit-tests.md | Write unit tests |
| 45 | 45-write-e2e-tests.md | Write E2E tests |
| 46 | 46-setup-observability.md | Setup observability (Sentry, dashboards) |
| 47 | 47-performance-and-a11y.md | Performance & A11y improvements |
| 48 | 48-release-checklist.md | Release checklist |
| 49 | 49-production-launch.md | Production launch |
| 50 | 50-post-launch-guardrails.md | Post-launch guardrails |

---

**Next Steps (for Executor):**
1. Generate 50 markdown files following the naming convention and template above.
2. Move original task descriptions + acceptance criteria into the respective files.
3. Update `docs/implementation_plan.md` to serve as an index linking to each new task file.
4. Commit and push in a feature branch, then open a draft PR for review.
