---
title: "High-Speed Serial Interface: Why the World Moved Beyond Parallel"
date: 2026-03-21T09:00:00+09:00
draft: false
description: "Explains the limitations of parallel interfaces—skew, EMI, pin count—and how high-speed serial interfaces (PCIe, SATA, USB, DDR, etc.) solve these challenges with differential signaling, clock recovery, and protocol innovation."
math: true
categories:
  - High Speed Interface
tags:
  - serial interface
  - parallel interface
  - clock recovery
  - skew
  - differential signaling
  - EMI
  - protocol
  - FPGA
  - DDR
---

# High-Speed Serial Interface

## Why Serial Interface?

### Summary of Parallel Interface Limitations
```text
Parallel (8-bit @ 1GHz)          Serial (1-bit @ 8GHz)
├─ Requires 9+ pins              ├─ Requires 2 pins (Diff Pair)
├─ Severe skew issues            ├─ No skew issues
├─ High EMI                      ├─ Low EMI (differential signal)
├─ High board complexity         ├─ Simple board
└─ Limited high-speed scaling    └─ Easy high-speed scaling
```

### Paradigm Shift
**"Multiple slow lanes" → "One fast lane"**

## Key Advantages of Serial Interface

### 1. Minimal Pin Count
```text
┌─────────────┐           ┌─────────────┐
│   TX IC     │   TX+     │   RX IC     │
│             ├──────────►│             │
│             │   TX-     │             │
│             ├──────────►│             │
└─────────────┘           └─────────────┘
```
- Communication possible with just **1 differential pair**
- Bidirectional: TX pair + RX pair = 4 pins total
- Reduced package cost, enables more channels

### 2. Skew Problem Solution
<iframe src="/high-speed-interface/Parallel-vs-Serial-Skew.html" width="100%" height="360" style="border:0;" title="Parallel vs Serial Skew animation"></iframe>

**Complexity Reduction:**
- **Parallel**: Must manage skew between 9 signals (CLK + 8 data)
  - Inter-lane skew: hundreds of ps tolerance
  - Requires length matching for all 9 traces
  - Extremely difficult at high frequencies

- **Serial**: Only manage skew within differential pair (TX+, TX-)
  - Intra-pair skew: few ps tolerance (±5ps typical)
  - Length matching for just 2 traces
  - Much simpler PCB design

- **Common-mode noise rejection** with differential signaling
- **Timing margin** drastically improved

### 3. High-Speed Operation Capability
- **Parallel**: Difficult above 1GHz
- **Serial**: 10Gbps, 25Gbps, 56Gbps possible
- Data rate = Clock frequency × Encoding efficiency

### 4. Reduced Electromagnetic Interference (EMI)
<iframe src="/high-speed-interface/Differential-Field-Animation.html" width="100%" height="360" style="border:0;" title="Differential pair field animation"></iframe>
- Differential signals reduce electromagnetic radiation
- Strong against external noise
- Easy FCC/CE certification

### 5. Simple PCB Design
- Manage only 2 traces
- Reduced length matching burden
- Can reduce layer count

## Challenges of Serial Interface

### 1. Clock Recovery
In parallel interfaces, clock is transmitted separately, but in serial, clock must be extracted from data.

{{< anim-serial-cdr >}}

**Solution**: CDR (Clock Data Recovery)
- PLL/DLL-based clock recovery
- Clock synchronization through edge detection

### 2. Maintaining DC Balance
Risk of clock recovery failure with consecutive '0's or '1's

```
Problem: 0000000000... ─────────── (no edges)
         1111111111... ────────────

Solution: 01010101... ─┐ ┌─┐ ┌─┐ ┌─ (abundant edges)
                       └─┘ └─┘ └─┘
```

**Solutions**: 
- **8b/10b encoding**: Guarantees DC balance
- **Scrambler**: Data randomization

### 3. Ensuring Data Integrity
Increased possibility of errors in high-speed transmission

**Solutions**:
- **CRC**: Error detection
- **FEC (Forward Error Correction)**: Error correction

### 4. Serialization/Deserialization
```
Parallel → Serial          Serial → Parallel
[8 bits]                   [1-bit stream]

┌──────────┐               ┌──────────┐
│Serializer│ ────────────► │Deserializ│
│          │  1-bit        │   er     │
└──────────┘               └──────────┘
```

