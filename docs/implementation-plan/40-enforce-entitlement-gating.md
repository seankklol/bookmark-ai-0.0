# Task 40 — Enforce entitlement gating

Branch Name: task/40-enforce-entitlement-gating

## Background and Motivation
Enforce entitlement gating on server for dashboard/API; unauthorized returns 402/redirect to `subscription-required`.

## Key Challenges and Analysis
- Consistent gating across services.

## High-level Task Breakdown
1. Create branch `task/40-enforce-entitlement-gating` from `main`.
2. Implement middleware checking subscription status.
3. Return 402 or redirect for unpaid users.
4. Verify acceptance criteria.

### Acceptance Criteria
- Paid vs. unpaid paths verified.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
