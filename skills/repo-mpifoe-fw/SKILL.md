---
name: repo-mpifoe-fw
description: >
  Repository reference and coding concerns for the mpifoe-fw firmware repo
  (IFoE management CPU firmware). Covers config-value justification rules,
  interrupt handling, reset-hook ordering, and EFTEST guards. Use when
  writing, reviewing, or debugging code in mpifoe-fw.
---

# mpifoe-fw — Repository Reference and Coding Concerns

This is reference knowledge applied when working on the mpifoe-fw firmware
(the IFoE management CPU firmware). When reviewing changes here, verify they
follow the concerns below in addition to the general practices in
`coding-instructions.md`.

## Configuration value justification (IFOESW-205)

There is an ongoing effort (IFOESW-205) to remove redundancy from
verification-derived configuration values and add proper justification.

- Applies to configuration initialization code, especially `#if 1` blocks
  carrying IFOESW-205 comments.
- Any modification to configuration values marked with IFOESW-205 comments
  **requires detailed explanation and cannot be approved without proper
  justification**. Specifically, such changes must include:
  1. Removal of any redundant values that can be derived from other sources
  2. Justification with citations from the architecture spec (e.g. the
     mission-mode configuration section)
  3. Detailed explanation of behavioural changes from the verification baseline
  4. Validation that changes are correct

## Interrupt handling

### Clear status bits after handling

Interrupt handlers must clear status bits after handling to prevent repeated
interrupts. Clear **after** the interrupt is fully handled, clear the correct
status register/bit, and clear even when delegating to a callback.

```c
✅ CORRECT:
void ex_packet_read_pending(ex_t *ex)
{
  if (ex->packet_read_pending_callback != NULL)
    ex->packet_read_pending_callback();

  /* Clear the status bit to prevent repeated interrupts */
  uint32_t status = BIT(VUL_EX_MR_PIO_CSR_S2R_STA_S2R_PKT_RD_PEND__LBN);
  sys_write32(status, ex->base + VUL_EX_CSR_PIO_S2R_STA__ADDR);
}

❌ INCORRECT:
void ex_packet_read_pending(ex_t *ex)
{
  if (ex->packet_read_pending_callback != NULL)
    ex->packet_read_pending_callback();
  /* Missing status bit clear - will cause repeated interrupts! */
}
```

Failing to clear interrupt status bits causes the interrupt to fire
repeatedly, wasting CPU and potentially causing instability.

### Architectural dependencies in interrupt routing

Question architectural implications when routing interrupts through unrelated
subsystems. A shared interrupt status register (multiple sources sharing one
status register) is a common source of inappropriate coupling.

Red flags:
- Routing packet/data interrupts through error-handling subsystems
  (e.g. packet notifications through the RAS driver)
- Creating dependencies between unrelated functional areas
- Mixing concerns (e.g. RAS driver handling non-error notifications)

Alternatives to consider:
- Main driver redispatches to appropriate handlers
- A separate IRQ dispatch layer that routes to multiple handlers
- Shared interrupt infrastructure that doesn't couple subsystems

```c
❌ QUESTIONABLE:
/* In RAS driver - handling packet notifications */
if (s2r_status & BIT(PACKET_READ_PENDING))
  ex_packet_read_pending(ex);  // Routing packet event through error handler

✅ BETTER:
a) Main EX driver handles interrupt, dispatches to RAS and packet handlers
b) Generic interrupt dispatcher routes to both RAS and packet subsystems
```

Flag for discussion when a dependency feels conceptually wrong, mixes
unrelated concerns, or works but violates design intent. Also ensure status
bits are cleared after handling (see above).

Rationale: architectural dependencies should align with functional
relationships; error-handling subsystems shouldn't handle non-error events;
better separation of concerns improves maintainability; consider the long-term
implications of dependency choices.

## Reset hook ordering and dependencies

Reset hooks must respect dependency ordering, especially during the FINI phase.

- Reset hooks run in **reverse order** during `RESET_PHASE_FINI` (higher
  priority runs first during fini).
- Components with dependencies need appropriate priority spacing; use the
  **2xx range** for reset-hook priorities to leave space for future additions.
- Always comment ordering constraints in the code.

Dependency examples:
- **Event queues (EVQ)** destroyed AFTER traffic queues (TXQ/RXQ), since
  TXQ/RXQ depend on EVQ. Priority: EVQ at 200, TXQ/RXQ at 201 (higher number
  runs first during FINI).
- **Doorbell queue** destroyed AFTER RX/TX queues (which depend on it):
  doorbell at a lower number than RX/TX.

Reset-hook client-type handling:
- Each hook should only free resources for its specific `client_id`.
- Do NOT use fallthrough to free resources for other client types — the
  framework calls hooks for each affected client separately (e.g. a PF FLR
  triggers hooks for VF first, then PF).

```c
✅ CORRECT:
/* Priority 200: EVQ must be destroyed after TXQ/RXQ (which are at 201) */
DECLARE_RESET_HOOK(evq_reset, 200, ...)
/* Priority 201: TXQ/RXQ destroyed before EVQ */
DECLARE_RESET_HOOK(traffic_reset, 201, ...)

// In hook body - only handle this client's resources:
case MC_CLIENT_TYPE_PF:
  fini_evq_index(EVQ_PF);  // Only PF's EVQ
  break;
case MC_CLIENT_TYPE_VF:
  fini_evq_index(EVQ_VF);  // Only VF's EVQ
  break;

❌ INCORRECT:
DECLARE_RESET_HOOK(evq_reset, 100, ...)      // Priority too low, no spacing
DECLARE_RESET_HOOK(traffic_reset, 100, ...)  // Same priority as EVQ!

case MC_CLIENT_TYPE_PF:
  fini_evq_index(EVQ_PF);
  __attribute__((fallthrough));  // Wrong! Framework handles VF separately
case MC_CLIENT_TYPE_VF:
  fini_evq_index(EVQ_VF);
  break;
```

Rationale: proper ordering prevents use-after-free and ensures clean teardown;
the framework handles multi-client resets automatically; comments document
ordering constraints for future maintainers; priority spacing allows future
hooks to be inserted.

## EFTEST guards

EFTEST-only functionality must be guarded with `#if defined(CONFIG_EFTEST)`.

- Guard both declarations and definitions.
- Functions using EFTEST-only fields must also be guarded.
- Even within `*_eftest.c` files, individual functions may still need guards.

```c
✅ CORRECT:
#if defined(CONFIG_EFTEST)
ifoe_ss_t *ifoe_ss_get_capturing_instance(uint32_t index) {
  ...
}
#endif

❌ INCORRECT:
// In production code without guard:
ifoe_ss_t *ifoe_ss_get_capturing_instance(uint32_t index) {
  ...
}
```
