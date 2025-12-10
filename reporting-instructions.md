# Reporting Instructions

This document contains instructions for how AI agents should create and manage reports, issues, and other tracking artifacts in external systems.

## Jira Issue Management

### AI Attribution Requirements

**Critical Rule**: All Jira issues, comments, and other content created by AI agents MUST include clear AI attribution using the same footer format as code commits.

#### Attribution Format

Use footer tags similar to `Signed-off-by`:
- `AI-reported-by:` - When AI created the entire issue/report
- `AI-commented-by:` - When AI added a comment to an existing issue
- `AI-amended-by:` - When AI made substantial modifications to existing content

**Format**: Include the AI tool name (e.g., "Claude", "GitHub Copilot", "GPT-4")

#### Issue Creation

When creating Jira issues via the MCP server, always include AI attribution at the end of the description:

**Example**:
```json
{
  "instance": "pensando",
  "project": "IFOESW",
  "summary": "Fix memory leak in packet processing",
  "description": "Memory is not being freed after packet processing completes, leading to gradual memory exhaustion over time.\n\nSteps to reproduce:\n1. Start packet processing\n2. Monitor memory usage\n3. Observe steady increase\n\nAI-reported-by: Claude",
  "issue_type": "Bug",
  "priority": "High"
}
```

#### Comments and Updates

When adding comments to existing Jira issues, include AI attribution at the end:

**Example**:
```
Analysis shows the memory leak is in the rx_buffer_pool cleanup path. 
The buffers are allocated in process_packet() but never freed when 
the connection closes.

Suggested fix: Add cleanup call in connection_close() handler.

AI-commented-by: Claude
```

#### Test Issues

When creating test issues for validation purposes:
- Clearly mark the summary with `[TEST]` prefix
- Include "This is a test issue and can be safely deleted" in the description
- Use appropriate labels like `test`, `automated`, `mcp-server`
- Set priority to Low unless testing priority handling
- Include AI attribution

**Example**:
```json
{
  "instance": "pensando",
  "project": "IFOESW",
  "summary": "[TEST] MCP Server Issue Creation Test - Please Delete",
  "description": "This is a test issue created to verify the Jira MCP server functionality. This issue can be safely deleted.\n\nAI-reported-by: Claude",
  "issue_type": "Task",
  "priority": "Low",
  "labels": ["test", "automated", "mcp-server"]
}
```

### Rationale

Transparency about AI-generated content helps teams:
- Understand the source of information and analysis
- Apply appropriate scrutiny to AI-generated content
- Track the effectiveness of AI assistance over time
- Maintain accountability in issue tracking systems
- Distinguish between human-created and AI-assisted content
- Maintain consistency with code commit attribution standards

## Future Reporting Systems

(Instructions for other reporting/tracking systems will be added here as needed)
