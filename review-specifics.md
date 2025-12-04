# Review Specifics Template

This file contains repository-specific coding style, architecture principles, and current priorities for automated review analysis. Copy and customize this template for each repository you review.

---

## Repository Identification

**Repository Name**: [e.g., mpifoe-emulator, sdp-firmware]

**Purpose**: [Brief description of what this repository does]

**Primary Languages**: [e.g., C, C++, Lua]

**Key Subsystems**:
- [Subsystem 1]: [Brief description]
- [Subsystem 2]: [Brief description]
- [Subsystem 3]: [Brief description]

---

## Coding Style Specifics

### Naming Conventions

**Function naming**:
- Public functions: [e.g., `module_function_name()` with module prefix]
- Static functions: [e.g., no prefix required, or specific pattern]
- Callback functions: [e.g., `on_event_name()` or specific pattern]

**Variable naming**:
- Global variables: [e.g., `g_variable_name` or specific pattern]
- Static variables: [e.g., `s_variable_name` or specific pattern]
- Local variables: [e.g., snake_case, camelCase, or specific pattern]
- Constants: [e.g., `UPPER_CASE` or specific pattern]

**Type naming**:
- Structs: [e.g., `struct module_name_s` or specific pattern]
- Typedefs: [e.g., `module_name_t` or specific pattern]
- Enums: [e.g., `enum module_state_e` or specific pattern]

**File naming**:
- Source files: [e.g., `module-name.c` or specific pattern]
- Header files: [e.g., `module-name.h` or specific pattern]

### Memory Management Patterns

**Allocation**:
- [e.g., "Use module-specific allocators, not direct malloc/free"]
- [e.g., "Allocate at pipeline start, free at pipeline end"]
- [e.g., "Use arena allocators for temporary allocations"]

**Ownership**:
- [e.g., "Caller owns returned pointers unless documented otherwise"]
- [e.g., "Use _take() suffix for functions that take ownership"]
- [e.g., "Use _borrow() suffix for functions that borrow references"]

**Error path cleanup**:
- [e.g., "Always unwind allocations in reverse order on error paths"]
- [e.g., "Use goto cleanup pattern for complex initialization"]
- [e.g., "Set pointers to NULL after freeing"]

### Error Handling Approaches

**Return values**:
- [e.g., "Return 0 on success, negative errno on failure"]
- [e.g., "Return pointer on success, NULL on failure"]
- [e.g., "Use specific error enum type"]

**Assertions**:
- [e.g., "Use assert() for programming errors (debug only)"]
- [e.g., "Use runtime checks for external/hardware errors"]
- [e.g., "Assert preconditions at function entry"]

**Logging**:
- [e.g., "Use LOG_ERR for errors, LOG_WRN for warnings, LOG_INF for info"]
- [e.g., "Include context (module, function) in log messages"]
- [e.g., "Log at point of detection, not propagation"]

### Module Organization

**File structure**:
- [e.g., "One module per file pair (.c/.h)"]
- [e.g., "Group related modules in subdirectories"]
- [e.g., "Keep platform-specific code in separate files"]

**Header organization**:
- [e.g., "Public API in main header, private in -internal.h"]
- [e.g., "Include guards: `#ifndef MODULE_NAME_H`"]
- [e.g., "Order: system headers, external headers, internal headers"]

**Source organization**:
- [e.g., "Static helpers at top, public API at bottom"]
- [e.g., "Group by functionality, not alphabetically"]
- [e.g., "Keep related functions together"]

---

## Architecture Principles

### Design Patterns in Use

**[Pattern Name 1]**: [e.g., Producer-Consumer]
- Where used: [e.g., "SDP streaming pipeline"]
- Key characteristics: [e.g., "Lock-free ring buffers between stages"]
- Common mistakes: [e.g., "Forgetting to handle backpressure"]

**[Pattern Name 2]**: [e.g., State Machine]
- Where used: [e.g., "Protocol handlers"]
- Key characteristics: [e.g., "Explicit state enum, transition functions"]
- Common mistakes: [e.g., "Missing state transitions, no invalid state handling"]

**[Pattern Name 3]**: [e.g., Dependency Injection]
- Where used: [e.g., "Hardware abstraction layer"]
- Key characteristics: [e.g., "Function pointers in config structs"]
- Common mistakes: [e.g., "Not validating function pointers before use"]

### Layering and Separation of Concerns

**Layer 1**: [e.g., Hardware Abstraction]
- Responsibilities: [e.g., "Direct hardware register access"]
- Must not: [e.g., "Contain business logic or protocol knowledge"]
- Depends on: [e.g., "Nothing (bottom layer)"]

**Layer 2**: [e.g., Device Drivers]
- Responsibilities: [e.g., "Device initialization, interrupt handling"]
- Must not: [e.g., "Know about higher-level protocols"]
- Depends on: [e.g., "Hardware abstraction layer"]

**Layer 3**: [e.g., Protocol Implementation]
- Responsibilities: [e.g., "Protocol state machines, message handling"]
- Must not: [e.g., "Access hardware directly"]
- Depends on: [e.g., "Device drivers"]

**Layer 4**: [e.g., Application Logic]
- Responsibilities: [e.g., "Business logic, orchestration"]
- Must not: [e.g., "Know about hardware details"]
- Depends on: [e.g., "Protocol layer"]

### Interface Contracts

**[Interface Name 1]**: [e.g., Stream Processing]
- Contract: [e.g., "process() returns number of items consumed"]
- Preconditions: [e.g., "Input buffer must be non-NULL"]
- Postconditions: [e.g., "Return value <= input count"]
- Thread safety: [e.g., "Not thread-safe, caller must synchronize"]

**[Interface Name 2]**: [e.g., Resource Allocation]
- Contract: [e.g., "alloc() returns NULL on failure"]
- Preconditions: [e.g., "Size must be > 0"]
- Postconditions: [e.g., "Returned pointer is aligned"]
- Thread safety: [e.g., "Thread-safe, uses internal locking"]

### Performance Considerations

**Critical paths**:
- [e.g., "Packet processing must complete in < 1ms"]
- [e.g., "Interrupt handlers must be < 100 instructions"]
- [e.g., "No dynamic allocation in real-time paths"]

**Optimization priorities**:
- [e.g., "Latency over throughput for control messages"]
- [e.g., "Memory efficiency over CPU for embedded targets"]
- [e.g., "Code size matters for firmware"]

**Profiling requirements**:
- [e.g., "Profile before optimizing"]
- [e.g., "Document performance assumptions in comments"]
- [e.g., "Add timing instrumentation for critical sections"]

---

## Current Priorities

### Active Refactoring Efforts

**[Effort Name 1]**: [e.g., "SDP Allocator Migration"]
- Goal: [e.g., "Replace malloc/free with custom allocators"]
- Status: [e.g., "In progress, 60% complete"]
- What to watch for: [e.g., "New code should use allocators, not malloc"]
- Related tickets: [e.g., "IFOESW-546, IFOESW-547"]

**IFOESW-205: Configuration Value Justification**
- Goal: Remove redundancy from verification-derived configuration values and add proper justification
- Status: Ongoing - applies to configuration initialization code
- What to watch for: Changes to configuration values (especially in `#if 1` blocks with IFOESW-205 comments) must include:
  1. Removal of any redundant values that can be derived from other sources
  2. Justification with citations from architecture spec (e.g., mission mode configuration section)
  3. Detailed explanation of behavioral changes from verification baseline
  4. Validation that changes are correct
- Related tickets: IFOESW-205
- Key requirement: Any modification to configuration values marked with IFOESW-205 comments requires detailed explanation and cannot be approved without proper justification

**[Effort Name 2]**: [e.g., "Error Handling Standardization"]
- Goal: [e.g., "Consistent error return codes across modules"]
- Status: [e.g., "Planning phase"]
- What to watch for: [e.g., "Don't introduce new error patterns"]
- Related tickets: [e.g., "IFOESW-XXX"]

### Known Technical Debt

**[Debt Item 1]**: [e.g., "Legacy Protocol Handler"]
- Issue: [e.g., "Uses global state, not thread-safe"]
- Impact: [e.g., "Blocks multi-instance support"]
- Plan: [e.g., "Rewrite in Q2 2025"]
- What to do: [e.g., "Don't make it worse, isolate dependencies"]

**[Debt Item 2]**: [e.g., "Inconsistent Logging"]
- Issue: [e.g., "Mix of printf and LOG_* macros"]
- Impact: [e.g., "Hard to filter logs"]
- Plan: [e.g., "Gradual migration"]
- What to do: [e.g., "New code must use LOG_* macros"]

### Areas Under Active Development

**[Area 1]**: [e.g., "New DMA Engine Support"]
- Status: [e.g., "Feature branch, merging soon"]
- Key changes: [e.g., "New DMA API, scatter-gather support"]
- Review focus: [e.g., "API design, error handling"]
- Contact: [e.g., "@developer-name for questions"]

**[Area 2]**: [e.g., "Firmware Update Mechanism"]
- Status: [e.g., "Design review"]
- Key changes: [e.g., "Bootloader integration"]
- Review focus: [e.g., "Security, rollback handling"]
- Contact: [e.g., "@developer-name for questions"]

### Deprecated Patterns to Avoid

**[Pattern 1]**: [e.g., "Direct Register Access"]
- Why deprecated: [e.g., "Breaks hardware abstraction"]
- Use instead: [e.g., "HAL functions"]
- Exceptions: [e.g., "None - always use HAL"]

**[Pattern 2]**: [e.g., "Busy-Wait Loops"]
- Why deprecated: [e.g., "Wastes CPU, blocks other tasks"]
- Use instead: [e.g., "Event-driven with callbacks"]
- Exceptions: [e.g., "Short delays < 10us in interrupt context"]

**[Pattern 3]**: [e.g., "Global Mutable State"]
- Why deprecated: [e.g., "Not thread-safe, hard to test"]
- Use instead: [e.g., "Context structs passed as parameters"]
- Exceptions: [e.g., "Hardware singleton resources only"]

---

## Common Issues

### Frequent Mistakes

**[Mistake 1]**: [e.g., "Forgetting to check return values"]
- Symptom: [e.g., "Crashes on error paths"]
- How to detect: [e.g., "Look for unchecked function calls"]
- How to fix: [e.g., "Add error checks, handle failures"]

**[Mistake 2]**: [e.g., "Off-by-one in buffer calculations"]
- Symptom: [e.g., "Buffer overruns, corruption"]
- How to detect: [e.g., "Review loop bounds, pointer arithmetic"]
- How to fix: [e.g., "Use < not <=, add assertions"]

**[Mistake 3]**: [e.g., "Missing volatile for hardware registers"]
- Symptom: [e.g., "Compiler optimizes away reads/writes"]
- How to detect: [e.g., "Check register access patterns"]
- How to fix: [e.g., "Add volatile qualifier"]

### Anti-Patterns to Watch For

**[Anti-Pattern 1]**: [e.g., "God Objects"]
- Description: [e.g., "Single struct/module doing too much"]
- Why bad: [e.g., "Hard to test, maintain, understand"]
- What to suggest: [e.g., "Split into focused modules"]

**[Anti-Pattern 2]**: [e.g., "Magic Numbers"]
- Description: [e.g., "Unexplained numeric constants"]
- Why bad: [e.g., "Unclear intent, hard to maintain"]
- What to suggest: [e.g., "Use named constants or enums"]

**[Anti-Pattern 3]**: [e.g., "Copy-Paste Code"]
- Description: [e.g., "Duplicated logic across files"]
- Why bad: [e.g., "Bug fixes needed in multiple places"]
- What to suggest: [e.g., "Extract common function"]

### Integration Gotchas

**[Gotcha 1]**: [e.g., "Firmware Version Compatibility"]
- Issue: [e.g., "API changes between firmware versions"]
- How to handle: [e.g., "Check version, use compatibility layer"]
- What to review: [e.g., "Version checks present, fallback logic"]

**[Gotcha 2]**: [e.g., "Endianness Mismatches"]
- Issue: [e.g., "Network vs. host byte order"]
- How to handle: [e.g., "Use ntohl/htonl macros"]
- What to review: [e.g., "Byte order conversions for wire formats"]

**[Gotcha 3]**: [e.g., "Interrupt Context Restrictions"]
- Issue: [e.g., "Can't call blocking functions in ISRs"]
- How to handle: [e.g., "Defer work to thread context"]
- What to review: [e.g., "ISR code only does minimal work"]

---

## Notes

- Update this file as patterns emerge from actual reviews
- Keep it focused on actionable, verifiable rules
- Remove outdated sections when priorities change
- Link to relevant documentation where appropriate
- Include examples of good and bad code when helpful

## Last Updated

**Date**: [YYYY-MM-DD]
**Updated by**: [Your name]
**Changes**: [Brief description of what was updated]
