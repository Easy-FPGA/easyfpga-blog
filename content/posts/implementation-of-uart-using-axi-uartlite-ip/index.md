---
title: "Implementation of UART with Xilinx FPGA and AXI Uartlite IP"
date: 2025-03-19T21:25:52+00:00
draft: false
description: "UART (Universal Asynchronous Receiver/Transmitter) is a serial communication protocol that enables asynchronous data transmission between devices with"
categories:
  - "UART"
tags:
  - "AMD"
  - "AXIUartlite"
  - "FPGA"
  - "MicroBlaze"
  - "UART"
  - "Verilog"
  - "Xilinx"
---

Overview

**What is UART?**

UART (Universal Asynchronous Receiver/Transmitter) is a hardware communication protocol used for serial communication between two devices. It is called asynchronous because it does not require a separate clock signal like SPI or I2C. Instead, both devices agree on a common data rate (**baud rate**) to ensure correct data transmission.

### **How Does UART Work?**

UART consists of two main data lines:

- TX (Transmitter) Line – Sends data.

- RX (Receiver) Line – Receives data.

Data is transmitted bit by bit in a frame format, which includes:

- Start Bit (1 bit)

- Data Bits (5 to 9 bits)

- Parity Bit (Optional, 1 bits)

- Stop Bit (1 or 2 bits)

Since UART is asynchronous, both devices must be configured with the same baud rate (e.g., 9600, 115200 bps).

StartD0D1D2D3D4D5D6D7Stop0101100101

- Start Bit = Always 0 (Indicated beginning of transmission).

- Data Bit = Actual data to be transmitted.

- Parity Bit = Optional error-checking mechanism.

- Stop Bit = Always 1 (Marks the end of the frame).

### **Key Features of UART**

- Full-duplex communication (TX and RX operate simultaneously).

- Baud rate must match on both sender and receiver.

- No clock signal needed (unlike SPI/I2C).

- Error detection using parity bits and stop bits.

### **Advantages and Disadvantages of UART**

advantages:

- simple and widely used.

- Requires only two wires (TX, RX).

- No clock synchronization needed.

disadvantages:

- Limited speed compared to SPI/I2C.

- Only supports communication between two devices (point-to-point)

- Data frame overhead due to start, stop, and parity bits.

**Where is UART Used?**

- Industrial applications (e.g., RS-232, RS-485).

- Embedded systems (e.g., microcontrollers, FPGA, and SoC communication).

- Serial devices (e.g., GPS modules, Bluetooth, and Wi-Fi modules).

- PC Communication (e.g., USB-to-serial converters for debugging).

### Two Ways to Implement UART Using AMD FPGA

There are two main approaches to implementing UART on a AMD FPGA:

- **Using AMD’s IP**

AXI Uartlite + RTL Controller

- AXI Uartlite + MicroBlaze

- **RTL Development**

Implementing UART using an IP core means utilizing pre-designed RTL hardware. You only designing the external interface that controls it (in this case, the AXI-Lite interface and control module).

Therefore, if you consider **'How can this be designed in RTL**?', you can understand of the control mechanism of the IP’s interface and registers.

### AMD’s IP + AXI-Lite Controller

ADM provides the **AXI Uartlite IP**, which allows UART implementation through an AXI-Lite interface.

To use this IP, you need to:

- Open **Vivado’s IP Catalog** and generate the **AXI UART Lite** **IP**.

- Configure parameters such as the **baud rate** and other settings.

This approach simplifies development, as it leverages AMD’s built-in IP instead of designing a custom UART controller from scratch.

![Vivado UART IP Caltalog](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image.png?w=855)*If you search for 'UART' in the IP Catalog, you will find two IPs. You can easily implement UART using AXI Uartlite.*

![AXI Uartlite Customize IP](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-1.png?w=1024)*Configuring parameters such as Baud Rate and others*

- AXI CLK: This IP utilizes AXI4-Lite as the user interface. Since UART control is managed through AXI4-Lite, you need to specify the AXI4-Lite clock frequency. By default, a 100 MHz clock is commonly used in AMD designs with MicroBlaze. You can keep this setting unchanged.

