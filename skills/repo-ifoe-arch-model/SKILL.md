---
name: repo-ifoe-arch-model
description: >
  Repository reference for the ifoe-arch-model repo (the IFoE architectural
  model, aka "ifoe-models"): its purpose, the two ways it is used, and its
  mpifoe-fw dependency. Use when writing, reviewing, or debugging code in
  ifoe-arch-model, or understanding how it relates to firmware and the
  subsystem model. (To build it on subsystem emulation use
  build-ifoe-arch-model-ss-emu.)
---

# ifoe-arch-model — Repository Reference

Reference knowledge applied when working on the ifoe-arch-model repository
(better named "ifoe-models").

## Purpose

**Primary purpose**: Contains an architectural model of IFoE.

**Secondary purpose**: Provides glue code so tests using the architectural
model API can use the same API when driving:
- Subsystem emulation (with firmware running to drive emulation)
- C model wrapper (with firmware running to drive model registers)

The same API works across the architectural model, subsystem emulation, and
the C model. That API is `ifoe_ss_model_t` and its three interchangeable
flavours (arch / emu / cmod); for the interface and what each flavour does
beneath it, see `repo-ifoe-arch-model-ss-model`.

## How it is used

The repository is used in two ways:
1. **With TE (Test Environment)** — to be described in the future.
2. **Semi-standalone on subsystem emulation** — see the
   `build-ifoe-arch-model-ss-emu` skill for the build process.

## External dependency: mpifoe-fw

**Location**: `external/mpifoe-fw` (subrepo within ifoe-arch-model)

**Purpose**: Main firmware repository that compiles firmware for real silicon
(Zephyr RTOS).

**How it's used in ifoe-arch-model**:
- The IFoE subsystem model doesn't include the management CPU.
- All firmware functionality must run natively on the X86 host.
- ifoe-arch-model compiles code from mpifoe-fw to drive the hardware.
- Provides sufficient stubs for Zephyr functionality to make this work.
- Goal: exercise as much real firmware as possible on subsystem emulation.
- Current scope: hardware abstraction layer and configuration logic.

**Key points**:
- Standard git submodule.
- Enables testing firmware code without real hardware.
- Simplifies testing and debug by running on the host system.
