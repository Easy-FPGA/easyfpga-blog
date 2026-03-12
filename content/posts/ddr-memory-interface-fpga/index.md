---
title: "Implementing a DDR Memory Interface on FPGA with Xilinx MIG"
date: 2025-09-04T03:20:49+00:00
draft: false
description: "Why external DDR memory is essential for FPGA designs, how to choose the right DDR type, and how to integrate it using the Xilinx Memory Interface Generator (MIG) IP."
math: true
categories:
  - "FPGA Design"
tags:
  - "AMD"
  - "DDR"
  - "FPGA"
  - "MIG"
  - "Vivado"
  - "Xilinx"
  - "Memory Interface"
---

FPGAs are optimised for massive parallelism, but their on-chip memory resources are limited. Block RAM (BRAM) and UltraRAM (URAM) are fast and easy to use, but they typically provide only a few tens of megabits to a few hundred megabits of capacity. When your application needs gigabytes — high-resolution video frames, machine-learning weight tables, large look-up tables — external **DDR memory** becomes essential.

## Why External DDR Memory?

| Memory Type | Location | Typical Capacity | Bandwidth | Access Latency |
|---|---|---|---|---|
| Distributed RAM | On-chip (LUT-based) | < 1 MB | Very high | 1 cycle |
| Block RAM (BRAM) | On-chip (dedicated) | up to ~200 MB | High | 1–2 cycles |
| UltraRAM (URAM) | On-chip (UltraScale+) | up to ~500 MB | High | 2–3 cycles |
| **External DDR** | Off-chip (dedicated chip) | **1–16+ GB** | Medium-High | ~50–100 ns |

For applications such as:
- HD/4K video frame buffers
- Neural network weight storage for inference
- Radar/sonar data logging
- High-speed data recording

…external DDR is not optional — it is a mandatory component.

## DDR Memory Types

| Type | Supply Voltage | Max Data Rate | Channels per Chip | Typical Applications |
|---|---|---|---|---|
| DDR3 | 1.5 V | ~2,133 MT/s | 1 | Consumer, video processing |
| DDR4 | 1.2 V | ~3,200 MT/s | 1 | High-performance servers, DSP, AI |
| DDR5 | 1.1 V | ~8,400 MT/s | 2 | Next-gen servers, data centres |
| LPDDR4/4X | 1.1 V / 0.6 V | ~4,266 MT/s | 2 | Mobile, battery-powered embedded |
| HBM2e | 1.2 V | ~3.2 Gbps/pin | 8 | AI accelerators, HPC |

> **MT/s** = MegaTransfers per second. DDR memory transfers data on both the rising and falling edge of the clock, so a 1600 MHz DDR4 clock yields 3200 MT/s.

**Effective bandwidth formula:**

$$\text{Bandwidth (MB/s)} = \text{MT/s} \times \frac{\text{Bus width (bits)}}{8}$$

Example: DDR4-3200 with a 32-bit bus → 3200 × 4 = **12,800 MB/s = 12.5 GB/s**

## Choosing the Right DDR Type

**High bandwidth (DSP, AI, radar):** DDR4 or DDR5. HBM offers up to 18× higher bandwidth than DDR5 but only appears in high-end Versal / Alveo devices.

**Power efficiency (battery-powered systems):** LPDDR4/4X. Originally designed for mobile, LPDDR is increasingly being evaluated for data-centre rack servers where power-per-performance is a key metric.

**Legacy / cost-sensitive:** DDR3 remains widely available and sufficient for many video and networking applications.

## DDR Chips and Modules

A single DDR chip typically has an **×8** (8-bit) or **×16** (16-bit) data bus width. To get wider buses:

- **DIMM (Desktop/Server):** 8× ×8-bit chips in parallel → 64-bit bus.
- **FPGA board design:** Connect 1–4 chips directly to the FPGA for 8, 16, 32, or 64-bit buses. The bus width is a key design decision — it directly scales the available bandwidth.

## FPGA–DDR Interface Implementation

Routing dozens of DDR signals with precisely matched trace lengths, implementing the DDR PHY, and managing bank activation, refresh, and command scheduling are all extremely complex tasks. FPGA vendors solve this with dedicated Memory Interface IP cores.

**AMD/Xilinx Memory Interface Generator (MIG):** MIG generates a complete DDR controller plus PHY for Xilinx 7-Series, UltraScale, and UltraScale+ FPGAs. It handles:

- Memory initialisation sequence
- Bank and row management
- Command reordering and scheduling
- Read/write levelling and training
- Optional ECC (Error Correcting Code)

The user interface presented by MIG is a simple AXI4 or native port — you do not need to understand the DDR command protocol to use it.

### Integrating MIG in Vivado

1. Open the **IP Catalog** in Vivado and search for "Memory Interface Generator".
2. Select MIG 7-Series or UltraScale MIS depending on your device.
3. In the configuration wizard, choose your DDR type, data width, and target speed.

![Select MIG from IP Catalog](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/09/image-1.png?w=1024)
*MIG in the Vivado IP Catalog*

![MIG configuration screen](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/09/image-2.png?w=1024)
*Key parameter: Memory Device Interface Speed. At 1066 MHz → 2132 MT/s. With an 8-bit data width, theoretical bandwidth = 2.132 GB/s.*

> **Tip:** To increase bandwidth, widen the data bus — use 16, 32, or 64-bit configurations rather than running at a higher clock rate.

![MIG block diagram](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/09/image-3.png?w=544)
*The user logic connects to the left side of the MIG block; the MIG drives the DDR bus directly.*

## DDR Efficiency Considerations

The theoretical peak bandwidth is rarely achievable in practice. Actual efficiency depends on:

- **Access pattern:** Sequential burst reads/writes achieve the highest efficiency. Random access (especially read-write mixing to the same bank) incurs additional precharge and activate latency.
- **Bank conflicts:** Accessing multiple rows in the same bank forces a precharge–activate sequence between commands, reducing throughput.
- **Refresh cycles:** DDR requires periodic refresh commands that temporarily suspend access.

![DDR efficiency example](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/09/image-4.png?w=478)
*Efficiency varies significantly by access pattern. Random mixed read/write to the same bank is worst-case.* ([AMD reference](https://docs.amd.com/r/en-US/ug585-zynq-7000-SoC-TRM/Theoretical-Bandwidth))

**Design recommendation:** Profile your application's access pattern early. If random access is unavoidable, consider adding a reorder buffer in your FPGA logic to coalesce requests into sequential bursts before sending them to the MIG.

## References

- [Xilinx/AMD UltraScale Memory Interface Solutions (PG150)](https://www.xilinx.com/support/documents/ip_documentation/mig/v7_1/pg150-ultrascale-mis.pdf)
- [Zynq-7000 SoC Technical Reference Manual — Theoretical Bandwidth](https://docs.amd.com/r/en-US/ug585-zynq-7000-SoC-TRM/Theoretical-Bandwidth)
