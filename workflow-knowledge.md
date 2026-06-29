# Workflow Knowledge

This file covers end-to-end multi-component workflows: tasks that span multiple repositories or artifacts, such as connecting the arch model, running diagnostics, and collecting results.

Most concrete workflows now live as Claude Code skills under `.claude/skills/`, which load on demand rather than always consuming context. For example, the SimNow launch and SLT datapath-test workflows are in `.claude/skills/simnow/`. Add new multi-component workflows as skills there.

For knowledge about building and developing *within* a specific repository, see `repo-specific-knowledge.md`.
For knowledge about operational/administrative tasks (release installation, artifact management), see `operational-knowledge.md`.