- Baud Rate: The baud rate determines the communication speed of the UART interface. It should be set to a value that is compatible with the receiving device to ensure proper communication.

- Data Bits: The number of data bits can be configured between 5 and 8. In most cases, 8 bits is the standard setting. This defines the number of bits transmitted between the Start and Stop bits.

- Parity: A parity bit can be added to detect errors in data transmission. This helps in verifying the integrity of the transmitted data.

![Block Diagram of AXI UART Lite](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-2.png?w=927)*Block Diagram of AXI UART Lite*

AXI4-Lite interface is connected to the internal UART Lite Register, and the Register is connected to the UART Control block. In other words, the UART is controlled by configuring the Register.

You need to understand the UART Register and its operation by referring to the next IP document.

IP related document: [https://docs.amd.com/v/u/en-US/pg142-axi-uartlite](https://docs.amd.com/v/u/en-US/pg142-axi-uartlite)

**Example Design**

The easiest and quickest way to understand the design is by utilizing the Example Design of the generated IP.

By running the simulation in the Example Design project, you can understand the basic operation and flow.

![How to execute "Open IP Example Design"](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-3.png?w=533)*Create IP example design*

![Block Diagram of UART Example Design](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-4.png?w=703)*Block Diagram of Example Design*

Through test bench code and simulation waveform, you can obtain information such as the sequences of command and registers access.

![UART Simulation wave form](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-5.png?w=1024)*Write (0x04, 0x00000041) through the AXI Lite Bus. Address 0x04 is the Tx FIFO.*

![UART Simulation wave form](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-6.png?w=1024)*You can notice the signal being output through the UART TX port. By checking the length of the Start Bit period, you can find the baud rate. 104.17 us → 9600 baud rate.*

In the case of the AXI UART Lite IP, a value is written to the TX FIFO register. Then, the data is transmitted through the **tx** port.

![Shows the test bench code of the UART example](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-7.png?w=580)*Looking at the testbench code, the 0x41 data is successfully sent, and the simulation ends*

Through simulation, you can understand the basic control method of how the UART IP operates.

The important part of the above method is designing the AXI-Lite interface. A control interface is ultimately needed. It should write or get data through the UART. This data must control the UART Register via AXI-Lite. One way to make this process easier is by using a micro-controller like **MicroBlaze**.

AXI-Lite + Micro Blaze

Since UART does not need high-speed signal processing, using a micro-controller to control it is also a feasible choice. This way, UART can be easily controlled using simple C code.

![A specific block design that includes Micro-Blaze and UART IP block.](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-8.png?w=1024)*Block design for controlling UART*

MicroBlaze and the AXI UART Lite IP are connected through the AXI Interconnect. In other words, peripheral devices connected to the AXI Interconnect are controlled based on the Address-Mapping.

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-13.png?w=1024)

In Vivado, you can export the hardware design to generate an XSA (Xilinx Support Archive) file. Then, using AMD's integrated development environment, Vitis IDE, you can run C projects based on this hardware.

![Vivado captrue of executing "Export Hardware"](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-9.png?w=433)*How to Export a Hardware Design Created in Vivado*

The development method using MicroBlaze can be referenced in the next document:

[https://docs.amd.com/r/en-US/ug1579-microblaze-embedded-design](https://docs.amd.com/r/en-US/ug1579-microblaze-embedded-design)

When you create a project in Vitis, the Board Support Package provides example code for each IP. The path to the example folder is as follows.

Example design: C:\Xilinx\Vitis\2022.2\data\embeddedsw\XilinxProcessorIPLib\drivers\uartlite_v3_7\examples

(The above path varies depending on the Vitis's version or installation directory)

![Low Level Example Code](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-10.png?w=597)*The Low-Level example stores the data to be transmitted in a buffer. It then places the data into the TX FIFO of the AXI UART Lite.*

The design can be easily created using the above example code. You can build the design and then use the generated ELF file for simulation.