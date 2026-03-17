---
title: "Top Module — Integrating TX & RX"
date: 2026-03-21T12:00:00+09:00
draft: true
description: "Building uart_top: port definitions, submodule instantiation, parameterization of CLK_FREQ and BAUD_RATE, and the design principles behind clean RTL interfaces."
categories:
  - "UART"
tags:
  - "FPGA"
  - "UART"
  - "RTL"
  - "SystemVerilog"
  - "Xilinx"
  - "EasyFPGA"
---
# Top Module — Integrating TX & RX

With TX and RX modules independently verified, the next step is integration: wrapping them in a `uart_top` module that presents a single, clean interface and ensures both sides derive their timing from the same parameters. This episode covers port design, parameterization strategy, and the integration patterns that make RTL blocks reusable across projects and FPGA vendors.

## System Architecture

`uart_top` integrates three submodules:

```
     CLK_FREQ, BAUD_RATE (parameters)
              │
    ┌─────────▼──────────────────────────┐
    │            uart_top                │
    │                                    │
    │  ┌──────────┐   baud_tick          │
    │  │ baud_gen │──────────────┐       │
    │  └──────────┘              │       │
    │                      ┌─────▼────┐  │
tx_start ──────────────────►          │  │
tx_data[7:0] ──────────────► uart_tx ├──► tx
tx_busy ◄───────────────────┤          │  │
    │                      └──────────┘  │
    │                      ┌──────────┐  │
rx ─────────────────────────► uart_rx ├──► rx_data[7:0]
    │                      │          ├──► rx_valid
    │                      │          ├──► parity_err
    │                      └──────────┘  │
    └────────────────────────────────────┘
```

- `uart_top` wraps `baud_gen`, `uart_tx`, and `uart_rx` as a single reusable UART core
- `baud_gen` generates `baud_tick` used for TX bit timing
- Ports are split cleanly into TX control/status and RX data/error outputs

## Top Module Port Definitions

```systemverilog
module uart_top (
    input  logic       clk,
    input  logic       rst_n,

    // TX interface
    input  logic       tx_start,
    input  logic [7:0] tx_data,
    output logic       tx_busy,
    output logic       tx,

    // RX interface
    input  logic       rx,
    output logic [7:0] rx_data,
    output logic       rx_valid,
    output logic       parity_err,
    output logic       frame_err
);
```

> **Port naming convention**: Separating TX and RX ports into labeled groups (`// TX interface`, `// RX interface`) makes instantiation self-documenting. When an upstream module wires up `uart_top`, the port names immediately communicate direction and function — reducing wiring errors in larger designs.

## Integration in uart_top

```systemverilog
logic baud_tick;

baud_gen #(
    .CLK_FREQ  (CLK_FREQ),
    .BAUD_RATE (BAUD_RATE)
) u_baud_gen (
    .clk       (clk),
    .rst_n     (rst_n),
    .baud_tick (baud_tick)
);

uart_tx u_tx (
    .clk       (clk),
    .rst_n     (rst_n),
    .baud_tick (baud_tick),
    .tx_start  (tx_start),
    .tx_data   (tx_data),
    .tx_busy   (tx_busy),
    .tx        (tx)
);

uart_rx #(
    .CLK_FREQ  (CLK_FREQ),
    .BAUD_RATE (BAUD_RATE)
) u_rx (
    .clk       (clk),
    .rst_n     (rst_n),
    .rx        (rx),
    .rx_data   (rx_data),
    .rx_valid  (rx_valid),
    .parity_err(parity_err),
    .frame_err (frame_err)
);
```

> **Why does RX receive `CLK_FREQ` and `BAUD_RATE` directly while TX only uses `baud_tick`?** TX advances its FSM on a shared `baud_tick` from `baud_gen` — TX and `baud_gen` are already synchronized. RX, however, triggers from a falling edge (Start Bit) at an arbitrary time, then must independently reconstruct the half-period delay and subsequent bit centers. It needs the raw parameters to compute `BAUD_CNT_MAX >> 1` and `BAUD_CNT_MAX - 1` internally.

## Parameterization

| Parameter | Meaning | Default |
|---|---|---|
| `CLK_FREQ` | Input clock frequency (Hz) | `50_000_000` |
| `BAUD_RATE` | UART baud rate (bps) | `115_200` |

These parameters propagate to both `baud_gen` and `uart_rx`, ensuring TX and RX timing are derived from the same constants in one place. Changing the baud rate requires editing exactly one location — the top-level parameter override — and the entire design adapts.

> **Portability in practice**: This module drops into any project by overriding two parameters at instantiation: `.CLK_FREQ(100_000_000), .BAUD_RATE(115_200)`. No manual recalculation of `BAUD_CNT_MAX` or counter widths is required — the localparam and `$clog2` expressions inside each submodule handle it automatically.

## Episode 6 Summary

- **`uart_top`** integrates **`baud_gen`**, **`uart_tx`**, and **`uart_rx`**
- **`baud_tick`** is generated by **`baud_gen`** and used for TX bit timing
- TX path: **`tx_start/tx_data`** in, **`tx/tx_busy`** out
- RX path: **`rx`** in, **`rx_data/rx_valid/parity_err/frame_err`** out

### Next Episode
> **Episode 7: FPGA Implementation & Hardware Test**
> Synthesis, ILA insertion, PuTTY setup, and troubleshooting

## Key Takeaways

- `uart_top` acts as the integration boundary — passing `CLK_FREQ` and `BAUD_RATE` as parameters ensures TX and RX always agree on timing without manual recalculation
- TX receives a shared `baud_tick` from `baud_gen`; RX counts from its own edge-triggered start and needs the raw parameters to reconstruct the half-period offset
- Using named port connections (`.port_name(signal)`) instead of positional connections is mandatory for maintainable RTL — especially once module ports change during iteration
- The `tx_busy` signal is the handshake mechanism: the caller must not assert `tx_start` while `tx_busy` is High, or the current byte transmission will be corrupted

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga
