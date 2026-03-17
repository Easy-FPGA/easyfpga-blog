---
title: "FPGA Implementation & Hardware Test"
date: 2026-03-22T12:00:00+09:00
draft: true
description: "End-to-end FPGA implementation: Vivado synthesis flow, ILA signal probing with mark_debug, XDC pin constraints, PuTTY terminal setup, and a systematic hardware troubleshooting guide."
categories:
  - "UART"
tags:
  - "FPGA"
  - "UART"
  - "RTL"
  - "SystemVerilog"
  - "Vivado"
  - "Xilinx"
  - "EasyFPGA"
---
# FPGA Implementation & Hardware Test

Simulation confirms logical correctness. FPGA implementation confirms whether the design meets timing, fits on real hardware, and behaves correctly when signals propagate through physical I/O. This episode covers the full path from RTL files to a running loopback test on an FPGA board: synthesis, ILA probing, XDC pin constraints, and a debugging workflow for when the terminal shows nothing.

## Implementation Flow

```
RTL Design (.sv)
      │
      ▼
 Synthesis (Vivado)
 ─ Logic mapping to LUT/FF
 ─ Resource report (LUT count, FF count)
      │
      ▼
 Implementation
 ─ Placement & Routing
 ─ Timing report (check WNS/WHS)
      │
      ▼
 Bitstream Generation
      │
      ▼
 Program FPGA (JTAG)
      │
      ▼
 Hardware Verification
 ─ ILA capture (internal signals at full clock speed)
 ─ Terminal (PuTTY / Tera Term)
```

## ILA (Integrated Logic Analyzer) Insertion

Add `(* mark_debug = "true" *)` attributes to signals you want to probe in hardware.

```systemverilog
// In uart_loopback_top.sv — mark internal signals for ILA
(* mark_debug = "true" *) logic [7:0] rx_data;
(* mark_debug = "true" *) logic       rx_valid;
(* mark_debug = "true" *) logic       tx_busy;
(* mark_debug = "true" *) logic       parity_err;
(* mark_debug = "true" *) logic       frame_err;
```

- Vivado automatically inserts an ILA core during implementation
- After bitstream programming: open **Vivado Hardware Manager** → set trigger on `rx_valid == 1`
- Inspect captured waveforms for RX data, parity, and frame validity at full clock speed

> **ILA vs printf**: In FPGA design, you cannot add print statements to RTL. The ILA is the equivalent — it captures hardware signal waveforms in on-chip BRAM with a configurable trigger, at full system clock speed, with zero impact on timing. Use ILA for any signal you would want to `$display` in simulation.

## Board-Level Wrapper (Physical Top)

`uart_top` is a UART core interface (parallel control/data + serial pins). For FPGA implementation, use a board-level top module that only exposes physical I/O ports (`clk`, `rst_n`, `rx`, `tx`) and keeps `tx_data/rx_data` internal.

```systemverilog
module uart_loopback_top (
     input  logic clk,
     input  logic rst_n,
     input  logic rx,
     output logic tx
);
     logic       tx_start, tx_busy;
     logic [7:0] tx_data, rx_data;
     logic       rx_valid, parity_err, frame_err;

     uart_top u_uart (
          .clk(clk), .rst_n(rst_n),
          .tx_start(tx_start), .tx_data(tx_data), .tx_busy(tx_busy), .tx(tx),
          .rx(rx), .rx_data(rx_data), .rx_valid(rx_valid),
          .parity_err(parity_err), .frame_err(frame_err)
     );

     // Loopback: retransmit each received byte when TX is idle.
     always_ff @(posedge clk or negedge rst_n) begin
          if (!rst_n) begin
               tx_start <= 1'b0;
               tx_data  <= '0;
          end else begin
               tx_start <= 1'b0;
               if (rx_valid && !tx_busy) begin
                    tx_data  <= rx_data;
                    tx_start <= 1'b1;
               end
          end
     end
endmodule
```

- `tx_data/rx_data` are internal bus signals in `uart_loopback_top`
- `.xdc` constraints apply only to external top-level ports (`clk`, `rst_n`, `rx`, `tx`)

## .xdc Pin Assignment (KCU105)

```tcl
# Clock
set_property PACKAGE_PIN <CLK_PIN> [get_ports clk]
set_property IOSTANDARD LVCMOS33 [get_ports clk]
create_clock -name sys_clk -period 20.000 [get_ports clk]

# UART
set_property PACKAGE_PIN <RX_PIN> [get_ports rx]
set_property PACKAGE_PIN <TX_PIN> [get_ports tx]
set_property IOSTANDARD LVCMOS33 [get_ports {rx tx}]

# Reset
set_property PACKAGE_PIN <RST_PIN> [get_ports rst_n]
set_property IOSTANDARD LVCMOS33 [get_ports rst_n]
```

> **Always verify pin assignments against your board schematic.** FPGA board revisions can change package pin assignments. Using the wrong `PACKAGE_PIN` may route the signal to an unconnected pad (silent failure) or to an incompatible I/O bank (DRC error during synthesis). The board's reference design `.xdc` file is the authoritative source.

## PuTTY / Tera Term Setup

```
Connection settings:
  ┌─────────────────────────────────────────┐
  │  Serial Port:  COMx  (check Device Mgr) │
  │  Baud Rate:    115200                   │
  │  Data Bits:    8                        │
  │  Parity:       Even                     │
  │  Stop Bits:    1                        │
  │  Flow Control: None                     │
  └─────────────────────────────────────────┘
```

- Type characters → the same character should echo back immediately (loopback design)
- If nothing appears: follow the troubleshooting guide below

## Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|---|---|---|
| **No output at all** | Wrong COM port or baud rate | Check Device Manager; match baud |
| **Garbled characters** | Baud rate mismatch | Verify `BAUD_CNT_MAX` calculation |
| **Characters shifted/wrong** | Wrong parity setting | Match 8E1 in both FPGA and terminal |
| **Timeout / lock-up** | Reset polarity or pin mismatch | Confirm `rst_n` is mapped correctly and released High |
| **ILA no trigger** | Wrong trigger signal | Check `mark_debug` attribute placement |
| **First char dropped** | TX starts too early | Check `tx_busy` / flow control logic |

> **Systematic debugging order**: (1) Confirm the COM port and baud rate in Device Manager. (2) Arm the ILA and trigger on `rx_valid`. (3) If `rx_valid` never triggers, the problem is at the RX input (pin, Start Bit detection, or baud rate). If it triggers but the byte value is wrong, the problem is bit ordering or parity. Start physical, work inward.

## Timing Closure Tips

After **Place & Route**, open the **Timing Report** (`Report Timing Summary`):

```
WNS (Worst Negative Slack) >= 0.0 ns  → PASS
WHS (Worst Hold Slack)     >= 0.0 ns  → PASS
```

For a simple UART core at 50 MHz, timing closure is straightforward. Problems arise when:
- The UART module is part of a larger design with a faster system clock
- Long combinational paths exist in the FSM next-state logic
- Reset synchronization is absent (metastability on `rst_n` assertion)

Common fixes:
- Use **pipelining** to break long combinational paths
- Add `set_max_delay` / `set_false_path` constraints for Clock Domain Crossing (CDC) signals
- Reduce clock frequency if WNS is severely negative

## Episode 7 Summary

- **Implementation flow**: Synthesis → P&R → Bitstream → Test
- **ILA**: `mark_debug` attribute → probe internal signals in hardware at full clock speed
- **.xdc**: correct `PACKAGE_PIN` and `IOSTANDARD` are critical
- **PuTTY setup**: 115200, 8E1, No flow control
- **Troubleshooting**: check baud rate, parity, reset, and pin assignments

### Next Episode
> **Episode 8: RTL Design Review & Improvements**
> FIFO buffering, dynamic baud rate, multi-byte framing protocol

## Key Takeaways

- The ILA (`mark_debug` attribute) is the FPGA engineer's equivalent of a print statement — it captures hardware signal waveforms in BRAM post-trigger without affecting timing
- Pin assignments in `.xdc` are unforgiving: the wrong `PACKAGE_PIN` produces silent failure; always cross-reference the board's reference design or schematic
- Start hardware debugging at the physical layer: COM port → baud rate → parity → then examine ILA waveforms; resist the urge to assume the RTL is wrong before proving the hardware connection works
- `WNS >= 0` and `WHS >= 0` in the timing report are pass/fail gates; a design with negative WNS is not timing-closed and will produce intermittent failures

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga
