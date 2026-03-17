---
title: "MicroBlaze: Soft-Core Processor on FPGA"
date: 2026-03-24T12:00:00+09:00
draft: true
description: "MicroBlaze architecture for FPGA engineers: soft-core vs hard-core comparison, AXI4-Lite peripheral bus, C-to-hardware execution model, and the Vivado-to-Vitis design flow."
categories:
  - "UART"
tags:
  - "FPGA"
  - "UART"
  - "MicroBlaze"
  - "AXI"
  - "Xilinx"
  - "EasyFPGA"
---
# MicroBlaze: Soft-Core Processor on FPGA

Episodes 5–8 showed UART implemented entirely in hardware. In Episodes 9–14, we take the alternative approach: a soft-core processor (MicroBlaze) drives the UART by writing C code to a register-mapped IP core. This is not just an academic exercise — soft-core processor designs appear frequently in production FPGA systems where control logic is complex enough to benefit from software expressibility and iterability.

## What is MicroBlaze?

**MicroBlaze** is a configurable 32-bit RISC **soft-core processor** from Xilinx/AMD.

| Feature | Description |
|---|---|
| **Type** | Soft-core — implemented in FPGA fabric (LUT/FF) |
| **Architecture** | 32-bit RISC, Harvard architecture |
| **Bus Interface** | AXI4 (data/instruction), AXI4-Lite (peripherals) |
| **Memory** | BRAM (Block RAM) for instruction and data |
| **Development** | C/C++ code in AMD Vitis IDE |
| **OS Support** | Bare-metal or FreeRTOS |

> MicroBlaze delivers embedded CPU capability without a dedicated processor chip. The entire CPU — ALU, register file, instruction decoder, pipeline — is synthesized from FPGA LUTs and flip-flops, just like any other digital logic.

## Soft-Core vs Hard-Core Processor

| Item | Soft-Core (MicroBlaze) | Hard-Core (Zynq PS ARM) |
|---|---|---|
| **Implementation** | FPGA LUT/FF fabric | Dedicated silicon |
| **Flexibility** | Fully configurable | Fixed architecture |
| **Performance** | Lower (fabric limited) | Higher (dedicated HW) |
| **Resource Cost** | ~800 LUTs (minimal config) | Fixed silicon area |
| **Availability** | Any Xilinx FPGA | Zynq / Zynq UltraScale only |
| **Use Case** | Moderate control, education | High-performance embedded |

> **When to choose MicroBlaze**: If your design needs a processor but you are using a non-Zynq FPGA (Artix-7, Kintex-7, UltraScale), MicroBlaze is the only option from Xilinx. For Zynq devices, the ARM hard-core (PS) is usually preferred for performance — but MicroBlaze remains useful for separate, deterministic real-time tasks that must not share an OS with general-purpose code.

## MicroBlaze Design Flow

```
Vivado (Hardware Design)
  1. Create Block Design
  2. Add MicroBlaze IP
  3. Run Block Automation (adds BRAM, debug module, reset)
  4. Add AXI peripherals (UART Lite, GPIO, etc.)
  5. Run Connection Automation
  6. Synthesize → Implement → Bitstream
  7. Export Hardware (.xsa) — include bitstream

Vitis (Software Development)
  8. Create Platform from .xsa
  9. Create Application Project (C/C++)
  10. Write application code (XUartLite_* APIs)
  11. Build → Launch on Hardware
  12. Debug via JTAG
```

The division between Vivado (hardware) and Vitis (software) mirrors the boundary between hardware description and C programming. The `.xsa` file is the handoff point — it contains the hardware memory map that Vitis uses to generate a Board Support Package (BSP) with correct base addresses.

## AXI4-Lite Interface

MicroBlaze communicates with peripherals via **AXI4-Lite** — a simplified memory-mapped register interface.

```
MicroBlaze CPU
    │
    ├──AXI4-Lite Master Bus──►  AXI UART Lite  (0x4060_0000)
    │                        ►  AXI GPIO       (0x4000_0000)
    │                        ►  AXI Timer      (0x4182_0000)
    │
    └──AXI4 (instruction)───►  AXI BRAM Ctrl  (0x0000_0000)
```

### Address Map (example)
| Peripheral | Base Address |
|---|---|
| AXI BRAM (instruction/data) | 0x0000_0000 |
| AXI UART Lite | 0x4060_0000 |
| AXI GPIO | 0x4000_0000 |

## C-to-Hardware Execution Model

```
C code (Vitis)
  printf("Hello UART\n");
       │
       ▼
Vitis BSP (Board Support Package)
  XUartLite_SendByte(UART_BASEADDR, ch);
       │
       ▼
MicroBlaze CPU
  SW instruction → AXI4-Lite write to UART TX_FIFO register
       │
       ▼
AXI UART Lite IP → drives uart_tx pin → Physical UART signal
```

The BSP (`xparameters.h`) defines peripheral base addresses so C code is portable across hardware revisions. This is the same abstraction layer as a UART driver in Linux or an RTOS — just at bare-metal level with no OS abstraction between the C code and the register writes.

## Why Use MicroBlaze + UART?

| Scenario | Benefit |
|---|---|
| Complex protocol logic | Move state machine to software — easier to iterate |
| Configuration at runtime | Change baud rate, parity via register writes |
| Multi-peripheral system | AXI interconnect ties everything together |
| Printf debugging | Route stdout to UART — the easiest debug channel |
| RTOS tasks | FreeRTOS handles concurrency above the hardware |

## Episode 9 Summary

- **MicroBlaze** = 32-bit RISC soft-core synthesized in FPGA fabric
- **AXI4-Lite** = lightweight memory-mapped bus for peripheral access
- **Design flow**: Vivado (HW) → export .xsa → Vitis (SW)
- **BSP** abstracts register addresses — C code uses library functions
- Best for: moderate control tasks, education, multi-peripheral systems

### Next Episode
> **Episode 10: AXI UART Lite IP**
> Register map, UART Lite vs 16550, IP configuration, and C driver patterns

## Key Takeaways

- MicroBlaze is a soft-core processor synthesized entirely in FPGA fabric; its performance is limited by LUT propagation delay, but it is available on any Xilinx FPGA regardless of device family
- AXI4-Lite connects the CPU to peripherals through a memory-mapped register bus; a C `write` instruction translates to an AXI address + data transaction on the bus
- The `.xsa` file is the handoff between Vivado (hardware) and Vitis (software); it contains the memory map that drives BSP generation and populates `xparameters.h`
- Choosing MicroBlaze trades FPGA resources (≥800 LUTs + BRAM) for the ability to update logic by recompiling C code rather than re-synthesizing HDL

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga
