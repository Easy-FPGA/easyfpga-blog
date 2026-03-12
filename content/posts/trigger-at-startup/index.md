---
title: "Trigger At StartUp"
date: 2025-03-28T08:44:30+00:00
draft: false
description: "The Trigger at Start up feature is used to configure the trigger settings of an ILA core in a design programming file (.bit or .pdi) so that it is pre"
categories:
  - "FPGA Design"
tags:
  - "Debug"
  - "FPGA"
  - "ILA"
  - "Trigger At StartUp"
  - "Xilinx"
---

When debugging hardware using the Hardware Manager, the device connection is established after a delay following the initial boot. Once connected, you can configure trigger conditions and begin debugging.

However, this delay means that the FPGA’s initial state cannot be examined using the Integrated Logic Analyzer (ILA), as it occurs before the device is accessible via the Hardware Manager.

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-27.png?w=323)*Illustrate a **Hardware Manager connected to a device via JTAG**,*

The Trigger At StartUp method enables you to define trigger conditions in the initial phase, immediately after the bitstream is downloaded and the system begins running. This allows you to debug the early stages of operation effectively.

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-29.png?w=1024)*Debugging at Device Startup*

The Trigger at Start up feature is used to configure the trigger settings of an ILA core in a design programming file (.bit) so that it is pre-armed to trigger immediately after device start up. 

To use the Trigger at Start up feature perform the following steps:

-  Run through the first pass of the ILA flow as usual to set up trigger condition.

Open the target, configure the device, and bring up the ILA Dashboard.

- Enter the trigger equations for the ILA core in the ILA Dashboard.

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-30.png?w=853)*Enter the trigger equations*

- From the Vivado Tcl command line, export the trigger register map file for the ILA core. This file contains all of the register settings to "stamp" back on the implemented netlist. The output from this is a single file.

Set the save folder path for the *.tas file

```
run_hw_ila -file D:/fpga_design/ila_trig.tas [get_hw_ilas hw_ila_1]
```

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-40.png?w=650)*Generate ila_trig.tas file*

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-41.png?w=723)*The *.tas file created in the filder*

- Go back and open the previously implemented routed design in VIvado IDE.

Project Mode: Use the Flow Navigator to open the implemented design

- Non-Project Mode: Open your routed checkpoint:%open_checkpoint <file>.dcp

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-42.png?w=224)*Open Implemented Design*

- At the Implemented Design Tcl Console, apply the trigger settings to the current design in memory, which is your routed netlist.

```
%apply_hw_ila_trigger ila_trig.tas
```

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-44.png?w=840)*Apply the trigger settings to the current design in memory*

- At the Implemented Design Tcl Console, write a new device image with Trigger at Start up settings using either the write_bitstream command for 7-series, UltraScale and UltraScale+ FPGAs or SoCs or the write_device_image command for Versal devices.

```
%write_bitstream -file ila_trigger_at_startup.bit
```

![](https://easyfpgadesign.wordpress.com/wp-content/uploads/2025/03/image-45.png?w=850)*write_bitstream command*

- Go back to the Hardware Manager and reconfigure with the new .bit file that you generated in the previous step.

Select the device in the hardware tree.

- Assign the .bit file generated in step 5.

- Program the device using the new .bit file

Once programmed, the new ILA core should immediately arm at start up. 

- You should see an indication in the Trigger Capture Status for the ILA core. If trigger or capture events have occurred, the ILA core is now populated with captured data samples.

For more detailed information, please refer to the linked document below.

[https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Trigger-At-Startup](https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging/Trigger-At-Startup)

[https://www.xilinx.com/video/hardware/debugging-at-device-startup.html](https://www.xilinx.com/video/hardware/debugging-at-device-startup.html)