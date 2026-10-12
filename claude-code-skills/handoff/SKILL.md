---
name: handoff
description: Compact the current conversation into a handoff document so a fresh session can continue with minimal re-loaded context. Use when the user wants to save state, pause a session, wrap up before a break, or explicitly asks for a handoff. Also use to resume from a prior handoff.
argument-hint: "What will the next session focus on?"
triggers:
  - /handoff
  - create a handoff
  - save state
  - let's pause here
  - I need to stop for today
  - wrap up this session
  - load handoff
  - resume from
  - continue where we left off
---

# Handoff

Two modes: **writing** a handoff (default) and **resuming** from one. Pick the mode based on what the user asked for.

## Writing a handoff

### Step 1 — Get the timestamp

Run a shell command to get the current date/time — do not guess it:

```
date +"%Y-%m-%d-%H%M"
```

(On Windows PowerShell: `Get-Date -Format "yyyy-MM-dd-HHmm"`)

This becomes `<timestamp>` in the output filename below.

### Step 2 — Gather the content

Review the conversation and draft each section below. **Omit any section with nothing to report** — don't leave empty placeholders.

- **Goal** — what this session was trying to accomplish. If the user passed arguments to the invocation (e.g. `/handoff <text>`), treat `<text>` as what the *next* session will focus on and tailor this section to set that up.
- **Completed** — concrete work done: files touched, features built, bugs fixed.
- **Decisions** — key choices made and *why*, not just what.
- **What Worked / What Didn't** — approaches tried and their outcome, so the next session doesn't repeat a dead end.
- **Open Issues / Known Constraints** — anything unresolved, blocking, or worth flagging.
- **Next Steps** — concrete, ordered action items for the next session.
- **References** — paths or URLs to specs, plans, ADRs, issues, commits, diffs that already capture detail. Link, don't duplicate their content.
- **Commands to Continue** — exact commands needed to pick up where this session left off (build, test, run), if applicable.
- **Suggested Skills** — which skills the next agent should invoke first, based on what kind of work remains.
- **Candidate for CLAUDE.md** — durable facts noticed this session (new conventions, standing decisions, constraints) that look reusable beyond this task. This is a flag for the user or the `claude-md-management:revise-claude-md` skill to act on later — never edit `CLAUDE.md` directly from this skill.

### Step 3 — Redact secrets

Scan the drafted content for anything that looks like an API key, token, password, connection string, or personally identifiable information. Replace each with `[REDACTED]` in place. This is a judgment-based re-read of your own draft, not a regex tool — err on the side of redacting when unsure.

### Step 4 — Write the file

Path: `handoffs/<timestamp>.md` relative to the current project's working directory (create the `handoffs/` directory if it doesn't exist).

Use this structure, including only the sections that have content:

```markdown
# Handoff — <timestamp>

## Goal
...

## Completed
...

## Decisions
...

## What Worked / What Didn't
...

## Open Issues / Known Constraints
...

## Next Steps
...

## References
...

## Commands to Continue
...

## Suggested Skills
...

## Candidate for CLAUDE.md
...
```

### Step 5 — Confirm and offer to clear

Print the file path to the user and ask them to confirm it looks right. Once confirmed, run `/clear` in the same turn. Never run it without that explicit go-ahead — `/clear` wipes visible context and is not reversible within the session.

## Resuming from a handoff

When the user asks to load, resume, or continue from a handoff:

1. Find the most recent file matching `handoffs/*.md` in the current project (sort by filename — the timestamp format sorts chronologically).
2. Read it.
3. Summarize its **Next Steps** section as the starting task list for this session, and mention its **Suggested Skills** section if present.
4. If no `handoffs/` directory or no matching file exists in the current project, say so plainly instead of guessing at prior context.
