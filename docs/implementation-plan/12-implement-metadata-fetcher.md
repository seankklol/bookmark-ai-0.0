# Task 12 — Implement metadata fetcher

Branch Name: task/12-implement-metadata-fetcher

## Background and Motivation
Implement metadata fetcher (Convex action) via JSDOM: IG-specific selectors else OG/Twitter/meta; thumbnail fallback priority `og:image` → favicon → server screenshot → placeholder.

## Key Challenges and Analysis
- _TBD by Planner/Executor_

## High-level Task Breakdown
1. Create branch `task/12-implement-metadata-fetcher` from `main`.
2. Build Convex action for metadata extraction following priority rules.
3. Handle error cases setting `status="error"`, `title=url`.
4. Validate against three real domains.
5. Verify acceptance criteria.

### Acceptance Criteria
- 3 real domains validated.
- Missing meta sets `status="error"` and `title=url`.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
