# Task 04 — Verify Vercel preview + SSR baseline

Branch Name: task/04-verify-vercel-ssr-baseline

## Background and Motivation
Verify Vercel preview and SSR baseline are working for the app. Ensure the Preview URL renders the `dashboard` route without runtime errors, SSR is active (initial HTML includes the dashboard shell), and no secrets are present in the client bundle (confirmed via build grep). Refs: `docs/app_flow_document.md`, `docs/frontend_guidelines_document.md`, `docs/backend_structure_document.md`.

## Key Challenges and Analysis
- _TBD by Planner/Executor_

## High-level Task Breakdown
1. Create branch `task/04-verify-vercel-ssr-baseline` from `main`.
2. Confirm Vercel project is connected and configured with `@vercel/react-router` preset and SSR enabled.
3. Open a PR to `main` to trigger a Vercel Preview deploy.
4. Validate the Preview URL renders the `dashboard` route without runtime errors.
5. Confirm SSR is active: initial HTML contains the dashboard shell and hydration attaches cleanly.
6. Run a production build and grep to ensure no secrets are present in client bundles.
7. Verify acceptance criteria.

### Acceptance Criteria
- Preview URL renders the `dashboard` route without runtime errors.
- SSR is active (initial HTML includes dashboard shell; hydration is clean).
- No secrets appear in client bundles (build grep shows none).

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
