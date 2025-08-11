# Task 36 — Add tunables & health page

Branch Name: task/36-add-tunables-and-health-page

## Background and Motivation
Add tunables (e.g., `SEARCH_EF`) read from central config; expose a server-only health page that echoes current constants.

## Key Challenges and Analysis
- Safely exposing diagnostic information behind auth.

## High-level Task Breakdown
1. Create branch `task/36-add-tunables-and-health-page` from `main`.
2. Extend config module to include tunables.
3. Implement `/admin/health` route behind auth displaying config values.
4. Verify acceptance criteria.

### Acceptance Criteria
- `/admin/health` behind auth displays config values.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
