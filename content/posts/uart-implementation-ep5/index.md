---
title: "Custom RTL Design: TX & RX"
date: 2026-03-18T12:00:00+09:00
draft: false
description: "Implementing UART TX and RX entirely in SystemVerilog: baud rate generator design, TX FSM, center-sampling RX FSM, and loopback testbench strategy."
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
# Custom RTL Design: TX & RX

This is the first hands-on RTL episode. We translate the UART protocol specification directly into synthesizable SystemVerilog — starting from the mathematical relationship between clock frequency and baud rate, then building the TX FSM and RX center-sampling logic. Each design decision is derived from first principles so you can reproduce and modify it for any FPGA or baud rate.

## Design Requirements

Target: **50 MHz system clock, 8E1 format**

| Requirement | Specification |
|---|---|
| **System Clock** | 50 MHz |
| **Baud Rate** | 9,600 / 115,200 / 460,800 bps |
| **Data Width** | 8 bits |
| **Parity** | Even parity |
| **Stop Bits** | 1 |
| **Format** | 8E1 (11 bits total per frame) |

**Baud counter maximum:**

$$\mathrm{BAUD}_{\mathrm{CNT,MAX}} = \frac{50{,}000{,}000}{115{,}200} \approx 434$$

The counter counts system clock cycles. When it reaches 434, one baud period has elapsed and it resets. The resulting one-cycle pulse (`baud_tick`) drives all TX bit transitions.

> **Integer division accuracy**: 50,000,000 ÷ 115,200 = 434.027... → we use 434. The error is 0.006%, accumulating to less than 0.07% across an 11-bit frame — well within the ±2–3% UART tolerance.

## Baud Rate Generator

Counts system clock cycles and produces a one-cycle **baud_tick** pulse.

```systemverilog
module baud_gen #(
    parameter CLK_FREQ  = 50_000_000,
    parameter BAUD_RATE = 115_200
)(
    input  logic clk, rst_n,
    output logic baud_tick
);
    localparam BAUD_CNT_MAX = CLK_FREQ / BAUD_RATE;  // 434

    logic [$clog2(BAUD_CNT_MAX)-1:0] cnt;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) cnt <= '0;
        else if (cnt == BAUD_CNT_MAX - 1) cnt <= '0;
        else cnt <= cnt + 1;
    end

    assign baud_tick = (cnt == BAUD_CNT_MAX - 1);
endmodule
```

`$clog2(BAUD_CNT_MAX)` automatically sizes the counter register — for 434 counts, 9 bits are required. Using `$clog2` keeps the code portable: changing `CLK_FREQ` or `BAUD_RATE` automatically adjusts the counter width without any manual recalculation.

## TX FSM — States

**IDLE → START → DATA (8 bits) → PARITY → STOP → IDLE**

```systemverilog
typedef enum logic [2:0] { IDLE, START, DATA, PARITY, STOP } tx_state_t;
tx_state_t state;
logic [2:0] bit_cnt;
logic [7:0] tx_shift;

always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin state <= IDLE; tx_out <= 1'b1; end
    else if (baud_tick) case (state)
        IDLE:   if (tx_start) begin
                    tx_shift <= tx_data; state <= START; tx_out <= 1'b0;
                end
        START:  begin tx_out <= tx_shift[0]; state <= DATA; bit_cnt <= 0; end
        DATA:   begin
                    tx_shift <= {1'b0, tx_shift[7:1]};
                    tx_out <= tx_shift[1];
                    if (bit_cnt == 6) state <= PARITY;
                    else bit_cnt <= bit_cnt + 1;
                end
        PARITY: begin tx_out <= ^tx_data; state <= STOP; end
        STOP:   begin tx_out <= 1'b1; state <= IDLE; end
    endcase
end
```

> **FSM design note**: The `tx_shift` register serializes the data. On each `baud_tick`, the register shifts right by one bit and `tx_out` takes the next bit in sequence. Even parity is `^tx_data` — the reduction XOR of all 8 data bits, computed combinationally from the original data rather than from the shifted register.

## RX FSM — Center Sampling

Sample each bit at its **center** for maximum noise margin.

$$\text{Center offset} = \frac{\mathrm{BAUD}_{\mathrm{CNT,MAX}}}{2} \approx 217\,\text{CLK cycles}$$

```systemverilog
typedef enum logic [2:0] { IDLE, START, DATA, PARITY, STOP } rx_state_t;

always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) state <= IDLE;
    else case (state)
        IDLE:   if (!uart_rx) begin baud_cnt <= '0; state <= START; end
        START:  if (baud_cnt == BAUD_CNT_MAX >> 1) begin
                    if (!uart_rx) begin state <= DATA; bit_cnt <= 0; end
                    else state <= IDLE;  // false start — glitch rejection
                end
        DATA:   if (baud_tick) begin
                    rx_shift <= {uart_rx, rx_shift[7:1]};
                    if (bit_cnt == 7) state <= PARITY;
                    else bit_cnt <= bit_cnt + 1;
                end
        PARITY: if (baud_tick) begin
                    parity_err <= (^rx_shift != uart_rx); state <= STOP;
                end
        STOP:   if (baud_tick) begin
                    if (uart_rx) rx_data_valid <= 1'b1;
                    else frame_err <= 1'b1;
                    state <= IDLE;
                end
    endcase
end
```

> **False start detection**: After the falling edge triggers the START state, the FSM waits half a bit period and re-checks `uart_rx`. If the line is already High again, the transition was noise — the FSM returns to IDLE without consuming a byte. This glitch rejection is critical on noisy lines and is the reason the half-period wait exists.

> **Shift direction in RX**: Data is shifted in MSB-first into `rx_shift` (`{uart_rx, rx_shift[7:1]}`). After 8 shifts, `rx_shift[0]` holds D0 (the first bit received = LSB of the byte), meaning the final `rx_shift` contains the byte in the correct bit order.

## Center Sampling — Animated

*The receiver must sample at the center of each bit period — not at the edge — to avoid noise contamination near signal transitions.*

<iframe src="/uart-html/UART Center Sampling.html" width="100%" height="230" style="border:none;border-radius:8px;display:block;margin:4px auto;"></iframe>

## Testbench Strategy

```systemverilog
// Loopback: connect TX output directly to RX input
assign uart_rx = uart_tx;

initial begin
    @(posedge clk); tx_data = 8'hA5; tx_start = 1;
    @(posedge clk); tx_start = 0;

    @(posedge rx_data_valid);
    assert (rx_data == 8'hA5) else $error("Loopback mismatch!");
    assert (!parity_err)      else $error("Parity error detected!");
end
```

The loopback testbench connects TX output directly to RX input, verifying end-to-end correctness in one simulation. If all bits, timing, and parity computations are correct, the received byte matches the sent byte and `parity_err` stays Low.

**Additional verification steps:**
- Confirm TX waveform: Start(0) → D0..D7 LSB-first → Parity → Stop(1)
- Check `baud_cnt` value at each RX sample point (should be ≈ BAUD_CNT_MAX/2 after the first, then BAUD_CNT_MAX-1 for subsequent)
- Parity error injection: force a wrong bit on the wire, confirm `parity_err` asserts
- Frame error injection: hold RX low during Stop bit, confirm `frame_err` asserts

## UART Frame Simulation Demo

*Vivado simulation waveform demo (xsim)*

<video controls preload="metadata" width="100%" style="display:block;border-radius:8px;background:#000;max-height:420px;" playsinline>
    <source src="uart_vivado_sim.mp4" type="video/mp4">
    Your browser does not support the video tag.
</video>

## Episode 5 Summary

- **Baud Generator**: `cnt == BAUD_CNT_MAX-1` → one-cycle `baud_tick`
- **TX FSM**: IDLE → START → DATA(8) → PARITY → STOP
- **RX FSM**: half-period delay → center sampling; validate Start and Stop bits
- **Even Parity**: `^tx_data` (reduction XOR)
- **Testbench**: loopback TX→RX; assert data and parity correctness

### Next Episode
> **Episode 6: Top Module — Integrating TX and RX**
> Clock wizard, reset logic, parameterization, reusable interface design

## Key Takeaways

- The baud generator uses integer division — `CLK_FREQ / BAUD_RATE` rounded down; the resulting error (~0.006% for 115200 bps at 50 MHz) is negligible within one UART frame
- `baud_tick` is a single-cycle pulse — advancing the TX FSM state only on `baud_tick` guarantees exactly one baud period per state
- RX center sampling delays `BAUD_CNT_MAX/2` after the Start Bit edge before the first sample; this provides maximum margin against clock drift and edge noise
- False start detection (re-check Start Bit at the half-period point) rejects glitches that would otherwise corrupt a received byte

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga
