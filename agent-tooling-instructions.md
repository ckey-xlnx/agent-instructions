# Agent Tooling Instructions

This document contains instructions specific to how AI agents should interact with development tools and system utilities. These are mechanical guidelines for tool usage, separate from coding style or review processes.

## Cline-Specific Instructions

### Experimental Features Configuration

Cline has experimental features that should be enabled for optimal performance:

1. **Native Tool Calling (NTC)** - Uses the LLM's native function calling format instead of XML-based tool invocation. Enabled by default since v3.39.0. This affects how the AI model communicates tool requests to Cline.

2. **Background Exec (Terminal Execution Mode)** - Controls how terminal commands are executed:
   - When enabled: Commands run asynchronously in the background, returning immediately without waiting for shell integration signals
   - Prevents hanging issues that occurred with VS Code shell integration
   - Added enhanced features in v3.46.0: command tracking, log file output, zombie process prevention (10-minute timeout), and clickable log paths in UI

**Recommended Settings**: Both features should be enabled for best reliability.

### Command Execution Tool Preference

**Preference Order** (use the first available option):

1. **Cline Native `execute_command` with Background Exec** (Preferred) - When "Background Exec" terminal execution mode is enabled, commands run asynchronously in the VS Code terminal and return immediately.

2. **Shell Integration MCP Server (`mcp-cli-exec`)** - Fallback if native execution has issues. Provides reliable command execution via MCP.

3. **VS Code Shell Integration** - Last resort fallback (can hang waiting for completion signals).

**Usage**:

**Option 1: Cline Native (Preferred)**
```xml
<execute_command>
<command>your-command-here</command>
<requires_approval>false</requires_approval>
</execute_command>
```

**Option 2: MCP Shell Integration (Fallback)**

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

**When to use MCP fallback**: Only use the MCP `mcp-cli-exec` server if you encounter issues with the native `execute_command` tool (e.g., hanging, shell integration problems).

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
