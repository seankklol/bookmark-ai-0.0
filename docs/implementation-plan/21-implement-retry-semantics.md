# Task 21 — Implement retry semantics

Branch Name: task/21-implement-retry-semantics

## Background and Motivation
Implement retry semantics: metadata failure → full pipeline retry; summary failure → summary+embed only.

## Key Challenges and Analysis
- Designing retry logic without infinite loops.
- Differentiating failure types.

## High-level Task Breakdown
1. Create branch `task/21-implement-retry-semantics` from `main`.
2. Implement retry logic for pipeline based on failure class.
3. Write tests simulating injected failures.
4. Verify acceptance criteria.

### Acceptance Criteria
- Manual toggle of injected failure proves correct retry path.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
