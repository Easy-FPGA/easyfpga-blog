---
title: "Critical Path and Pipelining in FPGA Design"
date: 2025-11-28T22:09:14+00:00
draft: false
description: "What a critical path is, how it limits your clock frequency, and how to fix it with pipelining — with SystemVerilog examples."
math: true
categories:
  - "FPGA Design"
tags:
  - "Critical Path"
  - "FPGA Design"
  - "Pipelining"
  - "Timing Closure"
  - "SystemVerilog"
---

## What Is a Critical Path?

The **critical path** is the longest combinational logic path between any two registers in a synchronous digital circuit — measured in propagation delay. It sets a hard lower bound on your clock period:

$$T_{clk} \geq T_{critical\_path} + T_{setup} + T_{hold}$$

$$F_{max} = \frac{1}{T_{clk}}$$

Everything else in your design can run faster. The critical path is the single bottleneck that caps the maximum clock frequency.

## A Concrete Example

```systemverilog
// Bad: long critical path
always_ff @(posedge clk) begin
    result <= a + b + c + d + e + f + g + h;  // 7 chained additions
end
```

**What happens in a single clock cycle:**

1. Clock edge arrives → registers release values `a, b, c …`
2. Adder 1: compute `a + b`
3. Adder 2: compute `(a+b) + c`
4. Adder 3: compute `((a+b)+c) + d`
5. … continue through all 7 additions …
6. Final sum must settle at `result`'s input **before** the next clock edge

**The problem:** Seven additions are chained end-to-end. If each adder has a 3 ns propagation delay:

| Metric | Value |
|---|---|
| Total critical path delay | 7 × 3 ns = 21 ns |
| Maximum clock frequency | 1 / 21 ns ≈ **47 MHz** |
| Can it run at 100 MHz? | **No** — the path is too long |

## Fix It With Pipelining

Pipelining inserts **registers between stages** so that each stage only needs to complete its smaller portion of work within a single clock cycle.

```systemverilog
// Good: pipelined — short critical path per stage
always_ff @(posedge clk) begin
    // Stage 1: four additions in parallel, then two partial sums
    temp1 <= a + b + c + d;   // 3 chained additions
    temp2 <= e + f + g + h;   // 3 chained additions
end

always_ff @(posedge clk) begin
    // Stage 2: combine the two partial sums
    result <= temp1 + temp2;  // 1 addition
end
```

**Effect:**

| Metric | Before | After |
|---|---|---|
| Worst-case stage depth | 7 additions | 3 additions |
| Critical path delay | 21 ns | 9 ns |
| Maximum clock frequency | ~47 MHz | ~111 MHz |
| Latency (cycles) | 1 | 2 |

The trade-off is latency: the result now takes 2 clock cycles instead of 1. Throughput, however, is unchanged — once the pipeline is full, a new result emerges every clock cycle.

## Key Concepts

**Breaking work into stages:** Every pipeline register is a "checkpoint" that saves the intermediate result. The next stage picks up from there on the following clock edge.

**Latency vs. throughput:** Pipelining adds latency (more cycles to produce the first result) but does not reduce throughput (results still emerge at one per cycle once the pipeline is filled).

**Identifying the critical path in Vivado:** After synthesis and implementation, open the Timing Summary report. The path with the **smallest positive slack** (or negative slack if timing is violated) is your critical path. Use the "Path Details" view to see every logic cell and net delay on that path.

**Common fixes beyond simple pipelining:**

| Technique | When to use |
|---|---|
| Register retiming | Move registers across logic boundaries without changing behaviour |
| Logic restructuring | Rebalance an adder tree to reduce depth (e.g., use a 4-2 compressor tree) |
| DSP inference | Map arithmetic to DSP48 slices, which have dedicated fast paths |
| Reduce fanout | High-fanout nets add routing delay — buffer or register them |

## Quick Rule of Thumb

If your design targets a clock frequency $F$ and the synthesis tool reports critical path slack of $-\Delta$, you typically need to reduce the combinational depth until the path is at least $\Delta$ ns shorter. Pipelining is the most reliable structural technique to achieve this.
