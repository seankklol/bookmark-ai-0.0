# Task 19 — Initialize Qdrant collection

Branch Name: task/19-initialize-qdrant-collection

## Background and Motivation
Initialize Qdrant collection `bookmarks_primary` with `{size:1536, distance:cosine}`, HNSW `{m:32, ef_construct:256}`; set query-time `search_ef=96`.

## Key Challenges and Analysis
- Correctly configuring collection parameters.
- Ensuring idempotency if collection exists.

## High-level Task Breakdown
1. Create branch `task/19-initialize-qdrant-collection` from `main`.
2. Create collection with specified parameters.
3. Verify collection schema via API.
4. Verify acceptance criteria.

### Acceptance Criteria
- Collection exists with correct schema; inspected via API.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
