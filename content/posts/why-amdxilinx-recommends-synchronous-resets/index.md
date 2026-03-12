---
title: "Why AMD(Xilinx) Recommends Synchronous Resets"
date: 2025-06-30T12:52:30+00:00
draft: false
categories:
  - "FPGA Design"
tags:
  - "AMD"
  - "FPGA"
  - "Reset"
  - "Xilinx"
---

[https://docs.amd.com/r/2021.1-English/ug949-vivado-design-methodology/When-and-Where-to-Use-a-Reset](https://docs.amd.com/r/2021.1-English/ug949-vivado-design-methodology/When-and-Where-to-Use-a-Reset)

## Advantages of Synchronous Resets

BenefitExplanation**Better Resource Mapping**Synchronous resets can be mapped directly to more FPGA resources (e.g., flip-flops inside DSPs, BRAMs).**Improved Logic Performance**Asynchronous resets can negatively affect general logic timing paths and increase routing complexity.**Simpler Routing**A global asynchronous reset might not increase control sets but does require routing reset wires to many elements, which can become complex.**Protects Memory Structures**Asynchronous resets may corrupt the contents of BRAMs, LUTRAMs, and SRLs during reset assertion—especially if those resources are driven by asynchronously reset registers.**Placement Flexibility**With synchronous resets, the logic can be more easily remapped during implementation for better packing and performance.**Compatibility with Dedicated Blocks**Some blocks like DSP48s and BRAMs only support synchronous resets, so using asynchronous resets can prevent proper inference into those blocks.

Reset HDL Coding Example

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/06/image-17.png?w=690)*Changing Asynchronous Reset into Synchronous Reset on Multipiler*

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/06/image-19.png?w=496)*Commenting Out Code with Reset Conditions*

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/06/image-20.png?w=234)*Separate Procedural Statements for Registers With and Without Reset*

The Optimal way to remove the resets is **to create separate sequential logic procedures with one for reset condition and the other for non-reset conditions.**

## Summary: In FPGAs...

ItemRecommended?Reason✅ **Synchronous reset**Commonly recommendedBetter for timing, tool support⚠️ **Asynchronous reset**Use with cautionNeeded for power-up init, but adds complexity✅ **Active-low reset**PreferredMore immune to noise, widely adopted standard✅ **Active-high reset**AcceptableLogically fine, but less common in industry