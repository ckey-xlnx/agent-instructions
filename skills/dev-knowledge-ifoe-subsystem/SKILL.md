---
name: dev-knowledge-ifoe-subsystem
description: >
  Reference for the IFoE subsystem (a.k.a. "station") — core IFoE plus the XRSEC,
  XRPFC, and XRMAC blocks beneath it that provide the ethernet ports. Use when
  reading, writing, reviewing, or debugging code, RTL, or models involving the
  IFoE subsystem, an IFoE station, XRSEC / XRPFC / XRMAC, or IFoE port modes.
  Covers the block stack below core IFoE and the 4/2/1 port modes (how many
  ethernet ports the station presents). Note: the subsystem *provides* ports but
  does not route streams to them — (accelerator id, path) -> port routing lives in
  core IFoE (path is an IFoE-only concept); see dev-knowledge-ifoe-core. For the
  GPU topology, station counts (MID/MI450) and the internal SDP path (DXS/DXM, SDP
  switch, DF) see dev-knowledge-mi450-gpu; for SDP see dev-knowledge-sdp-interface.
  Trigger phrases: "IFoE subsystem", "IFoE station", "station", "XRSEC", "XRPFC",
  "XRMAC", "IFoE ports", "IFoE port modes", "2x400G".
---

# IFoE Subsystem (Station)

The **IFoE subsystem** — also called a **station** — is the combination of:

- **Core IFoE** (see `dev-knowledge-ifoe-core`), and the three blocks beneath it:
- **XRSEC** — crypto
- **XRPFC** — flow control and buffers
- **XRMAC** — MAC functionality

## Ports

XRSEC / XRPFC / XRMAC together provide the **ports** to the ethernet fabric. The
number of ports the station presents depends on the operating mode: **4, 2, or 1**.
(For example, Helios runs IFoE in 2×400G mode = 2 ports — see
`dev-knowledge-helios`.)

The subsystem **provides** the ports; it does **not** decide which stream uses
which port. Stream-to-port routing is keyed on `(accelerator id, path)` and lives
in **core IFoE**, because *path* is an IFoE-only concept and the IFoE→XRSEC
interface already speaks in terms of ports. See `dev-knowledge-ifoe-core` for the
routing table and its port-loss redundancy.

## Where the station sits

A station attaches to the GPU's SDP fabric via core IFoE's two top-edge SDP
interfaces (through DXS/DXM). Station counts per MID, per-GPU topology, and the
internal SDP path (SDP switch -> DF -> GPU/DMA/HBM) are covered in
`dev-knowledge-mi450-gpu`.

## Naming note

The blocks were given as XRSEC (crypto), XRPFC (flow control/buffers), and XRMAC
(MAC). One source mention spelled the flow-control block "SRPFC"; it is recorded
here as **XRPFC** to match the XR-prefixed family. Confirm if the spelling matters
in a specific context.
