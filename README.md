# cn-shaping-skills

[Claude Code](https://claude.com/claude-code) skills for using [Shape Up](https://basecamp.com/shapeup) methods with [CreatorNotes](https://creatornotes.app) as the output target.

Built on [rjs/shaping-skills](https://github.com/rjs/shaping-skills). Same methodology, same affordance grammar, same conventions. The difference is the output: instead of writing markdown files to your repo, these skills create CreatorNotes canvases via the `cn` CLI. The canvas becomes the source of truth.

## Skills

### `/cn-breadboard`

Turn a Shape Up breadboard (UI affordances, code affordances, data stores, wiring) into a navigable CreatorNotes canvas. Each affordance is a typed note; each wire is a canvas edge with a label that carries control-flow or data-flow semantics.

Follows the upstream `/breadboarding` skill for the analysis step, then handles the CreatorNotes conversion via `cn`.

## Install

```bash
# 1. Clone this repo
git clone https://github.com/<org>/cn-shaping-skills.git ~/.local/share/cn-shaping-skills

# 2. Symlink each skill into your Claude Code skills directory
ln -s ~/.local/share/cn-shaping-skills/cn-breadboard ~/.claude/skills/cn-breadboard

# 3. Make sure the upstream shaping-skills are installed too
#    (cn-breadboard delegates to its /breadboarding procedure)
# See https://github.com/rjs/shaping-skills for instructions
```

## Prerequisites

- [Claude Code](https://claude.com/claude-code) installed and signed in
- [rjs/shaping-skills](https://github.com/rjs/shaping-skills) installed (cn-breadboard delegates to its `/breadboarding` skill)
- `cn` CLI installed and authenticated against your CreatorNotes workspace
- A workspace selected: `cn workspace select <id>`

## Conventions

| Breadboard element | CreatorNotes note type | Colour |
|--|--|--|
| UI affordance (U) | `Affordance` | pink `#FFB6C1` |
| Code affordance (N) | `Mechanism` | grey `#D3D3D3` |
| Data store (S) | `Store` | lavender `#E6E6FA` |
| Place (P) | `Place` | cream `#FFF3CD` |
| Chunk (collapsed subsystem) | `Chunk` | light blue `#B3E5FC` |

Types are created automatically on first use. Colours are re-pinned on every run.

Canvas edges carry semantics in their labels (`triggers`, `calls`, `writes <state>`, `reads`, `navigate`, etc.) because CreatorNotes canvas edges have no dashed-line support.

## Re-runs

Re-running `/cn-breadboard` against the same flow name **rewrites the existing canvas in place** rather than creating a new one. CreatorNotes auto-checkpoints before AI agent runs, so the previous state stays available via Compare Mode or one-click revert.

## Why a skill instead of an in-app feature

Putting breadboarding inside CreatorNotes as a first-class concept (custom node types, dashed edges, a "breadboard mode") would couple the app to one methodology. Today the affordances are typed notes and the wires are normal canvas edges. The app stays generic. The discipline lives in the skill.

## Credit

Built on [rjs/shaping-skills](https://github.com/rjs/shaping-skills). The methodology, the affordance grammar, the chunking convention, and the colour palette all come from there. Everything substantive in this repo is downstream of that work.
