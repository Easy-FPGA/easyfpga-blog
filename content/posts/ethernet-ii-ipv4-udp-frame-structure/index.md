---
title: "Ethernet II + IPv4 + UDP Frame Structure Reference"
date: 2025-09-22T14:39:59+00:00
draft: false
description: "A complete byte-level breakdown of an Ethernet II / IPv4 / UDP frame — the essential reference for FPGA-based network packet processing."
categories:
  - "Ethernet & Networking"
tags:
  - "Ethernet Frame"
  - "FPGA"
  - "IPv4"
  - "RTL"
  - "UDP"
  - "Packet Processing"
---

When implementing a network interface in RTL, you need a precise byte-offset map for every field in the frame. This post provides that reference for the most common combination: **Ethernet II + IPv4 + UDP**.

## Layer Stack

```
[ Preamble (7B) + SFD (1B) ]  ← handled by PHY/MAC, not in user datapath
[ Ethernet II header        ]
  [ IPv4 header             ]
    [ UDP header            ]
      [ Application data    ]
[ FCS / CRC-32 (4B)         ]  ← often stripped/added by MAC IP
                                   (configurable option in Xilinx TEMAC / Tri-MAC)
```

> **Note:** The preamble, SFD, and inter-packet gap (IPG) are inserted and stripped by the PHY/MAC layer and are typically **not visible** in the AXI-Stream user interface of a MAC IP core.

---

## Ethernet II Header (14 bytes)

Offsets are relative to the start of the Ethernet frame (byte 0 = first byte of the Destination MAC).

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0–5 | 6 B | **Destination MAC** | Broadcast = `FF:FF:FF:FF:FF:FF` |
| 6–11 | 6 B | **Source MAC** | Sender's MAC address |
| 12–13 | 2 B | **EtherType** | `0x0800` = IPv4, `0x86DD` = IPv6, `0x0806` = ARP |
| 14–… | variable | **Payload** | IPv4 packet starts here |
| last 4 B | 4 B | **FCS (CRC-32)** | Covers DA through end of payload |

---

## IPv4 Header (minimum 20 bytes)

Offsets within the IPv4 packet (i.e., from Ethernet byte offset 14).

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0 | 1 B | **Version / IHL** | Upper nibble = `4` (IPv4); lower nibble = header length in 32-bit words (usually `5` → 20 B) |
| 1 | 1 B | **DSCP / ECN** | (formerly ToS) |
| 2–3 | 2 B | **Total Length** | IPv4 header + payload in bytes |
| 4–5 | 2 B | **Identification** | Fragment ID |
| 6–7 | 2 B | **Flags + Fragment Offset** | Upper 3 bits = flags (`DF`, `MF`); lower 13 bits = fragment offset |
| 8 | 1 B | **TTL** | Decremented at each hop; drop when 0 |
| 9 | 1 B | **Protocol** | `0x11` (17) = UDP, `0x06` (6) = TCP, `0x01` = ICMP |
| 10–11 | 2 B | **Header Checksum** | One's complement of the IPv4 header only |
| 12–15 | 4 B | **Source IP** | e.g., `192.168.1.10` |
| 16–19 | 4 B | **Destination IP** | |
| 20–59 | 0–40 B | **Options** | Present only when IHL > 5 |

---

## UDP Header (8 bytes)

Offsets within the UDP segment (i.e., from Ethernet byte offset 34, assuming a 20-byte IPv4 header).

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0–1 | 2 B | **Source Port** | Ephemeral or well-known |
| 2–3 | 2 B | **Destination Port** | e.g., `0x0035` (53) = DNS |
| 4–5 | 2 B | **Length** | UDP header + data in bytes (minimum 8) |
| 6–7 | 2 B | **Checksum** | Optional in IPv4; covers pseudo-header + UDP header + data |
| 8–(8+N−1) | N B | **Data** | `N = UDP Length − 8` |

---

## Absolute Byte Offsets (No IPv4 Options)

For the common case of a 20-byte IPv4 header and no VLAN tag:

| Field | Ethernet-frame Offset |
|---|---|
| Destination MAC | 0–5 |
| Source MAC | 6–11 |
| EtherType | 12–13 |
| IPv4 Version/IHL | 14 |
| IPv4 Total Length | 16–17 |
| IPv4 Protocol | 23 |
| IPv4 Source IP | 26–29 |
| IPv4 Destination IP | 30–33 |
| UDP Source Port | 34–35 |
| UDP Destination Port | 36–37 |
| UDP Length | 38–39 |
| UDP Checksum | 40–41 |
| **UDP Payload starts** | **42** |

---

## RTL Implementation Tips

**Byte extraction in SystemVerilog (AXI-Stream, 8-bit byte lanes):**

```systemverilog
// Assuming rx_data[7:0] is valid, rx_byte_cnt tracks the current byte offset
logic [47:0] dst_mac;
logic [15:0] ethertype;
logic [7:0]  ip_protocol;
logic [15:0] udp_dst_port;

always_ff @(posedge clk) begin
    if (rx_valid && rx_ready) begin
        case (rx_byte_cnt)
            0:  dst_mac[47:40] <= rx_data;
            1:  dst_mac[39:32] <= rx_data;
            2:  dst_mac[31:24] <= rx_data;
            3:  dst_mac[23:16] <= rx_data;
            4:  dst_mac[15:8]  <= rx_data;
            5:  dst_mac[7:0]   <= rx_data;
            23: ip_protocol    <= rx_data;
            36: udp_dst_port[15:8] <= rx_data;
            37: udp_dst_port[7:0]  <= rx_data;
        endcase
    end
end
```

**FCS handling:** Most vendor MAC IP cores (Xilinx TEMAC, Tri-mode MAC) have an option to automatically append/verify FCS. When enabled, byte offsets above are correct. When disabled, subtract 4 bytes from the end of the frame to exclude FCS from payload processing.
