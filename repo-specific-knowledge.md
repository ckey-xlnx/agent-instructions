# Repository-Specific Knowledge

This file contains repository-specific knowledge applicable to all development tasks including code review, coding, and debugging.

---

## Automated Review Checklist

When performing automated code reviews, systematically check for these items:

### 1. TODO/FIXME/HACK Markers
**Rule**: ALL TODO, FIXME, HACK, XXX markers MUST include a Jira ticket reference.

**How to check**:
- Search diff for: `TODO`, `FIXME`, `HACK`, `XXX`
- Verify each has a ticket reference (e.g., `FWDEV-12345`, `IFOESW-678`)
- Flag any without tickets as blocking issues

**Examples**:
```c
✅ CORRECT:
/* FWDEV-12345: TODO: remove this workaround when hardware bug is fixed */
/* IFOESW-678: FIXME: this should use the allocator API */

❌ INCORRECT:
/* TODO: fix this later */
/* FIXME: memory leak here */
```

**Rationale**: TODOs without tickets never get done. All technical debt must be tracked.

### 3. Duplicate Declarations
**Rule**: Function prototypes must not be duplicated in the same header file.

**How to check**:
- When reviewing header file changes, check for duplicate function declarations
- Look for copy-paste errors where the same prototype appears twice
- Verify each function is declared exactly once

### 4. Implementation Completeness
**Rule**: Related code paths must have equivalent logic for mutual exclusion.

**How to check**:
- When adding validation to one command (e.g., "can't set capture if active")
- Check if the inverse command needs validation (e.g., "can't set active if capturing")
- Look for state machine transitions that need bidirectional checks
- Verify error paths are complete

**Example from r/29107**:
- `set_rx_capture_stations()` prevents capture mode on active stations ✓
- But `set_active_stations()` should also prevent activating capturing stations ✗

### 5. Temporary vs Permanent Code
**Rule**: Distinguish between permanent infrastructure and temporary workarounds.

**What is permanent**:
- New state fields that represent valid long-term states
- Iterator macros for valid states (e.g., `FOREACH_SS_CAPTURING`)
- APIs that will be needed after workarounds are removed

**What is temporary**:
- Workarounds for simulator/hardware bugs
- Mixed-state concepts that violate design intent (e.g., `ACTIVE_OR_CAPTURING`)
- Code that will be removed when a specific ticket is resolved

**How to check**:
- Question "temporary" markers - are they really temporary?
- Suggest separating permanent infrastructure into its own commit
- Ensure temporary workarounds are clearly marked with the ticket that will remove them

**Example**:
```c
✅ PERMANENT (should not be marked temporary):
bool rx_capture;  // Valid long-term state
FOREACH_SS_CAPTURING(ifoe)  // Meaningful iterator

❌ TEMPORARY (correctly marked):
/* TODO FWDEV-131680: Remove when stations can enter showtime without being active */
FOREACH_SS_ACTIVE_OR_CAPTURING(ifoe)  // Mixing two concepts
```

### 6. Workaround Cleanup Tickets
**Rule**: Temporary workarounds need separate tickets for their removal.

**How to check**:
- When a workaround is tied to a feature ticket (e.g., FWDEV-162485)
- That feature ticket will close when the feature ships
- Request a separate ticket to track workaround removal
- The workaround comment should reference the cleanup ticket

**Example**:
```c
❌ INCORRECT:
/* TODO FWDEV-162485: Workaround for Simnow bug */

✅ CORRECT:
/* TODO FWDEV-162485: Workaround for Simnow bug
 * TODO FWDEV-99999: Remove this workaround when bug is fixed */
```

### 7. Workaround Robustness
**Rule**: Analyze and document why workarounds work and their assumptions.

**How to check**:
- When reviewing a workaround, understand why it works
- Identify if it relies on accidental equalities or assumptions
- Document these assumptions in the code
- Flag fragile workarounds that may break if assumptions change

**Example from r/29107**:
```c
/* This workaround forces all queues to PB when rx_capture is enabled.
 * NOTE: This only works because MR_PIO_PCP == IFOE_RSP_QID (both are 2).
 * If either value changes, this workaround will break.
 */
```

### 8. Commit Structure
**Rule**: Separate permanent infrastructure from temporary workarounds in commits.

**How to check**:
- Review commit organization, not just individual changes
- Permanent infrastructure should be in separate commits
- Temporary workarounds should be clearly identified
- Preparatory commits should be separated from feature commits

**Example**:
```
✅ GOOD STRUCTURE:
Commit 1: Add FOREACH_SS_CAPTURING iterator (permanent)
Commit 2: Add rx_capture field and APIs (permanent)
Commit 3: Add MCDI command handler (permanent)
Commit 4: Temporary workaround for ACTIVE_OR_CAPTURING (temporary, clearly marked)

❌ POOR STRUCTURE:
Commit 1: Add everything mixed together
```

## Notes

- Update this file as patterns emerge from actual work
- Keep it focused on actionable, verifiable information
- Remove outdated sections when priorities change
- Link to relevant documentation where appropriate
- Include examples of good and bad code when helpful
