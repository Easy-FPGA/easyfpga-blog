---
title: "AXI with FPGA and SoC"
date: 2025-05-27T05:10:47+00:00
draft: false
description: "AXI, or Advanced eXtensible Interface, is part of the AMBA protocol by ARM, designed for high-performance system designs in FPGAs and SoCs. It include"
categories:
  - "FPGA Design"
tags:
  - "AMD"
  - "AXI"
  - "FPGA"
  - "Xilinx"
---

AXI stands for Advanced eXtensible Interface. It is a part of the AMBA (Advanced Microcontroller Bus Architecture) protocol family, developed by ARM. AXI is designed for high-performance, high-frequency system designs and is **widely adopted in FPGAs and SoCs.**

- Zynq SoC connects between Processing System (PS) and Programmable Logic (PL)

- Interconnection between Xilinx IP cores (e.g., Block RAM, DMA, TEMAC)

- Communication with MicroBlaze processor

- AXI Interconnect to connect multiple master/slave devices

## Types of AXI Interfaces

TypeDescriptionUse Case**AXI4**Full-featured, supports burst transfersMemory-mapped interfaces (e.g., DDR)**AXI4-Lite**Lightweight, no burst supportControl and configuration registers**AXI4-Stream**No address phase, continuous data flowHigh-speed streaming (video, 
audio)

When working with Xilinx IP cores, you'll often use one or more of these AXI interfaces, depending on your system needs.

## The AXI Channels

AXI4 and AXI4-Lite interfaces are composed of the following  independent channels:

- Write Address Channel (AW*)

- Write Data Channel (W*)

- Write Response Channel (B*)

- Read Address Channel (AR*)

- Read Data Channel (R*)

- Read Response Channel (RRESP)

Each channel uses VALID/READY handshake signals to transfer data reliably. This enables pipelining and parallelism, essential for high-performance designs.

## AXI4 (Memory Mapped)

AXI4 Memory-Mapped (AXI4-MM) is designed for high-performance memory-mapped communication between a master (e.g., CPU, DMA) and a slave (e.g., memory, custom IP). 

SignalDirectionDescription`awaddr`Master → SlaveWrite address`awvalid`Master → SlaveWrite address valid`awready`Slave → MasterWrite address accepted`wdata`Master → SlaveWrite data`wvalid`Master → SlaveWrite data valid`wready`Slave → MasterWrite data accepted`bresp`Slave → MasterWrite response (OKAY, SLVERR, etc)`bvalid`Slave → MasterWrite response valid`bready`Master → SlaveMaster ready to receive response`araddr`Master → SlaveRead address`arvalid`Master → SlaveRead address valid`arready`Slave → MasterRead address accepted`rdata`Slave → MasterRead data`rvalid`Slave → MasterRead data valid`rready`Master → SlaveMaster ready to accept read data`rresp`Slave → MasterRead response (OKAY, SLVERR, etc)

### Write Transaction

- Master sends write address over AW channel

- Master sends write data over W channel

- Slave send write response over B channel

### Read Transaction

- Master sends read address over AR channel

- Slave send read data the response over R channel 

### Key Features

FeatureDescription**Address-based**All data transactions use memory addresses**Supports bursts**Up to 256 data beats per burst**Separate channels**Independent channels for address, data, and responses**Out-of-order support**Responses can come back out-of-order

![AXI4-MM Simulation Write Transaction](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-7.png?w=1024)*Simulation Waveform of AXI4-MM Write Transaction*

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-8.png?w=1024)*AXI4-MM Write Channel*

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-9.png?w=1024)*AXI4-MM Write Response*

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-10.png?w=1024)*Simulation Waveform of AXI4-MM Read Transaction*

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-11.png?w=1024)*AXI4-MM Read Address Channel*

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-12.png?w=870)*AXI4-MM Read Response*

## AXI4-Lite

AXI4-Lite is a lightweight version of the AXI4 protocol, used for low-throughput control interfaces. It is typically used for accessing control and status registers in IP cores.

### Key Features

- No burst, only single data transfers

- Lower logic and routing resource usage

- Ideal for control/status register access

- Master (e.g., CPU) configures slave IP core

FeatureAXI4-MMAXI4-LiteBurst Transfers✅ Supported❌ Not supportedMax Transfer SizeUp to 256 beats1 beatComplexityHigherLowerUse CaseHigh data transferControl registers

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-4.png?w=1024)*Simulation waveform of AXI4-Lite Write Transaction*

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-5.png?w=1024)*Simulation waveform of AXI4-Lite Read Transcation*

## AXI Stream

AXI4-Stream is a simplified AXI protocol designed to transmit raw data without addressing. It is optimized for high-throughput, continuous data transfer, such as video, audio, or network packets.

### Features

- No address channel-simple interface

- Handshake using TVALID and TREADY

- Pipelined structure for high speed

- Supports TKEEP, TLAST, TUSER for flexibility

SignalDescriptionTVALIDSender indicates valid dataTREADYReceiver is ready to accept dataTDATAActual data being transmittedTLASTMarks the end of a frame or packetTKEEPByte qualifier for which bytes in TDATA are validTUSEROptional user-defined signal*AXI4-Stream Key Signals*

When TVALID is high and TREADY is high, a data transfer occurs on the TDATA bus. This handshake ensures safe and synchronized data transmission.

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-1.png?w=1023)*The TVALID and TDATA wait until TREADY is high.*

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-3.png?w=1024)*TLAST Marks the end of a frame packet*

AXI is the de facto standard interface in modern FPGA and SoC designs. A good understanding of AXI is essential for efficient and scalable hardware design.

Below is the link to the AXI website: [https://www.arm.com/architecture/system-architectures/amba/amba-specifications](https://www.arm.com/architecture/system-architectures/amba/amba-specifications)