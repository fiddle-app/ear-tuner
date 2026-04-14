# Context Cleanup Plan

Revised plan based on the `/audit` findings and Casey's feedback in the cleanup discussion. This plan will be incorporated into a log entry when the work is complete.

---

## Goals

1. **Single source of truth** — every piece of information documented in exactly one place; other files link to it
2. **Minimal auto-loaded context** — CLAUDE.md files stay lean (pointers + behavioral instructions); detailed reference docs are read on-demand via links
3. **Correct information** — fix stale contradictions between files
4. **Cross-project conventions at the right level** — dev process and folder conventions live at `Projects/` level, not buried in fiddle-specific files

---

## New Files to Create

### `Projects/folder-conventions.md`
What each standard folder means. Lightweight reference, frequently needed.

| Folder | Purpose |
|---|---|
| `specs/` | Durable design documents. Main spec: `{app}-spec.md`. Smaller specs use `YYYY-MM-DD_{desc}.md` naming so you know which are newer. Smaller specs eventually merge into the main one. |
| `handoffs/` | Ephemeral workflow artifacts (briefs, done reports, test artifacts). Clean out after feature is committed. |
| `log/` | Session work logs. Written by the `/log` skill. Don't look inside unless asked about previous work. |
| `backlog/` | Detailed pages for specific backlog items. Managed by Barry (backlog-manager skill). |
| `research/` | Active research topics. Short-lived; results end up in specs, log entries, or skulch. |
| `research/skulch/` | Archived/discarded research. Ignore unless specifically asked about old research. |

### `Projects/dev-process.md`
The full development process. Only needed when really building software. Content migrates here from `fiddle/architecture.md` Sections 9-10 and parts of Section 2.

Contents:
- Three development modes (vibe coding, spec-driven, adversarial test pass)
- Manual coder terminal workflow
- Spec format (durable, `specs/` folder)
- Brief and handoff formats (ephemeral, `handoffs/` folder)
- The `/handover` skill
- Retro-spec rhythm
- Coder subagent system prompt
- Tester subagent system prompt
- Codebase skeleton generation (ctags workflow)
- JSDoc convention
- Testing strategy (tiered testing, Vitest, Playwright, data-testid, post-hoc testing)

> [!question] Fiddle-specific content in dev-process.md
> The subagent prompts currently reference "Fiddle App family" and fiddle-specific conventions (vanilla JS, platform adapter). When Casey starts non-fiddle projects, these prompts may need to be parameterized or split into a generic template + fiddle overlay. For now, keep them as-is since fiddle is the only active project family.

A: 

### `ear-tuner/CLAUDE.md`
Concise module file for the current active project (~25 lines). See Phase 5 below.

### `media-markup/specs/media-markup-spec.md`
Receives the detailed content extracted from `media-markup/CLAUDE.md`. See Phase 3 below.

---

## Phase 1: Create the two Projects-level files

### `Projects/folder-conventions.md`
- Write the folder convention table (content above)
- Add a note that `backlog.md` files contain work item lists; `backlog/` folders contain detail pages
- Keep it short — under 30 lines

### `Projects/dev-process.md`
- Migrate content from `fiddle/architecture.md` Sections 9 and 10 (Development Workflow, Testing)
- Migrate JSDoc convention from `fiddle/architecture.md` Section 2
- Generalize where possible (the process patterns are cross-project; fiddle-specific examples stay as examples)
- Update all internal cross-references

### Update `Projects/CLAUDE.md`
- Add pointers to the two new files:
  ```
  - [folder-conventions.md](folder-conventions.md) — standard folder meanings (specs, log, backlog, research)
  - [dev-process.md](dev-process.md) — development workflow, testing strategy, spec/handoff formats
  ```
- Change "Ignore the `backlog.readme.md` file" to "Ignore files named `backlog-readme.md` or `backlog.readme.md`"
- The existing folder behavioral notes (ignore log/, skulch/, backlog/) stay — they're behavioral instructions, not reference docs. But they can reference `folder-conventions.md` for details.

---

## Phase 2: Slim down `fiddle/CLAUDE.md` (181 -> ~80 lines)

### Remove (content exists elsewhere or is redundant):

| Section | Lines | Reason |
|---|---|---|
| About the Developer | 12-17 | Covered by Casey's global Claude profile |
| Folder Structure diagram | 33-45 | SSOT is in `APPS.md` |
| Shared Data Architecture (full section) | 49-84 | SSOT is `APPS.md` (data flow) + `architecture.md` (data architecture). The version in CLAUDE.md is *stale* — still documents old media-annotations/ approach and wrong Tune List data source |
| Per-Tune Markdown Format | 88-90 | One-liner pointer is fine; the reference to `tune-hub/spec/tune-md-format.md` can stay as a single line |
| Coding Conventions | 107-117 | Fiddle-specific conventions (vanilla JS, no frameworks) stay in `architecture.md` Section 2. Cross-project conventions (JSDoc, etc.) move to `Projects/dev-process.md` |
| WPA -> Swift Portability | 120-158 | Fully covered in `architecture.md` Sections 2 and 7 |
| Backlog System (verbose) | 162-173 | Replace with 2-line note (see below) |

### Keep (behavioral instructions for Claude):

- **Project Overview** — trimmed to 3-4 lines (what this is, GitHub org, primary language)
- **Key References** — pointers to APPS.md, architecture.md (replaces the removed sections)
- **Shared Design System** — the behavioral rule "check `_shared/design/` before writing UI code" stays as a one-liner; the description of what's in it is in architecture.md Section 5
- **Per-Tune Markdown Format** — one-line pointer to `tune-hub/spec/tune-md-format.md`
- **Backlog** — trimmed to: "Managed by the backlog-manager skill ('Barry'). See `backlog.md` in this folder or any app folder."
- **Licensing & Attribution** — keep (behavioral instruction)

### Resulting structure (~80 lines):

```
# Fiddle App Family — Umbrella Claude Context
(intro paragraph, 4 lines)

## Key References
(pointers to APPS.md, architecture.md, 5 lines)

## Project Overview
(brief orientation, 4 lines)

## Shared Design System
(behavioral rule, 3 lines)

## Per-Tune Markdown Format
(one-line pointer)

## Backlog
(2 lines)

## Licensing & Attribution
(5 lines)
```

---

## Phase 3: Slim down `media-markup/CLAUDE.md` (167 -> ~35 lines)

### Create `media-markup/specs/media-markup-spec.md`

Move these sections from CLAUDE.md into the spec:
- Core Workflow (First Launch, Normal Launch, Annotating a File)
- Keyboard Shortcuts design
- Platform Adapter (full interface definition, code examples, module construction pattern)
- Detailed data architecture (SQL examples, attach pattern, preferences)

### Keep in CLAUDE.md (~35 lines):
- Purpose (3 lines)
- Status
- Target Platform — "Electron desktop-first. See [spec](specs/media-markup-spec.md) for rationale."
- Data Architecture summary — "Owns `media-markup.db`. Attaches `tunehub.db` read-only. See [spec](specs/media-markup-spec.md) and [architecture.md Section 4](../architecture.md)."
- Notes (media files stay in OneDrive, doesn't write to tunehub.db) — keep, these are quick-reference behavioral guardrails
- Link to parent CLAUDE.md

### Update `architecture.md` Section 7
- Change "The full adapter interface for MM is defined in media-markup/CLAUDE.md" to point to `media-markup/specs/media-markup-spec.md` instead.

---

## Phase 4: Trim `fiddle/architecture.md`

### Remove (migrating to `Projects/dev-process.md`):
- Section 9: Development Workflow (all of it — three modes, spec-driven, handoffs, subagent prompts, skeleton generation)
- Section 10: Testing (Vitest, Playwright, tiered testing, post-hoc testing)
- Section 2 partial: JSDoc Convention subsection

### Keep:
- Section 1: Platform Decisions
- Section 2: Technology Stack (minus JSDoc) — vanilla JS rationale, framework rejection, bundling
- Section 3: Performance Techniques
- Section 4: Data Architecture
- Section 5: Shared Design System
- Section 6: Data Robustness & Longevity
- Section 7: Platform Adapter Pattern
- Section 8: Electron Security
- Section 11: Development Order
- Section 12: Electron Shared Setup
- Section 13: Electron Packaging & Distribution
- Section 14: GitHub Repo Structure & Deploy Pipeline

### Add:
- Pointer at the top: "For development process, testing strategy, and folder conventions, see [dev-process.md](../dev-process.md) and [folder-conventions.md](../folder-conventions.md) at the Projects level."

### Renumber sections after removal.

---

## Phase 5: Create `ear-tuner/CLAUDE.md` (~25 lines)

```markdown
# Ear Tuner

See parent [fiddle/CLAUDE.md](../CLAUDE.md) for project context.

## Purpose

Ear training and pitch discrimination tool. Plays two tones and asks the user
to identify which is higher/lower or how they differ. Designed for fiddle
players developing relative pitch.

## Status

Built (WPA); design review pending.

## Target Platform

WPA today, served via GitHub Pages (`fiddle-app.github.io/ear`).
Future: Capacitor iOS wrap for iPhone/iPad + App Store.

## Key Files

- `index.html` — single-file app (HTML + CSS + JS)
- `sounds/` — audio sample files
- `specs/ear-tuner-spec.md` — design spec (fonts, colors, UI decisions)

## Known Issues

- iOS silent switch mutes Web Audio API output (workaround in place since commit 9d4d770)
- Bell volume reduced 17% to avoid startling users
```

---

## Phase 6: Fix contradictions and broken references

1. **Data flow stale info** — Removed from `fiddle/CLAUDE.md` in Phase 2 (the stale version). SSOT for data flow is `APPS.md`, which is already correct.

2. **`backlog-readme.md` broken reference** — Removed from `fiddle/CLAUDE.md` in Phase 2. Replaced with 2-line backlog note.

3. **`Projects/CLAUDE.md` backlog.readme.md reference** — Updated in Phase 1 to handle naming variations.

4. **Intonio** — Add to `APPS.md` registry:
   ```
   | Intonio | `intonio` | — | WPA (idea) | Idea | Live frequency spectrum visualization; identifies notes being played on fiddle, displays on musical staff with sharp/flat indication |
   ```
   Add to APPS.md folder structure diagram. Remove the confusing "Intonio" mention from `fiddle/CLAUDE.md` backlog section (which is being trimmed anyway).

5. **architecture.md Section 7 cross-reference** — Updated in Phase 3.

---

## Phase 7: Trim stubs

Replace `_shared/CLAUDE.md` and `microbreaker/CLAUDE.md` (15 lines each) with 3-line files:

```markdown
# {name}
See parent [fiddle/CLAUDE.md](../CLAUDE.md). Development not yet started.
```

---

## Execution Order

The phases have dependencies. Recommended execution order:

1. **Phase 1** — Create `Projects/folder-conventions.md` and `Projects/dev-process.md`. These are net-new files with no destructive changes.
2. **Phase 4** — Trim `architecture.md` (remove sections that moved to dev-process.md). Must happen after Phase 1 so the content has a home.
3. **Phase 2** — Slim `fiddle/CLAUDE.md`. Can happen after Phase 1 (needs folder-conventions.md to exist for links).
4. **Phase 3** — Slim `media-markup/CLAUDE.md` and create its spec. Independent of Phases 1-2.
5. **Phase 5** — Create `ear-tuner/CLAUDE.md`. Independent.
6. **Phase 6** — Fix remaining contradictions. Some are handled by Phases 2-4; the rest (Intonio in APPS.md, Projects/CLAUDE.md tweak) are independent.
7. **Phase 7** — Trim stubs. Independent, low priority.

Phases 3, 5, 6, and 7 are independent of each other and can be done in any order after Phase 1.

---

## Expected Outcome

| File | Before | After |
|---|---|---|
| `Projects/CLAUDE.md` | 76 lines | ~80 lines (minor additions) |
| `Projects/folder-conventions.md` | — | ~25 lines (new) |
| `Projects/dev-process.md` | — | ~280 lines (new, from architecture.md) |
| `fiddle/CLAUDE.md` | 181 lines | ~80 lines |
| `fiddle/architecture.md` | 815 lines | ~530 lines |
| `fiddle/APPS.md` | 100 lines | ~105 lines (Intonio added) |
| `media-markup/CLAUDE.md` | 167 lines | ~35 lines |
| `media-markup/specs/media-markup-spec.md` | — | ~130 lines (new, from CLAUDE.md) |
| `ear-tuner/CLAUDE.md` | — | ~25 lines (new) |
| `_shared/CLAUDE.md` | 15 lines | 3 lines |
| `microbreaker/CLAUDE.md` | 15 lines | 3 lines |

### Context load after cleanup

| Working in... | Files loaded | Before | After |
|---|---|---|---|
| ear-tuner/ | Projects + fiddle + ear-tuner | 257 | ~185 |
| media-markup/ | Projects + fiddle + media-markup | 424 | ~195 |
| tune-list/ | Projects + fiddle + tune-list | 315 | ~218 |
| minions/larry/ | Projects + fiddle + minions + larry | 349 | ~252 |
| _shared/ | Projects + fiddle + _shared | 272 | ~163 |

All contexts under 250-line budget (larry/ is tight but within tolerance).
