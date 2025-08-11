# Task 35 — Create central config module

Branch Name: task/35-create-central-config-module

## Background and Motivation
Create central config module `lib/config.ts` (server) and client-safe subset; ensure all controllers import from it.

## Key Challenges and Analysis
- Avoiding magic numbers elsewhere.
- Ensuring client build strips secrets.

## High-level Task Breakdown
1. Create branch `task/35-create-central-config-module` from `main`.
2. Implement config constants (SEARCH, INGEST, QDRANT).
3. Export client-safe subset without secrets.
4. Refactor controllers to use config.
5. Run static analysis to confirm no magic numbers remain.
6. Verify acceptance criteria.

### Acceptance Criteria
- Static analysis shows no magic numbers elsewhere; imports compile.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
