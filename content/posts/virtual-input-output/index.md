---
title: "Virtual Input/Output"
date: 2025-06-10T01:09:32+00:00
draft: false
categories:
  - "FPGA Design"
tags:
  - "AMD"
  - "Debug"
  - "FPGA"
  - "VIO"
  - "Virtual I/O"
---

[https://www.amd.com/en/products/adaptive-socs-and-fpgas/intellectual-property/vio.html](https://www.amd.com/en/products/adaptive-socs-and-fpgas/intellectual-property/vio.html)

When debugging an FPGA, there are times when you need to modify settings or parameters and monitor internal signals in real time. Setting up communication interfaces, such as receiving input from switches and keys or using UART, can be a complex process. However, AMD FPGAs offer a solution with the **Virtual I/O** IP, which allows these tasks to be easily performed via JTAG.

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/05/image-13.png?w=1024)*[https://docs.amd.com/v/u/en-US/pg159-vio](https://docs.amd.com/v/u/en-US/pg159-vio)*

**Xilinx Virtual Input/Output (VIO)** is a soft IP core provided by Xilinx that allows you to monitor and control internal FPGA signals in real time using the Vivado Integrated Logic Analyzer (ILA) without needing physical I/O pins.

## What is Xilinx VIO?

The **VIO core** acts like a virtual set of buttons, switches, and LEDs **inside the FPGA**. It can:

- Send values **into** your design (inputs),

- Read values **from** your design (outputs),

- Be connected to **internal signals** (not exposed to pins),

- Allow **real-time interaction** through the Vivado Hardware Manager (using JTAG).

## How Does It Work?

- The VIO core is added via the **IP Catalog** in Vivado.

- You configure the **number of inputs/outputs**, and **bit-widths**.

- Connect the VIO inputs/outputs to your internal signals in your design.

- During hardware debugging:

You can **set input values manually**.

- You can **observe output values in real-time**.

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/06/image-3.png?w=1024)*IP Generation*

```
`vio_eth i_vio_eth (
    .clk (gtx_clk),
    .probe_in0 (tx_state),
    .probe_in1 (rx_src_mac_addr),
    .probe_in2 (rx_arp_spa),
    .probe_out0 (perform_test_mode)

);`
```

IP integration 

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/06/image-4-edited.png)*VIO with Hardware manager*