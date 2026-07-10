---
name: prompt-builder
description: >
  Interviews the user about a development task (build, debug, or refactor/review),
  assesses its complexity, and generates a structured, copy-pasteable prompt for AI
  coding tools — written to prompts/{mode}-{timestamp}.md with an agent-dispatchable
  JSON companion file. Use when the user invokes /prompt-builder, asks for help
  writing or structuring a development prompt, or describes a coding task they want
  to hand to an AI coding assistant such as Claude Code, Cursor, or Copilot.
---

# Prompt Builder

Interview the user, then generate a structured development prompt in two formats: a
human-readable Markdown file and an agent-dispatchable JSON companion.

A good generated prompt is:

- **Scoped** — clear about what to do AND what not to touch
- **Contextual** — enough background to avoid wrong assumptions
- **Verifiable** — defines what "done" looks like
- **Mode-appropriate** — tailored to Build, Debug, or Refactor/Review

Follow the steps in order. Never skip the interview, but DO skip any question the
user has already answered — re-asking known information wastes their time.

---

## Step 1 — Detect the mode

Classify the task from what the user has said so far:

- **build** — new feature, file, endpoint, component; greenfield work
- **debug** — something is broken, erroring, or behaving unexpectedly
- **refactor** — existing code needs improving, cleaning up, or reviewing (covers code review requests too)

If the mode is unclear, ask directly:

> "Are you looking to **build** something new, **debug** a problem, or **refactor/review** existing code?"

State the detected mode back to the user so they can correct it.

## Step 2 — Interview (3–5 questions, conversational)

Ask **1–2 questions per message**, never the whole list at once. Use earlier answers
to decide what still needs asking. Stop as soon as you can fill the template —
usually 3–5 questions total. If the AskUserQuestion tool is available, use it with
concrete options where they exist.

**Build questions** (pick the unanswered ones):
1. What do you want to build? (one sentence)
2. What's the tech stack / language / framework?
3. Any constraints? (libraries to avoid, files to stay within, keep it simple)
4. How will we know it's working? (test passes, endpoint returns X, UI shows Y)
5. Any relevant context? (part of a larger system, follows an existing pattern)

**Debug questions:**
1. What's going wrong? (the symptom)
2. What's the error message or unexpected output, if any?
3. Which file / function / component is involved?
4. What environment? (language, framework, runtime, versions if relevant)
5. What have you already tried?

**Refactor/Review questions:**
1. What's the goal? (readability, deduplication, performance, assessment)
2. Which file(s) or section(s) should change?
3. What must NOT change? (behaviour, API surface, specific logic)
4. Any style or pattern conventions to follow?
5. How do we verify nothing broke? (existing tests pass, manual check)

**Delegation option:** if the user's initial description is long or ambiguous and the
Agent tool is available, dispatch the `prompt-interviewer` agent with the mode and
the raw description. It returns extracted requirements plus ranked gap questions —
then ask the user only the gaps. If the Agent tool is unavailable, do this analysis
inline.

## Step 3 — Assess complexity (the user makes the final call)

<!-- TUNE: this rubric drives the recommendation for when the heavyweight
     orchestrated template kicks in. Adjust the triggers to your team's taste. -->

First form a **recommendation**. Lean **complex** if any of these hold:

- Touches multiple files or modules
- Requires new routing, auth, data models, or API integration
- Has multiple distinct functional requirements
- Involves coordination between frontend and backend
- The user seems unsure about scope or implementation approach

Otherwise lean **simple**: single file or component, clearly scoped, one outcome,
obvious implementation path.

Then **ask the user whether the scope requires the extra effort** — never decide
silently. This counts as one of your 3–5 interview questions. State your
recommendation, the trigger(s) behind it, and what each choice means, e.g.:

> "This looks **complex** to me (touches both the API and the UI). Want the
> **orchestrated** prompt — plan-first, specialist subtasks, review phases — or
> a lighter **simple** single-template prompt?"

If the AskUserQuestion tool is available, use it with two options — "Simple —
quick single prompt" and "Complex — orchestrated with planning & review" — and
put your recommendation first, labelled "(Recommended)".

The user's choice is final: record it as `complexity` for Step 4 and the JSON
companion, even if it disagrees with the rubric.

## Step 4 — Compose the prompt

- **simple** → use the matching mode template below (Build, Debug, or Refactor/Review).
- **complex** → use the Orchestrated template, which wraps the mode template's
  sections and adds Plan / Execute / Final Review phases with specialist roles.

Adapt, don't force: **omit any section that has no real content** for this task
(e.g. skip "What I've Already Tried" if the user tried nothing) and rename sections
if the user's domain calls for it. Never emit placeholder text like "[TBD]" in the
final prompt.

Keep all content generic to the user's project — take stack, paths, and commands
from their answers, never from these examples.

**Delegation option:** if the Agent tool is available, you may dispatch the
`prompt-composer` agent with the mode, complexity, the interview results, and this
skill's base directory — so it can load the templates from this SKILL.md wherever
the plugin is installed. It renders the templates and writes both output files
(Step 5), then returns the file paths. Otherwise compose and write inline.

### Build template

