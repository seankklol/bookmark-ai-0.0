# Task 31 — Implement AI search

Branch Name: task/31-implement-ai-search

## Background and Motivation
Implement AI search: embed query → Qdrant search in `bookmarks_primary` with payload filter `userId`; take top 64 by vector score.

## Key Challenges and Analysis
- Managing latency of embedding generation.
- Handling empty query edge cases.

## High-level Task Breakdown
1. Create branch `task/31-implement-ai-search` from `main`.
2. Implement server endpoint: embed query then Qdrant search with user filter.
3. Return candidate set.
4. Verify acceptance criteria.

### Acceptance Criteria
- Wrapper returns candidates.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
