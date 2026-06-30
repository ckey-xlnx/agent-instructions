# Agent Instructions

This repository contains instruction files for AI-assisted code review and development.

## Prime Directive — Honesty and Epistemic Integrity

**This directive overrides all other instructions and applies at all times.**

- **Never fabricate.** Do not invent facts, file contents, command outputs, API responses, code behaviour, or any other information.
- **Never guess and present it as fact.** If you are uncertain, say so explicitly: "I don't know", "I'm not sure", or "I couldn't find that."
- **Never present unverified information as verified.** If something is an assumption, inference, or hypothesis, label it clearly as such before stating it.
- **When in doubt, do not act.** If you lack the information needed to complete a step safely and correctly, stop and ask rather than proceeding on a guess.
- **Absence of evidence is not evidence.** If you searched and found nothing, say "I searched and found nothing" — not "it doesn't exist" or "there is no X."

Violations of this directive — even well-intentioned ones — are more harmful than admitting uncertainty. A wrong answer delivered confidently wastes time, causes bugs, and erodes trust. An honest "I don't know" is always the correct response when knowledge is absent.

## AI Attribution — Always Required

Disclosing AI involvement is a corollary of the Prime Directive. **Any durable artifact you create or modify with AI assistance MUST carry an attribution footer naming the AI tool.** This is not optional and applies regardless of how small the contribution feels (trivial, purely-mechanical changes are the only exception, as noted per-artifact).

The exact footer vocabulary differs per artifact; consult the relevant source when you create one:
- **Commits** (`AI-authored-by:` / `AI-amended-by:`) — `coding-instructions.md` § AI Attribution
- **Code reviews** (`AI-review-delivered-by:` / `-aided-by:` / `-reviewed-by:`) — the `ckey-review` skill § Review Attribution
- **Jira issues / comments** — `reporting-instructions.md` § AI Attribution Requirements

When in doubt, attribute (and prefer the higher level of attribution). Omit only when AI was genuinely not involved.

## Before You Act — Mandatory Reading Gates

These are **hard gates**. You MUST read the specified material before performing the described
action. Do not treat these as advisory — they exist because these rules are frequently violated
without them.

| Before you do this...                        | You MUST first read                                                                    |
|----------------------------------------------|----------------------------------------------------------------------------------------|
| Execute ANY CLI command or MCP tool call     | `agent-tooling-instructions.md` §§ Cline-Specific Instructions, Command Execution: Two Tools |
| Use any `git` command                        | `agent-tooling-instructions.md` § Git Commands                                         |
| Create or comment on a Jira issue            | `reporting-instructions.md` § AI Attribution Requirements                              |
| Write, amend, or commit code                 | `coding-instructions.md` §§ Commit Message Structure, Whitespace Rules                 |
| Perform a code review                        | the `ckey-review` skill (`skills/ckey-review/`) — load it before reviewing             |
| Edit any file in `agent-instructions/`       | This file (`agent-instructions/AGENTS.md`) — read it in full before making changes   |

## Before You Act — Mandatory Action Gates

In addition to the reading gates above, certain conditions require a mandatory **action** before making any tool call. These are not advisory — they are hard requirements:

| If this condition is true...                                  | You MUST perform this action before any tool call                                                          |
|---------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The task is a **verification task** (see below)              | Read the "Verification Tasks" section below before taking any action                                       |

## Verification Tasks — Hard Rule

A **verification task** is any task using language such as: "prove", "check", "verify",
"demonstrate", "confirm", "iterate through and [check/prove/show]", "validate", "test".

**You MUST NOT make any changes when performing a verification task.**

When performing a verification task:

1. **Report findings only.** Do not fix, amend, or alter anything you discover.
2. If a failure or bug is found, **stop and report it explicitly**, including:
   - What failed and why (root cause)
   - What the fix would be
3. Then **ask the user whether to apply the fix** before taking any action.

The user may have reasons not to fix immediately (different ticket scope, pre-existing
issue, wants to triage first). Acting unilaterally removes their ability to decide.

❌ Wrong: discover failure → fix it → mention the change afterwards
✅ Correct: discover failure → report it → ask permission → fix only if authorised

## Workspace Structure Pattern

**IMPORTANT**: The VSCode workspace follows a consistent ordering pattern, but these folders may be at **completely different filesystem paths** — they are only "siblings" in the VSCode folder list, not on the filesystem.

The standard folder ordering is:

1. **First folder**: A workspace folder from within the `vscode-workspaces` repository. This is the primary working directory for the session and contains the `.code-workspace` file that defines the workspace.
2. **Second folder**: This `agent-instructions` repository.
3. **Remaining folders**: Code repositories that are the subject of the current task.

Example VSCode workspace configuration:
```
VSCode Workspace Folders:
1. /home/ckey/hg/vscode-workspaces/xcb/some-workspace    # Workspace folder (primary working directory)
2. /home/ckey/hg/agent-instructions                       # This repo (instructions)
3. /proj/some_project/ckey/work-repo                      # Actual project code (different path!)
4. /other/path/to/another-project                         # Another project (different path!)
```

**Key implications for task execution:**

1. **Don't assume filesystem proximity**: Workspace folders are NOT siblings on the filesystem — they are only grouped in the VSCode folder list
2. **Primary working directory is the first folder**: The session's CWD is the first workspace folder (from `vscode-workspaces`), not agent-instructions
3. **Reference these instructions**: While working in other folders (via full paths), continue to reference the instruction files in this repository
4. **Use full paths**: Always use absolute paths when referring to files in any workspace folder
5. **Session temporary files belong in the first folder**: Any files created during a session for task-coordination purposes (e.g. a Markdown file defining a task, a scratch file, a temporary plan) should be placed in the first workspace folder, not in a code repo or in agent-instructions

When starting a task, if it's unclear which folder the work should be done in, check the environment details for the "Primary workspace" hint or ask the user to clarify.

## Instruction Files

**coding-instructions.md**
- Commit message format and philosophy
- General coding style and patterns
- Work patterns (bug fixes, features, cleanup)
- Whitespace rules and TODO/FIXME requirements

**agent-tooling-instructions.md**
- Agent-specific tooling mechanics and usage guidelines
- How agents should interact with development tools
- Tool-specific flags and options for automated workflows
- Separate from coding style - focuses on tool execution

**infrastructure-knowledge.md**
- Agent-agnostic infrastructure facts: instances, URLs, which projects/repos live where
- GitHub accounts and repo mapping, Jira instances, ReviewBoard, other services
- Does NOT describe MCP access — see `mcp-configuration.md` for that

**mcp-configuration.md**
- How to access infrastructure services via MCP, per agent (Claude Code and Cline)
- AMD MCP platform endpoints, per-service server mapping, setup commands
- Agent-specific: server names, auth, tool call examples

**reporting-instructions.md**
- AI attribution requirements for external systems (Jira, etc.)
- Issue creation and comment formatting standards
- Test issue guidelines
- Maintains consistency with code commit attribution

**skills/**
- Claude Code skills: task-specific and reference content that loads on demand (progressive reveal) when a request matches the skill's description, rather than always consuming context
- Live in `skills/` in this repo; discovered globally via the `~/.claude/skills` symlink → `agent-instructions/skills` (set up by the dotfiles `link-dotfiles.sh`)
- Skill names follow a `<domain>-<kind>-<subject>` taxonomy (domains: `dev`, `admin`, `env`, `meta`; kinds: `workflow`, `knowledge`), with blessed shorthands `build-` (=dev-workflow-build), `repo-` (=dev-knowledge-repo), `infra-` (=admin-knowledge-infra), `bug-` (=dev-knowledge-bug, a single fixed/worked-around bug; named `bug-<project>-<number>-<slug>`), and `ckey-review` kept verbatim. The canonical definition lives in the `meta-writing-skills` skill.
- The skills hold knowledge that previously lived in standalone files, by category:
  - **dev-knowledge** — per-repo reference: `repo-mpifoe-fw`, `repo-simnow`, `repo-ifoe-arch-model` (plus each repo's `.coding-style.md` for fine-grained style)
  - **dev-workflow** — builds (`build-mpifoe-fw`, `build-simnow`, `build-asp`, `build-diag-tng`, `build-ifoe-arch-model-ss-emu`) and simulation (`dev-workflow-simnow-launch`, `dev-workflow-simnow-datapath-test`)
  - **admin-workflow** — `admin-workflow-simnow-install-release`
  - **admin-knowledge** — `infra-amd-sites`, `infra-grid-atl`, `admin-knowledge-firmware-postcodes`
  - **env-workflow** — `env-workflow-vscode-server-cleanup`
  - **code review** — `ckey-review`
  - **meta** — `meta-writing-skills`
- This list is illustrative, not exhaustive; the always-loaded skill descriptions are the source of truth for what exists. To create or migrate a skill, use the `meta-writing-skills` skill — it covers the taxonomy, layout, frontmatter, the one-level-deep discovery constraint, and the migration/verification pattern.

## Usage

- For normal coding tasks, reference **coding-instructions.md**, **agent-tooling-instructions.md**, the relevant **`repo-<name>`** reference skill, and **infrastructure-knowledge.md**
- When performing code reviews or analyzing commits, load the **ckey-review** skill and reference **coding-instructions.md**, the relevant **`repo-<name>`** reference skill, and **infrastructure-knowledge.md**
- For debugging tasks, reference **coding-instructions.md**, **agent-tooling-instructions.md**, the relevant **`repo-<name>`** reference skill, and **infrastructure-knowledge.md**
- For tool usage mechanics (debuggers, build systems, etc.), reference **agent-tooling-instructions.md**
- For infrastructure information (Jira instances, service mappings, etc.), reference **infrastructure-knowledge.md**
- For MCP server names, auth, and tool call syntax per agent, reference **mcp-configuration.md**
- When creating Jira issues or other external reports, reference **reporting-instructions.md**
- For operational/admin tasks, building, or end-to-end workflows, the relevant skill loads on demand (see **skills/** above)
