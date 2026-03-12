---
title: "Board-Level Understanding and Debugging for FPGA Engineers"
date: 2026-02-13T12:11:04+00:00
draft: false
description: "FPGA design is not just RTL — you need to understand power integrity, signal integrity, and hardware debugging techniques to ship working boards."
categories:
  - "FPGA Design Tips"
tags:
  - "Debug"
  - "FPGA"
  - "ILA"
  - "Schematic"
  - "Oscilloscope"
  - "SYSMON"
  - "Trigger"
  - "XADC"
  - "Signal Integrity"
  - "Power Integrity"
---

An FPGA does not work in isolation. No matter how correct your RTL is, the design will fail if the power supply is noisy, the clock does not reach the device cleanly, or a digital interface is mismatched at the board level. FPGA engineers who can diagnose hardware problems are significantly more effective than those who can only debug RTL in simulation.

This post covers the board-level skills that separate a junior FPGA engineer from a senior one.

## 1. Schematic and Datasheet Analysis

Before the board is built — and when it comes back from the fab — the schematic is your primary diagnostic tool.

**Pin mapping:** Study the FPGA datasheet before finalising pin assignments. Not all pins are equal: LVDS-capable pairs, high-speed transceiver pins, dedicated clock input pins, and multi-function I/O all have specific placement constraints. Your XDC/SDC constraint file must match the schematic exactly. Verify that external clock and reset signals connect to pins that the Vivado tool permits for those functions.

**Termination:** High-speed signal lines require controlled impedance and proper termination. Verify that termination resistor values in the schematic match the FPGA's expected signaling standard. If the FPGA has tunable internal termination (e.g., `TERM_100` in UltraScale I/O standards), ensure the schematic and HDL constraints agree.

**Power supply review:** FPGAs have multiple power rails — VCCINT (core), VCCIO (I/O per bank), VCCAUX (auxiliary clocking), VCCBRAM, etc. Each rail has a tight voltage tolerance (typically ±3–5%). Verify that the power sequencing order matches the datasheet power-on recommendation; powering rails in the wrong order can permanently damage the device.

## 2. Power Integrity Measurement

A stable power supply is the foundation of correct FPGA operation. Modern FPGAs draw transient currents of tens of amperes during configuration and switching events, which create voltage ripple on the supply rails.

**Oscilloscope setup for ripple measurement:**
- Switch the probe to **AC coupling** mode to block the DC component and amplify the small ripple on top of it.
- Set the vertical scale to 10–50 mV/div to resolve millivolt-level noise.
- Enable the 20 MHz bandwidth limiter to reduce oscilloscope self-noise if looking for low-frequency switching ripple.

**Power-up sequence verification:** Use a multi-channel oscilloscope to capture all supply rails simultaneously during power-on. Verify that each rail ramps up within the datasheet-specified time window and in the correct order. A rail that comes up late or out of sequence is a common source of startup failures that disappear once the board is fully powered.

## 3. Slow Serial Interface Debugging (I2C, SPI)

I2C and SPI are used extensively for sensor initialisation, DAC/ADC configuration, and EEPROM access. They are low frequency but easy to get wrong.

**Protocol decoding:** Modern oscilloscopes decode I2C and SPI in real time. Use this feature to read the actual address, register, and data values on the signal lines — not just look at the waveform shape. This immediately shows whether your RTL is generating the correct transaction sequence.

**Slew rate and glitch checking:** Too high a pull-up resistor value on an I2C bus slows the rise time and rounds the edges. Too low can cause overshoot and ringing. Look for waveforms that do not reach full VOH before the next transition, or glitches on the data line during an acknowledgement stretch.

## 4. High-Speed Signal Measurement and Triggering

At gigabit speeds, you cannot see signal defects with a basic edge trigger. Oscilloscope trigger capability becomes the measure of debugging expertise.

**Pulse width trigger:** Capture pulses that are only slightly too wide or too narrow — useful for detecting single-event glitches on clock or data lines.

**Eye diagram:** For PCIe, Gigabit Ethernet, or SERDES links, set the oscilloscope to infinite persistence and overlay thousands of bit transitions. The resulting "eye" pattern shows:
- **Eye height** → voltage noise margin
- **Eye width** → timing (jitter) margin
- **Eye opening** → whether the receiver can reliably sample the data

A closed or partially closed eye means the link will have errors, even if the frame rate appears acceptable from higher-level statistics.

**Serial trigger (pattern trigger):** Trigger the oscilloscope only when a specific bit pattern appears on a serial line (e.g., the Ethernet preamble `0x55 55 55 55 55 55 55 D5`). This isolates exactly the packet boundary you want to examine without manually scrolling through hours of captured data.

## 5. In-Chip Debugging Tools

**Vivado ILA (Integrated Logic Analyser):** Insert ILA probes anywhere in your HDL design to capture on-chip signals into Block RAM and stream the result to your PC over JTAG. Supports complex trigger conditions — combinational expressions, sequences of events, counters.

**Quartus SignalTap:** Intel's equivalent in-system logic analyser, with similar capabilities.

**XADC / System Monitor (SYSMON):** Every AMD/Xilinx FPGA contains a built-in 12-bit ADC connected to internal temperature and voltage sensors. Reading these in real time tells you:

- **Die temperature:** If logic density or clock frequency is too high, die temperature rises sharply. Use SYSMON alarms to shut down before thermal damage occurs. Compare temperature readings at different clock frequencies to verify your cooling solution (heat sink, fan) is adequate.
- **Internal supply voltages (VCCINT, VCCAUX, VCCBRAM):** If voltage droops when the design is fully switching, the VRM (Voltage Regulator Module) on the board may be undersized, or bulk decoupling capacitors near the device may be insufficient.

## 6. Common Real-World Scenario

> "Simulation passes, but the sensor returns no data on the actual board."

**Step 1:** Attach oscilloscope probes to the I2C SCL and SDA lines.

**Step 2:** Set up I2C protocol decoding. Trigger on the START condition.

**Step 3:** Observe: the signal fails to reach 3.3 V — it tops out at ~1.8 V.

**Step 4:** Check the schematic. The level shifter supplying the 3.3 V side has its VCCA pin connected to a rail that is not powered up at the time the I2C transaction is attempted, leaving the pull-up inactive.

**Resolution:** Fix the power sequencing or switch the pull-up to an always-on rail.

This cycle — **form hypothesis → measure with instruments → hardware correction** — is the core debugging loop that distinguishes experienced FPGA engineers. Simulation can verify that the RTL logic is correct. Instruments tell you whether the board is actually delivering the environment the RTL expects.
