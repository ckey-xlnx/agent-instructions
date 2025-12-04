# Agent Tooling Instructions

This document contains instructions specific to how AI agents should interact with development tools and system utilities. These are mechanical guidelines for tool usage, separate from coding style or review processes.

## Cline-Specific Instructions

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

(Additional agent-specific tooling instructions will be added here as needed)
