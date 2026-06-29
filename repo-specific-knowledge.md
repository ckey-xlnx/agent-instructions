# Repository-Specific Knowledge

Repository-specific knowledge now lives in per-repo **reference skills** under `.claude/skills/`, which load on demand when working on that repo rather than always consuming context:

- `repo-mpifoe-fw` — mpifoe-fw firmware: config-justification, interrupt handling, reset-hook ordering, EFTEST guards
- `repo-simnow` — simnow: external dependencies (ifoe_ss_model)
- `repo-ifoe-arch-model` — ifoe-arch-model: purpose, usage paths, mpifoe-fw dependency

Build procedures live in `build-*` skills; firmware postcodes in `firmware-postcodes`. Generic coding and commit practices (including the former review checklist) live in `coding-instructions.md`; per-repo code style lives in each repo's `.coding-style.md`.

Add new repo-specific knowledge as a `repo-<name>` reference skill, not here.
