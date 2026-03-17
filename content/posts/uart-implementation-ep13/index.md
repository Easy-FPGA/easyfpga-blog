---
title: "Co-Simulation"
date: 2026-03-28T12:00:00+09:00
draft: true
description: "Hardware-software co-simulation: loading compiled ELF into BRAM simulation models, writing a bit-level UART stimulus testbench, and verifying AXI bus transactions in waveforms."
categories:
  - "UART"
tags:
  - "FPGA"
  - "UART"
  - "MicroBlaze"
  - "AXI"
  - "Simulation"
  - "Xilinx"
  - "EasyFPGA"
---
# Co-Simulation

## What is Co-Simulation?

**Co-simulation** = simulating **software (C/ELF) and hardware (RTL)** together.

```
┌─────────────────────────────────────────────────────────┐
│                    Co-Simulation                         │
│                                                         │
│  MicroBlaze CPU  ──AXI──►  AXI UART Lite (RTL)         │
│  (executes .elf)               │                        │
│                                ▼                        │
│                    uart_tx ──► External TB              │
│                    uart_rx ◄── (inject stimulus)        │
└─────────────────────────────────────────────────────────┘
```

Co-simulation is the most thorough verification method before FPGA programming: it runs the compiled C application in a simulated MicroBlaze, which drives the actual RTL of the AXI UART Lite IP. Bugs in either the software or the hardware produce an incorrect waveform that can be inspected deterministically and reproduced exactly.

### Benefits
- Verify software and hardware **behave correctly together before FPGA programming**
- Debug without physical hardware
- Repeatable: deterministic simulation means bugs are consistently reproducible

## Co-Simulation Setup Flow

```
1. Vivado: Export Simulation Files
   - File → Export → Export Simulation
   - Simulator: Questa / Vivado Simulator
   - Include: .tcl, .sh, IP simulation models

2. Associate .elf file with BRAM
   - In simulation .tcl: set_property SIM_ELF_FILE {app.elf} [get_cells u_bram]

3. Write external testbench
   - Instantiate the BD wrapper
   - Drive uart_rxd with test stimulus
   - Monitor uart_txd for expected data

4. Run Simulation
   - vsim / xsim -g (GUI mode for waveforms)
   - Check timing, AXI transactions, UART waveform
```

## ELF Association with BRAM

The compiled C application (`.elf`) must be loaded into Block RAM at simulation start.

```tcl
# In simulation .tcl — associate .elf with BRAM
set_property SIM_ELF_FILE {
    d:/vitis_workspace/uart_loopback/Debug/uart_loopback.elf
} [get_cells {bd_inst/microblaze_0_local_memory/lmb_bram}]
```

This initializes the BRAM simulation model's contents to match the compiled program. MicroBlaze then begins fetching and executing instructions from address `0x0000_0000` (the BRAM base) after reset is released — exactly as it would on real hardware.

## Testbench Structure

```systemverilog
module tb_cosim;
    // DUT: Block Design wrapper
    design_1_wrapper u_bd (
        .sys_clk    (clk),
        .reset_n    (rst_n),
        .uart_rxd   (uart_rx_stim),   // our stimulus
        .uart_txd   (uart_tx_mon)     // monitor output
    );

    // Clock: 100 MHz
    initial clk = 0;
    always #5 clk = ~clk;

    // Stimulus sequence
    initial begin
        rst_n = 0; uart_rx_stim = 1;
        #200 rst_n = 1;
        #50_000;                    // Wait for MicroBlaze boot
        uart_send_byte(8'hA5);
        #10_000;
        $display("Test complete");
        $finish;
    end

    // Monitor TX
    always @(posedge uart_tx_mon_valid)
        $display("Received byte: 0x%02X", uart_tx_received_byte);
endmodule
```

## UART Bit-Level Stimulus Task

```systemverilog
// Send one UART byte: 115200 baud, 8E1
task uart_send_byte(input [7:0] data);
    integer i;
    localparam BIT_PERIOD = 8680;  // ns at 115200 bps (1/115200 × 10^9)
    // Start bit
    uart_rx_stim = 0; #BIT_PERIOD;
    // Data bits (LSB first)
    for (i = 0; i < 8; i++) begin
        uart_rx_stim = data[i]; #BIT_PERIOD;
    end
    // Even parity bit
    uart_rx_stim = ^data; #BIT_PERIOD;
    // Stop bit
    uart_rx_stim = 1; #BIT_PERIOD;
endtask
```

> **Bit period calculation**: `1 / 115200 bps × 10^9 = 8680.6 ns`. The integer approximation `8680 ns` introduces a 0.007% error per bit, accumulating to 0.077% across an 8E1 frame — negligible. The task encodes exactly the same protocol as `uart_tx` in hardware: Start(0), 8 data bits LSB-first, even parity (`^data`), Stop(1).

## Timing Considerations

| Parameter | Value | Notes |
|---|---|---|
| **Simulation time step** | 1 ns | Adequate for 115200 baud |
| **Bit period at 115200** | 8,680 ns | `#8680` delay in testbench |
| **Full frame (8E1)** | 95,480 ns | 11 bits × 8,680 ns |
| **MicroBlaze boot time** | ~50,000 ns | Processor starts executing after reset |
| **Min simulation time** | > 500 µs | Allow MB to initialize and respond |

> **Long simulation times**: MicroBlaze requires approximately 50 µs (50,000 clock cycles at 100 MHz) after reset release before executing the first instruction — BRAM initialization and cache warm-up. When a co-simulation "produces no output," check simulation time first: `run 2ms` is a safe minimum for a loopback test.

## Viewing AXI Transactions in Waveform

Add AXI bus signals to the waveform viewer:

```
Vivado Simulator / Questa:
  Design Hierarchy → bd_inst → axi_interconnect → M00_AXI
  Add: AWADDR, AWVALID, AWREADY, WDATA, WVALID, WREADY
  Add: ARADDR, ARVALID, ARREADY, RDATA, RVALID, RREADY
```

Typical AXI write transaction (MicroBlaze to UART TX_FIFO):
```
AWADDR  = 0x40600004  (TX_FIFO offset 0x04)
WDATA   = 0x000000A5  (character to send)
AWVALID = 1, AWREADY = 1, WVALID = 1, WREADY = 1
→ AXI write complete → UART IP begins serializing
```

Confirming this AXI transaction in the waveform verifies the complete chain: C code → MicroBlaze CPU → AXI bus → UART Lite register → serial output.

## Episode 13 Summary

- **Co-simulation**: runs compiled C (.elf) with RTL in the same simulator
- **ELF association**: load application binary into BRAM simulation model before sim starts
- **Testbench**: drive `uart_rxd`; monitor `uart_txd`; verify loopback echo
- **Bit-level stimulus**: `uart_send_byte` task with `#BIT_PERIOD` delay
- **AXI waveform**: confirm MicroBlaze writes to UART TX_FIFO register

### Next Episode
> **Episode 14: MicroBlaze UART Hardware Test**
> Real FPGA test — bitstream download, terminal, Vitis debugger

## Key Takeaways

- Co-simulation loads the compiled `.elf` into the BRAM simulation model so MicroBlaze executes the actual application code alongside the RTL — bugs in either show up in the same waveform
- MicroBlaze requires approximately 50 µs after reset before the first instruction executes; always run simulations for at least 500 µs to capture a complete loopback transaction
- The `uart_send_byte` testbench task encodes the full bit-level protocol: Start(0), 8 data bits LSB-first, even parity (`^data`), Stop(1) — this is the same encoding `uart_tx` uses in hardware
- AXI bus transaction waveforms (AWADDR, WDATA) confirm the complete path from C `printf` through the CPU, interconnect, and IP to the physical UART line

## Code and References

- RTL source repository: https://github.com/Easy-FPGA/easyfpga-uart-core-sv.git
- YouTube: https://www.youtube.com/@easy-fpga
