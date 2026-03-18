---
title: "Final Recap — UART on FPGA"
date: 2026-03-30T12:00:00+09:00
draft: true
description: "Complete course recap: custom RTL vs MicroBlaze approaches compared side by side, key UART, FPGA, and AXI concepts consolidated, and a learning roadmap for the next topics."
categories:
  - "UART"
series:
  - "UART on FPGA"
tags:
  - "FPGA"
  - "UART"
  - "RTL"
  - "MicroBlaze"
  - "AXI"
  - "Xilinx"
  - "EasyFPGA"
---
# Final Recap — UART on FPGA

You have now implemented UART on FPGA in two fundamentally different ways — as pure RTL hardware, and as software running on a soft-core processor. This final episode consolidates the most important concepts from all 15 episodes and maps them to the next topics worth exploring.

## Course Structure Overview

| Episode | Topic | Approach |
|---|---|---|
| **01** | What is UART? | Fundamentals |
| **02** | Frame Format | Fundamentals |
| **03** | RS-232 & RS-485 | Fundamentals |
| **04** | Implementation Methods | Overview |
| **05** | Custom RTL Design | RTL |
| **06** | Top Module Integration | RTL |
| **07** | FPGA Implementation & Test | RTL |
| **08** | RTL Review & Improvements | RTL |
| **09** | MicroBlaze Architecture | MicroBlaze |
| **10** | AXI UART Lite IP | MicroBlaze |
| **11** | Block Design Construction | MicroBlaze |
| **12** | Vitis C Code | MicroBlaze |
| **13** | Co-Simulation | MicroBlaze |
| **14** | Hardware Test | MicroBlaze |
| **15** | Final Recap | Summary |

## Approach 1 — Custom RTL (Episodes 5–8)

**Implement UART entirely in Verilog/SystemVerilog**

### What You Built
- `baud_gen`: `CLK_FREQ / BAUD_RATE` integer division → one-cycle `baud_tick` pulse
- `uart_tx`: FSM (IDLE→START→DATA→PARITY→STOP), LSB-first, even parity via `^data`
- `uart_rx`: Start Bit edge detection, half-period wait, center sampling, parity and frame error detection
- `uart_top`: single integration point with `CLK_FREQ`/`BAUD_RATE` parameters
- EP08 improvements: FIFO buffering, dynamic baud rate, multi-byte framing, break detection, interrupt outputs

### Key Skills
- FSM design in SystemVerilog (`typedef enum`, state register, next-state logic)
- Clock domain timing and constraint-driven design (`.xdc`)
- ILA signal probing with `mark_debug` attribute
- Systematic hardware debugging (COM port → baud → ILA waveform)

## Approach 2 — MicroBlaze + AXI UART Lite (Episodes 9–14)

**Connect AXI UART IP to a soft-core processor**

### What You Built
- Vivado Block Design: MicroBlaze + AXI Interconnect + UART Lite + BRAM
- Vitis Platform from `.xsa` + auto-generated BSP (`xparameters.h`)
- Loopback C application using `XUartLite_*` APIs
- Co-simulation: compiled `.elf` + RTL UART IP simulated together
- Hardware test: Launch on Hardware + Vitis Memory view register inspection

### Key Skills
- AXI4-Lite peripheral integration via Block Design automation
- BSP (Board Support Package) architecture and `xparameters.h` portability
- Vitis embedded software development and JTAG debugging
- Hardware/software co-simulation with ELF association

## Two Approaches — When to Use Which

| Criteria | Custom RTL | MicroBlaze + UART IP |
|---|---|---|
| **Protocol complexity** | Simple, fixed | Complex, changeable |
| **Timing requirements** | Strict, cycle-accurate | Relaxed, software-managed |
| **Resource constraints** | Very tight (~50 LUTs) | Moderate or higher (800+ LUTs + BRAM) |
| **Development speed** | Slower | Faster for SW engineers |
| **Target platform** | Any FPGA | Xilinx only |
| **Learning goal** | RTL, FSM, timing | SoC, AXI, embedded SW |
| **Maintenance** | HDL changes needed | Software update only |

> **In practice**: most production FPGA designs use both approaches — a hard-coded RTL UART for low-level, deterministic communication and a processor-SoC for high-level protocol handling. The boundary is determined by latency requirements and update cadence.

## Key Concepts Consolidated

### UART Protocol
- **Frame**: Start(0) + Data[8] LSB-first + Parity + Stop(1); 11 bits total for 8E1
- **Baud rate**: `BAUD_CNT_MAX = CLK_FREQ / BAUD_RATE` — integer division is accurate enough within one frame
- **Center sampling**: offset = `BAUD_CNT_MAX / 2` — maximum margin against clock drift and edge noise
- **Even parity**: `^data[7:0]` (reduction XOR) — one gate in hardware, one character in Verilog

### FPGA/RTL Design
- **Clock management**: derive `baud_tick` from the system clock; never use asynchronous logic for baud timing
- **ILA**: `mark_debug` attribute → real-time internal signal probing without stopping the design
- **Parameterization**: `CLK_FREQ`/`BAUD_RATE` parameters propagated through the hierarchy make the design portable

### MicroBlaze SoC
- **BSP**: `xparameters.h` maps hardware register addresses to C constants — the `.xsa` handoff drives this
- **AXI4-Lite**: memory-mapped register bus; C `write` → AXI AWADDR/WDATA transaction → peripheral register
- **Co-simulation**: `.elf` loaded into BRAM simulation model; software and hardware execute in the same simulator
- **Debug**: Vitis JTAG debugger + Memory view for AXI register inspection at runtime

## Learning Roadmap — What's Next?

```
UART Fundamentals (✓ This course)
      │
      ├──► SPI & I2C on FPGA
      │     (synchronous serial, master/slave, multi-device addressing)
      │
      ├──► AXI Bus Deep Dive
      │     (AXI4-Full burst transactions, AXI Stream, AXI DMA)
      │
      ├──► Zynq SoC
      │     (ARM hard-core PS + programmable logic PL, unified memory map)
      │
      ├──► High-Speed SerDes
      │     (LVDS, PCIe, USB 3.0, JESD204B, 8b10b/64b66b line encoding)
      │
      └──► FPGA DSP
            (FIR filters, FFT, fixed-point arithmetic, IP cores)
```

## Resources

| Resource | Link |
|---|---|
| **RTL source code** | https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git |
| **YouTube channel** | https://www.youtube.com/@easy-fpga |
| **Vivado / Vitis download** | https://www.xilinx.com/support/download.html |
| **MicroBlaze reference** | UG984 (MicroBlaze Processor Reference Guide) |
| **AXI UART Lite product guide** | PG142 (AXI UART Lite Product Guide) |

## Thank You!

**What you built in this series:**

- A complete custom UART TX/RX core in SystemVerilog — baud generator, TX FSM, center-sampling RX FSM, even parity, frame and parity error detection (~150 lines of synthesizable code)
- A full MicroBlaze SoC on FPGA with AXI UART Lite, C loopback application, and co-simulation testbench
- Both designs verified in simulation and tested on real FPGA hardware

**The most important takeaway**: the two approaches are not competing — they complement each other. Understanding how to implement a protocol in RTL gives you the depth to debug it when it fails in a processor-driven system. Understanding how to drive it from C code gives you the flexibility to extend it when specifications change.

> Subscribe at https://www.youtube.com/@easy-fpga for more FPGA tutorials.

## Key Takeaways

- UART is a foundation protocol — the frame format, baud timing, and error detection patterns here reappear in SPI, I²C, and more complex serial standards; the mental model transfers
- Custom RTL (EP05–08): `baud_gen` + `uart_tx` FSM + `uart_rx` center sampling is the complete minimal UART in roughly 150 lines of SystemVerilog — know every line
- MicroBlaze (EP09–14): Block Design + Vitis C code is the complete minimal SoC — the `.xsa` handoff and BSP generation are the workflow patterns you will repeat for every AXI peripheral you add
- Next steps: SPI and I²C extend the same RTL concepts to synchronous protocols; AXI Bus Deep Dive extends the MicroBlaze patterns to high-performance burst transfers

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga
