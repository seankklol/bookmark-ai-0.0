# Task 41 — Implement tracked redirect action

Branch Name: task/41-implement-tracked-redirect

## Background and Motivation
Implement tracked redirect (`/r/:id`) (Convex HTTP action): increment `clickCount` server-side then 302 to stored normalized URL; prevent open-redirect.

## Key Challenges and Analysis
- Security to prevent malicious redirects.

## High-level Task Breakdown
1. Create branch `task/41-implement-tracked-redirect` from `main`.
2. Implement HTTP action incrementing clickCount and redirecting.
3. Add validation ensuring only known bookmark URLs.
4. Verify acceptance criteria.

### Acceptance Criteria
- Only known bookmark URLs are used.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
