# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Ibex is a 32-bit RISC-V CPU core written in SystemVerilog, maintained by lowRISC. It is heavily parametrizable and targets embedded control applications. The primary build/simulation tool is **FuseSoC**; configurations are defined in `ibex_configs.yaml` and translated to FuseSoC parameters via `util/ibex_config.py`.

## Common Commands

### Simple System (Verilator simulation)
```bash
make build-simple-system          # Build Verilator simulation
make run-simple-system            # Run with a program
make build-csr-test && make run-csr-test  # Build and run CSR testbench
```

### Compliance & Regression
```bash
make build-riscv-compliance       # Build compliance test infrastructure
```

### UVM Core Verification (requires Xcelium + Spike)
```bash
make --keep-going IBEX_CONFIG=opentitan SIMULATOR=xlm ISS=spike TEST=<test_name>
```

### Lint
```bash
make lint-core-tracing            # RTL lint via FuseSoC/Verilator
make python-lint                  # Lint Python utilities
```

### Formatting
```bash
clang-format -i <file>.cc         # Format C/C++ files
git clang-format                  # Format only changed C/C++ files
# Verible is used for SV formatting (checked in CI via pr_lint.yml)
```

### OSS Formal Verification (Nix environment required)
The formal flow uses Yosys + rIC3 and is resource-intensive (100+ GB RAM recommended). See `dv/formal/` and `ci/ci-formal.yml`.

## RTL Architecture

**Pipeline**: 2 or 3 stages depending on configuration.

| Stage | Module | Description |
|-------|--------|-------------|
| IF | `ibex_if_stage.sv` | PC management, instruction fetch, optional ICache |
| ID | `ibex_id_stage.sv` | Decode, register reads, compressed instruction expansion |
| EX/WB | `ibex_ex_block.sv`, `ibex_wb_stage.sv` | ALU, multiply/divide, load/store, optional writeback stage |

**Key modules:**
- `ibex_top.sv` — Top-level wrapper; selects register file variant, instantiates optional lockstep
- `ibex_core.sv` — Main CPU; instantiates all pipeline stages
- `ibex_controller.sv` — Hazard detection, stall/flush control
- `ibex_cs_registers.sv` — All CSRs, privilege modes
- `ibex_pmp.sv` — Physical Memory Protection
- `ibex_pkg.sv` — All enums, parameters, and types shared across the design
- `ibex_lockstep.sv` — Dual-core lockstep for security-critical configs
- `ibex_dummy_instr.sv` — Constant-time dummy instruction insertion (security)

**Register file variants** (selected via `RegFile` parameter): `ibex_register_file_ff.sv`, `ibex_register_file_fpga.sv`, `ibex_register_file_latch.sv` — all with optional ECC variants.

**Multiply/divide**: Two implementations — `ibex_multdiv_fast.sv` (1-cycle or 3-cycle) and `ibex_multdiv_slow.sv` (iterative).

## Configurations

Defined in `ibex_configs.yaml`. Key verified configs:

| Config | ISA | Pipeline | Notes |
|--------|-----|----------|-------|
| `small` | RV32IMC | 2-stage, 3-cycle mult | Minimal area |
| `opentitan` | RV32IMCB | 3-stage, 1-cycle mult, ICache, PMP, ECC | Production/OpenTitan |
| `maxperf` | RV32IMC | 3-stage, 1-cycle mult, BranchTargetALU | Max performance |

Use `util/ibex_config.py` to translate a named config to FuseSoC `--param` flags.

## Verification Structure

- `dv/uvm/core_ibex/` — UVM testbench with Spike co-simulation (requires commercial tools)
- `dv/riscv_compliance/` — RISC-V compliance test suite (Verilator)
- `dv/cs_registers/` — CSR-focused C++/SV DPI testbench
- `dv/formal/` — Formal trace equivalence against Sail ISS model
- `dv/cosim/` — Spike + Ibex co-simulation infrastructure
- `formal/` — FPV targets: `data_ind_timing/` (constant-time security) and `icache/`

## Toolchain & Dependencies

Version pins are in `ci/vars.env`. Key tools:
- **Verilator** v4.210 — RTL simulation and lint
- **Verible** v0.0-3622-g07b310a3 — SV formatting
- **lowrisc RISC-V GCC toolchain** (rv32imcb) — Software compilation
- **FuseSoC + Edalize** — Build system
- **Spike** (lowRISC fork) — ISS for co-simulation

Nix flake (`flake.nix`) provides a reproducible environment for the formal flow.

Vendor dependencies are in `vendor/` and managed via `util/vendor.py` with `.vendor.hjson` lock files alongside each vendored component.

## Coding Style

- SystemVerilog: follow [lowRISC Verilog Coding Style Guide](https://github.com/lowRISC/style-guides/blob/master/VerilogCodingStyle.md)
- C/C++: run `clang-format` before committing
- Commits: 50-char subject, imperative present tense, 72-char wrapped body