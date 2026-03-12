---
title: "Ethernet II vs IEEE 802.3: Key Differences Explained"
date: 2025-09-22T14:42:35+00:00
draft: false
description: "The 2-byte field at offset 12 in an Ethernet frame means completely different things in Ethernet II and IEEE 802.3 — here is why it matters."
categories:
  - "Ethernet & Networking"
tags:
  - "Ethernet"
  - "Ethernet II"
  - "Ethernet version"
  - "IEEE 802.3"
  - "EtherType"
---

If you have ever looked at an Ethernet frame in Wireshark and wondered why the 2-byte field at offset 12 is sometimes labelled "EtherType" and sometimes "Length", this post explains the reason.

## 1. Ethernet II (also called DIX Ethernet)

**Origin:** Defined jointly by **DEC, Intel, and Xerox in 1980** — hence the alternative name "DIX Ethernet".

**Frame structure:**

| Field | Size | Meaning |
|---|---|---|
| Destination MAC | 6 B | Target station address |
| Source MAC | 6 B | Sending station address |
| **EtherType** | **2 B** | Upper-layer protocol identifier |
| Payload | 46–1500 B | Network-layer packet |
| FCS (CRC-32) | 4 B | Frame check sequence |

The 2-byte field carries an **EtherType value** — a number that directly identifies which protocol is encapsulated in the payload:

| EtherType | Protocol |
|---|---|
| `0x0800` | IPv4 |
| `0x86DD` | IPv6 |
| `0x0806` | ARP |
| `0x8100` | 802.1Q VLAN tag |

**Why it matters:** Because EtherType directly names the upper protocol, IP stacks can demultiplex packets with a single table lookup. No additional header is required.

**Status today:** Ethernet II is the **de-facto standard for all IPv4/IPv6 traffic worldwide**.

## 2. IEEE 802.3 Ethernet (Original Standard)

**Origin:** Standardised by the **IEEE 802.3 committee in 1983**.

**Frame structure:**

| Field | Size | Meaning |
|---|---|---|
| Destination MAC | 6 B | Target station address |
| Source MAC | 6 B | Sending station address |
| **Length** | **2 B** | Number of bytes in the MAC data field (46–1500) |
| LLC header | 3 B | 802.2 Logical Link Control (DSAP, SSAP, Control) |
| Optional SNAP | 5 B | Carries an OUI + EtherType for protocol identification |
| Payload | variable | |
| FCS (CRC-32) | 4 B | |

Because the third field is a **length** rather than a protocol ID, the frame on its own cannot tell you which upper-layer protocol the payload contains. An 802.2 LLC header (and optionally a SNAP header) must follow immediately to carry that information.

**Status today:** Largely confined to legacy IBM/Novell networks and some industrial protocols. Uncommon in modern IP networks.

## 3. Summary Comparison

| | Ethernet II (DIX) | IEEE 802.3 |
|---|---|---|
| 2-byte field meaning | EtherType — upper protocol number | Length — payload byte count |
| Upper-protocol identification | Direct (EtherType value) | Indirect (requires 802.2 LLC / SNAP) |
| Primary users | IPv4, IPv6, ARP — **global internet standard** | Legacy IBM/Novell, 802.2-based protocols |
| Max payload | 1500 B (MTU) | 1500 B (same) |
| Prevalence today | Overwhelming majority | Niche / legacy |

## 4. How to Tell Them Apart

Look at the numeric value of the 2-byte field at offset 12 in the frame:

- **≥ 0x0600 (1536 decimal):** This is an **EtherType** → Ethernet II frame.
- **≤ 0x05DC (1500 decimal):** This is a **Length** → IEEE 802.3 frame (look for LLC/SNAP after it).

**Examples:**
- `0x0800` (2048) → Ethernet II carrying IPv4
- `0x86DD` (34525) → Ethernet II carrying IPv6
- `0x05DC` (1500) or less → IEEE 802.3 + LLC

> **Practical takeaway:** Every IPv4/IPv6 packet you capture in Wireshark is encapsulated in an **Ethernet II** frame. The IEEE 802.3 variant is effectively invisible in modern IP networks.
