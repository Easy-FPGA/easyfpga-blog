---
title: "What is UART?"
date: 2026-03-16T12:00:00+09:00
draft: true
description: "UART fundamentals for FPGA engineers: asynchronous communication basics, frame structure intuition, baud rate, clock drift, and practical use cases."
categories:
  - "UART"
tags:
  - "FPGA"
  - "UART"
  - "RTL"
  - "Xilinx"
  - "EasyFPGA"
---
# What is UART?

**UART — Universal Asynchronous Receiver/Transmitter**

A hardware communication protocol for **serial communication** between two devices.

UART is one of the oldest digital communication protocols still in active use. Despite being designed alongside early teletype systems, it appears in virtually every FPGA board, microcontroller, and development kit — because it needs only two wires and every terminal emulator on every operating system speaks it. This episode covers the fundamentals: what asynchronous communication really means, how baud rate works, and why clock drift is the central engineering challenge when implementing UART from scratch.

### Key Features

| Feature | Description |
|---|---|
| **Asynchronous** | No dedicated clock signal between devices |
| **Simple Interface** | Only TX (Transmit) and RX (Receive) pins needed |
| **Full-Duplex** | Can send and receive data simultaneously |
| **Configurable Baud Rate** | Adjust transmission speed for different applications |

## Synchronous vs Asynchronous Communication

### Synchronous Communication
```
Clock  ──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──
         └─┘  └─┘  └─┘  └─┘  └─┘  └─┘
Data   ───X 0  X 1  X 1  X 0  X 1  X──
```
- Transmitter and receiver share the **same clock**
- Data sampled on each clock edge → **reliable and fast**

### Asynchronous Communication (UART)
```
TX/RX  ─────┐S│D0│D1│D2│D3│D4│D5│D6│D7│P│E┌─
            └─────────────────────────────┘
             Start    Data Bits        Stop
```
- Each device uses its **own internal clock**
- Receiver detects the **start bit** (falling edge) to synchronize

## Simple Hardware Interface

Only **2 signal wires** needed — TX and RX

```
┌──────────────────┐              ┌──────────────────┐
│       FPGA       │              │       MCU        │
│                  │              │                  │
│         UART_TX  ├─────────────►│  UART_RX         │
│                  │              │                  │
│         UART_RX  │◄─────────────┤  UART_TX         │
│                  │              │                  │
└──────────────────┘              └──────────────────┘
```

- TX connects to RX: **cross-connect**
- Common **GND** is required
- Optional: CTS/RTS for hardware flow control

## Baud Rate

**Baud Rate** — the number of signal changes (symbols) per second

$$\text{Bit Duration} = \frac{1}{\text{Baud Rate}}$$

| Baud Rate | Bit Duration | Typical Use |
|---|---|---|
| 9,600 | 104.2 µs | Sensors, legacy devices |
| 115,200 | 8.68 µs | PC debugging, common default |
| 460,800 | 2.17 µs | High-speed embedded |
| 1,000,000 | 1.00 µs | Industrial, short distance |

> **115,200 bps** is the most commonly used default baud rate.

## Why is UART Considered Low-Speed?

### Clock Drift Problem

Each device uses an **independent clock**, causing **frequency offset** to accumulate over time.

```
Transmitter:  ──┬──┬──┬──┬──┬──┬── (8.68 µs intervals — ideal)
Receiver:     ──┬──┬──┬──┬──┬──┬── (8.69 µs intervals — 1% off)
                                          ↑ Sampling drift
```

- Accumulated drift → wrong bit sampled → **framing error**
- Maximum clock tolerance: typically **±2–3%**

### Solutions
- **Oversampling** (16× or 8×): sample at the center of each bit
- **Short frames**: 10–11 bits per frame limits drift exposure
- **Re-sync per frame**: receiver resets timing on every Start bit

> **Engineering note**: Oversampling works by dividing each bit period into 16 time slots. The receiver samples at the center three slots and takes a majority vote. This tolerates both clock offset and signal transition glitches near bit edges — the reason UART reliably operates with ±2% clock tolerance between two independent oscillators.

## Clock Drift Visualization

```
TX sends:   S  D0  D1  D2  D3  D4  D5  D6  D7  P  E
            ↓
            Falling edge → RX resets timing

RX samples (ideal):   ●    ●    ●    ●    ●    ●    ●    ●
                      D0   D1   D2   D3   D4   D5   D6   D7

RX samples (drifted): ●    ●    ●    ●    ●    ●    ●   ●
                      D0   D1   D2   D3   D4   D5   D6  [D7 → wrong!]
```

- Even a small clock mismatch can shift the sampling point into the wrong bit window
- The receiver re-synchronizes its timing on **every Start Bit**

## Clock Drift — Animated Visualization

*0x55 frame (alternating bits) — watch sample points drift into the wrong bit window as offset accumulates*

<iframe src="/uart-html/UART Clock Drift.html" width="100%" height="450" style="border:none;border-radius:8px;display:block;margin:4px auto;"></iframe>

## UART vs SPI vs I²C

| Feature | UART | SPI | I²C |
|---|---|---|---|
| **Type** | Asynchronous | Synchronous | Synchronous |
| **Data Lines** | 2 (TX, RX) | 4 (MOSI, MISO, SCLK, SS) | 2 (SDA, SCL) |
| **Speed** | Up to ~1 Mbps | Tens of Mbps | Up to 3.4 Mbps |
| **Multi-device** | Peer-to-peer | Master-Slave (SS per device) | Multi-master/slave |
| **Wiring** | Simplest | Moderate | Simplest for multi-device |
| **Use Case** | Debug, GPS, BT | Flash, ADC, displays | Sensors, EEPROM |

## UART Applications

### Common Use Cases

| Category | Examples |
|---|---|
| **Embedded Debug** | MCU/FPGA serial console (printf, log output) |
| **Wireless Modules** | Bluetooth (HC-05), Wi-Fi (ESP8266), GPS |
| **PC Connection** | USB-to-UART bridge (FT232, CP2102) |
| **Industrial** | RS-232 (15 m), RS-485 (1200 m) physical layer |
| **Sensor Interface** | LIDAR, IMU, GNSS modules |

## Episode 1 Summary

- **UART** = Universal Asynchronous Receiver/Transmitter
- **Asynchronous**: no shared clock — only TX and RX lines required
- **Baud Rate** sets transmission speed (115200 bps most common)
- **Clock Drift** makes UART unsuitable for high-speed, long-distance links
- **Oversampling** (16×) improves receiver sampling reliability
- Simpler than SPI/I²C but limited to **point-to-point** communication

### Next Episode
> **Episode 2: UART Frame Format**
> Start bit, Data bits, Parity bit, Stop bit — detailed frame structure

## Key Takeaways

- UART is asynchronous: no shared clock — both sides must independently agree on baud rate and stay within ±2–3% frequency accuracy
- The Start Bit falling edge resets accumulated clock drift at the start of every frame; this is why UART tolerates small clock mismatches at all
- Oversampling (16×) divides each bit period into slots and samples the center — providing strong margin against both clock drift and line glitches
- Cross-connecting TX→RX is the most common wiring mistake — always verify the cross-connect before debugging the RTL

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga






