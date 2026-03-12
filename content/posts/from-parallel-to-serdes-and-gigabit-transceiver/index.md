---
title: "From Parallel Bus to High-Speed SERDES and Gigabit Transceivers"
date: 2025-08-21T02:41:58+00:00
draft: false
description: "Why parallel buses hit a wall at gigabit speeds, how SERDES solves the problem, and a look at AMD's GTH, GTY, and GTM transceivers."
categories:
  - "High Speed Interface"
tags:
  - "FPGA"
  - "Gigabit Transceiver"
  - "High Speed Communication"
  - "LVDS"
  - "SERDES"
  - "CDR"
  - "PAM4"
---

## The Limits of Parallel Buses

Traditional parallel data buses were efficient at low clock rates, but as clock frequencies increased they ran into fundamental physical barriers. The core problem is **timing skew**: with many parallel data lanes plus a separate clock lane, each signal travels a slightly different path length on the PCB and through the package, arriving at the receiver at slightly different times. As the clock period shrinks, even a small skew becomes a significant fraction of a bit period, and data errors follow.

Beyond skew, parallel buses demand large numbers of pins, wide PCB routing, and significant board area. The combination of tight timing requirements and high pin counts made parallel interfaces increasingly impractical above a few hundred megabits per second per signal.

## LVDS: A Step Forward, but Still Parallel

LVDS (Low-Voltage Differential Signaling) was widely adopted in the mid-1990s as a physical-layer standard that reduced power consumption and EMI compared to single-ended TTL/CMOS. By sending a signal as a differential pair — a small voltage swing between two complementary wires — LVDS achieved better noise immunity and higher speeds.

However, LVDS retained the fundamental weakness of parallel buses and hit its own ceiling:

- **Speed limit:** The LVDS standard specifies a maximum of **655 Mbps per lane**. Some vendors pushed proprietary extensions to ~3 Gbps, but that still falls far short of the 10+ Gbps needed by modern protocols.
- **Skew across lanes:** To reach 1+ Gbps aggregate bandwidth, you need many parallel LVDS lanes plus a separate clock lane — which reintroduces exactly the skew problem SERDES was designed to eliminate.
- **Long-distance limitations:** LVDS struggles with 8K video resolutions and transmission distances beyond a few metres, where cable skew and power loss degrade signal integrity.

In short, LVDS overcame the limitations of legacy parallel interfaces but could not scale to the multi-gigabit world. The industry needed a technology that collapsed clock and data into a single serial stream.

## SERDES: Solving Skew at the Root

SERDES (Serializer/Deserializer) converts wide parallel data into a single high-speed differential serial stream, transmits it over one differential lane, and reconstructs the parallel data at the far end. Because there is only one signal lane per link, skew between clock and data is eliminated by design.

**Architecture overview:**

**Transmit side (Serializer):**
- Encoder — adds overhead bits needed for clock recovery
- Multiplexer — combines N parallel bits (e.g., 64 bits) into a single serial bitstream
- Differential driver — launches the signal onto the transmission medium

**Receive side (Deserializer):**
- CDR (Clock and Data Recovery) — reconstructs the receive clock from the incoming bitstream
- Sampler — latches each bit at the precisely recovered clock edge
- Demultiplexer — splits the serial stream back into N parallel words
- Decoder — strips overhead bits and delivers clean user data

The CDR circuit is the most critical block in the receiver. Because no separate clock is sent, the receiver must extract timing information from the data transitions themselves. The encoder on the transmit side adds controlled overhead bits (e.g., via 8B/10B or 128B/130B line coding, or scrambling) to guarantee sufficient edge density for the CDR.

**CDR implementation approaches:**

| Method | How it works | Trade-offs |
|---|---|---|
| Oversampling | Samples each bit N×, majority vote selects the correct value | Requires very high-frequency internal clock; high power |
| PLL-based | Phase detector measures phase error between incoming edges and VCO; loop adjusts VCO phase | Low overhead; dominant method in modern multi-Gbps SerDes |

## AMD Gigabit Transceivers: Hardware Engines

AMD (formerly Xilinx) embeds hard-IP multi-gigabit transceiver (MGT) blocks directly into its FPGAs. These blocks are fully self-contained serial I/O engines — they handle equalization, CDR, serialization, and protocol encoding in silicon, freeing the FPGA fabric for user logic.

### GTH and GTY Transceivers

**GTH** is found in 7-Series and UltraScale FPGAs (Kintex, Artix families). It supports up to **16.3 Gbps** per lane and is well suited for 10G Ethernet, PCIe Gen3, CPRI, JESD204B, and similar protocols.

**GTY** is the high-speed MGT in UltraScale and UltraScale+ devices (Virtex, Kintex UltraScale+). It extends the line rate to **32.75 Gbps** per lane, supporting 25G Ethernet, 100G Ethernet (4×25G), and high-bandwidth interconnects.

Both GTH and GTY use NRZ (Non-Return-to-Zero) signaling, where each bit is encoded as one of two voltage levels.

### GTM Transceivers

**GTM** is the flagship transceiver in AMD's Versal Adaptive SoC family. Built on a 7 nm process, it operates from **9.5 Gbps to 112 Gbps** per lane and introduces **PAM4 (Pulse-Amplitude Modulation, 4-level)** signaling alongside NRZ.

PAM4 encodes 2 bits per symbol using four distinct voltage levels, doubling the effective data rate for a given baud rate. This is essential for 100G and 400G Ethernet and next-generation data centre interconnects where NRZ has reached its practical signaling limit.

| Transceiver | FPGA Family | Max Line Rate | Modulation | Typical Protocols |
|---|---|---|---|---|
| GTH | 7-Series, UltraScale | 16.3 Gbps | NRZ | 10G Eth, PCIe Gen3, CPRI |
| GTY | UltraScale+ | 32.75 Gbps | NRZ | 25G/100G Eth, PCIe Gen4 |
| GTM | Versal | 112 Gbps | NRZ / PAM4 | 100G/400G Eth, PCIe Gen5 |

## Reference

- AMD UltraScale GTM Transceivers User Guide: [UG581](https://docs.amd.com/v/u/en-US/ug581-ultrascale-gtm-transceivers)
