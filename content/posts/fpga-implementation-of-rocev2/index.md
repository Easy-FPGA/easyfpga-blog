---
title: "FPGA Implementation of RoCEv2"
date: 2025-06-17T12:21:53+00:00
draft: false
categories:
  - "Ethernet & Networking"
tags:
  - "CMAC"
  - "ERNIC"
  - "FPGA"
  - "RoCEv2"
---

## What is the RoCE v2?

RoCEv2, or RDMA over Converged Ethernet version2 is a network protocol that enable Remote Direct Memory Access (RDMA) over standard Ethernet networks using UDP/IP. It's designed to deliver high-throughput, low-latency communication while minimizing CPU usage-making it ideal for data centers, high-performance computing, and storage systems.

The link below is a lecture on RDMA. It provides a lot of useful information. [https://www.coursera.org/learn/the-fundamentals-of-rdma-programming/home/info](https://www.coursera.org/learn/the-fundamentals-of-rdma-programming/home/info)

## RoCEv2 Packet Format

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/06/image-13.png?w=867)

- Ethernet Header

Standard Ethernet frame header.

- IP Header

IPv4 or IPv6 depending on the network configuration.

- UDP Header

Destination Port: 4791 (reserved for RoCE v2)

- InfiniBand Transport Headers

BTH (Base Transport Header): Contains OpCode, packet sequence number (PSN), and other control bits.

- Optional Headers: Depending on the RDMA operation. you may see:

RETH (RDMA Extended Transport Header) for RDMA READ/WRITE

- AETH (Acknowledgement Extended Header) 

- DETH (Datagram Extended Transport Header)

- Atomic Headers for atomic operations

- Payload

The actual RDMA data being tranferred.

- ICRC (Invariant CRC)

Ensures data integrity across the RDMA transport layer.

## RoCE v2 Implementation Technologies

### 🔹 **Hardware (FPGA, NIC):**

- **Mellanox (NVIDIA) ConnectX-4** and later support RoCE v2

- **AMD/Xilinx FPGAs** can use the **ERNIC IP (RoCE Engine)** for RoCE v2 implementation

### 🔹 **Software (Linux environment):**

- **`rdma-core`** library (includes **libibverbs** and **librdmacm**)

- Use **`ibv_*` APIs** to create QPs (Queue Pairs) and perform data transfer

## FPGA Implementation

For an FPGA Implementation, You need two IP cores: a 100G MAC and an ERNIC.

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/06/image-14.png?w=1024)*Integrated CMAC Block for 100 Gb/s Ethernet*

- Integrated 100G Ethernet Subsystem [https://www.amd.com/en/products/adaptive-socs-and-fpgas/intellectual-property/cmac_usplus.html](https://www.amd.com/en/products/adaptive-socs-and-fpgas/intellectual-property/cmac_usplus.html)

Supprots CAUI-10, CAUI-4, 100GAUI-2, 100GAUI-4

- 100 Ethernet MAC, and RS-FEC logic + PCS Block

- ERNIC (Xilinx IP) [https://www.amd.com/en/products/adaptive-socs-and-fpgas/intellectual-property/ef-di-ernic.html](https://www.amd.com/en/products/adaptive-socs-and-fpgas/intellectual-property/ef-di-ernic.html)

RoCE v2 packet parsing/generation

- Memory interface for DMA

- UDP/IP encapsulation and decapsulation

- ICRC generation/check