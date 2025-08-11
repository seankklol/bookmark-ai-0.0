# Task 39 — Implement Convex HTTP webhook route

Branch Name: task/39-implement-convex-http-route

## Background and Motivation
Implement Convex HTTP route to handle Polar webhooks; verify Polar signature; upsert subscription state; idempotent handling.

## Key Challenges and Analysis
- Secure signature verification.

## High-level Task Breakdown
1. Create branch `task/39-implement-convex-http-route` from `main`.
2. Implement HTTP route validating Polar signature.
3. Upsert subscription state with idempotency.
4. Replay same event to confirm no-op.
5. Verify acceptance criteria.

### Acceptance Criteria
- Replay of same event is a no-op (200).

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
