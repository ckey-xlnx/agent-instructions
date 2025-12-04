# Christopher Key's Coding Style & Work Patterns

## Commit Message Structure

### Format
- **Subject Line**: Always include ticket reference (e.g., `IFOESW-XXX:`, `SIEEMU-XXX:`)
- **Descriptive Summary**: Concise, lowercase description after colon
- **Body**: Multi-line explanations for non-trivial changes, separated by blank line from subject
- **Prefix for Cleanup**: Use `cleanup:` prefix for refactoring/non-functional changes
- **AI Attribution**: Must include attribution footer when AI contributes to the commit (see AI Attribution section below)

### Examples
```
IFOESW-366: move the assertion up the callstack

Rather than asserting immediately if SDP is backpressured by lack of
tags, we return a suitable error and let our caller assert.  Our
caller is in a suitable position to defer processing and a future
commit with do just that.

This commit exists to make that future commit smaller and more
readable.
```

```
IFOESW-366: cleanup: SPaG
```

```
IFOESW-548: remove the bad (but innocuous) code
```

### AI Attribution

All AI contributions to commits must be properly attributed using footer tags similar to `Signed-off-by`.

**Format:**
- `AI-authored-by:` - When AI generated the entire commit
- `AI-amended-by:` - When AI made non-trivial changes to human-authored work

**Usage Rules:**
1. **AI-authored commits**: Use when AI wrote the code/changes from scratch
   ```
   IFOESW-1234: implement new SDP retry logic
   
   Add exponential backoff when SDP ports are unavailable.
   
   AI-authored-by: GitHub Copilot
   ```

2. **AI-amended commits**: Use when AI made substantial modifications (not just formatting/trivial fixes)
   ```
   IFOESW-5678: refactor memory allocation in rx2 stage
   
   Convert to heap allocation with proper ownership transfer.
   
   AI-amended-by: GitHub Copilot
   ```

3. **Trivial AI changes** (formatting, simple typo fixes, etc.) do not require attribution

4. **Tool specification**: Include the AI tool name (e.g., "GitHub Copilot", "Claude", "GPT-4")

This ensures transparency about the origin of code and helps track the effectiveness of AI assistance.

## Commit Philosophy

### Incremental Changes
- Break large features into small, logical commits
- Each commit should be self-contained and buildable
- Create preparatory commits that make future changes smaller and more readable
- Explicitly call out when a commit exists to simplify a future commit

### Explaining Design Intent
- Always document the "why" behind architectural decisions
- Call out deviations from design intent and explain rationale
- Note when code placement is temporary or conceptually wrong but pragmatic
- Explain future plans for moving/refactoring code

### Example Pattern
```
IFOESW-546: provide allocators for the SDP streaming types

The design intent is that the start of pipelines allocate the
streaming types passed along and the end of the pipelines free them.

Doing this directly with malloc/free is a bad idea for several reasons:

 - Because of our complex build environment, the start and end of pipelines
   may be built by different compilers.  Having malloc/free span such a
   boundary is dangerous.

 - The allocations are fixed size.  We can optimise for this, e.g. maintaining
   linked list of already allocated objects.

This commit introduces the allocators.  A subsequent commit will use them.
```

## Code Style

### Language Focus
- Primary languages: C (most common), C++, some Lua scripting
- Hardware modeling and emulation code
- Interface with Zephyr firmware and hardware definitions

### Comments
- Explain deviations from design intent
- Document why code is in unexpected locations
- Note future cleanup plans
- Use TODO/FIXME comments for known issues
- Keep comments focused on "why" not "what"

### TODO/FIXME Requirements
**NEVER add TODO, FIXME, HACK, XXX, or similar markers without a Jira reference.**

It's acceptable to leave work incomplete and tag it for future completion, but every such marker MUST include a ticket reference to track the work.

✅ **Correct:**
```c
/* IFOESW-1234: TODO: remove this once firmware no longer needs it */
/* IFOESW-5678: FIXME: this should use the allocator API */
/* IFOESW-9012: HACK: temporary workaround until hardware bug is fixed */
```

❌ **Incorrect:**
```c
/* TODO: fix this */
/* FIXME: memory leak here */
/* HACK: remove later */
```

This ensures all technical debt is tracked and can be prioritized appropriately.

### General Principles
- Separate concerns (e.g., transport vs. logic)
- Write clear, maintainable code that expresses intent

## Typical Work Patterns

### Bug Fixes
- Identify root cause (memory leaks, incorrect pointer math, missing state checks)
- Minimal fixes that address the core issue
- Add assertions to catch similar issues in debug builds

### Feature Implementation
- Start with preparatory refactoring
- Add infrastructure (allocators, queues, state tracking)
- Implement feature incrementally
- Clean up after implementation complete

### Integration Updates
- Pull in external dependencies (`external/mpifoe-fw`)
- Update internal APIs to match external changes
- Provide implementations for new driver functions
- Update configuration and type mappings

### Cleanup Work
- Remove unused/dead code (`strip unused cruft`)
- Fix misleading naming or logging
- Correct violations of design intent
- Improve code clarity without functional changes

## Notes
- Typos in commit messages are acceptable (focus is on clarity of intent)
- Tag code for future removal/cleanup when needed
- Prefer small, focused commits over large omnibus changes
- Always verify changes compile and maintain build integrity

## Whitespace Rules
- **No trailing whitespace**: Lines must not end with whitespace characters
- **Blank lines must be empty**: Even blank lines should contain no whitespace
- This applies to all file types: source code, documentation, configuration files

## Repository-Specific Style
For code style details specific to individual repositories (naming conventions, memory management patterns, architecture principles, etc.), refer to `.coding-style.md` in the repository root.
