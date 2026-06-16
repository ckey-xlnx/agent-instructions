# Agent Tooling Instructions

This document contains instructions specific to how AI agents should interact with development tools and system utilities. These are mechanical guidelines for tool usage, separate from coding style or review processes.

## Cline-Specific Instructions

### Experimental Features Configuration

Cline has two notable experimental features:

1. **Native Tool Calling (NTC)** — Uses the LLM's native function calling format instead of XML-based tool invocation. Enabled by default since v3.39.0. This is purely about how tool call *syntax* is formatted; it has no bearing on how commands are executed.

2. **Background Exec (Terminal Execution Mode)** — A GUI setting that controls how the built-in `execute_command` tool runs commands. There are two possible modes:
   - **"Background Exec"** (recommended): Commands run asynchronously in the background, returning immediately. Avoids VS Code shell integration issues. Features added in v3.46.0: command tracking, log file output, zombie process prevention (10-minute timeout), clickable log paths in UI.
   - **"VSCode Terminal"**: Commands run via VS Code's terminal/shell integration. **This mode is broken and unreliable — do not use it.**

**Recommended Settings**: Both NTC and Background Exec should be enabled.

### Command Execution: Two Tools

There are exactly two tools available for running shell commands:

1. **`execute_command`** — Cline's built-in tool. Its behaviour depends entirely on which sub-mode is active (see below).
2. **MCP `mcp-cli-exec` server** — A separate MCP server providing reliable command execution. Use this when `execute_command` is not usable.

### Determining the Active `execute_command` Mode

The active mode is snapshotted at session start and does not change during a session. It is reflected in your system prompt — look at how `execute_command` is described:

- **Background Exec mode**: The system prompt will describe the tool with language about background execution, async behaviour, log files, or similar.
- **VSCode Terminal mode**: The system prompt will describe the tool with language such as *"commands are run in the user's VSCode terminal"*.

### What To Do Based on the Active Mode

#### Mode: "Background Exec" — `execute_command` is working correctly

Use `execute_command` directly. Set `requires_approval` appropriately (true for potentially dangerous/destructive operations, false for safe ones).

```xml
<execute_command>
<command>your-command-here</command>
<requires_approval>false</requires_approval>
</execute_command>
```

#### Mode: "VSCode Terminal" — `execute_command` is broken

**Do not use `execute_command`.** Warn the user at the start of the session:

> ⚠️ **Warning**: Cline's `execute_command` tool is set to "VSCode Terminal" mode, which is known to be broken and unreliable. Command execution will fall back to the `mcp-cli-exec` MCP server. Please switch Cline's terminal execution mode to "Background Exec" in the Cline settings.

Then use the **MCP `mcp-cli-exec` server** for all command execution:

For simple commands:
```xml
<use_mcp_tool>
<server_name>mcp-cli-exec</server_name>
<tool_name>cli-exec-raw</tool_name>
<arguments>{"command": "ls -la"}</arguments>
</use_mcp_tool>
```

For commands in a specific directory:
```xml
<use_mcp_tool>
<server_name>mcp-cli-exec</server_name>
<tool_name>cli-exec</tool_name>
<arguments>{
  "workingDirectory": "/path/to/directory",
  "commands": "git status"
}</arguments>
</use_mcp_tool>
```

For multiple commands sequentially:
```xml
<use_mcp_tool>
<server_name>mcp-cli-exec</server_name>
<tool_name>cli-exec</tool_name>
<arguments>{
  "workingDirectory": "/path/to/directory",
  "commands": ["git status", "git log --oneline -5"]
}</arguments>
</use_mcp_tool>
```

### GDB (GNU Debugger)

When using GDB, always include the `--batch` flag to prevent interactive paging of output:

```bash
# Good - output streams directly without paging
gdb --batch -ex "command1" -ex "command2" program

# Bad - requires manual paging through output
gdb -ex "command1" -ex "command2" program
```

**Rationale**: Interactive paging requires manual user intervention to view output, which breaks the automated workflow. The `--batch` flag ensures all output is displayed at once and GDB exits automatically after executing commands.

**Common GDB usage patterns**:
```bash
# Backtrace from core dump
gdb --batch -ex "bt" program core

# Print variable values
gdb --batch -ex "print variable_name" program

# Disassemble function
gdb --batch -ex "disassemble function_name" program

# Multiple commands
gdb --batch -ex "set pagination off" -ex "bt full" -ex "info registers" program core
```

## General Tool Usage Guidelines

### Non-Interactive Tool Execution

**Critical Rule**: Tools cannot be run interactively. All commands must complete without requiring user input or manual paging.

#### Git Commands

**git diff and similar paging commands**:

**Critical Rule**: Always use `--no-pager` for git commands that could potentially page output, regardless of whether you expect the output to be short. This ensures consistent, non-interactive behavior.

```bash
# Good - output streams directly without paging
git --no-pager diff
git --no-pager log
git --no-pager log --oneline -10  # Use --no-pager even for short output
git --no-pager show

# Bad - requires manual paging through output
git diff
git log
git log --oneline -10  # Wrong even if output is expected to be short
git show
```

**git add (staging changes)**:

**Critical Rule**: Never use `git add -A`, `git add .`, or `git add --all`. These commands stage all changes in the repository, which can accidentally include untracked files, temporary files, or unrelated changes.

```bash
# Good - explicitly stage only the files you intend to commit
git add path/to/specific/file.c
git add path/to/another/file.h path/to/third/file.c

# Good - stage all tracked files that have been modified (but not untracked files)
git add -u

# Bad - stages everything including untracked files
git add -A
git add .
git add --all
```

**Rationale**: Using broad staging commands leads to accidental commits of:
- Temporary test files
- Debug scripts
- Build artifacts
- Personal configuration files
- Files from unrelated work in progress

Always review what you're staging and be explicit about which files belong in each commit.

**git rebase -i (interactive rebase)**:
```bash
# Good - programmatically provide rebase instructions
GIT_SEQUENCE_EDITOR="sed -i 's/^pick/edit/'" git rebase -i HEAD~3

# Good - use environment variable for complex edits
GIT_SEQUENCE_EDITOR="cat > /tmp/rebase-todo && sed -i '2s/pick/squash/' /tmp/rebase-todo" git rebase -i HEAD~3

# Bad - opens editor for user interaction
git rebase -i HEAD~3
```

**Common interactive rebase operations**:
```bash
# Squash last 2 commits
GIT_SEQUENCE_EDITOR="sed -i '2s/^pick/squash/'" git rebase -i HEAD~2

# Edit a specific commit
GIT_SEQUENCE_EDITOR="sed -i '1s/^pick/edit/'" git rebase -i HEAD~1

# Reorder commits (swap first two)
GIT_SEQUENCE_EDITOR="sed -i '1h;2,1H;1d;2G'" git rebase -i HEAD~2

# Drop a commit
GIT_SEQUENCE_EDITOR="sed -i '2d'" git rebase -i HEAD~3
```

#### Other Common Interactive Tools

**less, more, and pagers**:
```bash
# Good - use cat or redirect to file
cat large_file.txt
command | cat

# Bad - requires manual paging
less large_file.txt
command | less
```

**Text editors (vim, nano, etc.)**:
```bash
# Good - use non-interactive tools
sed -i 's/old/new/' file.txt
echo "content" > file.txt

# Bad - opens interactive editor
vim file.txt
nano file.txt
```

**Rationale**: Automated workflows cannot handle interactive prompts, paging, or editor sessions. All operations must be scriptable and complete without user intervention.

(Additional agent-specific tooling instructions will be added here as needed)
