---
title: "Block Design Construction"
date: 2026-03-26T12:00:00+09:00
draft: true
description: "Step-by-step Vivado Block Design: adding MicroBlaze and UART Lite, running Block Automation and Connection Automation, assigning AXI addresses, and exporting hardware to Vitis."
categories:
  - "UART"
tags:
  - "FPGA"
  - "UART"
  - "MicroBlaze"
  - "AXI"
  - "Vivado"
  - "Xilinx"
  - "EasyFPGA"
---
# Block Design Construction

Vivado's Block Design canvas replaces manual AXI interconnect wiring with a graphical drag-and-drop interface backed by automated connection logic. Understanding what Block Automation and Connection Automation actually add — and when to override them — is the key skill for building MicroBlaze-based SoC designs efficiently.

## Block Design Overview

We will build the following system:

```
                    ┌─────────────────────────────────────────┐
                    │           Block Design                   │
                    │                                         │
clk_100mhz ────────►│ Clock Wizard ─► MicroBlaze ─► AXI Interconnect
                    │                      │                  │
rstn ──────────────►│ Proc Sys Reset        │                  │
                    │                      ▼                  │
                    │                AXI BRAM Ctrl ──► BRAM   │
                    │                                         │
                    │                AXI UART Lite            │
                    └──────────────────────│───────────────────┘
                                           ▼
                                      uart_rx / uart_tx
```

## Step 1: Add MicroBlaze

```
1. Create Block Design (BD)
2. Add IP → search "MicroBlaze" → double-click
3. Click "Run Block Automation"
   Options:
     - Preset: Microcontroller
     - BRAM: 32KB (local memory)
     - Debug: Enable (JTAG via MDM)
     - Interrupt Controller: No (for now)
   Click OK
```

Block Automation adds:
- `mdm_1` (MicroBlaze Debug Module) — required for JTAG breakpoints in Vitis
- `Processor System Reset` — ensures the CPU releases reset after the clock stabilizes
- `AXI BRAM Controller` + `blk_mem_gen` — instruction and data memory
- `Clocking Wizard` — generates a clean system clock from the board's oscillator input

> **Why 32KB BRAM?** A minimal C application with BSP overhead typically fits within 20–30 KB. If the Vitis linker reports an overflow (`region 'lmb_bram_memory' overflowed`), double the BRAM size here and re-synthesize. The BRAM consumes FPGA RAMB36 blocks, not LUTs.

## Step 2: Add AXI UART Lite

```
1. Add IP → "AXI UART Lite"
2. Double-click to configure:
   - Baud Rate: 115200
   - Data Bits: 8
   - Use Parity: Yes → Even
3. Click "Run Connection Automation"
   - Check: S_AXI → /microblaze_0/M_AXI_DP
   - Connect uart_rtl to external port
   Click OK
```

Connection Automation adds:
- AXI4-Lite connection from MicroBlaze to UART Lite via the AXI Interconnect
- External ports `uart_rtl_0_rxd` and `uart_rtl_0_txd` on the Block Design boundary

## Step 3: Assign Addresses

Open **Address Editor** tab:

```
Name                  | Base Address | Range  | High Address
----------------------|--------------|--------|-------------
microblaze_0/BRAM     | 0x0000_0000  | 32K    | 0x0000_7FFF
microblaze_0/UART     | 0x4060_0000  | 4K     | 0x4060_0FFF
```

- Click **Auto Assign Address** to let Vivado assign automatically
- Or manually type `0x4060_0000` for UART Lite

> **Why address assignment matters**: The address assigned here is directly encoded into the auto-generated `xparameters.h` as `XPAR_UARTLITE_0_BASEADDR`. If you change it in the Address Editor but forget to re-export the `.xsa` and regenerate the BSP in Vitis, your C code will write to the wrong address and the UART will appear non-functional.

## Step 4: Create HDL Wrapper

```
Sources panel → Right-click Block Design (.bd file)
→ "Create HDL Wrapper"
→ Select: Let Vivado manage wrapper and auto-update
→ Click OK
```

This generates a top-level SystemVerilog/Verilog file that instantiates the entire block design. The wrapper is the synthesis entry point — it connects block design ports to FPGA I/O pins via the `.xdc` constraint file.

## Step 5: .xdc Constraints

```tcl
# Board clock (verify PACKAGE_PIN against your board schematic)
set_property PACKAGE_PIN AE5  [get_ports sys_clk]
set_property IOSTANDARD  LVDS [get_ports sys_clk]
create_clock -name sys_clk -period 5.000 [get_ports sys_clk]

# UART (connected to on-board USB-UART bridge)
set_property PACKAGE_PIN AH11 [get_ports uart_rtl_0_rxd]
set_property PACKAGE_PIN AH12 [get_ports uart_rtl_0_txd]
set_property IOSTANDARD LVCMOS33 [get_ports {uart_rtl_0_rxd uart_rtl_0_txd}]

# Reset button
set_property PACKAGE_PIN AN8 [get_ports reset_n]
set_property IOSTANDARD LVCMOS33 [get_ports reset_n]
```

## Step 6: Synthesis → Implementation → Bitstream

```
Flow Navigator:
  1. Generate Bitstream (runs Synthesis + Implementation automatically)
  2. Check timing:
     - WNS >= 0: timing closed
  3. Export Hardware (File → Export → Export Hardware)
     - Check "Include bitstream"
     - Path: project.xsa
```

> **Always include the bitstream in the `.xsa`**. Without it, Vitis cannot program the FPGA during "Launch on Hardware" — you would need to manually open Vivado Hardware Manager, program the FPGA, then return to Vitis. Exporting with the bitstream allows Vitis to do all of this in one step from the Run menu.

## Episode 11 Summary

- **Block Automation**: one-click adds BRAM, debug module, clock wizard, reset
- **Connection Automation**: one-click AXI wiring from MicroBlaze to peripherals
- **Address Editor**: assign and verify AXI memory-mapped addresses
- **HDL Wrapper**: wrap the BD into a synthesizable top-level module
- **Export .xsa with bitstream**: Vitis needs the bitstream to program the FPGA

### Next Episode
> **Episode 12: Vitis C Code**
> Platform creation, BSP, loopback C code, printf via UART

## Key Takeaways

- Block Automation and Connection Automation handle the AXI wiring — understand what they added (BRAM, reset synchronizer, debug module) so you can diagnose wiring issues when they arise
- The Address Editor assignment becomes `XPAR_UARTLITE_0_BASEADDR` in the BSP — always re-export `.xsa` after address changes and rebuild the platform in Vitis
- "Create HDL Wrapper (Let Vivado manage)" is the standard choice — it regenerates the wrapper automatically whenever the block design changes
- Export hardware with "Include bitstream" to enable Vitis to program the FPGA and load the ELF in a single "Launch on Hardware" action

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga
