---
name: dev-knowledge-mi450-gpu
description: >
  Reference for MI450 GPU topology as it relates to IFoE: the MID / station
  hierarchy, the GPU-internal SDP path (DXS/DXM, SDP switch, DF to GPU/DMA/HBM),
  the GPU-to-GPU communication policy (which SDP virtual channel and which address
  space for in-domain vs cross-tray traffic), and MMHUB/L1BIU address translation.
  Use when reading, writing, reviewing, or debugging code, RTL, or models
  involving the MI450 GPU, MIDs (Media IO Die), how many IFoE stations a GPU has,
  DXS / DXM, the SDP switch, the data fabric (DF), GPU-to-GPU comms, SDP VC6/VC7
  selection, SPA / NPA / GVA addresses, or MMHUB / L1BIU translation. For the IFoE
  core block and its VC ingest/egress mechanism see dev-knowledge-ifoe-core; for
  the IFoE station block stack and port modes see dev-knowledge-ifoe-subsystem;
  for rack/tray deployment (Helios) see dev-knowledge-helios; for SDP see
  dev-knowledge-sdp-interface. Trigger phrases: "MI450", "MID", "Media IO Die",
  "IFoE stations per GPU", "DXS", "DXM", "SDP switch", "data fabric", "DF",
  "GPU to GPU comms", "VC6", "VC7", "SPA", "NPA", "GVA", "MMHUB", "L1BIU".
---

# MI450 GPU (IFoE-relevant topology)

This skill captures the MI450 GPU topology that matters for IFoE. For the IFoE
blocks themselves see `dev-knowledge-ifoe-core` and `dev-knowledge-ifoe-subsystem`.

> Sourcing note: this content is dictation-sourced and not yet cross-referenced
> against a written spec. Items marked **UNVERIFIED** below are explicitly
> uncertain. Note that the SPA/NPA/GVA/translation questions will **not** be
> answered by the IFoE MAS — they need a different (GPU / addressing) source.

## Hierarchy: station -> MID -> GPU

- A **Media IO Die (MID)** can host up to **18 IFoE subsystems (stations)**.
- An **MI450 GPU** has **2 MIDs** — so up to 36 stations per GPU in principle.

In practice, the **SerDes lanes are shared** for other purposes (e.g. connection
to a CPU), so having all 36 IFoE stations active at once is **rare**. How lanes
are actually allocated is a deployment choice — see `dev-knowledge-helios` for the
Helios allocation.

## Internal SDP path

Core IFoE's two top-edge SDP interfaces attach to the GPU's SDP fabric through
**DXS** and **DXM**:

- IFoE **Completer** interface  <- **DXS** Originator interface
- IFoE **Originator** (initiator) interface -> **DXM** Completer interface

DXS and DXM are reached via the **SDP switch (SDP SW)**, which connects into the
**DF (Data Fabric)**. DF routes to **GPU, DMA engines, HBM**, etc.

```
                 SDP SW            DF
GPU / DMA / HBM <------> DXS/DXM <----> (DXS.Orig -> IFoE.Completer)
                                        (IFoE.Orig -> DXM.Completer)
```

## GPU-to-GPU communication policy

How two GPUs communicate depends on whether they are in the same tray ("domain")
or different trays. IFoE carries the traffic in both cases; the difference is the
SDP virtual channel and the address space used:

| Peer relationship          | Considered  | SDP VC | Address space |
|----------------------------|-------------|--------|---------------|
| Same tray (2 GPUs in tray) | in-domain   | **VC7**| **SPA** (system physical address)  |
| Different trays            | (cross-tray)| **VC6**| **NPA** (network physical address) |

This is the **policy** side of VC selection. The **mechanism** lives in IFoE:
IFoE ignores the VC on SDP ingress and **synthesises** it on egress from its
knowledge of which accelerator ids are local vs remote (see
`dev-knowledge-ifoe-core`). ("Tray" / domain membership comes from the Helios
deployment — see `dev-knowledge-helios`.)

## Address translation (MMHUB / L1BIU)

- **Outgoing** transactions: **MMHUB** performs translation, from **GVA** to
  **NPA/SPA**, before the transaction is handed to IFoE via DXS.
- **Incoming** transactions: **L1BIU** translates / validates the incoming
  address.

**UNVERIFIED** (flagged by the domain expert; source is *not* the IFoE MAS):

- **GVA** is believed to expand to **"guest virtual address"** — unconfirmed.
- On the L1BIU inbound path, **SPA** may be **untranslated** while **NPA** is
  **definitely translated** — the SPA case is uncertain.

Confirm these against a GPU/addressing source before relying on them.
