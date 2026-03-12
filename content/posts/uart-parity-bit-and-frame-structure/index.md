---
title: "UART Parity Bit and Frame Structure"
date: 2025-07-16T12:02:34+00:00
draft: false
description: "UART frames consist of a start bit, data bits, an optional parity bit, and stop bit(s). Parity can be even, odd, or omitted. Even parity is calculated"
categories:
  - "UART"
tags:
  - "FPGA"
  - "Parity"
  - "UART"
---

## UART Frame Format

UART (Universal Asynchronous Receiver/Transmitter) sends data in a structured frame that includes the following components:

- **Start Bit**: Always `0`; indicates the beginning of the frame.

- **Data Bits**: The actual payload, typically 5 to 8 bits.

- **Parity Bit** *(optional)*: Used for simple error detection.

- **Stop Bit(s)**: One or two bits set to `1`; mark the end of the frame.

## Parity Bit Modes

- **Even Parity**: The parity bit is set such that the total number of `1`s (including the parity bit) is even.

- **Odd Parity**: The parity bit ensures the total number of `1`s is odd.

- **No Parity**: Parity bit is omitted.

### Example: Even Parity Calculation

Let's consider an example with ** 0x29**:

- **Binary**: `00101001` → Number of `1`s = 3 (odd)

- **Even Parity**: Add parity bit `1` → Total `1`s = 4 (even)

- ✅ *Result*: For even parity, the parity bit for `0x29` is `1`

## Verilog Implementation with Reduction XOR

```
`logic [7:0] tx_data = 8'b10101010;
parity_bit <= (parity_mode) ? ~^tx_data : ^tx_data;`
```

Using the reduction XOR operator (^), you can easily calculate the parity:

- `^tx_data` returns the XOR of all bits in `tx_data`

- For **even parity**, `parity_bit = ^tx_data`

- For **odd parity**, use `~^tx_data`

## Simulation Behavior

In the simulation waveform, you can clearly observe that the parity bit is transmitted immediately after the data bits. This follows the standard UART sequence.

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/07/image.png?w=1024)*Simulation waveform under even parity mode*

## RX-Side Parity Check Logic

```
`parity_bit_rx   <= rx_line; // Received parity bit from UART frame

// Parity bit computed from received data
parity_bit_calc <= parity_mode ? ~^shift_reg : ^shift_reg; 

if (parity_bit_rx != parity_bit_calc)
    rx_error <= 1'b1; // Set error flag if parity doesn't match`
```

The receiver recalculates the expected parity from the received data and compares it with the parity bit received over the wire. If the values differ, it flags a parity error.