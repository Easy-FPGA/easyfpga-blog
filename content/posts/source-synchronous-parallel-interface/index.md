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

<div class="anim-container" style="background:#1a1a2e;border-radius:10px;padding:24px;margin:24px 0;font-family:monospace;">
  <div style="color:#a0aec0;font-size:0.85rem;margin-bottom:12px;">Global Synchronous — Clock skew accumulates across traces</div>
  <canvas id="globalClkCanvas" width="680" height="180" style="width:100%;border-radius:6px;"></canvas>
  <div style="display:flex;gap:16px;margin-top:10px;font-size:0.8rem;">
    <span style="color:#f6e05e;">■ CLK</span>
    <span style="color:#68d391;">■ D[0] (short trace)</span>
    <span style="color:#fc8181;">■ D[1] (long trace)</span>
    <span style="color:#76e4f7;">■ D[2] (longer trace)</span>
  </div>
</div>

<script>
(function() {
  const canvas = document.getElementById('globalClkCanvas');
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;
  const colors = ['#f6e05e','#68d391','#fc8181','#76e4f7'];
  const labels = ['CLK','D[0]','D[1]','D[2]'];
  const delays = [0, 0, 8, 18]; // px delay (skew)
  let t = 0;

  function drawWave(y, color, delay, period, high, label) {
    ctx.strokeStyle = color;
    ctx.lineWidth = 2;
    ctx.beginPath();
    const rowH = 28;
    for (let x = 0; x < W; x++) {
      const phase = ((x - delay + t) % period + period) % period;
      const val = phase < period / 2 ? high : high + rowH;
      if (x === 0) ctx.moveTo(x, y + val);
      else ctx.lineTo(x, y + val);
    }
    ctx.stroke();
  }

  function render() {
    ctx.clearRect(0, 0, W, H);
    ctx.fillStyle = '#1a1a2e';
    ctx.fillRect(0, 0, W, H);
    const period = 80;
    const rowSpacing = 42;
    delays.forEach((d, i) => {
      drawWave(i * rowSpacing + 4, colors[i], d, period, 0, labels[i]);
    });
    // skew arrows
    ctx.strokeStyle = '#ff6b6b';
    ctx.setLineDash([3,3]);
    ctx.lineWidth = 1;
    [120, 200, 280].forEach(xBase => {
      ctx.beginPath(); ctx.moveTo(xBase, 8); ctx.lineTo(xBase, H - 8); ctx.stroke();
      ctx.beginPath(); ctx.moveTo(xBase + delays[2], 8); ctx.lineTo(xBase + delays[2], H - 8); ctx.stroke();
    });
    ctx.setLineDash([]);
    t = (t + 0.6) % period;
    requestAnimationFrame(render);
  }
  render();
})();
</script>

**The core problem:** As trace lengths diverge, each data bit arrives at a slightly different time. The receiver must sample all bits with a single clock edge — but the valid window shrinks as skew grows.

---

## Source-Synchronous: Send the Clock with the Data

The source-synchronous approach eliminates the global clock problem by having the **transmitter send its own sampling strobe alongside the data**.

In DDR memory, this is the **DQS (Data Strobe)** signal — one strobe per byte lane, traveling with its 8 data bits (DQ).

<div class="anim-container" style="background:#1a1a2e;border-radius:10px;padding:24px;margin:24px 0;font-family:monospace;">
  <div style="color:#a0aec0;font-size:0.85rem;margin-bottom:12px;">Source-Synchronous — DQS strobe travels with DQ data</div>
  <canvas id="ssCanvas" width="680" height="220" style="width:100%;border-radius:6px;"></canvas>
  <div style="display:flex;gap:16px;margin-top:10px;font-size:0.8rem;flex-wrap:wrap;">
    <span style="color:#f6e05e;">■ DQS (strobe)</span>
    <span style="color:#68d391;">■ DQ[0]</span>
    <span style="color:#fc8181;">■ DQ[1]</span>
    <span style="color:#76e4f7;">■ DQ[2]</span>
    <span style="color:#b794f4;">■ DQ[3]</span>
    <span style="color:#ff6b6b;">↕ sample point</span>
  </div>
</div>

<script>
(function() {
  const canvas = document.getElementById('ssCanvas');
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;
  const groupDelay = 22; // whole group shifts together
  const colors = ['#f6e05e','#68d391','#fc8181','#76e4f7','#b794f4'];
  const labels = ['DQS','DQ[0]','DQ[1]','DQ[2]','DQ[3]'];
  const intraSkew = [0, 2, -2, 3, -1]; // tiny intra-pair skew (ps-level)
  let t = 0;
  const period = 80;
  const rowSpacing = 40;

  function drawWave(y, color, skew, isDQS) {
    ctx.strokeStyle = color;
    ctx.lineWidth = isDQS ? 2.5 : 1.8;
    ctx.beginPath();
    const rowH = isDQS ? 22 : 20;
    for (let x = 0; x < W; x++) {
      const phase = ((x - groupDelay - skew + t) % period + period) % period;
      const val = phase < period / 2 ? 0 : rowH;
      if (x === 0) ctx.moveTo(x, y + val);
      else ctx.lineTo(x, y + val);
    }
    ctx.stroke();
  }

  function render() {
    ctx.clearRect(0, 0, W, H);
    ctx.fillStyle = '#1a1a2e';
    ctx.fillRect(0, 0, W, H);

    labels.forEach((l, i) => {
      drawWave(i * rowSpacing + 6, colors[i], intraSkew[i], i === 0);
    });

    // sample points at DQS rising edges
    ctx.strokeStyle = '#ff6b6b';
    ctx.lineWidth = 1.5;
    ctx.setLineDash([4,3]);
    for (let x = groupDelay; x < W; x += period) {
      const xPos = (x - t % period + period) % period + groupDelay;
      if (xPos > 0 && xPos < W) {
        ctx.beginPath(); ctx.moveTo(xPos, 0); ctx.lineTo(xPos, H); ctx.stroke();
      }
    }
    ctx.setLineDash([]);

    // DQS label
    ctx.fillStyle = '#f6e05e';
    ctx.font = '11px monospace';
    ctx.fillText('DQS', 4, 20);

    t = (t + 0.6) % period;
    requestAnimationFrame(render);
  }
  render();
})();
</script>

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
