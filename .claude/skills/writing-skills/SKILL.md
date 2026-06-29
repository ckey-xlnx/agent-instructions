---
name: writing-skills
description: >
  How to author a new Claude Code skill in this repo (agent-instructions):
  directory layout, frontmatter, naming, discovery constraints, and the
  migration pattern. Use when asked to create, write, add, or migrate a skill,
  or move content out of the always-loaded instruction files into a skill.
---

# Writing a Skill (in agent-instructions)

Skills move task-specific or reference content out of the always-loaded
instruction files so it loads on demand (progressive reveal) only when its
description matches the request.

## Hard constraint: one level deep

A skill MUST live at exactly:

```
.claude/skills/<name>/SKILL.md
```

**Subdirectories are NOT discovered.** `.claude/skills/group/<name>/SKILL.md`
is silently ignored (Claude Code issue #10238, open as of mid-2026). This was
confirmed empirically here: nested skills never appeared in the model's
available-skills list; flattening fixed it immediately.

Encode grouping in the **name prefix**, not a directory:
- `simnow-launch`, `simnow-datapath-test`, `simnow-install-release`
- `repo-mpifoe-fw` (per-repo reference)

## File format

```markdown
---
name: <kebab-case-name>          # match the directory name
description: >
  One or more sentences. LEAD with the words a request would contain.
  List concrete trigger phrases — the model under-triggers on vague
  descriptions, so more phrasings is safer than fewer.
---

# Title

Body: the procedure or reference content.
```

- `name` and `description` are the only required frontmatter fields.
- The description is the *only* thing always loaded; it is what the model
  matches against. Spend effort here. Combined name+description text is capped
  (~1,536 chars) so put the key use case first.
- Add `disable-model-invocation: true` only for skills that should be
  explicit-only (`/name`), e.g. a destructive action you never want
  auto-triggered.

## Two kinds of content

- **Task skill** — step-by-step procedure, triggered by a verb ("build X",
  "launch Y"). Action-oriented.
- **Reference skill** — conventions / domain knowledge applied inline,
  triggered by context ("working on repo Z"). Per-repo reference triggers
  reliably because the repo name/paths appear in the request.

## Discovery mechanics

- Skills here are version-controlled in `agent-instructions/.claude/skills/`
  and discovered globally via the `~/.claude/skills` symlink (set up by the
  dotfiles `link-dotfiles.sh`).
- A **new top-level skill dir** is only scanned at startup — it needs a
  session restart before it is discovered. A new skill added under an
  already-scanned `.claude/skills/` is picked up without restart in practice,
  but restart-then-verify is the safe path.
- Trigger reliability is imperfect (~50% on vague descriptions). For any
  **mandatory** behaviour (a hard gate), keep the gate in AGENTS.md and point
  it at the skill — do not rely on implicit triggering alone.

## Migration pattern (moving content out of an always-loaded file)

1. Create the skill (`.claude/skills/<name>/SKILL.md`) with the content.
2. Remove that content from the source file; leave a slim orienting stub/index
   pointing at the skill.
3. Update AGENTS.md references (file index, usage section, and any mandatory
   reading gate — repoint the gate at the skill, don't drop it).
4. Restart, then trigger-test with a fresh Agent: confirm the skill appears in
   the available-skills list AND that the agent loads it WITHOUT being handed
   the file path. (A subagent reading the file by path is NOT proof of
   discovery — that mistake was made early in this migration.)
5. Commit with files staged explicitly by name; use the `AI-authored-by: Claude`
   footer (see coding-instructions.md). Exclude unrelated untracked files.

## Verifying a skill actually triggers

The honest test: a fresh agent, given a realistic request, loads the skill on
its own. Check the available-skills list shows the skill, and that the agent
did not just `find`/`Read` the markdown by path. If it only worked because a
path was supplied, it did NOT trigger.
