---
name: repo-simnow
description: >
  Repository reference for the simnow repo: external dependencies and how they
  are managed. Use when writing, reviewing, or debugging code in simnow,
  especially when touching externals/ or the ifoe_ss_model dependency. (To
  build SimNow use build-simnow; to install a release use admin-workflow-simnow-install-release.)
---

# simnow — Repository Reference

Reference knowledge applied when working on the simnow repository.

## External dependency: ifoe_ss_model

**Location**: `externals/ifoe_ss_model`

**Purpose**: Provides the IFOE subsystem model for simulation.

**How it works**:
- Functions like a git submodule but is managed through CMake.
- The required revision is specified in `externals/CMakeLists.txt`.
- When building with ifoe_ss_model support enabled, `externals/CMakeLists.txt` will:
  1. Clone the repository if not already present
  2. Check out the correct revision as specified in the CMakeLists.txt
  3. Make it available for the build

**Key points**:
- Not a true git submodule — managed by the build system.
- Revision updates are done by modifying `externals/CMakeLists.txt`.
- The repository is only cloned/updated when building with ifoe_ss_model support.
