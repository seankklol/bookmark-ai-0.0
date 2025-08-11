# Task 18 — Build Qdrant server wrapper

Branch Name: task/18-build-qdrant-wrapper

## Background and Motivation
Build Qdrant server wrapper (upsert/query/delete) that always includes `userId` in payload & filters queries by `userId`.

## Key Challenges and Analysis
- Enforcing user filter on every request.
- Ensuring wrapper is generic and reusable.

## High-level Task Breakdown
1. Create branch `task/18-build-qdrant-wrapper` from `main`.
2. Implement wrapper with upsert, query, delete methods.
3. Ensure each query includes user filter condition.
4. Write unit tests rejecting unfiltered queries.
5. Verify acceptance criteria.

### Acceptance Criteria
- Unit test fails any unfiltered query attempt.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
