---
name: dev-knowledge-ifoe-subsystem
description: >
  Reference for the IFoE subsystem (a.k.a. "station") — core IFoE plus the XRSEC,
  XRPFC, and XRMAC blocks beneath it — and how it connects to the SDP switch, data
  fabric, and the wider MI450 GPU. Use when reading, writing, reviewing, or
  debugging code, RTL, or models involving the IFoE subsystem, an IFoE station,
  XRSEC / XRPFC / XRMAC, IFoE ports, IFoE port routing/redundancy, DXS/DXM, the
  SDP switch, or how IFoE fits into a MID / MI450. Covers the block stack below
  core IFoE, the 4/2/1 port modes, accelerator-id+path port routing for redundancy,
  the SDP-side connectivity (IFoE Completer <- DXS Originator, IFoE Originator ->
  DXM Completer, via SDP SW into DF to GPU/DMA/HBM), and the station-count
  hierarchy (up to 18 stations per MID, 2 MIDs per MI450). For the core IFoE block
  itself (streams, sequencing, replay) see dev-knowledge-ifoe-core; for SDP see
  dev-knowledge-sdp-interface. Trigger phrases: "IFoE subsystem", "IFoE station",
  "station", "XRSEC", "XRPFC", "XRMAC", "IFoE ports", "IFoE port routing", "DXS",
  "DXM", "SDP switch", "MID", "Media IO Die", "MI450 IFoE".
---

# IFoE Subsystem (Station)

The **IFoE subsystem** — also called a **station** — is the combination of:

- **Core IFoE** (see `dev-knowledge-ifoe-core`), and the three blocks beneath it:
- **XRSEC** — crypto
- **XRPFC** — flow control and buffers
- **XRMAC** — MAC functionality

## Ports

XRSEC / XRPFC / XRMAC together provide multiple **ports** to the ethernet fabric.
The number of ports depends on the operating mode: **4, 2, or 1**.

### Stream-to-port routing and redundancy

Streams are routed to ports via a **table indexed by `(accelerator id, path)`**
— the same stream-identity fields established by core IFoE. Because the mapping is
table-driven, streams can be re-routed across ports, giving **redundancy against
port loss**: if a port goes down, its streams can be moved to surviving ports.

## SDP-side connectivity

Core IFoE's two top-edge SDP interfaces connect into the SDP fabric:

- The IFoE **Completer** interface connects to the **DXS Originator** interface.
- The IFoE **Originator** (initiator) interface connects to the **DXM Completer**
  interface.

DXS and DXM are reached via the **SDP switch (SDP SW)**. The SDP switch in turn
connects into the **DF (Data Fabric)**, which routes to **GPU, DMA engines, HBM**,
etc.

Direction summary:

```
DXS.Originator ---> IFoE.Completer        (into IFoE, request side)
IFoE.Originator ---> DXM.Completer         (out of IFoE, toward fabric)
        (DXS / DXM reached via SDP SW -> DF -> GPU / DMA / HBM)
```

## Hierarchy: station -> MID -> MI450

- A **Media IO Die (MID)** can have up to **18 stations**.
- An **MI450 GPU** has **2 MIDs**.

So a single MI450 can host up to 36 IFoE stations (2 MIDs × 18).

## Naming note

The blocks were given as XRSEC (crypto), XRPFC (flow control/buffers), and XRMAC
(MAC). One source mention spelled the flow-control block "SRPFC"; it is recorded
here as **XRPFC** to match the XR-prefixed family. Confirm if the spelling matters
in a specific context.
