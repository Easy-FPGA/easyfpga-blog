---
title: "FPGA product lines"
date: 2025-03-19T04:30:17+00:00
draft: false
description: "AMD's FPGA product families, categorized into UltraScale+, UltraScale, and 7 Series, utilize varying semiconductor processes (16nm, 20nm, and 28nm res"
categories:
  - "FPGA Basics"
tags:
  - "AMD"
  - "Artix"
  - "FPGA"
  - "Kintex"
  - "Portfolio"
  - "Spartan"
  - "UltraScale+"
  - "Xilinx"
---

Let's explore the AMD (Xilinx) FPGA product families for reference

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-19.png?w=1024)*[https://www.amd.com/en/products/adaptive-socs-and-fpgas/fpga.html](https://www.amd.com/en/products/adaptive-socs-and-fpgas/fpga.html)*

Looking at the Portfolio on the AMD FPGA (Xilinx) website, you'll notice it is divided into three categories. These categories are UltraScale+, UltraScale, and 7 Series at the top. This categorization of product families is based on semiconductor process technology. UltraScale+ is manufactured on the 16nm FinFET process. UltraScale is manufactured on the 20nm process. 7 Series is manufactured on the 28nm process.

Finer process technologies offer several advantages. They provide higher chip performance. They also allow the configuration of more logic cells in the same area and result in lower power consumption.

**FPGA Family****Target Application****Key Features****Zynq ****UltraScale****+ ****MPSoC**Integrated ARM Cortex-A53 + FPGAEmbedded systems, AI/Edge computing**Spartan ****UltraScale****+**Low-cost, simple logic processingBasic FPGA logic, minimal DSP & memory**Artix UltraScale+**Low power, cost-effectiveIndustrial controllers, sensor interfaces**Kintex UltraScale+**Balanced performance & costVideo processing, wireless communication, 5G**Virtex UltraScale+**Highest performance, high-speed interfacesData centers, networking, AI acceleration**Versal ACAP**AI engines, highest processing powerMachine learning, 6G, high-speed data acceleration

Even within the same Series, product families are divided into Spartan, Artix, Kintex, and Virtex. Each product family has a different Target Application based on price and performance. In other words, there are differences in internal architecture optimization (e.g., number of LUTs, DSP blocks, transceiver speed, memory interface).

Spartan and Artix can be seen as low-cost, high-efficiency chips, while Kintex and Virtex require mid-to-high-level performance and high costs.

**Feature****Spartan UltraScale+****Artix UltraScale+****Kintex UltraScale+****Virtex UltraScale+****Logic Cells (LUT Count)**Up to 218KUp to 308KUp to 1,843KUp to 3,780K**DSP Blocks**Up to 384Up to 1,200Up to 3,528Up to 12,288**Block RAM (BRAM)**Very SmallSmallLargerLargest (HBM Support)**Transceiver Speed**Up to 16.3GbpsUp to 16.3GbpsUp to 32.75GbpsUp to 58Gbps**PCIe Support**Up to Gen4x8 or Gen4x4Up to Gen4x4 or Gen3x8Up to Gen4x8 or Gen3x16Up to Gen3x16, Gen4x8**Ethernet Support**No high-speed EthernetUp to 10GUp to 100GUp to 100G**DDR Support**DDR4 (up to 2,400Mbps)DDR4 (up to 2,133Mbps)DDR4 (up to 2,666Mbps)DDR4/HBM (up to 4,266Mbps)

The table above compares the detailed specifications of product families within UltraScale+. The Spartan family was released later compared to other product families. Some of its features also outperform Artix in certain areas.

(AMD states that this product family was released in anticipation of the explosive growth in sensors and connected devices like robots)

Product families are mainly divided from Spartan to Virtex. This division is based on the amount of Logic Cells. It also depends on the internal memory blocks and the supported speed of Transceivers.

The link below provides a selection guide for UltraScale+ products. You can check the detailed specifications differences of internal blocks there.

[https://docs.amd.com/v/u/en-US/ultrascale-plus-fpga-product-selection-guide](https://docs.amd.com/v/u/en-US/ultrascale-plus-fpga-product-selection-guide) UltraScale+ product selection guide