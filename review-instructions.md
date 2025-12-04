# Automated Review Request Feedback Process

## Overview

This document describes the process for partially automating feedback on review requests using AI assistance. The system provides four main capabilities:

1. **Summary of pending reviews** - Quick overview of reviews meeting specific criteria
2. **Detailed analysis** - In-depth review with suggested comments and controversy flags
3. **Interactive feedback submission** - Guided process for leaving review comments
4. **Learning from feedback** - Suggestions for updating review-specifics.md based on actual feedback patterns

## Process Flow

### A. Summary of Pending Review Requests

**Purpose**: Quickly identify which reviews need attention without deep analysis.

**Criteria for filtering**:
- Status (pending, submitted, discarded, all)
- Assigned users or groups
- Repository
- Time range
- Number of results

**Output format**:
```
Review Request #XXXX - [Repository Name]
Author: [Name]
Summary: [Brief description]
Files changed: X files, +YYY/-ZZZ lines
Age: X days
Reviewers: [List]
Status: [pending/submitted]
```

**Efficiency requirements**:
- Use list_review_requests API call only
- No diff fetching at this stage
- Minimal processing overhead
- Quick scan of metadata only

### B. Detailed Analysis of Review Requests

**Purpose**: Provide actionable feedback suggestions and identify reviews requiring human attention.

**Analysis sources** (in priority order):
1. `coding-instructions.md` - General coding and commit practices
2. `review-instructions.md` - This document describing the review process
3. `review-specifics.md` - Repository-specific style, architecture, and priorities

**Analysis steps**:

1. **Fetch review data**:
   - Review request details
   - All diffs
   - Diff content (patches)
   - Existing reviews and comments
   - File context where needed

2. **Commit message analysis**:
   - Verify ticket reference format (e.g., `IFOESW-XXX:`, `SIEEMU-XXX:`)
   - Check for descriptive summary after colon
   - Validate `cleanup:` prefix usage for non-functional changes
   - Verify AI attribution (`AI-authored-by:`, `AI-amended-by:`) when applicable
   - Check for multi-line explanations on non-trivial changes
   - Assess whether commit philosophy is followed (incremental changes, design intent explanation)

3. **Existing review analysis**:
   - Fetch all existing reviews and their comments
   - Read replies to understand ongoing discussions
   - Identify unresolved issues or open questions
   - Note areas where clarification or additional feedback may be helpful

4. **Code change analysis**:
   - Check for TODO/FIXME/HACK/XXX without Jira references
   - Verify whitespace rules (no trailing whitespace, empty blank lines)
   - Assess commit size and logical separation
   - Check for unexplained design deviations
   - Verify repository-specific style from review-specifics.md
   - Look for common bug patterns (memory leaks, pointer issues, missing checks)
   - Assess code clarity and maintainability

5. **Controversy detection**:
   Flag reviews that need human attention when:
   - Large architectural changes without adequate explanation
   - Deviations from established patterns without justification
   - Complex refactoring affecting multiple subsystems
   - Changes to critical paths (memory management, hardware interfaces)
   - Conflicts with repository-specific priorities in review-specifics.md
   - Existing review comments indicate disagreement
   - Changes that appear to violate design intent

**Output format**:

```
=== Review Request #XXXX Analysis ===

CONTROVERSY FLAG: [YES/NO]
Reason: [If flagged, explain why human attention is needed]

COMMIT MESSAGES:
Commit: [hash] - [subject line]
✓ Ticket reference present
✗ Missing explanation for non-trivial change
✓ AI attribution included
Suggestion: Add body explaining why this refactoring is needed

CODE CHANGES:

File: path/to/file.c
Lines: 45-67

Issue: TODO without Jira reference
Current: /* TODO: optimize this later */
Suggested comment: "This TODO needs a Jira reference. Consider creating a ticket or removing if not actionable."

File: path/to/other.c
Lines: 123-145

Issue: Trailing whitespace on lines 130, 142
Suggested comment: "Please remove trailing whitespace (lines 130, 142)"

File: path/to/feature.c
Lines: 200-250

Observation: Large function added without explanation
Suggested comment: "This is a substantial new function. Could you add a comment explaining the design approach and why it's placed in this file?"

POSITIVE OBSERVATIONS:
- Clean separation of concerns in transport layer changes
- Good incremental approach across 3 commits
- Clear commit messages explaining design intent

OVERALL ASSESSMENT:
[Summary of review quality and main concerns]

RECOMMENDED ACTION:
[Ship it / Request changes / Needs discussion]
```

