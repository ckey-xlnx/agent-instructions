---
name: dev-knowledge-helios
description: >
  Reference for Helios — the rack-level deployment of MI450 GPUs and how IFoE is
  configured within it. Use when reading, writing, reviewing, or debugging code,
  RTL, or models involving Helios, a Helios rack, a Helios tray, the SerDes lane
  allocation (MID0 -> CPU, MID1 -> IFoE), the 2x400G IFoE port mode in Helios, or
  Helios IFoE port counts. Covers the rack composition (36 compute trays + 6
  switches), the tray (4 GPUs), how SerDes lanes are split between CPU and IFoE,
  the 2x400G two-port IFoE mode, and the resulting per-GPU / per-tray port counts,
  plus the in-domain (same-tray) vs cross-tray distinction that drives GPU-to-GPU
  comms. For the GPU topology and GPU-to-GPU VC/address policy see
  dev-knowledge-mi450-gpu; for the IFoE blocks see dev-knowledge-ifoe-core and
  dev-knowledge-ifoe-subsystem. Trigger phrases: "Helios", "Helios rack", "Helios
  tray", "compute tray", "SerDes lanes", "MID0 / MID1 lanes", "2x400G", "IFoE port
  count", "GPUs per tray", "in-domain vs cross-tray".
---

# Helios (Rack Deployment)

**Helios** is a **rack** built from MI450 GPUs. This skill captures the
Helios-specific deployment facts; for the GPU chip itself see
`dev-knowledge-mi450-gpu`.

> Sourcing note: dictation-sourced, not yet cross-referenced against a written
> spec.

## Composition

- A **Helios rack** = **36 compute trays** + **6 switches**.
- A **Helios tray** = **4 GPUs**.

## SerDes lane allocation

On each GPU in a Helios tray, the two MIDs' SerDes lanes are dedicated by function:

- **MID0** SerDes lanes → **CPU connection** (all of them).
- **MID1** SerDes lanes → **IFoE** (all of them).

(This is the concrete instance of the "SerDes lanes are shared for other purposes"
caveat noted in `dev-knowledge-mi450-gpu`: in Helios, MID0 goes entirely to the
CPU, so only MID1 carries IFoE.)

## IFoE port mode and port counts

All IFoE in Helios runs in **2×400G mode**, so each IFoE subsystem presents
**2 ports** (see `dev-knowledge-ifoe-subsystem` for port modes).

This yields:

- **36 IFoE ports per GPU**
- **144 IFoE ports per tray** (36 × 4 GPUs)

## In-domain vs cross-tray

The tray boundary defines the communication "domain" used for GPU-to-GPU traffic:

- Two GPUs **in the same tray** → **in-domain**.
- GPUs **in different trays** → cross-tray.

Both go through IFoE, but they use different SDP virtual channels and address
spaces (VC7+SPA in-domain, VC6+NPA cross-tray). That policy is described in
`dev-knowledge-mi450-gpu`.
