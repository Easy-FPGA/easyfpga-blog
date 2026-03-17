---
title: "UART Frame Format"
date: 2026-03-16T12:00:00+09:00
draft: true
description: "A practical guide to UART frame fields: start/data/parity/stop bits, LSB-first ordering, overhead, and throughput implications."
categories:
  - "UART"
tags:
  - "FPGA"
  - "UART"
  - "RTL"
  - "Xilinx"
  - "EasyFPGA"
---
# UART Frame Format

Each UART transmission carries data inside a precisely defined frame. Understanding the purpose of every field — not just its name — is what separates a developer who can implement UART from scratch from one who only configures a vendor IP. This episode dissects the frame bit by bit, covering the transmission order convention, parity computation, and the overhead implications of different format choices.

## UART Frame Structure

Every UART transmission is based on a fixed **Frame** structure.

```
  Idle  Start   D0   D1   D2   D3   D4   D5   D6   D7  Parity  Stop  Idle
───────┐                                                                ┌────
  1    │  0  │ b0 │ b1 │ b2 │ b3 │ b4 │ b5 │ b6 │ b7 │   P   │  1  │  1 |
       └─────┴────┴────┴────┴────┴────┴────┴────┴────┴───────┴──────────┘
            ←──────────────── 1 Frame ─────────────────────────────→
              1bit   ←──── 8 bits (LSB first) ────→   1bit   1bit
```

| Field | Bits | Value | Purpose |
|---|---|---|---|
| **Start Bit** | 1 | Always **0** | Signals start of frame (falling edge) |
| **Data Bits** | 5–9 | Data | Actual data payload (LSB first) |
| **Parity Bit** | 0–1 | Even/Odd | Optional error detection |
| **Stop Bit** | 1–2 | Always **1** | Signals end of frame, returns to idle |

## Frame Transmission — Animated

*0xA5 frame (8E1) — bits transmitted one at a time, LSB first. Watch each field highlighted in sequence.*

<iframe src="/uart-html/UART Frame Transmission.html" width="100%" height="400" style="border:none;border-radius:8px;display:block;margin:4px auto;"></iframe>

## Start Bit & Stop Bit

### Start Bit
- Line is normally **High (1, Idle)**
- Transmission begins with a **Low (0)** → **Falling Edge**
- Receiver uses this falling edge to **re-synchronize its timing**

### Stop Bit
- Always **High (1)**
- Returns the line to Idle state
- **1 Stop bit** (standard), 2 Stop bits (legacy devices, slow MCUs)

```
  Idle      Start      Data ...      Stop     Idle
───┐          ↓                       ↑──────────
   └──────────┘   ~~~data~~~   ───────┘
       Falling Edge                  Back to High
```

## Data Bits & Bit Order

### Data Width Options
- **5, 6, 7, 8** bits — 8-bit is most common (ASCII, byte data)
- **9** bits — special use (e.g., multi-drop address flag)

### Transmission Order: LSB First

```
Data to send: 0xA5 = 1010_0101b

Bit order:   D0  D1  D2  D3  D4  D5  D6  D7
TX value:     1   0   1   0   0   1   0   1
              ↑ LSB                  MSB ↑

TX waveform: D0 → D1 → D2 → D3 → D4 → D5 → D6 → D7
```

- The **LSB (D0)** is always transmitted first
- For 0xA5, the first bit on the wire is: **1** (LSB)

> **Why LSB first?** Early UART hardware used a shift register that naturally shifted bits out from the LSB end. This is opposite to the MSB-first convention most software developers expect — so always verify bit ordering explicitly in simulation waveforms before assuming the RX design is correct.

## Parity Bit

Optional field for **single-bit error detection**

### Even Parity
Set parity bit so that the **total number of 1s (data + parity) is even**

### Odd Parity
Set parity bit so that the **total number of 1s (data + parity) is odd**

| Data (0b10010100) | Count of 1s | Even Parity | Odd Parity |
|---|---|---|---|
| 1 0 0 1 0 1 0 0 | 3 (odd) | **1** (make even) | **0** (keep odd) |

```systemverilog
// Even parity using reduction XOR
parity_bit = ^tx_data;       // 1 if count of 1s is odd
// Odd parity
parity_bit = ~^tx_data;
```

> Parity detects single-bit errors only. **Any even number of bit errors** (2, 4, ...) cancel out and go undetected. For reliable data integrity over longer payloads, use a CRC checksum at the application layer.

## Checksum for Multi-Byte Data

For longer payloads, use a **Checksum** instead of per-byte parity

### Checksum Calculation Example

```
Data bytes: 0x12, 0x34, 0x56

Sum:  0x12 + 0x34 + 0x56 = 0x9C

Checksum (1 byte): 0x9C  (discard overflow beyond 1 byte)
```

### Advantages over Parity
- **Block-level protection**: verifies integrity of the entire data block
- **Detects multiple bit errors** that simple parity cannot catch
- **Lower overhead** for long transfers: one checksum per block

## UART Communication Overhead

**8N1 format** (8 data bits, No parity, 1 stop bit) — most common configuration

$$\text{Overhead} = \frac{\text{Non-data bits}}{\text{Total bits}} = \frac{2}{10} = 20\%$$

| Config | Total Bits | Data Bits | Overhead |
|---|---|---|---|
| 8N1 | 10 | 8 | **20%** |
| 8E1 | 11 | 8 | **27%** |
| 8N2 | 11 | 8 | **27%** |
| 7E1 | 10 | 7 | **30%** |

### Effective Throughput (115200 bps, 8N1)
$$\text{Effective throughput} = 115200 \times \frac{8}{10} = 92{,}160 \text{ bps} = 11{,}520 \text{ bytes/s}$$

## Frame Format Summary

```
115200 bps, 8E1:

  Idle  │S│ D0 │ D1 │ D2 │ D3 │ D4 │ D5 │ D6 │ D7 │ Par│Stp│  Idle
────────┘ └────┴────┴────┴────┴────┴────┴────┴────┴────┴───┘ ──────
         │←────────────────── 8.68µs × 11 = 95.5µs ──────────────→│
         1bit              8 data bits               1bit  1bit
        (Start)                                    (Parity)(Stop)
```

### Key Points
- **Start Bit**: always 0, falling edge → timing synchronization
- **Data Bits**: 8 bits, LSB first, actual payload
- **Parity Bit**: even parity, detects 1-bit errors
- **Stop Bit**: always 1, returns to idle

## Episode 2 Summary

- UART frame = Start(1) + Data(5–9) + Parity(0–1) + Stop(1–2)
- **Start Bit**: always 0; falling edge resets receiver timing
- **Data**: LSB transmitted first; 8 bits is the standard
- **Parity**: even/odd for 1-bit error detection (optional)
- **Stop Bit**: always 1; returns line to idle
- **8N1 overhead**: 20% (10 bits total, 8 data bits)

### Next Episode
> **Episode 3: RS-232 & RS-485**
> Converting UART logic signals to physical electrical standards

## Key Takeaways

- UART frame = Start (1b, always 0) + Data (5–9b, LSB first) + Parity (1b, optional) + Stop (1–2b, always 1) — every field at exactly the right timing; missing any field causes a framing error
- LSB-first transmission is counterintuitive for software developers — verify the bit order explicitly in simulation before signing off on the RX design
- Even parity is the reduction XOR of all data bits: `parity = ^data[7:0]`; the receiver recalculates and compares on every received byte
- 8N1 carries 20% overhead (2 framing bits per 10-bit frame); 8E1 adds 1 more bit but enables hardware parity error detection at the receiver

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga







