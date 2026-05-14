---
name: cn-breadboard
description: Turn a Shape Up breadboard (UI affordances, code affordances, data stores, wiring) into a navigable CreatorNotes canvas via the cn CLI. Use when a user wants the output of /breadboarding as a CreatorNotes canvas instead of a markdown file in the repo. Triggers on "/cn-breadboard", "breadboard this in CN", "make a breadboard canvas".
---

# cn-breadboard

You are converting a Shape Up breadboard into a CreatorNotes canvas. The canvas is the source of truth. Nothing gets written to the repo.

This skill builds on the upstream [rjs/shaping-skills](https://github.com/rjs/shaping-skills) `/breadboarding` skill. It handles the OUTPUT step only; the affordance tables come from the upstream procedure.

---

## Before you start

Verify:

```bash
cn --version              # the cn CLI is installed
cn auth status            # authenticated
cn workspace current      # a workspace is selected
```

If anything fails, ask the user to fix it before continuing. Do not try to install or authenticate on their behalf.

---

## Procedure

### Step 1 — Produce the breadboard (if it doesn't exist)

If the user already has a completed breadboard (Places, UI affordances, Code affordances, Data Stores tables with Wires Out / Returns To columns filled in), skip to Step 2.

Otherwise, follow the upstream `/breadboarding` procedure to produce one. The upstream skill is at `~/.claude/skills/breadboarding/skill.md`. Read it, follow it, then come back here.

Before continuing, confirm with the user: "Here is the breadboard. Build the canvas?"

### Step 2 — Pin the type colours

The `cn` CLI auto-creates types on first use, but to keep colours pinned to the breadboarding palette, run these every time (idempotent):

```bash
cn types update Affordance --color "#FFB6C1" 2>/dev/null || cn types create Affordance --color "#FFB6C1"
cn types update Mechanism  --color "#D3D3D3" 2>/dev/null || cn types create Mechanism  --color "#D3D3D3"
cn types update Store      --color "#E6E6FA" 2>/dev/null || cn types create Store      --color "#E6E6FA"
cn types update Place      --color "#FFF3CD" 2>/dev/null || cn types create Place      --color "#FFF3CD"
cn types update Chunk      --color "#B3E5FC" 2>/dev/null || cn types create Chunk      --color "#B3E5FC"
```

### Step 3 — Decide: new canvas or rewrite in place

If a canvas titled `Breadboard: <flow name>` already exists in the workspace, REWRITE it in place. Do not create a new one. CreatorNotes auto-checkpoints before AI agent runs, so the previous state is recoverable with one click in the UI.

```bash
cn canvas list --json | jq -r '.[] | select(.title == "Breadboard: <flow name>") | .id'
```

If empty, create it:

```bash
cn canvas create "Breadboard: <flow name>" --json
```

If rewriting, fetch existing nodes and bulk-remove them first:

```bash
cn canvas get <canvasId> --json
cn canvas bulk-remove <canvasId> --nodes '<all-node-ids>'
```

### Step 4 — Create one note per affordance

Use `cn notes create --notes` with a JSON array. One note per row across all four tables (Places, UI, Code, Stores).

Type mapping:

| Breadboard element | Note type | Colour |
|--|--|--|
| UI affordance (U) | `Affordance` | pink `#FFB6C1` |
| Code affordance (N) | `Mechanism` | grey `#D3D3D3` |
| Data store (S) | `Store` | lavender `#E6E6FA` |
| Place (P) — anchor note | `Place` | cream `#FFF3CD` |
| Chunk — collapsed subsystem | `Chunk` | light blue `#B3E5FC` |

Note content (markdown):
- **Title:** `U1: Search input` — affordance ID followed by the affordance label
- **Body:** 2–4 sentences. Cover: what it does, where the data comes from, where its output goes. Mention component / file name if relevant.
- **Cross-references:** wire the breadboard's Wires Out and Returns To columns into the body as `[@<key>: title](relationship:triggers)` mentions. These become relationship records you can navigate.

Write each note's markdown to a temp file in `/tmp/cn-breadboard/<key>.md`, then bulk-create:

```bash
cn notes create --notes '[
  {"key":"u1","type":"Affordance","markdownFile":"/tmp/cn-breadboard/u1.md"},
  {"key":"n1","type":"Mechanism","markdownFile":"/tmp/cn-breadboard/n1.md"},
  ...
]' --json
```

Capture the displayIds (e.g., `AFFORDANCE-5`, `MECHANISM-12`) from the JSON response — you'll need them for bulk-add.

### Step 5 — Lay out the canvas

Use a section-per-Place layout:

- **Hero richtext** at top, medium size, indigo `#6366F1`, with the colour legend
- **Section text heading** per Place at X=100 (e.g., "P1: Workspace shell")
- **Affordances** placed in rows within each Place section

Standard positions:
- 3-column grid: X = 400, 1000, 1600 (note width 400, gap 200)
- 2-column centred: X = 700, 1300
- 1-column centred: X = 1000
- Section heading: X = 100, sits 60px above first row
- Vertical stride per row: 240 (180 note height + 60 gap)
- Gap between sections: 120

Add the hero first to capture its real height from the server response (`estimatedHeight`), then compute the rest of the Y positions.

```bash
cn canvas add-richtext <canvasId> --content-file /tmp/cn-breadboard/hero.md --size medium --color "#6366F1" --x 640 --y 100 --json
```

For section headings:

```bash
cn canvas add-text <canvasId> --text "P1: <place name>" --size heading --x 100 --y <Y>
```

For the notes (bulk):

```bash
cn canvas bulk-add <canvasId> --notes '[
  {"noteId":"AFFORDANCE-5","x":400,"y":700},
  ...
]'
```

### Step 6 — Add edges with semantic labels

CreatorNotes canvas edges have no line-style support. Labels carry the semantics. Pick the verb that fits the wiring:

| Label family | When to use |
|--|--|
| `triggers`, `click`, `type` | UI affordance triggers a code path |
| `calls`, `schedules`, `signals` | Code calls / schedules / fires another |
| `writes <state>` | Code writes to a store |
| `reads`, `input`, `<thing> data` | Data flows from a store or producer to a consumer |
| `navigate`, `opens`, `cancel` | Place transitions (the user moves to a different Place) |

```bash
cn canvas add-edge <canvasId> --source <fromNodeId> --target <toNodeId> --label "calls"
```

Add edges sparingly. Include the load-bearing narrative arcs (3-25 edges typical), not every wire. The breadboard tables in chat are the complete truth; the canvas is the navigable visualisation.

### Step 7 — Return the canvas URL

Output:
- The canvas URL: `https://<server>/ws/<workspaceId>?canvasId=<canvasId>`
- 2-3 of the most important note URLs so the user can jump straight in
- A short summary of what's on the canvas (sections, affordance count, edge count)

---

## Conventions reference

### Types

| Type | Maps to | Colour |
|--|--|--|
| Affordance | UI affordance (U) | pink `#FFB6C1` |
| Mechanism  | Code affordance (N) | grey `#D3D3D3` |
| Store      | Data store (S) | lavender `#E6E6FA` |
| Place      | Bounded context (P) | cream `#FFF3CD` |
| Chunk      | Collapsed subsystem | light blue `#B3E5FC` |

### Edge label families

- **Control flow:** `triggers`, `click`, `type`, `calls`, `schedules`, `signals`
- **Data flow:** `writes <state>`, `reads`, `<thing> data`, `input`
- **Navigation:** `navigate`, `opens`, `cancel`

### Layout

- 3-col grid: X = 400, 1000, 1600
- 2-col centred: X = 700, 1300
- 1 centred: X = 1000
- Section heading: X = 100, 60px above row
- Row vertical stride: 240 (180 note + 60 gap)
- Section gap: 120
- Hero richtext: X = 640, medium size, indigo `#6366F1`

---

## Re-runs

If `/cn-breadboard` is invoked again for the same scope:

1. Find the existing canvas by name (`Breadboard: <flow>`)
2. Rewrite it in place — wipe existing nodes, re-run Steps 4–6
3. Canvas Versioning auto-checkpoints before the run, so the previous state is one click away in the CreatorNotes UI

Never create a numbered duplicate (`Breadboard: Foo v2`). The canvas URL must stay stable so other notes can keep linking to it.

---

## What to do when something goes wrong

- **Type already exists with different colour:** `cn types update` should fix it. If it errors, surface the error to the user.
- **bulk-create rejects a type:** check the type name is exactly `Affordance` / `Mechanism` / `Store` / `Place` / `Chunk` — case-sensitive PascalCase.
- **bulk-add 207 with per-item errors:** the canvas may have stale node references. Surface the errors to the user with the failing rows.
- **Canvas with this name already exists, user wanted a new one:** ask the user. Default to rewriting in place; only fork to a new canvas if they confirm.

---

## Credit

Built on [rjs/shaping-skills](https://github.com/rjs/shaping-skills). The methodology, the affordance grammar, the chunking convention, and the colour palette all come from there. This skill only redirects the output to a CreatorNotes canvas.
