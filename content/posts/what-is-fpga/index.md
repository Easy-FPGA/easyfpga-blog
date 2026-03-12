---
title: "What is FPGA?"
date: 2025-03-19T04:07:35+00:00
draft: false
description: "The content explains the concept of Field Programmable Gate Arrays (FPGA) using a breadboard analogy for prototyping. It details how components on a b"
categories:
  - "FPGA Basics"
tags:
  - "FPGA"
  - "HDL"
---

FPGA: Field Programmable Gate Array

### To take the Bread Board as an example…

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-14.png?w=1024)*Bread Board ([https://en.wikipedia.org/wiki/Breadboard#/media/File:Ultrasound-PreAmp-Breadboard.jpg](https://en.wikipedia.org/wiki/Breadboard#/media/File:Ultrasound-PreAmp-Breadboard.jpg))*

The photo above shows a breadboard. Breadboards are used to temporarily test prototypes. They usually have strips on both sides to extend power lines. The interior is divided into abcde and fghij sections where components or wires can be connected. This allows you to implement desired functions and test them with frequent changes.

You can think of the components plugged into the breadboard as modules designed in **HDL**. The wires connecting them can be viewed as wire declarations in HDL. The placement of the components is similar to the **Placement** process in FPGA Implementation. Connecting the wires is akin to the **Routing** process.

### Hardware described in HDL

You can think of FPGA as describing the process of connecting ICs on a breadboard. It uses wires and HDL (Hardware Description Language). The difference, though, is that FPGAs already have many CLBs (Configuration Logic Blocks) and Interconnect resources inside. They implement functions by programming these resources.

In addition, several components are available. These include Input/output block circuitry (IOBs) and Clock Management Tiles (CMTs). There is also Block RAM and DSP slices. Additionally, there are Gigabit Transceivers that are capable of high-speed serial data transmission. SoC products with ARM CPUs built-in are included too. Although there are various internal blocks, you can think of them as basically consisting of programmable blocks and routing resources.

You can check the content related to the FPGA structure in the links below.

[https://docs.amd.com/r/en-US/ug1291-viv/FPGA-Architecture](https://docs.amd.com/r/en-US/ug1291-viv/FPGA-Architecture)

[https://docs.amd.com/r/2024.1-English/ug1393-vitis-application-acceleration/Understanding-an-FPGA-Architecture](https://docs.amd.com/r/2024.1-English/ug1393-vitis-application-acceleration/Understanding-an-FPGA-Architecture)

[https://docs.amd.com/r/en-US/ug574-ultrascale-clb/Differences-from-Previous-Generations](https://docs.amd.com/r/en-US/ug574-ultrascale-clb/Differences-from-Previous-Generations)