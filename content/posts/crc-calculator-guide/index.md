---
title: "CRC Calculator & RTL Generator: Complete Guide"
date: 2026-03-13T00:00:00+00:00
draft: false
categories:
  - "RTL Design"
  - "FPGA"
tags:
  - "CRC"
  - "CRC-32"
  - "SystemVerilog"
  - "FPGA"
  - "RTL"
  - "Error Detection"
---

CRC (Cyclic Redundancy Check) is the most widely used error-detection mechanism in digital communication — from Ethernet frames to USB packets to SSD data integrity. This post explains how CRC works mathematically and shows how to use the [free online CRC Calculator & RTL Generator](/tools/crc-calculator/) to produce ready-to-simulate SystemVerilog code for your FPGA design.

---

## What Is CRC?

CRC appends a checksum to a data block so the receiver can detect corruption. The transmitter computes a CRC value over the data and appends it; the receiver recomputes the CRC and compares. A mismatch means a transmission error occurred.

### Why CRC Instead of a Simple Sum?

The simplest error-detection method is summing all bytes. The problem: if two bits flip simultaneously in opposing directions, the total sum is unchanged and the error goes undetected — especially with **burst errors**.

CRC is based on **polynomial division over GF(2)** and is guaranteed to detect:
- All single-bit errors
- All burst errors up to the CRC width (e.g., any 32-bit burst with CRC-32)
- Most longer random errors with very high probability

---

## Mathematical Foundation

### Polynomial Arithmetic over GF(2)

CRC treats a bit stream as a polynomial with binary coefficients. In GF(2):

- Addition = XOR (`1+1=0`, `1+0=1`)
- Multiplication = AND (`1×1=1`, `1×0=0`)
- Subtraction = XOR (same as addition)

The bit string `11010011` represents:

$$D(x) = x^7 + x^6 + x^4 + x^1 + x^0$$

CRC is the **remainder** when the data polynomial $D(x)$ is divided by a fixed **generator polynomial** $G(x)$:

$$\text{CRC} = D(x) \cdot x^r \bmod G(x)$$

where $r$ is the CRC width in bits.

### Generator Polynomial

Each CRC standard defines its own generator polynomial. For CRC-32:

$$G(x) = x^{32} + x^{26} + x^{23} + x^{22} + x^{16} + x^{12} + x^{11} + x^{10} + x^8 + x^7 + x^5 + x^4 + x^2 + x + 1$$

In hexadecimal (dropping the implicit leading $x^{32}$ term): `0x04C11DB7`.

---

## CRC Parameters Explained

Five parameters fully define the behavior of any CRC standard:

| Parameter | Description | CRC-32 Value |
|-----------|-------------|--------------|
| **Width** | CRC output bit width | 32 |
| **Poly** | Generator polynomial (MSB implicit) | `0x04C11DB7` |
| **Init** | Initial value loaded into the CRC register | `0xFFFFFFFF` |
| **RefIn** | Reflect (bit-reverse) each input byte before processing | `true` |
| **RefOut** | Reflect the final CRC register before output | `true` |
| **XorOut** | XOR mask applied to the final output | `0xFFFFFFFF` |

### Why RefIn / RefOut?

In serial hardware, data is often transmitted LSB-first. Setting RefIn=true reverses the bit order of each input byte before feeding it through the CRC shift register, aligning the hardware shift direction with the software calculation. RefOut=true reflects the final register value before outputting — required by standards like Ethernet that also process bits LSB-first.

Ethernet (CRC-32) uses RefIn=RefOut=true. CRC-16/CCITT uses RefIn=RefOut=false. **Getting these wrong produces a completely different result**, so always verify against a known test vector.

---

## Common CRC Standards

