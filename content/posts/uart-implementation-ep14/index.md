---
title: "MicroBlaze UART — Hardware Test"
date: 2026-03-29T12:00:00+09:00
draft: true
description: "Running the MicroBlaze UART design on real FPGA hardware: exporting bitstream, updating the Vitis platform, launching the application via JTAG, terminal verification, and Vitis debugger walkthrough."
categories:
  - "UART"
tags:
  - "FPGA"
  - "UART"
  - "MicroBlaze"
  - "Vitis"
  - "Xilinx"
  - "EasyFPGA"
---
# MicroBlaze UART — Hardware Test

Simulation verified correctness in software cycles. Hardware test reveals what simulation cannot: clock routing, I/O timing, power-on behavior, and physical signal integrity. This episode runs the complete MicroBlaze UART design on a real FPGA board and uses the Vitis debugger to inspect the running application.

## Hardware Test Overview

```
Vivado:
  Generate Bitstream (with .elf embedded in BRAM init)
         │
         ▼ .xsa (includes bitstream)
Vitis:
  Update Platform (re-import .xsa)
         │
         ▼
  Run → Launch on Hardware (JTAG)
         │
         ▼  FPGA programmed + ELF loaded into BRAM
  Open Serial Terminal (115200, 8E1)
         │
         ▼
  Type characters → verify loopback echo
```

## Step 1: Export Hardware with Bitstream

```
Vivado → File → Export → Export Hardware
  ☑ Include bitstream
  Path: <project>/design_1_wrapper.xsa
Click OK
```

> Including the bitstream in the `.xsa` allows Vitis to program the FPGA during "Launch on Hardware" in a single step. Without the bitstream, you must manually open Vivado Hardware Manager, program the FPGA, then return to Vitis. Always include the bitstream.

## Step 2: Update Vitis Platform

When the hardware design changes, the platform must be refreshed:

```
Vitis → Platform Project (platform_0)
  Right-click → Update Hardware Specification
  Browse to new .xsa file
  Click OK

Then: Right-click Platform → Build
(Rebuilds BSP libraries with new hardware)
```

> **Always rebuild the platform after updating the hardware specification.** Skipping this leaves the BSP reflecting the old address map — the application may fail silently, writing to stale addresses. The rebuild also regenerates `xparameters.h` with any address changes.

## Step 3: Launch on Hardware

```
Vitis → Application Project
  Right-click → Run As → Launch on Hardware (Single Application Debug)

This performs:
  1. Programs FPGA with bitstream from .xsa
  2. Downloads .elf to BRAM via JTAG
  3. Releases MicroBlaze reset → CPU begins execution
```

**Prerequisites:**
- FPGA board connected via USB (JTAG)
- Xilinx cable drivers installed
- Board power on

## Step 4: Serial Terminal Setup

```
Open terminal application (PuTTY / RealTerm / Tera Term):
  Port:         COMx  (check Device Manager → Ports)
  Baud Rate:    115200
  Data Bits:    8
  Parity:       Even
  Stop Bits:    1
  Flow Control: None
```

**Verify loopback:**
- Type any character in the terminal
- The same character should echo back immediately
- If `printf` is enabled: startup log messages appear before the loopback begins

## Step 5: Vitis Debugger

Debug the C application while it runs on real FPGA hardware:

```
Vitis → Run → Debug As → Launch on Hardware (Single Application Debug)

In Debug perspective:
  - Set breakpoints: click left margin of C source line
  - Run: F8 (Resume)
  - Step: F6 (Step Over), F5 (Step Into)
  - Inspect variables: Variables view
  - Memory: Window → Show View → Memory
```

### Inspecting AXI UART Registers in Memory View

```
Memory → Add Memory Monitor → Address: 0x40600000
Displays: RX_FIFO (0x00), TX_FIFO (0x04), STAT_REG (0x08), CTRL_REG (0x0C)
```

You can watch `STAT_REG` live: bit[0] (`RX_VALID`) transitions High when a byte is in the FIFO. This confirms the AXI address mapping is correct and the hardware is responding — the fastest way to rule out an address assignment mismatch.

## RTL Design vs MicroBlaze Comparison

| Item | Custom RTL (EP05–07) | MicroBlaze + UART IP (EP09–14) |
|---|---|---|
| **Logic implementation** | All in Verilog HDL | C software + IP core |
| **Baud rate change** | Recompile HDL, re-synthesize | Change one constant, recompile C |
| **Protocol extension** | Modify RTL state machine | Edit C code |
| **Resource usage** | Very low (~50 LUTs) | High (MicroBlaze + BRAM) |
| **Latency** | Cycle-accurate, deterministic | Software-dependent |
| **Debug** | ILA waveform | Vitis debugger + register view |
| **Best for** | High performance, portability | Complex control, rapid SW iteration |

## Troubleshooting Hardware Test

| Symptom | Likely Cause | Fix |
|---|---|---|
| FPGA not programmed | JTAG cable/driver issue | Check USB cable; reinstall drivers |
| Terminal no output | Wrong COM port or baud | Verify COMx in Device Manager |
| Garbled characters | Parity mismatch | Confirm 8E1 on both sides |
| Program crashes | BRAM overflow | Increase BRAM depth in BD |
| Breakpoint not hit | .elf not rebuilt | Clean + rebuild application |
| AXI read returns 0 | Wrong base address | Verify `XPAR_UARTLITE_0_BASEADDR` |

## Episode 14 Summary

- **Export .xsa with bitstream**: enables Vitis to program FPGA directly
- **Update Platform**: always update after hardware changes
- **Launch on Hardware**: programs FPGA + loads .elf in one step
- **Serial terminal**: 115200 8E1, No flow control
- **Vitis Debugger**: breakpoints, variable inspection, Memory view for AXI regs
- **RTL vs MB**: RTL = performance; MicroBlaze = software flexibility

### Next Episode
> **Episode 15: Final Recap**
> Two-approach summary, key concepts, what to learn next

## Key Takeaways

- "Update Hardware Specification" + "Build Platform" in Vitis must follow every Vivado hardware export — skipping leaves the BSP inconsistent with the actual hardware mapping
- The Vitis Memory view at `0x40600000` displays AXI UART Lite registers live; watching `STAT_REG[0]` transition is the fastest way to confirm the AXI address mapping is correct in hardware
- The RTL approach (~50 LUTs) is cycle-accurate and vendor-agnostic; the MicroBlaze approach (800+ LUTs + BRAM) pays in resources for protocol changes via C code edits rather than HDL re-synthesis
- Launching in Vitis with JTAG debug programs the FPGA, loads the ELF, and releases reset in one action — this is the standard hardware iteration loop for embedded MicroBlaze development

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga
