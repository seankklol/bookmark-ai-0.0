# Task 15 — OpenAI summarization action

Branch Name: task/15-openai-summarization-action

## Background and Motivation
Implement OpenAI summarization action with the locked prompt; enforce Unicode grapheme ≤ 300 with safe truncation.

## Key Challenges and Analysis
- Handling multi-byte grapheme counting (emojis, accents).
- Ensuring prompt remains locked and version controlled.

## High-level Task Breakdown
1. Create branch `task/15-openai-summarization-action` from `main`.
2. Implement Convex action calling OpenAI with locked prompt.
3. Add grapheme counter utility and safe truncation logic (≤ 300).
4. Write unit tests for strings with emojis.
5. Verify acceptance criteria.

### Acceptance Criteria
- Strings with emojis counted correctly.

## Project Status Board
- [ ] Todo
- [ ] In Progress
- [ ] Done

## Current Status / Progress Tracking

## Executor's Feedback or Assistance Requests