| Name | Width | Poly | Init | XorOut | RefIn | RefOut | Used In |
|------|-------|------|------|--------|-------|--------|---------|
| CRC-32 | 32 | `04C11DB7` | `FFFFFFFF` | `FFFFFFFF` | ✓ | ✓ | Ethernet, ZIP, PNG |
| CRC-32C | 32 | `1EDC6F41` | `FFFFFFFF` | `FFFFFFFF` | ✓ | ✓ | iSCSI, SCTP |
| CRC-16/IBM | 16 | `8005` | `0000` | `0000` | ✓ | ✓ | USB, ARC |
| CRC-16/CCITT | 16 | `1021` | `FFFF` | `0000` | ✗ | ✗ | HDLC, X.25, XModem |
| CRC-16/MODBUS | 16 | `8005` | `FFFF` | `0000` | ✓ | ✓ | Modbus RTU |
| CRC-8 | 8 | `07` | `00` | `00` | ✗ | ✗ | 1-Wire, SMBus |

---

## Using the Online Tool

👉 **[Open CRC Calculator & RTL Generator](/tools/crc-calculator/)**

### Step 1 — Select a Preset

Click one of the preset buttons at the top (e.g., **CRC-32 (Ethernet)**). All parameters fill in automatically. For a non-standard CRC, click **Custom** and enter the Poly, Init, and XorOut values manually.

### Step 2 — Enter Input Data

Three input modes are supported:

**HEX mode** (default) — enter bytes as hex tokens, read left-to-right:
```
31 32 33 34 35 36 37 38 39    ← "123456789" as ASCII hex bytes
```
A token longer than 2 hex digits is split into bytes from the left. Example: `78563412` → bytes `[0x78, 0x56, 0x34, 0x12]`.

**ASCII mode** — type text directly:
```
123456789
```

**Binary mode** — 8-bit groups separated by spaces:
```
00110001 00110010 00110011
```

### Step 3 — Calculate

Click **Calculate**. The result appears in HEX, Decimal, Binary, and Verilog literal format. Any value can be copied to clipboard with one click.

**Verification**: The CRC-32 of the ASCII string `123456789` must be `0xCBF43926`. This is the official test vector for CRC-32. The **Self-Test** section at the bottom of the tool automatically verifies all six standard presets against their known vectors.

### Step 4 — Generate RTL

Scroll to the **RTL Generator** card:

1. Set **Data Width** — how many bits per clock your FPGA design processes (8 / 16 / 32 / 64)
2. **Module Name** is auto-populated (e.g., `crc32_w32`)
3. Click **⚡ Generate RTL + Testbench**

Two tabs appear: **RTL** (the module) and **Testbench** (self-checking simulation). Use **📋 Copy** or **⬇ Download** to save the `.sv` files.

---

## Understanding the Generated RTL

The output module follows this structure:

```systemverilog
module crc32_w32 (
    input  logic        clk,
    input  logic        resetn,       // Active-low synchronous reset
    input  logic [31:0] data_in,      // Input data word (one per clock)
    output logic [31:0] crc_out,      // Registered CRC state
    output logic [31:0] crc_final     // crc_out ^ XOR_OUT (final CRC value)
);
    logic [31:0] crc_reg;
    logic [31:0] crc_next;

    // Parallel CRC combinational logic (unrolled GF(2) matrix)
    always_comb begin
        crc_next[0] = crc_reg[1] ^ crc_reg[2] ^ ... ^ data_in[0] ^ ...;
        // one line per output bit
    end

    // CRC state register
    always_ff @(posedge clk) begin
        if (!resetn) crc_reg <= 32'hFFFFFFFF;   // Init value
        else         crc_reg <= crc_next;
    end

    assign crc_out   = crc_reg;                 // (bit-reversed if RefOut=true)
    assign crc_final = crc_out ^ 32'hFFFFFFFF;  // XorOut applied
endmodule
```

### The Key Idea: Parallel CRC via GF(2) Matrix

A bit-serial CRC shift register processes one bit per clock. For FPGA designs that must handle 32 or 64 bits per clock, we derive a **linear transformation matrix** that maps the current CRC state $\mathbf{c}$ and input data $\mathbf{d}$ to the next state in a single combinational stage:

$$\mathbf{c}_{\text{next}} = \mathbf{A} \cdot \mathbf{c} \oplus \mathbf{B} \cdot \mathbf{d}$$

where $\mathbf{A}$ encodes which CRC bits feed each output bit, and $\mathbf{B}$ encodes which data bits feed each output bit. Since all matrix entries are 0 or 1, the matrix-vector product becomes a tree of XOR gates. The RTL generator pre-computes these matrices and unrolls each output bit equation — resulting in a **single-cycle combinational stage** regardless of data width.

### Port Reference

| Port | Direction | Description |
|------|-----------|-------------|
| `clk` | input | Clock, sampled on rising edge |
| `resetn` | input | Active-low synchronous reset — loads `Init` value into `crc_reg` |
| `data_in` | input | Input data word (`Data Width` bits wide) |
| `crc_out` | output | Registered CRC state; valid one clock after each `data_in` |
| `crc_final` | output | `crc_out ^ XorOut`; present only when `XorOut ≠ 0` |

### Processing a Continuous Stream

Apply successive data words without asserting reset between them. `crc_final` holds the running CRC one cycle after each word.

```
clk:       __|‾|_|‾|_|‾|_|‾|_|‾|_
resetn:    __|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|_
data_in:   X | W0 | W1 | W2 | W3 | X
crc_final: X |  X | C1 | C2 | C3 | FINAL
```

To start a new CRC frame, assert `resetn` for one clock, then deassert.

---

## Running the Simulation

The generated testbench runs two self-checking tests:

| Test | What It Checks |
|------|----------------|
| **Single word** | CRC after processing one data word |
| **4-word chain** | CRC after processing four consecutive words without reset |

### Icarus Verilog (free)

```bash
iverilog -g2012 -o sim crc32_w32_tb.sv crc32_w32.sv && vvp sim
```

### Vivado xsim

```bash
xvlog --sv crc32_w32.sv crc32_w32_tb.sv
xelab -debug typical crc32_w32_tb
xsim crc32_w32_tb --runall
```

Expected output:

```
============================================================
CRC RTL Builder — Testbench: crc32_w32
Algorithm : CRC-32 (Ethernet/802.3)
============================================================
PASS: Single word                  | 0x...
PASS: 4-word chain                 | 0x...
============================================================
2 PASSED  0 FAILED
ALL TESTS PASSED ✓
```

---

## FAQ

**Q: My result doesn't match another CRC tool — why?**  
Check RefIn, RefOut, Init, and XorOut first. RefIn/RefOut differences alone will produce a completely different result. Use the **Self-Test** section to compare against six official known-good vectors instantly.

**Q: How do I choose Data Width?**  
Match the AXI-Stream `tdata` width of your design. 1GbE commonly uses 8 or 32 bits; 10GbE uses 64 bits; 100GbE up to 512 bits. Wider data width generates more XOR gates but the timing path always spans a single combinational stage.

**Q: Which output should I use — `crc_out` or `crc_final`?**  
If `XorOut ≠ 0` (CRC-32, CRC-32C, etc.) use `crc_final`. If `XorOut = 0` (CRC-16/IBM, CRC-8, etc.) `crc_out` is the final value and `crc_final` is not generated.

**Q: Can I generate a custom-width CRC, like CRC-24?**  
Yes. Select **Custom**, set Width to 24, and enter the Poly, Init, and XorOut for your target standard.

---

## Wrapping Up

CRC is a critical block in any FPGA communication interface design, but the combination of parameters and the parallel-implementation math can be tricky to get right. The [EasyFPGA CRC Calculator](/tools/crc-calculator/) lets you validate parameters instantly against known test vectors and generate synthesizable, self-verifying SystemVerilog in seconds — no installation required.
