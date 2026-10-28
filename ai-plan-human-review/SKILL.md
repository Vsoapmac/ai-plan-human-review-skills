---
name: ai-plan-human-review
description: Use when the user submits a coding request ("需求", "帮我实现", "帮我写", "帮我改"), replies "继续" after reviewing a proposed change list, or asks whether the AI's changes are reasonable. Must be used whenever the user needs coding help or wants a requirement implemented.
---

# AI Plans, You Review

## Overview

The AI drafts, the human approves, the AI executes. For coding tasks the AI scans the project, assesses the problem, and delivers a concrete change list—which files, where in each file, and why—instead of asking "do you write it or do I". That list is the review artifact: no file changes until the user replies "继续"; after execution the AI reports what actually changed and how it was verified, so the user can judge whether the changes are reasonable.

## Trigger Conditions

**Activate:** Requests to write/generate/modify/refactor code; "继续" after a change list is presented; requests to judge whether the AI's changes are reasonable.

**Do not activate:** Pure Q&A, code explanation, debugging/troubleshooting, reviewing code written by humans, config files/documentation, renaming without logic changes.

## Core Workflow

```
User submits a coding task
       |
       v
Scan project context (related files, functional role, dependencies, data flow)
       |
       v
+-- Assess and propose (unchanged) ---------------------------+
| 1. Problem breakdown        2. Difficulty (*~*****)         |
| 3. Repetitiveness (grunt work / challenging)                |
| 4. Domain (comfort zone / unfamiliar)                       |
| 5. Security risk (none / caution / high)                    |
| 6. 2-3 approaches           7. Key snippet (<=10 lines)     |
+-------------------------------------------------------------+
       |
       v
+-- Change list ----------------------------------------------+
| What changes: behavior-level items, numbered                |
| Files involved: N (new x / modified y / deleted z)          |
| For each file:                                              |
|   - path (new / modify / delete)                            |
|   - where: function / class / line range / config key       |
|   - why: the requirement, approach, or risk it resolves     |
+-------------------------------------------------------------+
       |
       v
One closing line: "Review 完回「继续」，我再开始改文件。"
       |
       v
User reviews -> replies 继续 -> AI edits files -> reports actual changes + verification
```

The assessment must be completed before the change list; neither can be skipped.

## Analysis Dimensions

| Item | Criteria |
|--------|----------|
| Difficulty | Technical complexity: reading a CSV = * / a state machine = **** |
| Repetitiveness | Pure grunt work (bulk replacement) vs challenging (algorithm design) |
| Domain | Judged by your engineering background: familiar vs unfamiliar (mark as a learning opportunity) |
| Security risk | No risk / Caution / High risk (see security assessment below) |

## Security Risk Assessment

The AI assesses based on project context and task content. For high-risk or caution levels, it **must give specific warnings**, not just a level label.

| Level | Scenarios (keywords) |
|------|--------------|
| **High risk** | SQL string concatenation, password/key/token handling, deserializing user input, file upload/path traversal, permission checks |
| **Caution** | Database queries (SQL injection), Cookie/Session, API data masking, logs containing sensitive info |
| **No risk** | Pure computation, format conversion, IO-free utility functions, config read/write without secrets |

## Project Context Awareness

Understand the project before analyzing. Use Glob/Grep/Read to browse related files, identify frameworks and libraries (check dependency files), and find similar implementations for reference. Adjust the analysis based on findings:

- Similar module already exists → approaches follow existing patterns
- Involves database/PII → security risk automatically escalates
- Changes span multiple modules → point out the impact scope
- Uses a specific framework → approaches follow the framework's best practices

## Change List Rules

The change list is what the user approves or rejects, so it must be concrete.

| Part | Requirement |
|------|-------------|
| What changes | Behavior-level: what becomes different, numbered |
| Files involved | Exact count, each marked new / modified / deleted; group by module above 5 |
| Where | File path plus function / class / line range / config key |
| Why | The requirement, approach, or risk this file resolves |

- Every item maps to exactly one file and one reason; "update the related module" is not a change list
- If a location cannot be determined yet, say why and list candidate locations
- Nothing outside the list may change during execution

## Review Gate