### C. Interactive Feedback Submission

**Purpose**: Guide the reviewer through leaving feedback based on analysis suggestions.

**Process**:

1. Present analysis from step B
2. For each suggested comment, ask:
   - "Include this comment? [yes/no/edit]"
   - If edit: allow modification of suggested text
   - Track which suggestions are accepted/rejected

3. Ask for overall review decision:
   - Ship it (approve)
   - Request changes
   - Comment only (no explicit approval/rejection)

4. Ask for top-level review text:
   - Body top (appears above comments)
   - Body bottom (appears below comments)

5. Determine AI attribution level (see Review Attribution section below)

6. Confirm before posting:
   - Show complete review preview
   - List all comments to be posted
   - Show attribution footer that will be included
   - Ask for final confirmation

7. Submit review via API with appropriate attribution

8. Report results:
   - Confirmation of successful posting
   - Link to review
   - Summary of what was posted

**Interaction principles**:
- Default to including suggested comments unless explicitly rejected
- Allow free-form additions at any point
- Support batch operations (e.g., "include all whitespace comments")
- Provide escape hatch to cancel before posting

## Review Attribution

All AI contributions to code reviews must be properly attributed using footer tags similar to commit attribution (see coding-instructions.md).

**Format:**
- `AI-review-delivered-by:` - When all assessment was human, AI just automated the process
- `AI-review-aided-by:` - When review was collaborative between human and AI
- `AI-review-reviewed-by:` - When majority of contribution was from AI

**Usage Rules:**

1. **AI-review-delivered-by**: Use when the human reviewer:
   - Made all assessment decisions independently
   - AI only automated the mechanical process of posting
   - Human wrote all review comments from scratch
   - AI provided no analysis or suggestions that were used
   
   ```
   This looks good, please fix the whitespace issues on lines 45-47.
   
   AI-review-delivered-by: Claude
   ```

2. **AI-review-aided-by**: Use when the review was collaborative:
   - AI provided analysis and suggestions
   - Human reviewed, modified, and approved suggestions
   - Human added significant additional context or comments
   - Mix of AI-suggested and human-written feedback
   
   ```
   The function renaming improves debuggability. However, please also
   add a comment explaining why unique names are important here.
   
   AI-review-aided-by: Claude
   ```

3. **AI-review-reviewed-by**: Use when AI did the majority of the work:
   - AI performed the analysis and generated most comments
   - Human approved with minimal modifications
   - Most feedback came directly from AI suggestions
   - Human primarily validated rather than created content
   
   ```
   Please address the following issues:
   - TODO on line 45 needs Jira reference
   - Trailing whitespace on lines 130, 142
   - Missing error handling in new function
   
   AI-review-reviewed-by: Claude
   ```

**Attribution Guidelines:**

- **Always include attribution** when AI was involved in any capacity
- **Be honest** about the level of AI contribution
- **Default to higher attribution** when uncertain (e.g., use "AI-review-reviewed-by" rather than "AI-review-aided-by" if mostly AI-generated)
- **Include tool name** (e.g., "Claude", "GitHub Copilot", "GPT-4")
- **Omit attribution** only when AI was not used at all

**Examples:**

Fully automated posting of human review:
```
Ship It!

AI-review-delivered-by: Claude
```

Collaborative review:
```
Good refactoring. A few suggestions:
- Consider adding unit tests for the new allocator
- The error path on line 67 could use a comment

AI-review-aided-by: Claude
```

Mostly AI-generated review:
```
This change looks good overall. Please address:
- Missing Jira reference in TODO on line 45
- Trailing whitespace on lines 130, 142
- Function could benefit from a docstring

AI-review-reviewed-by: Claude
```

**Transparency Benefits:**

- Helps other reviewers understand the source of feedback
- Tracks effectiveness of AI-assisted reviews
- Maintains trust in the review process
- Allows for appropriate weight to be given to different types of feedback
- Enables learning about which AI contributions are most valuable

### D. Learning from Feedback

**Purpose**: Improve review-specifics.md based on actual feedback patterns.

**Data sources**:
1. Feedback submitted through the tool (step C)
2. Feedback scraped from API for reviews I've authored

**Analysis approach**:

1. **Pattern detection**:
   - Identify recurring comment themes
   - Track which suggestions are consistently accepted/rejected
   - Note repository-specific issues that appear frequently
   - Detect new coding patterns or anti-patterns

2. **Gap identification**:
   - Find issues caught in reviews but not in automated analysis
   - Identify missing rules in review-specifics.md
   - Detect outdated guidance that no longer applies

