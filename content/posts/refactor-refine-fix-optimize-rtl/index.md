---
title: "Refactor, Refine, Fix, and Optimize: Precise Terminology for RTL Development"
date: 2026-02-19T00:47:21+00:00
draft: false
description: "Using the right word when reviewing or requesting RTL changes makes communication faster and less ambiguous. Here is what each term means in an FPGA context."
categories:
  - "FPGA Design Tips"
tags:
  - "Fix/Correct"
  - "Optimize"
  - "Refactor"
  - "Refine"
  - "RTL"
  - "Code Quality"
---

In FPGA and RTL development, four terms are frequently used to describe code changes: **refactor**, **refine**, **fix/correct**, and **optimize**. They are often used interchangeably in casual conversation, but they mean distinctly different things. Using the right term helps your team understand immediately what kind of change is being made — and whether to expect any behavioural differences.

## Refactor

**Definition:** Restructure internal implementation without changing externally observable behaviour.

A refactored module passes the same test vectors before and after the change. Simulation output is bit-identical. Timing may or may not improve.

**Goal:** Readability, maintainability, reducing duplication.

**RTL examples:**

- Replace a hardcoded LFSR feedback expression with a `for` loop or `generate` statement so the polynomial degree is a parameter.
- Split a monolithic 500-line `always` block into a separate FSM block and a separate datapath block for clearer code structure.
- Rename signals from `sig_123` to `s_tx_byte_valid` to make their purpose self-documenting.
- Replace a repeated pattern used in three modules with a shared RTL function or common module instantiation.

> *"I am refactoring the CRC module — output is unchanged, just making it parameterisable."*

## Refine

**Definition:** Improve an already-working implementation to bring it closer to a target specification, reference model, or desired accuracy.

A refinement change typically produces slightly different (and more correct) output, but the core algorithm is not broken — it is just not perfectly matching the spec yet.

**Goal:** Accuracy improvement, fine-tuning, alignment with reference.

**RTL examples:**

- Synchronise the reset seed value in an RTL scrambler with the initial state in the software reference model — the LFSR was working but starting from a different seed.
- Adjust polynomial tap positions in an LFSR by one bit to match the exact specification in a 3GPP or PCIe standard document.
- Tune the fixed-point scaling factor in a filter so that the quantization error matches the target SNR from the golden model.

> *"We need to refine the scrambler — the polynomial is correct but the initial seed is off by one."*

## Fix / Correct

**Definition:** Eliminate a definite bug or mismatch. Something is observably wrong and must be made right.

This is the most direct category — there is a clear before (broken) and after (working) state.

**Goal:** Bug elimination, correctness restoration.

**RTL examples:**

- The CRC polynomial register was initialised to `0x00` instead of `0xFF` — output never matches reference.
- An off-by-one error in a state counter causes the FSM to generate one extra idle cycle, breaking packet framing.
- A missing `else` branch leaves a latch inferred in a synthesised design, causing unpredictable RTL/gate-level simulation mismatch.

> *"This is a bug fix — the polynomial was wrong, the output was completely incorrect."*

## Optimize

**Definition:** Improve measurable resource usage, timing, or power without changing the functional specification.

An optimised design is functionally equivalent to the pre-optimisation version but runs faster, uses fewer resources, or consumes less power.

**Goal:** Timing closure, LUT/DSP/BRAM reduction, power reduction.

**RTL examples:**

- Restructure an adder tree from a linear chain (critical path ∝ N) to a balanced binary tree (critical path ∝ log₂N) to improve Fmax.
- Replace a synchronous clear on a high-fanout reset net with a gate-enable style to reduce hold-time violations at the expense of a slightly larger circuit.
- Move a register stage across a long combinational path (retiming) to equalise pipeline stage depths and allow a higher clock frequency.
- Merge two independent `case` statements that operate on the same signal into one to reduce LUT packing overhead.

> *"After optimization, the LFSR now synthesises to 32 fewer LUTs and timing margin improved to +1.2 ns."*

---

## Summary Table

| Term | Observable behaviour changes? | Typical review comment |
|---|---|---|
| **Refactor** | No | "Same output, cleaner structure" |
| **Refine** | Slightly (more accurate) | "Closer to spec / reference model" |
| **Fix / Correct** | Yes (from wrong to right) | "Bug fixed" |
| **Optimize** | No (functionally equivalent) | "Faster / smaller / lower power" |

Using the correct term in a code review, commit message, or design review prevents misunderstandings like: *"You said you were refactoring — why did the test vector change?"*
