# Backlog Management System

The backlogs in this project (`backlog.md` and `backlog-done.md`) are managed by the **backlog-manager skill**. You can add items manually in any loose format — the skill will clean them up, assign IDs, and sort everything for you.

-----

## Quick Start

### Status and type at a glance

**Priority (`P` column) — shown first because it drives sort order:**

|Value|Meaning                        |
|-----|-------------------------------|
|P1   |Urgent                         |
|P2   |Normal (default if unspecified)|
|P3   |Someday / maybe                |

**Status (`S` column):**

|Value|Meaning                             |
|-----|------------------------------------|
|`.`  |Open / not started                  |
|`~`  |In progress                         |
|`!`  |Blocked                             |
|`x`  |Done (moved to `backlog-done.md`)   |
|`-`  |Dropped (moved to `backlog-done.md`)|

**Item type (first letter of ID):**

|Letter|Type         |Description                                                                     |
|------|-------------|--------------------------------------------------------------------------------|
|B     |Bug          |Defects, regressions, broken behavior                                           |
|C     |Coding       |New features, enhancements, implementation                                      |
|D     |Design       |UI/UX, visual design, mockups, style decisions                                  |
|H     |Human        |Actions only a human can take (call someone, record something, solicit feedback)|
|P     |Planning     |Architecture decisions, scoping, research, design spikes                        |
|T     |Testing      |QA, validation, device testing, user testing                                    |
|X     |Cross-project|Coordination tasks touching multiple projects or shared resources               |

-----

### Adding items manually

Just append a plain line to the bottom of `backlog.md`. You don’t need to worry about formatting, IDs, or priority. The skill will handle it on the next cleanup.

If you know the type, lead with the letter and a colon:

```
C: add support for ABC file import
B: volume control unresponsive on first tap
H: ask Rhys to review the tuning display
```

Include a priority anywhere in the line if you want to specify one:

```
P1 C: fix the focus-loss silence bug
H: ask Rhys to review the tuning display P3
B: volume control unresponsive P1 on first tap
```

If you omit priority, the skill defaults to P2.

You can add loose lines anywhere in the file — including between existing table rows. The skill will find and process them wherever they appear. If you copy an existing row and paste it nearby to use as a starting point, delete the numeric part of the ID (leave just the category letter) or delete the ID entirely. If a duplicate ID is detected, the first occurrence is kept and the second gets a fresh ID assigned.

To mark an item done, put an `x` anywhere near its current status indicator:

```
.x    x.    ~x    x~    !x
```

-----

### Invoking the skill

- **Clean up** the backlog (reformat loose lines, assign IDs, sort by priority):
  - *“Clean up the backlog”*
- **Add** an item (skill adds, formats, and re-sorts immediately):
  - *“Add a backlog item to have Rhys review the app and give feedback”*
  - *“Add a P1 bug — losing audio after focus loss”*
  - *“Add a someday item to explore a dark mode”*
- **Update** status or priority (re-sorts after every update):
  - *“Mark B1 done”*
  - *“Mark C3 blocked”*
  - *“Drop P2 — not pursuing that approach”*
  - *“Change H1 to P1”*
- **Query** the backlog:
  - *“What’s still open in the coding category?”*
  - *“Show me everything that’s blocked”*
  - *“What P1 items are left?”*
- **Initialize** a new backlog in the current folder:
  - *“Initialize a new backlog here”*
- **Add** a child project to this backlog family:
  - *“Add a new child project called TuneReview”*
- **Disconnect** all child backlogs from this family:
  - *“Disconnect all child backlogs”*
  - ⚠️ The skill will warn you and list every file that will be modified before proceeding.

-----

### Cross-referencing items in other backlogs

Each `backlog.md` has a collapsible **Prefixes** section at the top listing the other projects in this family and their prefix letters. To reference an item in another backlog, use square brackets with the prefixed ID:

```
[EC3]    (Ear Tuner, item C3)
[PP1]    (Parent/family backlog, item P1)
[HC2]    (TuneHub, item C2)
```

The parent backlog always uses the reserved prefix **`P`**. No child project may use `P` as its prefix.

Use cross-references in the Notes field of any item:

```
| X1 | P2 | . | Coordinate settings icon across apps | See [ED1], [MD1] |
```

When cleanup encounters a cross-reference like `[EC3]`, it converts it to a clickable relative hyperlink and adds an invisible HTML anchor tag to the target row’s Notes cell so the link resolves correctly:

```markdown
[EC3](../ear-tuner/backlog.md#EC3)
```

The anchor in the target file looks like this (appended after any existing Notes text):

```
| C3 | P2 | . | Add Copy Log button | Needs C1 <a id="EC3"></a> |
```

Anchor tags are only added when a row is actually referenced — they serve as a useful signal that something elsewhere is pointing to that item. When cleanup adds an anchor to a file in another project folder, it will tell you so you know to commit and push that repo as well.

-----

## Backlog Format

### backlog.md

```markdown
# Ear Tuner — Backlog

<details>
<summary>Prefixes (for referencing items in other backlogs)</summary>

| Prefix | Project |
|--------|---------|
| P | Fiddle App Family (parent) |
| B | MicroBreaker |
| E | Ear Tuner (this project) |
| I | Intonio |
| H | TuneHub |
| M | Media Markup |
| L | TuneList |

</details>

| ID | P  | S | Description | Notes |
|----|-----|---|-------------|-------|
| B1 | P1 | ! | Fix focus-loss audio silence bug | iOS only |
| C1 | P2 | ~ | Implement localStorage logging circular buffer | Group 2 |
| H1 | P2 | . | Decide on max log retention count | Before C1 |
| C2 | P3 | . | Add Copy Log button to Settings | Needs C1 |
```

### backlog-done.md

```markdown
# Ear Tuner — Done & Dropped

| ID | P  | S | Description | Notes |
|----|-----|---|-------------|-------|
| B2 | P2 | x | Fix BFCache iOS issue | |
| P1 | P3 | - | Investigate Web Audio worklet approach | Superseded by simpler fix |
```

Done items are appended in the order they were completed and never re-sorted.

-----

## Sort Order

After cleanup, open items are sorted by priority, then by status within each priority:

```
P1 !  (urgent + blocked)
P1 ~  (urgent + in progress)
P1 .  (urgent + not started)
P2 !
P2 ~
P2 .
P3 !
P3 ~
P3 .
```

-----

## ID Sequences

- Each category letter has its own sequence: `B1 B2 B3`, `C1 C2 C3`, etc.
- Numbers are never reused, even after an item is dropped or completed.
- The current high-water mark for each category is stored in `backlog-meta.json`. Do not edit that file manually.
- All ID letters are uppercase.

-----

## Prefixes

- The parent backlog always uses the reserved prefix **`P`**.
- Child prefixes are chosen when the child is added and must be unique and must not be `P`.
- The skill will display all in-use prefixes when prompting for a new child prefix.

-----

## backlog-meta.json

This file records the project’s position in the hierarchy and the current ID high-water marks.

### Parent folder

```json
{
  "level": "parent",
  "prefix": "P",
  "name": "Fiddle App Family",
  "children": [
    { "name": "MicroBreaker", "prefix": "B" },
    { "name": "Ear Tuner", "prefix": "E" },
    { "name": "Intonio", "prefix": "I" },
    { "name": "TuneHub", "prefix": "H" },
    { "name": "Media Markup", "prefix": "M" },
    { "name": "TuneList", "prefix": "L" }
  ],
  "ids": { "B": 0, "C": 0, "D": 0, "H": 0, "P": 0, "T": 0, "X": 0 }
}
```

### Child folder

```json
{
  "level": "child",
  "prefix": "E",
  "name": "Ear Tuner",
  "parent": { "name": "Fiddle App Family", "prefix": "P" },
  "siblings": [
    { "name": "MicroBreaker", "prefix": "B" },
    { "name": "Intonio", "prefix": "I" },
    { "name": "TuneHub", "prefix": "H" },
    { "name": "Media Markup", "prefix": "M" },
    { "name": "TuneList", "prefix": "L" }
  ],
  "ids": { "B": 0, "C": 0, "D": 0, "H": 0, "P": 0, "T": 0, "X": 0 }
}
```

A parent with no children registered yet is treated identically to a standalone backlog — just a parent waiting for children to be added.

-----

## Skill Behavior and Guardrails

**Initialization:**

- Creates `backlog.md`, `backlog-done.md`, and `backlog-meta.json` in the current folder.
- Always initializes as a parent-level backlog. Children are added separately.
- Copies `backlog-readme.md` into the folder.
- Will not overwrite existing backlog files.

**Adding a child project:**

- Must be run from the parent folder context.
- Skill displays all in-use prefixes and prompts for the new child’s prefix.
- `P` is always shown as reserved for the parent.
- Creates backlog files in the child folder if none exist. If files already exist, updates `backlog-meta.json` and the prefix table only — never overwrites backlog content.
- Refuses if run from a child folder (two-level limit).
- Refuses if the target child folder already contains a parent-level backlog.

**Disconnecting all children:**

- Removes hierarchy from `backlog-meta.json` in the parent and all child folders.
- Leaves prefix tables in `backlog.md` files intact but adds a note that the backlog has been disconnected from its family.
- Skill explicitly lists all files that will be modified and requires confirmation before proceeding.
- All affected backlogs become independent parent-level backlogs after disconnect.

-----

## Folder Structure

Each folder in the hierarchy contains:

|File               |Purpose                           |
|-------------------|----------------------------------|
|`backlog.md`       |Active items                      |
|`backlog-done.md`  |Completed and dropped items       |
|`backlog-meta.json`|ID counters and hierarchy metadata|

The parent folder also contains `backlog-readme.md` (this file). Child folders do not get their own copy.

Child folders may be part of the same source control repo as the parent, or they may be separate repos wired in as git submodules. Either way, cross-references between backlogs rely on relative file paths, so the parent-child folder hierarchy must remain stable on disk. If you use submodules, VS Code will resolve relative links correctly in its markdown preview.