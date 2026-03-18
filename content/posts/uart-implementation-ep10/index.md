---
title: "AXI UART Lite IP"
date: 2026-03-25T12:00:00+09:00
draft: true
description: "Deep dive into AXI UART Lite: register map, UART Lite vs AXI 16550, IP Catalog configuration, and both BSP driver and direct register-access C code patterns."
categories:
  - "UART"
series:
  - "UART on FPGA"
tags:
  - "FPGA"
  - "UART"
  - "MicroBlaze"
  - "AXI"
  - "Xilinx"
  - "EasyFPGA"
---
# AXI UART Lite IP

Understanding the AXI UART Lite register map is the bridge between the hardware abstraction (AXI bus) and the software abstraction (BSP driver functions). This episode covers how the IP is configured in the IP Catalog, what each register does at the bit level, and how to write C code — both using the BSP and via direct register access — to control UART from a MicroBlaze application.

## UART IP Options in Vivado

Xilinx provides two UART IP cores:

| Feature | **AXI UART Lite** | **AXI 16550** |
|---|---|---|
| **Standard** | Xilinx proprietary | 16550 compatible (industry standard) |
| **FIFO depth** | 16 bytes RX/TX | 16 bytes RX/TX (FIFO enabled) |
| **Baud Rate** | Fixed at IP config time | Runtime-adjustable via divisor register |
| **Parity** | Even/Odd/None | Even/Odd/None |
| **Interrupts** | RX data valid, TX empty | Full interrupt set |
| **Resource Usage** | Lower | Higher |
| **Recommendation** | **Most projects** | Legacy 16550 compatibility required |

> **AXI UART Lite is simpler and smaller**. Choose AXI 16550 only if your application requires runtime baud rate changes or must be compatible with existing 16550 software drivers (e.g., porting Linux drivers to bare-metal).

## AXI UART Lite Register Map

Base address (example): **0x4060_0000**

| Offset | Register | Access | Description |
|---|---|---|---|
| `0x00` | `RX_FIFO` | Read | Receive FIFO data register |
| `0x04` | `TX_FIFO` | Write | Transmit FIFO data register |
| `0x08` | `STAT_REG` | Read | Status register (full/empty/parity/frame err) |
| `0x0C` | `CTRL_REG` | R/W | Control register (RST_TX, RST_RX, EN_INTR) |

### Status Register Bits (`STAT_REG`)
| Bit | Name | Description |
|---|---|---|
| [0] | `RX_VALID` | RX FIFO has data |
| [1] | `RX_FULL` | RX FIFO full |
| [2] | `TX_EMPTY` | TX FIFO empty |
| [3] | `TX_FULL` | TX FIFO full |
| [4] | `INTR_ENABLED` | Interrupt enabled |
| [5] | `OVERRUN` | RX overrun error |
| [6] | `FRAME_ERR` | Stop bit error |
| [7] | `PARITY_ERR` | Parity error |

> **Register-mapped I/O in practice**: The CPU accesses UART hardware by reading and writing memory addresses. Polling `RX_VALID` at bit[0] of `STAT_REG` is the C pattern: `while (!(Xil_In32(UART_BASE + 0x08) & 0x01));` — identical in concept to reading a status register on any embedded MCU. The FPGA fabric simply decodes the AXI address and connects it to the IP's internal register file.

## Adding the IP in Vivado

```
Vivado → Block Design view:
1. Press '+' (Add IP) → search "AXI UART Lite"
2. Double-click to add
3. Customize IP dialog:
   - Baud Rate: 115200
   - Data Bits: 8
   - Use Parity: Yes (Even)
   - Click OK
4. Run Connection Automation
   - Connects S_AXI to AXI4-Lite Interconnect
   - Connects uart_rtl to board interface (if defined)
5. Assign Address: 0x4060_0000 in Address Editor
```

## AXI UART Lite C API (BSP)

The Vitis BSP provides `XUartLite` driver functions:

```c
#include "xuartlite.h"
#include "xparameters.h"

// Initialization
XUartLite uart;
XUartLite_Initialize(&uart, XPAR_UARTLITE_0_DEVICE_ID);

// Send a single byte
XUartLite_SendByte(XPAR_UARTLITE_0_BASEADDR, 'H');

// Receive a single byte (blocking)
u8 data = XUartLite_RecvByte(XPAR_UARTLITE_0_BASEADDR);

// Send a string
u8 msg[] = "Hello FPGA\n";
XUartLite_Send(&uart, msg, sizeof(msg));
```

## Direct Register Access (Alternative)

For learning or ultra-low overhead:

```c
#include "xparameters.h"
#include "xil_io.h"

#define UART_BASE  XPAR_UARTLITE_0_BASEADDR
#define TX_FIFO    0x04
#define STAT_REG   0x08
#define TX_FULL    (1 << 3)
#define RX_VALID   (1 << 0)

// Poll-send one byte
void uart_send(u8 ch) {
    while (Xil_In32(UART_BASE + STAT_REG) & TX_FULL);  // Wait if TX full
    Xil_Out32(UART_BASE + TX_FIFO, ch);
}

// Poll-receive one byte
u8 uart_recv(void) {
    while (!(Xil_In32(UART_BASE + STAT_REG) & RX_VALID));  // Wait for data
    return (u8)Xil_In32(UART_BASE);
}
```

> **Direct register access vs BSP functions**: The BSP functions are convenient and portable. Direct register access is educational — it makes the hardware interface explicit and helps you understand what the BSP is doing internally. For production code, use the BSP; for debugging a hardware/software integration issue, direct access eliminates the driver layer as a variable.

## Episode 10 Summary

- **AXI UART Lite**: lighter, simpler, recommended for most designs
- **AXI 16550**: legacy-compatible, runtime baud rate adjustment
- **Register map**: RX_FIFO(0x00), TX_FIFO(0x04), STAT(0x08), CTRL(0x0C)
- **BSP functions**: `XUartLite_SendByte`, `XUartLite_RecvByte`, `XUartLite_Send`
- **Direct register access**: `Xil_In32` / `Xil_Out32` for low-overhead control

### Next Episode
> **Episode 11: Block Design Construction**
> MicroBlaze + UART Lite — complete Vivado Block Design walkthrough

## Key Takeaways

- AXI UART Lite has 4 registers at fixed offsets (0x00–0x0C); knowing the register map lets you write UART drivers and debug hardware/software integration without relying on BSP functions
- `STAT_REG[0]` (`RX_VALID`) is the polling check before reading; `STAT_REG[3]` (`TX_FULL`) prevents TX overrun — always check these flags before accessing the FIFOs
- The base address in C code must match the assignment in the Vivado Address Editor — a mismatch produces silent failure (writes go nowhere, reads return 0)
- BSP driver functions are preferred for production code; direct `Xil_In32`/`Xil_Out32` is the right tool when debugging whether an AXI address mapping is correct

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga
