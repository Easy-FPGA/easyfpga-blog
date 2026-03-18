---
title: "RTL Design Review & Improvements"
date: 2026-03-23T12:00:00+09:00
draft: true
description: "Evolving the basic UART RTL: FIFO buffering to prevent byte loss, runtime baud rate selection, multi-byte framing with SOF/EOF markers, break detection, and an interrupt interface."
categories:
  - "UART"
series:
  - "UART on FPGA"
tags:
  - "FPGA"
  - "UART"
  - "RTL"
  - "SystemVerilog"
  - "Xilinx"
  - "EasyFPGA"
---
# RTL Design Review & Improvements

The basic UART core from Episodes 5–6 is correct but minimal. Real applications require more: buffering for when the host is busy, baud rate changes at runtime, and multi-byte packet framing. This episode systematically identifies each limitation of the minimal design and shows how to address it — without over-engineering for unused features.

## Current Design — Limitations

| Limitation | Description |
|---|---|
| **No FIFO buffer** | Byte is lost if processor doesn't read in time |
| **Fixed baud rate** | Baud rate fixed at compile time — not reconfigurable |
| **Single-byte only** | No multi-byte framing or packet structure |
| **No flow control** | Receiver can't signal sender to pause |
| **No break detection** | Line stuck-low condition not detected |

> **Which limitation matters for your application?** For a simple UART debug console, fixed baud rate and no FIFO are acceptable. For a production sensor interface at high data rates, FIFO is mandatory. For industrial protocols (Modbus, DMX512), multi-byte framing and break detection are required. Add improvements only for the use case at hand — each adds complexity.

## Improvement 1: FIFO Buffer

Replace the simple register with a **circular FIFO** buffer.

```systemverilog
module uart_fifo #(
    parameter DEPTH = 16,
    parameter WIDTH = 8
)(
    input  logic             clk, rst_n,
    input  logic             wr_en,
    input  logic [WIDTH-1:0] wr_data,
    input  logic             rd_en,
    output logic [WIDTH-1:0] rd_data,
    output logic             full, empty
);
    logic [WIDTH-1:0] mem [DEPTH-1:0];
    logic [$clog2(DEPTH)-1:0] wr_ptr, rd_ptr;
    logic [$clog2(DEPTH):0]   count;

    assign full  = (count == DEPTH);
    assign empty = (count == 0);
    // ... write/read pointer logic ...
endmodule
```

> **FIFO depth calculation**: At 115,200 bps, a new byte arrives every 95.5 µs. If your host processor may be preempted for up to 1 ms (e.g., interrupt latency), you need at least `ceil(1000 / 95.5) ≈ 11` bytes of FIFO depth to avoid overflow. A depth of 16 is the standard minimum; 64 or 256 is safer for OS-driven hosts.

## Improvement 2: Dynamic Baud Rate

Use a lookup table to set `BAUD_DIV` at runtime.

```systemverilog
logic [1:0] baud_sel;         // 00=9600, 01=115200, 10=460800
logic [15:0] baud_div;

always_comb
    case (baud_sel)
        2'b00: baud_div = 16'd5208;   //  50 MHz / 9600
        2'b01: baud_div = 16'd434;    //  50 MHz / 115200
        2'b10: baud_div = 16'd108;    //  50 MHz / 460800
        default: baud_div = 16'd434;
    endcase

always_ff @(posedge clk or negedge rst_n)
    if (!rst_n || cnt == baud_div - 1) cnt <= '0;
    else cnt <= cnt + 1;

assign baud_tick = (cnt == baud_div - 1);
```

> **Caution with dynamic baud rate**: Never change `baud_sel` while a frame is in progress. The `baud_div` change takes effect on the next counter reload — mid-frame causes a timing corruption of the current byte. Add a guard: only allow `baud_sel` changes when both TX and RX FSMs are in the IDLE state.

## Improvement 3: Multi-Byte Frame Protocol

Define a simple packet structure with SOF/EOF markers.

```
Packet format:
┌──────┬──────────┬────────────────────┬──────────┬──────┐
│ SOF  │  Length  │     Data bytes     │ Checksum │  EOF │
│ 0x7B │  1 byte  │  N bytes (N=Len)   │  1 byte  │ 0x7D │
└──────┴──────────┴────────────────────┴──────────┴──────┘
  '{'                                                '}'
```

### Frame FSM States
```
WAIT_SOF → RECV_LEN → RECV_DATA → RECV_CKS → WAIT_EOF
    │                                              │
    └──────── error: back to WAIT_SOF ◄───────────┘
```

The checksum is a byte sum of all data bytes. On checksum mismatch or missing EOF, the FSM resets to `WAIT_SOF` and flags a protocol error. This structure can be extended to CRC-16 for stronger error detection.

## Improvement 4: Break Detection

A **break condition** = line stays LOW for more than one full frame duration.

```systemverilog
logic [15:0] break_cnt;
logic        break_detected;

always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) break_cnt <= '0;
    else if (uart_rx) break_cnt <= '0;   // Line high: reset counter
    else break_cnt <= break_cnt + 1;      // Line low: count up
end

// Break if line stays low > 2 full frames (22 bits at current baud)
assign break_detected = (break_cnt > 2 * BAUD_CNT_MAX * 11);
```

Break detection is required by LIN bus (automotive) and DMX512 (stage lighting), where a break condition is a deliberate protocol-level signaling event that marks the start of a packet.

## Improvement 5: Interrupt Interface

```systemverilog
output logic irq_rx_valid,   // New byte received
output logic irq_tx_ready,   // TX FIFO has space
output logic irq_parity_err, // Parity error detected
output logic irq_frame_err,  // Frame error detected
output logic irq_break        // Break condition detected
```

Connect these to a CPU interrupt controller for software-driven handling. In a pure RTL design (no CPU), these can trigger a downstream processing FSM directly.

## Updated Architecture

```
                ┌──────────────────────────────────────┐
                │            uart_top (improved)        │
                │                                       │
uart_rx ───────►│ RX FSM ──► RX FIFO ──► data_out      │
                │                ↕                      │
                │            Flow Ctrl                  │
                │                ↕                      │
uart_tx ◄───────│ TX FSM ◄── TX FIFO ◄── data_in       │
                │                                       │
                │  Interrupt Controller ──► irq[4:0]    │
                └──────────────────────────────────────┘
```

## Episode 8 Summary

- **FIFO buffer**: prevents byte loss on processor latency
- **Dynamic baud rate**: LUT-based `baud_div` selection at runtime
- **Multi-byte framing**: SOF/EOF markers, length field, checksum
- **Break detection**: line stuck-low longer than 2 full frames
- **Interrupt interface**: event signals for processor integration

### Next Episode
> **Episode 9: MicroBlaze — Soft-Core Processor on FPGA**
> MicroBlaze architecture, AXI4-Lite interface, Vivado → Vitis flow

## Key Takeaways

- Add a FIFO when the receiving application cannot service bytes within one baud period; calculate the required depth from application interrupt latency divided by the byte period at the configured baud rate
- Dynamic baud rate requires a guard condition (check IDLE state) before `baud_sel` changes — mid-frame baud changes corrupt the current byte
- The multi-byte frame protocol (SOF + Length + Data + Checksum + EOF) is a practical template for binary application-layer protocols over UART
- Break detection is a named protocol feature in LIN and DMX512 — implement it only when the target protocol specification explicitly requires it

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga
