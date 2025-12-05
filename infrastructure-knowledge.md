# Infrastructure Knowledge

This document contains general infrastructure information that applies across projects, including service mappings, tool configurations, and other operational knowledge.

## Jira Instance Mapping

Multiple Jira instances are used across different projects. When referencing Jira tickets or using Jira tools, use the following mapping:

### Project to Instance Mapping

**Pensando Atlassian Instance** (`pensando`)
- URL: https://pensando.atlassian.net
- Projects:
  - **IFOESW-*** - IFOE Software (legacy and Oktek interaction)

**OnTrack Internal Instance** (`ontrack_internal`)
- URL: https://ontrack-internal.amd.com
- Purpose: Normal AMD internal tickets
- Projects:
  - **FWDEV-*** - Firmware Development
  - (Other internal AMD projects)

**OnTrack External Instance** (`ontrack_external`)
- URL: https://ontrack.amd.com
- Purpose: Tickets shared with AMD customers/suppliers
- Projects:
  - (Customer/supplier collaboration projects)

**AMD Atlassian Instance** (`amd`)
- URL: https://amd.atlassian.net
- Projects:
  - **DMFPMSN-*** - AMD-specific project

### Projects Requiring Investigation

- **SIEEMU-*** - Instance location to be determined

### Usage Notes

- When using the Jira MCP server tools, specify the instance explicitly based on the project prefix
- For automated reviews, check the ticket prefix to determine the correct instance
- If you encounter a project prefix not listed here, ask for clarification on which instance to use

### Example Usage

```
# Fetch an IFOESW ticket from Pensando
get_issue(instance="pensando", issue_key="IFOESW-205")

# Search for FWDEV tickets in OnTrack Internal
search_issues(instance="ontrack_internal", jql="project = FWDEV AND status = Open")

# Fetch a DMFPMSN ticket from AMD
get_issue(instance="amd", issue_key="DMFPMSN-12345")
```

## Other Infrastructure Information

(Additional infrastructure knowledge will be added here as needed, such as:
- Build system configurations
- CI/CD pipeline details
- Repository locations and purposes
- Development environment setup
- Testing infrastructure
- Deployment procedures)
