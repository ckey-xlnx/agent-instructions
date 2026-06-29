# Operational Knowledge

This file covers operational/administrative tasks: installing releases, managing artifacts, deploying software to lab environments, and similar tasks that are only weakly tied to a specific repository.

Concrete procedures now live as Claude Code skills under `.claude/skills/`, which load on demand rather than always consuming context. For example, SimNow release installation is in `.claude/skills/simnow/install-release/`. Add new operational procedures as skills there.

For knowledge about building and developing *within* a specific repository, see `repo-specific-knowledge.md`.
For knowledge about end-to-end multi-component workflows (e.g. running tests across SimNow + arch model + diags), see `workflow-knowledge.md`.
