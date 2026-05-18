# sci-skills

Cursor Agent Skills for manuscript figure workflows (R / ggplot2 / cowplot).

## Skills

| Folder | Purpose |
|--------|---------|
| `manuscript-figure-curation/` | Curated `MANUSCRIPT_ROOT`, `figure_manifest.yaml`, per-panel `build_*`, volcano / ggplot2 heatmap patterns, **重画刊例** alignment. |
| `manuscript-composite-layout/` | Whole-figure layout from `design.txt` → NPC boxes → `cowplot::ggdraw` + `draw_plot`. |

Each skill is a directory containing **`SKILL.md`** (Cursor skill front matter + body).

## Install

Copy the skill folders into your project or user skills tree:

- **Project:** `<repo>/.cursor/skills/<skill-name>/`
- **User (global):** `~/.cursor/skills/<skill-name>/`

Restart Cursor or reload skills if your client caches them.

## Repo

<https://github.com/lssb/sci-skills>