```markdown
## Task
[One sentence: what to build]

## Context
- Tech stack: [language / framework / libraries]
- Location: [file, module, or area of the codebase]
- Background: [how this fits into the larger system]

## Requirements
- [Requirement 1]
- [Requirement 2]

## Constraints
- Do NOT modify: [files, behaviour, or APIs to leave untouched]
- Avoid: [libraries, patterns, or approaches to steer clear of]
- Follow: [conventions or architectural patterns already in use]

## Acceptance Criteria
- [ ] [Verifiable outcome]
- [ ] [How to verify: e.g. run the project's test command, check endpoint returns X]
```

### Debug template

```markdown
## Problem
[One sentence: what's going wrong]

## Environment
- Language / framework: [e.g. Python 3.12 / FastAPI]
- Relevant file(s): [paths]
- Runtime / OS: [if relevant]

## Symptom
[What the user sees — error message, wrong output, crash]

## Steps to Reproduce
1. [Step]
2. [Step]

## What I've Already Tried
- [Attempt and result]

## Goal
[What "fixed" looks like — e.g. "the test passes", "no more 500 error"]
```

### Refactor / Review template

```markdown
## Task
[One sentence: the refactor or review goal]

## Target
- File(s): [paths]
- Scope: [function, class, module, or whole file]

## Goal
[Why: reduce duplication, improve readability, extract reusable logic, assess quality]

## Constraints
- Do NOT change: [behaviour, API surface, logic that must stay]
- Follow: [existing patterns, style guide, naming conventions]

## Verification
- [ ] [Existing tests still pass]
- [ ] [Manual check: X still works]
- [ ] [No change to public API / signatures]
```

### Orchestrated template (complex tasks only)

Start with the matching mode template's sections above, then append:

```markdown
## Before You Begin
Read the codebase to understand existing patterns, conventions, and architecture.
Do not write any code yet.

---

## Phase 1: Plan (do not write code yet)

Produce a structured implementation plan:

### Requirements Restatement
[Concise restatement of what will be built/fixed/refactored]

### Implementation Phases
Break work into small, independent subtasks. Each subtask must:
- Touch as few files as possible
- Have a single clear outcome
- Be assigned a specialist role (table below)
- Be marked sequential or parallel

### Specialist Roles
| Role | Responsibility |
|------|---------------|
| Implementer | Write the feature/fix code |
| TDD Guide | Write tests first (RED → GREEN → REFACTOR) |
| Code Reviewer | Check quality, error handling, size limits, dead code |
| Security Reviewer | Check secrets, injection, input validation |
| Build Fixer | Resolve build/type errors with minimal changes only |
| Doc Updater | Update documentation to reflect changes |

### Dependencies & Risks
- Dependencies: [external services, APIs, packages]
- HIGH / MEDIUM / LOW risks with mitigations

### Estimated Complexity: [HIGH / MEDIUM / LOW]

**WAITING FOR CONFIRMATION**: Proceed with this plan? Reply yes/no/modify before
any code is written.

---

## Phase 2: Execute (after plan is approved)

- **Subagent isolation**: treat each subtask as a fresh context; pass only what's
  explicitly needed, carry no assumptions between subtasks.
- **TDD for implementation subtasks**: write failing tests first, verify they fail,
  implement minimally, verify they pass, then refactor.
- **Quality review after each subtask**: spec compliance (no more, no less), errors
  handled, no hardcoded values or secrets, input validated. Fix Critical/Important
  findings before the next subtask.
- **Parallelise** independent subtasks; keep dependent ones in order.
- **Checkpoint after each phase**: tests passing/failing, build status, completed
  tasks, blockers, next steps.

---

## Phase 3: Final Review

1. Run the project's full verification (type check, lint, tests, build).
2. Whole-change review: security, quality, test coverage, docs.
3. Fix any Critical or High findings before marking complete.
```

## Step 5 — Write the output files

1. Get a filesystem-safe timestamp by running: `date +%Y%m%d-%H%M%S`
   (never guess the time; no colons or spaces in filenames).
2. Write the Markdown prompt to `prompts/{mode}-{timestamp}.md` relative to the
   project root, where `{mode}` is `build`, `debug`, or `refactor`. Create the
   `prompts/` directory if it doesn't exist. Never put user-supplied text in the
   filename.
3. Write the JSON companion to `prompts/{mode}-{timestamp}.json` (same basename):

```json
{
  "mode": "build | debug | refactor",
  "complexity": "simple | complex",
  "prompt": "<the full Markdown prompt text>",
  "subtasks": [
    {
      "role": "Implementer",
      "description": "One-sentence subtask outcome",
      "dependencies": [0],
      "parallel": true
    }
  ],
  "acceptance_criteria": ["Verifiable outcome 1", "Verifiable outcome 2"]
}
```

- `subtasks` is present only for complex tasks (omit the key for simple ones);
  `dependencies` lists the zero-based indexes of subtasks that must finish first;
  `parallel: true` means it can run alongside other subtasks whose dependencies
  are also met.
- `acceptance_criteria` mirrors the checklist in the Markdown prompt, one string
  per criterion, always present.
- Additional keys may be added later; consumers must ignore unknown keys.

4. Show the user the full generated prompt in the conversation as well — the files
   are for reuse, the chat copy is for immediate review.

## Step 6 — Offer follow-ups

Offer at least two of:

- "Want me to refine any section?"
- "Should I make the acceptance criteria more specific?"
- "Want me to tighten the scope / split this into two prompts?"
- (simple tasks only) "Want the orchestrated version instead, with planning and
  specialist roles?"

Apply requested refinements by rewriting **both** output files, keeping the same
filenames.
