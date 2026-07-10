# prompt-builder

A Claude Code plugin that interviews you about a development task, then generates a
structured, copy-pasteable prompt for AI coding tools (Claude Code, Cursor, Copilot,
etc.) — as both a human-readable Markdown file and an agent-dispatchable JSON file.

Pure Markdown + JSON. No packages, build tools, or setup scripts.

## Install

**Option A — as a plugin** (recommended):

```
claude plugin marketplace add <path-or-git-url-of-this-repo>
claude plugin install prompt-builder@prompt-builder
```

(or interactively: `/plugin marketplace add …` then `/plugin install prompt-builder`)

**Option B — copy into a project.** Copy these into your project's `.claude/`:

```
skills/prompt-builder/   →  .claude/skills/prompt-builder/
agents/*.md              →  .claude/agents/
hooks/hooks.json         →  merge the "hooks" block into .claude/settings.json
```

If your project already has a `.claude/settings.json`, merge rather than overwrite.
This repo's own `.claude/` directory is exactly that copy-install, kept in sync for
local development — use it as a reference.

Either way there's no other setup: skills and agents are auto-discovered.

## Repo layout

```
.claude-plugin/
├── plugin.json               # plugin manifest (name, version)
└── marketplace.json          # lets this repo be added as a marketplace directly
skills/prompt-builder/SKILL.md # the /prompt-builder skill (templates live here)
agents/prompt-interviewer.md   # extracts requirements & gap questions
agents/prompt-composer.md      # renders templates, writes output files
hooks/hooks.json               # prints generated prompt paths after writes
.claude/                       # in-repo dev install of the same files
```

## Usage

```
/prompt-builder
```

or just describe a task ("I want to build...", "this endpoint is failing...").
The skill then:

1. **Detects the mode** — Build, Debug, or Refactor/Review
2. **Interviews you** — 3–5 focused questions, one or two at a time, skipping
   anything you already said
3. **Assesses complexity** — recommends simple (single file, clear scope) vs
   complex (multi-file, coordination), then asks *you* whether the scope
   warrants the extra orchestrated effort; your choice is final
4. **Generates the prompt** — mode-specific template for simple tasks; for complex
   tasks, an orchestrated prompt with a planning phase, specialist roles, and
   subagent execution instructions
5. **Writes two files** and offers refinements

A hook prints the generated file path after every write, e.g.:

```
📄 prompt-builder: generated prompt written to prompts/build-20260708-143012.md
```

## Output

Files land in `prompts/` at your project root, named `{mode}-{timestamp}`:

- **`prompts/build-20260708-143012.md`** — the prompt, ready to paste into any AI
  coding assistant
- **`prompts/build-20260708-143012.json`** — the same prompt as structured data:

```json
{
  "mode": "build",
  "complexity": "complex",
  "prompt": "<full markdown prompt text>",
  "subtasks": [
    {
      "role": "Implementer",
      "description": "Create the endpoint skeleton",
      "dependencies": [],
      "parallel": true
    }
  ],
  "acceptance_criteria": ["GET /health returns 200"]
}
```

`subtasks` appears only for complex tasks; `dependencies` holds zero-based indexes
of prerequisite subtasks. Consumers should ignore unknown keys.

## Customising

- **Complexity rubric**: edit Step 3 of `skills/prompt-builder/SKILL.md` (marked
  with a `TUNE` comment) to change when the heavyweight orchestrated template is
  recommended. (Copy-installs: edit your `.claude/skills/` copy instead.)
- **Templates**: all four templates live in the skill file's Step 4 — edit them in
  place; the `prompt-composer` agent reads them from there, so there's one source
  of truth.
- **Hook**: the notification hook in `hooks/hooks.json` uses POSIX `grep`/`sed`
  only (no jq). Remove it (or the merged block in `.claude/settings.json`) if you
  don't want the notification.

## Uninstall

Plugin install: `claude plugin uninstall prompt-builder`. Copy-install: delete the
three Markdown files and remove the merged entries from `.claude/settings.json`.
Optionally delete the `prompts/` directory.
