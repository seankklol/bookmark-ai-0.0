# Task 20 — Extend orchestrator (summary → embed → Qdrant upsert)

Branch Name: task/20-extend-orchestrator

## Background and Motivation
Extend ingestion orchestrator: after summarization, embed, then Qdrant upsert; update Convex fields `embVersion`, `primaryVectorId`.

## Key Challenges and Analysis
- Ensuring orchestrator flow and error handling.
- Maintaining atomic updates.

## High-level Task Breakdown
1. Create branch `task/20-extend-orchestrator` from `main`.
2. Extend orchestrator pipeline to perform summary → embed → Qdrant upsert.
3. Update Convex document fields accordingly.
4. Verify UI shows summary and bookmark becomes searchable.
5. Verify acceptance criteria.

### Acceptance Criteria
- Fresh save shows summary text and becomes searchable.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
