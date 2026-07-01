---
name: meta-writing-skills
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
skills/<name>/SKILL.md
```

**Subdirectories are NOT discovered.** `skills/group/<name>/SKILL.md`
is silently ignored (Claude Code issue #10238, open as of mid-2026). This was
confirmed empirically here: nested skills never appeared in the model's
available-skills list; flattening fixed it immediately.

Encode grouping in the **name prefix**, not a directory. Every skill name
MUST start with a category prefix from the taxonomy below.

## Naming taxonomy (canonical)

Skill names are `<domain>-<kind>-<subject>`.

**Domains:** `dev` (developing the thing) · `admin` (operating/knowing the
thing itself, as distinct from developing it) · `env` (the user's working
environment) · `meta` (authoring skills / tooling about this repo).

**Kinds:** `workflow` (a task you perform — verb-triggered) · `knowledge`
(reference applied inline — context-triggered).

The `dev`/`admin` split is deliberate: knowledge of how to *develop* a thing
(e.g. firmware build flags) is `dev`; knowledge of the thing *itself* or how to
operate it (e.g. site codes, firmware postcodes) is `admin`.

**Blessed shorthands** — these expand to a `<domain>-<kind>` prefix plus a
conventional subject segment; use the shorthand, not the long form:

| Shorthand | Expands to | For |
|-----------|------------|-----|
| `build-`  | `dev-workflow-build-`   | building a specific target |
| `repo-`   | `dev-knowledge-repo-`   | per-repository reference |
| `infra-`  | `admin-knowledge-infra-`| infrastructure reference (sites, grids) |
| `bug-`    | `dev-knowledge-bug-`    | a single fixed/worked-around bug (regression recognition) |
| `ckey-review` | (verbatim)          | the review skill — kept short as a command |

Skills with no blessed shorthand use the full prefix, e.g.
`dev-workflow-simnow-launch`, `admin-workflow-simnow-install-release`,
`admin-knowledge-firmware-postcodes`, `env-workflow-vscode-server-cleanup`,
`meta-writing-skills`. A new shorthand can be added here later if a prefix
becomes common enough to be worth abbreviating.

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

## The two kinds (workflow vs knowledge)

The `<kind>` segment reflects which of these a skill is:

- **workflow** — step-by-step procedure, triggered by a verb ("build X",
  "launch Y"). Action-oriented.
- **knowledge** — conventions / domain reference applied inline, triggered by
  context ("working on repo Z"). Per-subject reference triggers reliably
  because the subject name/paths appear in the request.

## The `bug-` kind (documenting a fixed/worked-around bug)

A `bug-` skill (`dev-knowledge-bug-<subject>`) documents a **single** bug that
has been fixed or has a workaround in place. Its primary purpose is **regression
recognition** — so that if the same symptoms resurface later, the bug is
recognised as previously-seen rather than re-investigated from scratch. It also
serves as durable documentation for any workaround still in place.

Scope each `bug-` skill to one bug (or one tightly-related cluster).

**The subject segment MUST carry the bug's tracking reference** so the name
uniquely and recognisably identifies the bug. The form is
`bug-<project>-<number>-<slug>`, where `<project>` is the tracking project,
`<number>` is the bug number within it, and `<slug>` is a short symptom/component
hint — e.g. `bug-demi400-8833-no-paused-stream-ack` (DEMI400 project, bug 8833).
Do not use a generic name like `bug-stream-issue`; the project+number is what
makes one bug distinguishable from another in the skills list.

Lead the description with the **observable symptoms** — the failure mode, error
text, or misbehaviour a future request would describe — because that is what a
regression report will contain, not the root cause (which is unknown when the
regression recurs). A useful body covers:

- **Symptoms** — how it manifests (error messages, wrong values, hangs), stated
  in the terms an observer would use.
- **Root cause** — what was actually wrong, once known.
- **Fix / workaround** — what was done, where, and whether it's a true fix or a
  workaround still load-bearing.
- **How to confirm a recurrence** — the quickest check that distinguishes this
  bug from a look-alike.
- Links (`[[name]]`) to the relevant `repo-` / `dev-knowledge-` skills.

This is `dev-knowledge` (reference applied inline, context-triggered), not a
workflow. It complements — does not replace — the commit/PR that carries the
fix: the commit explains the change; the skill makes the symptoms findable.

## Sharing and personal content

These skills are shared with teammates working on the same project, who have
access to the same org, sites, and `/proj` paths. So AMD-internal, site, and
project specifics are all fine to include as-is — the only content that is
genuinely *not* shared is what is personal to one user.

Keep skills shareable by default:

- **Parameterize personal paths** rather than hardcoding them. Prefer `$HOME`,
  `$USER`, `/scratch/$USER/...`, `/proj/.../$USER/...` over `/home/ckey/...` etc.
  This makes the skill *correct* for every teammate, not merely flagged as
  wrong. Where a concrete value aids understanding, show it as an example and
  say to substitute their own.
- **Mark the rare irreducibly-personal span** — a genuine personal convention
  or choice that is not a project fact — with an XML comment marker so a
  teammate sees it should be adapted, not taken as gospel:

  ```
  Output dir <!-- personal --> `/proj/vulcano_dump2_ner/$USER/simnow/packaging`
  is a personal convention; use any writable path.
  ```

  The `<!-- personal -->` marker is greppable and visually distinct from prose.
  Nothing needs stripping on share — the marker just flags "adapt this".

- **A deliberately-shared resource owned by someone else stays literal** — don't
  parameterize it. If a skill points at a *specific* shared checkout, script, or
  file that every teammate reads at the same location (e.g. a colleague's script
  at `.../users/<their-name>/...`, or a shared tool checkout), leave the literal
  path: substituting `$USER` would break it. Only *your own* work paths get
  parameterized. Caveat: don't mistake a personal path for a shared one — a path
  under your own area is shared-as-is *only if you actually intend teammates to
  use that exact copy*; otherwise point the skill at a genuinely common location.

An entire skill that is inherently personal (e.g. a named custom process) is
fine as its own skill; its name signals ownership and teammates can ignore it.

## Discovery mechanics

- Skills here are version-controlled in `agent-instructions/skills/`
  and discovered globally via the `~/.claude/skills` symlink → that directory
  (set up by the dotfiles `link-dotfiles.sh`).
- A **new top-level skill dir** is only scanned at startup — it needs a
  session restart before it is discovered. A new skill added under the
  already-scanned skills directory is picked up without restart in practice,
  but restart-then-verify is the safe path.
- Trigger reliability is imperfect (~50% on vague descriptions). For any
  **mandatory** behaviour (a hard gate), keep the gate in AGENTS.md and point
  it at the skill — do not rely on implicit triggering alone.

## Migration pattern (moving content out of an always-loaded file)

1. Create the skill (`skills/<name>/SKILL.md`) with the content.
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