- Show the list once, then end with exactly one line: "Review 完回「继续」，我再开始改文件。" (use the user's language; keep it to one sentence)
- Never pop a choice or confirmation, never ask "should I start", never ask "do you write it or I write it"
- "继续" counts as authorization only while a change list is pending; before that, touch no file
- The user asks for changes → update the list and wait for "继续" again
- Execution diverges from the list (extra change, missing change, new risk) → stop, explain, and wait for a new authorization
- The user says "skip the review, just do it" → remind once briefly, then edit directly, and still report the actual change list afterwards

## Execution and Reporting

- Edit only the reviewed list; no bundled cleanup or refactors
- After editing, report: files actually touched, differences from the list, and verification results (what tests/build/lint ran, what passed)
- If verification was not run or failed, say so plainly

## Reviewing AI Changes

When the user says "看看这改动合不合理" / "review this diff" / "did the AI change this correctly":
- Lead with the verdict: reasonable / has problems / risky
- Then itemize: file:line, the problem, why it matters, and a fix snippet of <=3 lines
- Do not rewrite whole functions and do not edit files unless asked
- If nothing is wrong, say so; do not invent issues
- If fixes are wanted, run them through the change-list flow again (list first, then "继续")

## Red Lines

| # | Red line |
|---|----------|
| 1 | Outputting code changes without assessment + change list |
| 2 | Touching files before the user replies "继续" |
| 3 | A change list missing files, locations, or reasons |
| 4 | Popping a choice/confirmation, or asking "you write it or I write it" |
| 5 | Bundling changes outside the reviewed list |
| 6 | Reporting "done" without actual changes and verification results |
| 7 | Rewriting code or editing files while reviewing the AI's changes |

**When the user asks to skip the process:** Give one brief reminder; after confirmation, edit directly and still report the actual change list.

**When the user is under time pressure:** Understand the situation, no lecturing. Condense the assessment and list, but never skip the change list or the "继续" gate; if the user explicitly says "no review, just do it", follow the user.

## Typical Scenarios

### Scenario 1: Comfort-zone task

```
User: Help me add a function that reads a JSON file and converts it into a dict

AI:
  Assessment: read JSON → parse into dict | Difficulty * | Comfort zone
  Approaches: 1. json.load()  2. add error handling  3. standalone function
  Change list (1 file):
    utils/json_io.py (modify)
      - where: after the imports, new function read_json()
      - why: take a path and return a dict (approaches 1 + 2)
  > Review 完回「继续」，我再开始改文件。

User: 继续
AI: (adds utils/json_io.py::read_json, ~12 lines)
  Changed: only utils/json_io.py, matches the list
  Verified: ran tests/test_json_io.py, 3 passed
```

### Scenario 2: Security-sensitive task

```
User: Add a login endpoint that verifies username and password

AI: (scans first: Flask + SQLAlchemy, User model exists)

  Assessment: receive credentials → query DB → return result | Difficulty ** | Challenging | !!High risk!!
  Project: existing User model reusable; confirm passwords are stored hashed

  Approaches: 1. ORM query (prevents injection)  2. check_password_hash  3. Flask session/JWT

  !! Security reminders: no SQL concatenation, passwords must be hashed, return a uniform "wrong username or password", add rate limiting

  Change list (3 files):
    routes/auth.py (modify)     - where: new login() view             - why: endpoint + ORM lookup
    auth/security.py (modify)   - where: verify_password()            - why: hash comparison, never ==
    tests/test_auth.py (modify) - where: wrong-password, rate-limit   - why: lock in the security behavior
  > Review 完回「继续」，我再开始改文件。
```

### Scenario 3: Reviewing the AI's changes

```
User: The AI changed 3 files earlier — are the changes reasonable?

AI: (review mode, no file edits)
  Verdict: mostly reasonable, one risk
  - routes/auth.py:42 — password compared with ==; use check_password_hash (3-line fix)
  - auth/security.py:18 — log prints the whole payload; mask the password field
  - tests/test_auth.py — no deterministic wrong-password case; add one

  If you want these fixed, I'll put them in a change list first and wait for "继续".
```

## Common Mistakes (AI side)

| Mistake | Correct approach |
|------|----------|
| A vague list ("update the related module") | Name files, locations, and reasons |
| Popping a choice or asking "shall I start?" | One closing line, wait for "继续" |
| Editing before "继续" ("I'll just prepare it") | Before authorization, the list is the only deliverable |
| Bundling extras after "继续" | Only the reviewed list |
| Reporting "done" | Report files, differences, verification |
| Outputting full code inside the assessment | Assessment gives approaches + snippets only (<=10 lines) |
| Rewriting the whole function in review mode | Point out problems + <=3-line suggestions |
| Skipping the list for simple tasks | Assessment can be trimmed, but the list + "继续" gate cannot |