3. **Suggestion generation**:
   ```
   === Suggested Updates to review-specifics.md ===
   
   Based on 15 reviews over the past 2 weeks:
   
   NEW RULE SUGGESTION:
   Section: Memory Management
   Observation: 8 comments about missing free() calls in error paths
   Suggested addition:
   "Always verify error paths properly free allocated resources. 
   Common pattern: if allocation fails partway through initialization,
   ensure cleanup unwinds all prior allocations."
   
   PATTERN REFINEMENT:
   Section: Naming Conventions
   Observation: 5 comments about inconsistent prefix usage
   Current rule: "Use module prefix for public functions"
   Suggested refinement: "Use module prefix for public functions. 
   For the SDP module, use 'sdp_' prefix. For RX2 stage, use 'rx2_'."
   
   OUTDATED RULE:
   Section: Hardware Interfaces
   Observation: No comments about this in 20 reviews
   Suggested action: Review if this guidance is still relevant
   ```

4. **Update workflow**:
   - Present suggestions for review
   - Allow acceptance/rejection/editing of each suggestion
   - Generate updated review-specifics.md content
   - Optionally create a commit with the changes

**Learning frequency**:
- On-demand: "Analyze my recent feedback and suggest updates"
- Periodic: After every N reviews (configurable)
- Triggered: When pattern threshold is reached (e.g., same issue 3+ times)

## Configuration

### Review Specifics Structure

Each repository should have a review-specifics.md file containing:

1. **Repository identification**
   - Name and purpose
   - Primary languages
   - Key subsystems

2. **Coding style specifics**
   - Naming conventions
   - Memory management patterns
   - Error handling approaches
   - Module organization

3. **Architecture principles**
   - Design patterns in use
   - Layering and separation of concerns
   - Interface contracts
   - Performance considerations

4. **Current priorities**
   - Active refactoring efforts
   - Known technical debt
   - Areas under active development
   - Deprecated patterns to avoid

5. **Common issues**
   - Frequent mistakes
   - Anti-patterns to watch for
   - Integration gotchas

### Tool Integration

The system integrates with ReviewBoard via MCP server:
- `list_review_requests` - Get review list
- `get_review_request` - Get review details
- `get_review_diffs` - Get diff list
- `get_diff_content` - Get actual patches
- `get_reviews` - Get existing reviews
- `get_review_comments` - Get review comments
- `get_review_replies` - Get replies to reviews
- `post_review` - Submit review feedback
- `post_review_reply` - Post a reply to an existing review

### Generating Review Replies

When analyzing a review request, always check for existing reviews and their comments. If there are comments that warrant a response:

1. **Read existing comments and replies** using `get_review_comments` and `get_review_replies`
2. **Identify opportunities for helpful replies**:
   - Questions that need answers
   - Clarifications that would help the discussion
   - Additional context that supports or refines a comment
   - Acknowledgment of valid concerns
3. **Generate appropriate replies** using `post_review_reply` when:
   - You can provide useful clarification
   - There's an ongoing discussion that needs input
   - A comment raises important points worth addressing
4. **Include proper attribution** in reply body_bottom field (e.g., "AI-review-delivered-by: Claude")

**Important**: Replies should add value to the discussion. Don't reply just to acknowledge - only reply when you have substantive information to contribute.

## Best Practices

### For Automated Analysis
- Always read all three instruction files before analysis
- Consider repository context from review-specifics.md
- Flag uncertainty - better to ask than assume
- Provide specific line numbers and file paths
- Quote actual code in suggestions
- Explain the "why" behind each suggestion

### For Human Reviewer
- Review controversy flags first
- Use automated suggestions as starting point, not gospel
- Add context and nuance that AI cannot
- Override suggestions when they miss important context
- Provide feedback on suggestion quality to improve learning

### For Learning System
- Track both accepted and rejected suggestions
- Look for patterns across multiple reviews
- Don't over-fit to individual cases
- Suggest rules that are actionable and verifiable
- Keep review-specifics.md focused and practical

## Privacy and Security

- Review content may contain proprietary code
- Ensure AI analysis respects confidentiality
- Don't leak review content across repositories
- Store learning data appropriately
- Allow opt-out of learning features if needed

## Future Enhancements

Potential additions to consider:
- Diff-based learning (what changes are commonly requested)
- Reviewer consistency tracking
- Time-to-review metrics
- Integration with CI/CD results
- Automated "Ship It" for trivial changes meeting all criteria
- Cross-repository pattern detection