**Solution**: SERDES (Serializer/Deserializer)

## Major Standards and Protocols

### PCI Express (PCIe)
- **Speed**: Gen1 (2.5GT/s) ~ Gen5 (32GT/s)
- **Encoding**: Gen1/2 uses 8b/10b, Gen3+ uses 128b/130b
- **Lanes**: x1, x4, x8, x16 configurations

### USB
- **USB 2.0**: 480Mbps
- **USB 3.0**: 5Gbps (8b/10b)
- **USB 3.1**: 10Gbps
- **USB4**: 40Gbps

### SATA
- **SATA 1.0**: 1.5Gbps
- **SATA 2.0**: 3Gbps
- **SATA 3.0**: 6Gbps
- **Encoding**: 8b/10b

### Ethernet
- **1000BASE-T**: 1Gbps
- **10GBASE-R**: 10Gbps (64b/66b)
- **25G/50G/100G Ethernet**: 25Gbps+

### DisplayPort / HDMI
- **DisplayPort 1.4**: 32.4Gbps (4 lanes)
- **HDMI 2.1**: 48Gbps

### Aurora, RapidIO
- Xilinx and other FPGA manufacturer high-speed serial protocols

## Serial Interface Layer Architecture

```
┌─────────────────────────────────────┐
│   Application Layer                 │  ← User data
├─────────────────────────────────────┤
│   Transaction/Link Layer            │  ← Packetization, flow control
│   - Packet framing                  │
│   - CRC                             │
│   - Flow control                    │
├─────────────────────────────────────┤
│   Physical Coding Sublayer (PCS)    │  ← Encoding/Decoding
│   - 8b/10b or 64b/66b encoding      │
│   - Scrambling                      │
├─────────────────────────────────────┤
│   Physical Media Attachment (PMA)   │  ← SERDES
│   - Serialization                   │
│   - Deserialization                 │
│   - Clock Recovery                  │
├─────────────────────────────────────┤
│   Physical Medium Dependent (PMD)   │  ← Physical layer
│   - TX Driver                       │
│   - RX Equalizer                    │
└─────────────────────────────────────┘
```

## Bandwidth Comparison

### Parallel DDR3-1600
```
64-bit × 1600MT/s = 102.4 Gbps (theoretical)
Actual: 12.8 GB/s = 102.4 Gbps
Pin count: 64 data + control/clock = 80+ pins
```

### PCIe Gen3 x16
```
16 lanes × 8Gbps = 128 Gbps
Actual: 128b/130b encoding = ~125.9 Gbps
Pin count: 16 × 2 (differential) × 2 (TX/RX) = 64 pins (data only)
```

### Efficiency Analysis
- Bandwidth per pin: PCIe significantly higher
- Scalability: PCIe easily scales by adding lanes
- Transmission distance: Serial far superior

## Key Technologies in Serial Interface Implementation

### 1. 8b/10b Encoding
- Guarantees DC balance
- Supports clock recovery
- Control code insertion

### 2. Scrambler
- Data pattern randomization
- EMI distribution
- Improved clock recovery

### 3. CRC
- Data integrity verification
- Packet-level error detection

### 4. SERDES
- Parallel ↔ Serial conversion
- High-speed I/O interface
- CDR (Clock Data Recovery)

## Design Considerations

### Signal Integrity
- **Impedance matching**: 100Ω differential impedance
- **Loss compensation**: Pre-emphasis, Equalization
- **Jitter management**: Jitter budget calculation

### Power Consumption
```
Power = C × V² × f × α

High-speed serial:
- f is high but fewer switching events
- Small capacitance
→ Overall power can be better than parallel
```

### Test and Verification
- **BERT (Bit Error Rate Test)**: Measure bit error rate
- **Eye Diagram**: Evaluate signal quality
- **Jitter analysis**: Evaluate timing accuracy

## Conclusion

High-speed serial interface has become the standard for modern high-speed communications for the following reasons:

### Advantages
✓ Fewer pins
✓ Higher data transfer rates
✓ Simpler PCB design
✓ Lower EMI
✓ Excellent scalability

### Challenges to Overcome
- Clock recovery
- DC balance
- Data integrity
- Serialization/deserialization

These challenges have been solved through technologies such as **8b/10b encoding**, **Scrambler**, **CRC**, and **SERDES**.

---
*Next: Detailed principles of 8b/10b encoding*
