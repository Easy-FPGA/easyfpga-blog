---
title: "Quantization for CNN Inference on FPGA"
date: 2025-11-24T23:03:14+00:00
draft: false
description: "Why integer quantization is essential for running CNN models on FPGA, how the conversion works, and a concrete fixed-point example."
math: true
categories:
  - "CNN on FPGA"
tags:
  - "FPGA"
  - "Quantization"
  - "Fixed-Point"
  - "CNN"
  - "Deep Learning"
  - "Hardware Accelerator"
---

## What Is Quantization?

In signal processing, **quantization** is the process of mapping a continuous range of values to a discrete (integer) set. In deep learning and hardware acceleration, it specifically refers to:

> Converting floating-point (FP32, FP16, or BF16) model weights and activations to lower-bit integers (INT8, INT4, etc.) in order to reduce memory footprint and computational cost.

Quantization is the bridge that makes neural networks practical on resource-constrained hardware.

## Why Quantization Is Essential for FPGA CNN Implementation

FPGAs have a fixed amount of logic, DSP slices, and BRAM. Floating-point arithmetic is expensive in all three dimensions:

| Operation | FP32 Multiply | INT8 Multiply |
|---|---|---|
| DSP slices (Xilinx) | 3 | 1 (or shared) |
| LUT usage | High | Low |
| BRAM for weights | 4 bytes / weight | 1 byte / weight |
| Throughput (at same clock) | 1× | ~4× |

A typical mid-range CNN (ResNet-18, MobileNet-v2) has millions of weights. At FP32 that is hundreds of megabytes — far beyond what any FPGA can hold on-chip. At INT8 it drops to tens of megabytes, enabling full on-chip weight storage in BRAM/URAM on high-end parts, or fitting in external DDR with manageable bandwidth requirements.

**For edge devices and low-power applications** (surveillance cameras, industrial inspection, drones), quantization is not optional — the design simply cannot close at target power without it.

## The Core Goal

> Map floating-point values to integers **while minimising information loss**.

The fundamental quantization equation is:

$$\text{real\_value} = \text{stored\_integer} \times \text{scale\_factor}$$

Where `scale_factor` calibrates the integer back to the original floating-point range.

## Step-by-Step Conversion Example

Suppose you want to represent the floating-point weight **0.3508980572** using an `ap_fixed<8,2>` type — 8 total bits, 2 integer bits, 6 fractional bits.

**Step 1: Determine the scale factor**

With 6 fractional bits, the scale factor is $2^6 = 64$.

**Step 2: Scale the value**

$$0.3508980572 \times 64 = 22.457...$$

**Step 3: Round to integer**

$$\lfloor 22.457 \rceil = 22_{10} = 0\text{x}16 = 0\text{b}00010110$$

**Step 4: Reconstruction (dequantization)**

$$22 / 64 = 0.34375$$

The quantization error is $|0.3508980572 - 0.34375| = 0.00715$, or about 2% relative error.

**Binary layout of `0b00010110` in `ap_fixed<8,2>`:**

```
Bit 7  Bit 6  Bit 5  Bit 4  Bit 3  Bit 2  Bit 1  Bit 0
  0      0      0      1      0      1      1      0
  ↑↑                  ↑ ← fractional part (010110 = 22/64 = 0.34375) →
  integer part (00 = 0)
```

## Quantization Approaches

### Post-Training Quantization (PTQ)
Convert a pre-trained FP32 model to INT8 after training is complete. Requires a small calibration dataset to determine the optimal scale factors per layer. Fast and simple — no retraining needed.

### Quantization-Aware Training (QAT)
Fold quantization error into the training loop so the model learns to compensate. Achieves higher accuracy than PTQ, especially at very low bit widths (INT4 and below).

## Application to LeNet-5 on FPGA

For a concrete end-to-end example, consider LeNet-5 (see [LeNet-5 Implementation on FPGA](../lenet5-implementation-on-fpga/)):

- Train in PyTorch with FP32.
- Apply PTQ with INT8 quantization on all convolutional and fully-connected weights.
- Export weights as `int8_t` arrays.
- Initialise BRAM in Vivado using `$readmemh` with the quantized weight file.
- Implement each MAC as an 8×8 multiply with an accumulator wide enough to avoid overflow.

With INT8 weights, LeNet-5's ~61,000 parameters occupy only ~62 KB — easily fitting in the BRAM of even an Artix-7 35T.

## Key Points

- Quantization is **not optional** for practical FPGA CNN implementation — FP32 is too expensive in DSP and BRAM resources.
- INT8 is the current industry sweet spot: ~4× memory reduction, ~4× throughput improvement, <1% accuracy degradation for most standard models when using PTQ with calibration.
- The `ap_fixed<W,I>` type in Xilinx HLS and `logic signed [W-1:0]` in RTL are the standard representations for fixed-point inference.

## Reference

- [What is INT8 Quantization and Why Is It Popular for Deep Neural Networks? (MathWorks)](https://kr.mathworks.com/company/technical-articles/what-is-int8-quantization-and-why-is-it-popular-for-deep-neural-networks.html)
