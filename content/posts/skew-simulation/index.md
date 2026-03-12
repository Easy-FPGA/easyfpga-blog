---
title: "Skew Simulation"
date: 2025-06-16T23:09:38+00:00
draft: false
categories:
  - "High Speed Interface"
tags:
  - "FPGA"
  - "Skew"
---

## What is Skew?

Skew refers to the timing difference in signal arrival between multiple paths that are supposed to be synchronized.

![skew, waver form](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/06/image-10.png?w=848)*shows a skewed waveform between data lines.*

## Why Skew Simulation Is Necessary

If you simulate your design assuming everything is perfectly aligned, you may encounter issues during actual testing. In real PCBs, skew naturally exists, which is why techniques like length matching are necessary.

However, even with length matching, the impact of skew becomes more significant as signal speed increases. Therefore, it's important to address skew issues during simulation.

## What Problems Can Skew Cause?

- Data Corruption

In parallel data buses, if bits arrive at different times, the receiver might sample a mix of old and new values, resulting in incorrect data being latched.

- Mismatch Between Simulation and Real Hardware

Simulations often assume ideal timing. But in real hardware, trace lengths, loading, and environmental factors introduce skew, which can lead to unexpected failures not seen in simulation.

## How to Simulate Skew

Verilog example:

```
`assign delayed_data[0] = data[0];
assign #1 delayed_data[1] = data[1]; // 1ns delay
assign #2 delayed_data[2] = data[2]; // 2ns delay
assign #3 delayed_data[3] = data[3]; // 3ns delay
assign #0 delayed_data[4] = data[4];
assign #0 delayed_data[5] = data[5];
assign #0 delayed_data[6] = data[6];
assign #0 delayed_data[7] = data[7];`
```

You can Intentionally add delay to simulate skew and see how your design reacts.

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/06/image-12.png?w=893)*Simulation Waveform with skew applied*