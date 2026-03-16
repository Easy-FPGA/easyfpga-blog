---
title: "UART Implementation Methods"
date: 2026-03-16T12:00:00+09:00
draft: false
description: "Three practical ways to build UART on FPGA: vendor IP, soft processor plus UART IP, and custom RTL, with trade-offs in flexibility, performance, and development effort."
categories:
  - "UART"
tags:
  - "FPGA"
  - "UART"
  - "RTL"
  - "MicroBlaze"
  - "Xilinx"
---
# UART Implementation Methods

## Three Approaches to UART on FPGA

```
┌─────────────────────────────────────────────────────────┐
│                 UART on FPGA — 3 Methods                │
├─────────────────┬──────────────────┬────────────────────┤
│  Method 1       │  Method 2        │  Method 3          │
│  Vendor IP Core │  Soft Processor  │  Custom RTL        │
│                 │  + UART IP       │  Design            │
├─────────────────┼──────────────────┼────────────────────┤
│  Easy & fast    │  Software-driven │  Full control      │
│  AXI-compatible │  Highly scalable │  Best performance  │
│  Limited flex   │  High resources  │  Longer dev time   │
└─────────────────┴──────────────────┴────────────────────┘
```

## Method 1: Vendor IP Core

**Use a verified UART IP from Xilinx/Intel — add via IP Catalog with a few clicks**

### Pros
- **Quick integration**: drag-and-drop from IP Catalog
- **AXI-compatible**: connects directly to MicroBlaze, Zynq ARM via AXI4-Lite
- **Pre-verified**: stable, vendor-tested IP
- **Interrupt support**: built-in hardware interrupt
- **Low resource usage**: optimized implementation

### Cons
- **Limited flexibility**: hard to modify baud rate or interface behavior
- **No software stack**: requires a processor for control logic
- **Vendor lock-in**: Xilinx/Intel specific

## Method 2: Soft Processor + UART Peripheral

**Connect UART IP to a soft-core CPU (MicroBlaze / Nios II) via AXI bus**

### Pros
- **Software-driven control**: implement complex logic in C code
- **OS support**: FreeRTOS or bare-metal
- **Scalable**: connect multiple peripherals via AXI interconnect

### Cons
- **High resource usage**: CPU + BRAM + peripherals
- **Latency**: software scheduling adds delay
- **Toolchain complexity**: requires both Vivado and Vitis

```
MicroBlaze ──AXI4-Lite──► AXI UART Lite ──► TX/RX
```

## Method 3: Custom RTL Design

**Implement UART TX/RX directly in Verilog/SystemVerilog**

### Pros
- **Full control**: customize timing, bit width, parity, and more
- **Lightweight**: only the logic you need → minimal resource usage
- **Best performance**: cycle-accurate operation
- **Portable**: works on any FPGA platform (Xilinx, Intel, Lattice...)
- **Educational**: builds deep understanding of FSM and timing design

### Cons
- **Development time**: requires design, simulation, and verification
- **Debug complexity**: needs ILA or oscilloscope to troubleshoot

## Full Comparison

| Feature | Vendor IP | Soft Processor + IP | Custom RTL |
|---|---|---|---|
| **Ease of Use** | Very high (drag-and-drop) | Medium (HW+SW integration) | Low (requires expertise) |
| **Customization** | Limited | High (via software) | Complete control |
| **Performance** | High, deterministic | Depends on SW efficiency | Highest, cycle-accurate |
| **Resource Usage** | Low | High (CPU + memory) | Low to moderate |
| **AXI Integration** | Native | Native via processor | Manual |
| **Learning Value** | Low | Medium | **High** |

> **Quick prototype** → Vendor IP
> **Software control needed** → Soft Processor
> **Learning / optimization / portability** → Custom RTL

## Lecture Structure

```
Episodes 5–7:   Custom RTL Design
                (Baud Generator, TX FSM, RX FSM, Testbench, FPGA implementation)

Episodes 9–13:  MicroBlaze + AXI UART Lite IP
                (Block Design, Vitis C Code, Co-Simulation)

Episode 15:     Final comparison and recap
```

## Episode 4 Summary

- **Method 1 (Vendor IP)**: fast, pre-verified, native AXI support
- **Method 2 (Soft Processor)**: C code control, highly extensible
- **Method 3 (Custom RTL)**: full control, best performance, highest learning value
- This lecture covers **both RTL design and MicroBlaze** approaches

### Next Episode
> **Episode 5: Custom RTL Design**
> Requirements → TX FSM → RX FSM → Verilog implementation

## Key Takeaways

- Build UART logic with timing assumptions made explicit
- Verify behavior with both simulation and hardware-oriented checks
- Keep interfaces minimal and move complexity into reusable RTL blocks

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga






