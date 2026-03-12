---
title: "Source-Synchronous Signaling: How High-Speed Parallel Interfaces Stay in Sync"
date: 2026-03-12T10:00:00+09:00
draft: false
description: "Why global clock distribution fails at high speeds, and how source-synchronous interfaces like DDR's DQ+DQS pair solve the timing problem."
categories:
  - "High Speed Interface"
tags:
  - "source-synchronous"
  - "parallel interface"
  - "DDR"
  - "skew"
  - "signal integrity"
  - "DQS"
---

## Overview

As parallel interfaces push to higher speeds, keeping all data bits synchronized becomes increasingly difficult. This post explains why global clock distribution hits a wall — and how **source-synchronous** signaling solves it.

---

## Signal Propagation on PCB

Before diving into timing, it helps to understand how fast signals actually travel on a PCB:

- Typical propagation speed on FR4: **15–17 cm/ns**
- A 1 cm trace length difference ≈ **60–70 ps** of timing skew

At 1 GHz (1 ns period), 70 ps of skew already consumes **7% of your entire timing budget** from a single centimeter of mismatch. As data rates climb, this becomes unmanageable.

---

## The Problem with Global Synchronous Clocking

In a globally synchronous system, one central clock is distributed to all receivers across the board.

{{< anim-global-sync >}}

**The core problem:** As trace lengths diverge, each data bit arrives at a slightly different time. The receiver must sample all bits with a single clock edge — but the valid window shrinks as skew grows.

---

## Source-Synchronous: Send the Clock with the Data

The source-synchronous approach eliminates the global clock problem by having the **transmitter send its own sampling strobe alongside the data**.

In DDR memory, this is the **DQS (Data Strobe)** signal — one strobe per byte lane, traveling with its 8 data bits (DQ).

{{< anim-source-sync >}}

**Key insight:** The DQS strobe and DQ data experience the *same* board-level delay — they travel the same path. The receiver simply uses DQS edges to sample DQ, regardless of how long the traces are.

---

## Global Synchronous vs. Source-Synchronous

| | Global Synchronous | Source-Synchronous |
|--|--|--|
| Clock source | Central, shared | Transmitted with data |
| Skew challenge | All traces vs. clock tree | Only intra-pair (DQ↔DQS) |
| Trace matching | Every signal to clock | DQ matched to DQS per byte lane |
| Example | PCI, ISA | DDR2/3/4/5, LPDDR |
| Speed limit | ~200–400 MHz | 3200+ MT/s |

---

## DDR4 in Practice

DDR4 organizes its signals into **byte lanes**. Each lane has:
- 8× DQ bits
- 1× DQS (differential) — the strobe for that lane
- 1× DM (data mask)

```
Byte Lane 0          Byte Lane 1
DQ[7:0] ←→ DQS0     DQ[15:8] ←→ DQS1
```

The FPGA MIG (Memory Interface Generator) or PHY automatically handles:
- **Write Leveling** — aligns DQS to the memory clock on write
- **Read Training** — calibrates DQ sampling window on read

These calibration routines compensate for any remaining board-level skew at startup.

---

## PCB Design Guidelines

- **Intra-pair matching** (DQ↔DQS within same byte lane): within **±10 ps** (≈ 1–2 mm on FR4)
- **Inter-pair matching** (between byte lanes): within **±25 ps** — relaxed because each lane has its own strobe
- Use serpentine (meander) routing for length matching
- Keep all DQ/DQS traces on the same PCB layer with consistent impedance (typically 40–50 Ω differential)
- Minimize stubs and via transitions

---

## Summary

- Global synchronous clocking struggles beyond a few hundred MHz due to skew accumulation across the board
- Source-synchronous moves the reference clock to the data source — the strobe travels with the data, so board delay cancels out
- DDR memory's DQ+DQS pairing is the canonical example: each byte lane has its own strobe, limiting skew control to just that group
- PCB design requires intra-pair trace matching to within picoseconds; write leveling and read training handle the rest at runtime
