---
name: dev-knowledge-sdp-interface
description: >
  Reference for the SDP (Scalable Data Port) interface — the SoC-15 point-to-point
  bus between an Originator and a Completer. Use when reading, writing, reviewing,
  or debugging code, RTL, or models that involve SDP, SDP channels, SDP credits,
  or SDP connect/disconnect. Covers the Originator/Completer roles, the four core
  message types (Req, OrigData, RdRsp, WrRsp), the credit-based flow-control model
  (who grants vs. holds credits, initial credit, remember/forget on disconnect),
  the six port-control signals (OrigClkReq/Ack/Ctl, CompClkReq/Ack/Ctl) and the
  connect/disconnect handshake, and the Vld/Rdy data+credit transfer convention.
  Trigger phrases: "SDP", "SDP interface", "SDP request/response", "SDP credit",
  "ReqVld", "ReqCreditVld", "OrigClkReq", "CompClkAck", "what does OrigClkCtl do",
  "SDP connect/disconnect", "SDP originator/completer".
---

# SDP (Scalable Data Port) Interface

The SDP interface carries **SDP requests and responses** between two agents over a
point-to-point link. It is the SoC-15 Scalable Data Port; the authoritative
reference is the *SoC-15 Architecture Specification — Scalable Data Port (SDP)*
(Rev 0.21). This skill is a working summary of the link-level mechanics most often
needed when developing or reviewing SDP code, not a full transcription of the spec.

## Roles

Two sides, asymmetric in what they carry but symmetric in their control signalling:

- **Originator** — issues requests. **Produces** `Req` and `OrigData`;
  **consumes** `RdRsp` and `WrRsp`.
- **Completer** — satisfies requests. **Produces** `RdRsp` and `WrRsp`;
  **consumes** `Req` and `OrigData`.

(`OrigData` is the Originator Data channel — write data, and probe-response data
when probes are supported. `RdRsp`/`WrRsp` are the read- and write-response
channels.)

> Scope note: the full SDP interface has **eight** physical channels — the four
> above plus Probe Request, Probe Response, and Response Acknowledge. This skill
> focuses on the four core data/response channels; the credit and handshake
> mechanics described here generalise to all of them.

## Credit-based flow control

The interface is **credit driven**, per message type (Req, OrigData, RdRsp, WrRsp
each have their own independent credit pool).

The rule, stated two equivalent ways:

- **The consumer of a message type grants (issues) credits for that type.**
  The producer of that type may send only when it holds positive credit, and each
  send **consumes** one credit.
- Equivalently: **the producer of a message type is the credit *holder*** — it
  receives the credits and spends them. Credits flow opposite to the data.

Worked direction for each type:

| Message type | Producer (holds/spends credit) | Consumer (grants credit) |
|--------------|--------------------------------|--------------------------|
| `Req`        | Originator                     | Completer                |
| `OrigData`   | Originator                     | Completer                |
| `RdRsp`      | Completer                      | Originator               |
| `WrRsp`      | Completer                      | Originator               |

So the Completer grants Req/OrigData credit to the Originator; the Originator
grants RdRsp/WrRsp credit to the Completer. A credit represents one unit of buffer
space in the consumer (e.g. one request buffer, or one aligned data block).

Credit counters are **saturating** — the holder must record incoming credits and
must not let the count wrap past its maximum.

**At start of day (reset), all credit is zero.** Nothing can be sent until the
consumer issues credit.

## Initial credit, and remember/forget across disconnect

The agent that controls whether initial credit is (re)issued is the **credit
holder** = the **message producer** (the side that *receives* that credit type).
It expresses this on connect via *its own* `ClkCtl` signal (see below):

- **On connect**, the producer signals whether it wants initial credit for the
  channels it transmits on. The consumer honours the request, granting initial
  credit **commensurate with its internal storage** (buffer space).
  - The Originator, via `OrigClkCtl`, controls initial credit for `Req` and
    `OrigData` (the two types it produces / receives credit for).
  - The Completer, via `CompClkCtl`, controls initial credit for `RdRsp` and
    `WrRsp`.
  - A single `ClkCtl` covers **both** channels that side produces — there is no
    per-channel granularity in the connect request.
- **On disconnect**, that same credit holder chooses to either:
  - **(a) remember** its credit — and therefore *not* request initial credit on
    reconnect (`ClkCtl` inactive at the next connect), or
  - **(b) forget** its credit — and request initial credit on reconnect
    (`ClkCtl` active at the next connect).

## Port-control signals and the connect/disconnect handshake

Connection is mediated by **six** signals, three driven by each side, **wholly
symmetric**:

| Driven by Originator | Driven by Completer | Role |
|----------------------|---------------------|------|
| `OrigClkReq`         | `CompClkReq`        | connect request (1) / disconnect request (0) |
| `OrigClkAck`         | `CompClkAck`        | acknowledge the *other* side's Req |
| `OrigClkCtl`         | `CompClkCtl`        | initial-credit control, sampled at the Req edge |

Handshake semantics (mirror-image in both directions):

- **Connect**: `OrigClkReq` 0→1 requests connection; `CompClkAck`=1 acknowledges.
  Symmetrically, `CompClkReq` 0→1 is acknowledged by `OrigClkAck`=1. When all four
  Req/Ack signals are asserted the port is in its **active state** and information
  can flow.
- **Initial credit**: if a side's `ClkCtl` is **high** at the moment *its own*
  `Req` transitions 0→1, that side is requesting initial credit be (re)issued for
  the channels it produces. `ClkCtl` low ⇒ do not (re)issue. The requester must
  hold `ClkCtl` stable until its connection request is acknowledged.
- **Disconnect**: driving `Req` 0→1→**0** requests disconnect; the other side's
  `Ack` de-asserting is the acknowledgement. (`ClkCtl` state at the falling edge of
  `Req` additionally distinguishes a *full* disconnect from a *one-sided* / temporary
  disconnect under the enhanced-port-control feature — consult the spec for that
  detail when needed.)

> `OrigClkCtl` / `CompClkCtl` belong to the **Enhanced Port Control** feature.
> If that feature is absent, initial-credit negotiation is handled by a means
> outside the SDP spec.

## Data and credit transfer — Vld/Rdy

Every channel — both the **data/message** channels and the **credit-release**
sub-channels — uses a **Valid/Ready** handshake:

- Data/message: e.g. `ReqVld` / `ReqRdy`, `OrigDataVld` / `OrigDataRdy`,
  `RdRspVld` / `RdRspRdy`, `WrRspVld` / `WrRspRdy`.
- Credit grant: e.g. `ReqCreditVld` / `ReqCreditRdy` (and the matching
  `*CreditVld` / `*CreditRdy` on the other channels). The credit-release
  sub-channel carries flow control in the direction *opposite* to its data
  channel.

A transfer occurs on an active clock edge when **`Vld && Rdy`** are both high.

**`Rdy` is implementation-dependent.** Some implementations provide a real `Rdy`
(used for clock-domain pacing / synchronisation); others omit it, in which case it
is **implicitly 1** (always ready → a transfer happens on every cycle `Vld` is
asserted). For point-to-point connections the `Rdy` input on outbound channels is
typically tied active. Note: `Vld`/`Rdy` is for **pacing/synchronisation only**,
**not** for flow control — flow control is the credit system. A producer must
already hold a credit before asserting `Vld`.

## Quick mental model

1. Consumer grants credit (buffer space) → producer accumulates it.
2. Producer spends a credit to send a message (`Vld`, transfers when `Rdy`).
3. Consumer frees the buffer → releases a credit back on the credit sub-channel.
4. Connect/disconnect is a symmetric Req/Ack handshake; `Ctl` at the Req edge says
   whether the credit holder wants its initial credit (re)issued, and the holder
   remembers-or-forgets credit across a disconnect.
