# Task 17 — Implement embeddings action

Branch Name: task/17-implement-embeddings-action

## Background and Motivation
Implement embeddings action using `text-embedding-3-small` (1536-d). Gate: ensure Convex organization; compose text: always title + (IG caption or meta description).

## Key Challenges and Analysis
- Ensuring text composition order and fallback.
- Handling empty title rejection.

## High-level Task Breakdown
1. Create branch `task/17-implement-embeddings-action` from `main`.
2. Implement embeddings action with specified model and dimensions.
3. Compose embedding text with title + caption/description.
4. Reject empty title with clear error.
5. Write unit tests.
6. Verify acceptance criteria.

### Acceptance Criteria
- Empty title rejects embed with clear error.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
