---
name: firmware-postcodes
description: >
  Reference for interpreting firmware postcodes / status registers across the
  IFoE firmwares (ASP-FMC FW_STATUS_REG, and mpifoe-fw postcodes). Use when
  asked what a postcode or FW_STATUS_REG value means, to decode a boot/error
  code, to find where postcode enums are defined, or when debugging firmware
  boot progress or hangs.
---

# Firmware Postcodes

Reference for decoding firmware postcodes / status registers across the IFoE
firmwares. Each firmware has its own scheme; find the relevant section below.

## ASP-FMC postcodes (FW_STATUS_REG)

The ASP (Application Security Processor) reports firmware execution progress
through the `FW_STATUS_REG` register — a 32-bit composite view of the current
firmware state across three log levels.

### FW_STATUS_REG format

The register value has format `0xAATTSSEE`:

| Bits    | Field   | Log Level    | Description              |
|---------|---------|--------------|--------------------------|
| 31-24   | `0xAA`  | -            | Fixed FMC domain prefix  |
| 23-16   | `TT`    | TRACE_LOG    | Current trace/progress   |
| 15-8    | `SS`    | STATUS_LOG   | Current status           |
| 7-0     | `EE`    | ERROR_LOG    | Error code (if any)      |

### How values are updated

Each log level updates only its portion of the register:
- `post_code(code, TRACE_LOG)` → sets bits 23-16 to `code`, preserves other bits
- `post_code(code, STATUS_LOG)` → sets bits 15-8 to `code`, preserves other bits
- `post_code(code, ERROR_LOG)` → sets bits 7-0 to `code`, preserves other bits

### Common error codes (bits 7-0)

From the `FMC_RETCODE` enum in `common/src/include/fmc_postcode.h`:

| Value | Constant                            | Meaning                           |
|-------|-------------------------------------|-----------------------------------|
| 0x00  | `FMC_OK`                            | Success / No error                |
| 0x01  | `FMC_ERR_GENERIC`                   | Generic error                     |
| 0x0C  | `FMC_ERR_TIMEOUT`                   | Timeout                           |
| 0x10  | `FMC_ERR_DATA_CORRUPTION`           | Data corruption                   |
| 0x19  | `FMC_ERR_CCP_SHA`                   | SHA operation failed              |
| 0x32  | `FMC_CALIPTRA_ERROR`                | Caliptra error detected           |
| 0x93  | `FMC_RECOVERY_FW_VALIDATION_FAIL`   | Recovery FW validation failed     |
| 0xEE  | `FMC_ERR_SOCKET_DOWN_BOOT_OVERFLOW_ERROR` | Boot overflow with socket down |

### Common trace codes (bits 23-16)

From the `FMC_TRACE` enum showing boot progress:

| Value | Constant                    | Meaning                           |
|-------|-----------------------------|-----------------------------------|
| 0x01  | `FMC_LOAD_SUCCESS`          | Stage 1 bootloader started        |
| 0x02  | `FMC_INIT_LIBROM`           | LIBROM initialization             |
| 0x04  | `FMC_LOAD_VALIDATE_IMAGE`   | Loading/validating FW image       |
| 0x08  | `FMC_ART_INIT`              | Initializing ART communication    |
| 0x0A  | `FMC_PLAT_INIT`             | Platform data initialization      |
| 0x12  | `FMC_SPI_FLASH_INIT`        | SPI flash directory init          |
| 0x14  | `FMC_HSP_INIT`              | HSP initialization                |
| 0x20  | `FMC_MULTI_SOCKET_BOOT`     | Multi-socket boot                 |
| 0xAA  | `FMC_JUMP_TO_BOOTLOADER`    | Jumping to bootloader             |

### Key source files

- `common/firmware/post_code.h` — API declarations
- `common/firmware/post_code.c` — implementation
- `common/src/include/fmc_postcode.h` — error/trace code enumerations

### Reading the register

Example `FW_STATUS_REG = 0xAA0A0000`:
- Prefix: `0xAA` (valid FMC postcode)
- Trace: `0x0A` = `FMC_PLAT_INIT` (platform initialization in progress)
- Status: `0x00` (no status)
- Error: `0x00` (no error)

Example error state `FW_STATUS_REG = 0xAA0A000C`:
- Trace: `0x0A` = last trace was platform init
- Error: `0x0C` = `FMC_ERR_TIMEOUT` (timeout occurred)

## mpifoe-fw postcodes

_TODO: document the mpifoe-fw postcode scheme (format, where the enum/values
are defined, and how to read them). Known data point so far: the mpifoe
postcode monitor shows `0x800f0000` at SimNow steady state (see the
simnow-launch skill)._
