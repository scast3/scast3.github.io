---
title: "Oscilloscope Software/Hardware Co-design on Zynq-7010 SoC"
date: 2025-12-10
summary: "Designed an oscilloscope on an fpga in vhdl"
tags: ["FPGA", "VHDL", "Verification", "Xilinx"]
github: "https://github.com/scast3/oscilloscope"
tools: ["Vivado", "Vitis"]
status: "complete"          # complete | in-progress
weight: 1                   # lower = appears first in listings
---


## Overview

For this project, I designed a fully functional two-channel oscilloscope built on a Xilinx Zynq-7010 SoC. The system captures analog signals using the AD7606 ADC and renders waveforms in real time over HDMI. Rather than using a dedicated oscilloscope MCU, the design splits responsibility across two domains on the same chip: programmable logic (PL) which is implemented in VHDL, handles the acquisition and video pipeline which is very timing-dependent, while the ARM Cortex-A9 processing system (PS) runs embedded C firmware for user control through a command line interface.

The two domains communicate through a custom AXI4-Lite slave peripheral, giving the ARM processor memory-mapped access to control registers, status flags, and live sample data in the FPGA fabric.

---

![fpga](board.jpg)

## System Architecture

```
  +---------------------+        AXI4-Lite Bus        +----------------------+
  |  ARM Cortex-A9 (PS) | <-------------------------> |   PL (VHDL Fabric)   |
  |                     |                             |                      |
  |  Vitis C firmware   |   slv_reg0: CH1 data (R)    |  acquireToHDMI       |
  |  - UART CLI         |   slv_reg1: CH2 data (R)    |  +-----------------+ |
  |  - TTC0 ISR         |   slv_reg2: status (R)      |  | ADC FSM         | |
  |  - Trigger control  |   slv_reg3: control (W)     |  | Sample timer    | |
  |  - Function gen     |   slv_reg4: trig volt (W)   |  | Trigger logic   | |
  |                     |   slv_reg5: trig time (W)   |  | HDMI renderer   | |
  +---------------------+                             |  +-----------------+ |
                                                      |                      |
                                        AD7606 ADC -->|  16-bit parallel bus |
                                        HDMI output <-|  TMDS serializer     |
                                                      +----------------------+
```

The top-level VHDL entity `acquireToHDMI` instantiates two sub-modules: a datapath (`acquireToHDMI_datapath`) handling the ADC interface, waveform buffering, and video rendering, and a Moore FSM (`acquireToHDMI_fsm`) generating the control word that sequences all operations. The AXI wrapper instantiates this top-level and exposes its ports as memory-mapped registers to the PS.

---

## Programmable Logic — VHDL Design

The PL design follows a classic **datapath / FSM separation**. The datapath (`acquireToHDMI_datapath`) contains all registers, counters, BRAMs, comparators, and pixel converters as structural VHDL instantiations. The FSM (`acquireToHDMI_fsm`) is a pure Moore machine that observes a status word (`sw`) from the datapath and drives a control word (`cw`) back to it — the two modules communicate only through these two buses, with no direct logic between them.

### Control Word / Status Word Interface

The FSM has no knowledge of signal values — it only sees a vector of condition bits (the status word) and outputs a vector of control bits (the control word). Every datapath resource — counters, registers, BRAM write enables — is gated by a dedicated bit in `cw`. This makes the state outputs in the FSM completely readable as a lookup table: each state drives a fixed `cw` pattern with named bit positions defined in the shared package.

**Key status word (sw) bits observed by the FSM:**

| Bit | Signal | Source in datapath |
|---|---|---|
| `BUSY_SW` | AD7606 busy pin | External ADC pin |
| `SHORT_DELAY_DONE_SW` | Short counter == target | `shortCompare` |
| `LONG_DELAY_DONE_SW` | Long counter == target | `longCompare` |
| `FULL_SW` | Write address == screen width | `cmp_BRAM_full` |
| `SAMPLE_SW` | Sample counter == rate target | `sampleCompare` |
| `TRIG_CH1_SW` | CH1 rising edge detected | Trigger comparators |
| `TRIG_CH2_SW` | CH2 rising edge detected | Trigger comparators |
| `STORE_SW` | SR latch — BRAM fill active | SR latch process |
| `SINGLE_SW` / `FORCED_SW` | Mode from PS control reg | AXI slv_reg3 |

**Key control word (cw) bits driven by the FSM:**

| Bit(s) | Function |
|---|---|
| `CONVST_CW` | Assert ADC conversion start |
| `CS_CW`, `RD_CW` | ADC chip select and read strobe |
| `RESET_AD7606_CW` | ADC hardware reset |
| `DATA_STORAGE_CH1_WRITE_CW` | BRAM write enable for CH1 |
| `DATA_STORAGE_CH2_WRITE_CW` | BRAM write enable for CH2 |
| `TRIG_CH1_WRITE_CW` | Load trigger sample register CH1 |
| `TRIG_CH2_WRITE_CW` | Load trigger sample register CH2 |
| `SET_STORE_FLAG_CW` / `CLEAR_STORE_FLAG_CW` | Set/clear the SR latch |
| `DATA_STORAGE_COUNTER_CW` | Count/hold/reset BRAM write address |
| `SHORT_DELAY_COUNTER_CW` / `LONG_DELAY_COUNTER_CW` | Count/hold/reset delay timers |
| `SAMPLING_COUNTER_CW` | Count/hold/reset sample interval timer |

### FSM State Machine

The FSM has 22 states sequencing the full acquisition pipeline. The major flow is:

```
RESET_STATE -> LONG_DELAY -> ADC_RST
    |
    +-- (forced mode) --> WAIT_FORCED --> (single pulse) --> SET_STORE_FLAG
    |
    +-- (trigger mode) --> CLEAR_STORE_FLAG
    |
    v
BEGIN_CONVST -> ASSERT_CONVST -> BUSY_0 -> BUSY_1
    |
    v
READ_CH1_LOW -> WRITE_CH1_TRIG or WRITE_CH1_BRAM -> READ_CH1_HIGH
    |
    v
RST_SHORT -> READ_CH2_LOW -> WRITE_CH2_TRIG or WRITE_CH2_BRAM -> READ_CH2_HIGH
    |
    v
WAIT_END_SAMP_INT -> (FULL?) -> BRAM_FULL -> loop
                  -> (trigger hit, not full) -> SET_STORE_FLAG
                  -> (no trigger) -> BEGIN_CONVST
```

At each ADC read state, the FSM branches based on `STORE_SW`: if the SR latch is set (BRAM fill is active), it routes the sample to BRAM (`WRITE_CH1_BRAM`); otherwise it routes it only to the trigger comparator registers (`WRITE_CH1_TRIG`). This ensures samples are compared against the threshold continuously but only written to BRAM once a trigger has been detected.

### ADC Interface (AD7606)

The AD7606 is an 8-channel, 16-bit SAR ADC with a parallel readout interface. The FSM drives `CONVST`, `CS`, `RD`, and `RESET` in the correct sequence — asserting conversion start, waiting for the `BUSY` flag to deassert (states `BUSY_0` → `BUSY_1`), then clocking out the 16-bit result. Two short-delay counters in the datapath provide the required ADC setup and hold timing. Sampling rate is controlled by a 4-to-1 mux (`sampleMux`) that selects between four preset counter targets based on the 2-bit `sampleRate_select` from the PS.

### Trigger Logic

Trigger detection uses a two-sample edge detector implemented structurally in the datapath. For each channel, two chained `genericRegister` instances capture consecutive ADC samples, and two `genericCompare_Signed` instances compare each against the signed threshold voltage:

- **Sample 1 comparator:** checks `sample1 > threshold` (rising condition)
- **Sample 2 comparator:** checks `sample2 < threshold` (pre-crossing condition)

The trigger fires when both conditions are true simultaneously — sample 2 was below threshold and sample 1 crossed above it. This detects a rising edge crossing rather than just a level threshold, preventing false triggers on a flat signal sitting above the threshold.

**Trigger mode** (`forced_mode = 0`) — the FSM loops through `BEGIN_CONVST` continuously, writing samples only to the trigger registers, until `TRIG_CH1_SW` fires. Then `SET_STORE_FLAG` enables BRAM writes and the next `VIDEO_WIDTH` samples fill the display buffer.

**Forced mode** (`forced_mode = 1`) — the FSM parks in `WAIT_FORCED` and only proceeds on a `SINGLE_SW` pulse from the PS, immediately setting the store flag and capturing one frame.

### HDMI Video Output and Waveform Rendering

The datapath instantiates a `clk_wiz_0` PLL to derive the pixel clock (`videoClk`) and a 5x clock (`videoClk5x`) for TMDS serialization from the system clock. The `videoSignalGenerator` produces standard `HS`, `VS`, and `DE` timing signals along with pixel coordinates (`pixelHorz`, `pixelVert`).

Each channel's BRAM is dual-port: port A is clocked on the system clock and written by the FSM during acquisition; port B is clocked on the pixel clock and read during display. The read address is `pixelHorz - L_EDGE`, mapping each horizontal pixel directly to a stored sample. A `toPixelValue` converter scales the 16-bit signed ADC value to a vertical pixel coordinate, and a `genericCompare` checks whether the current `pixelVert` matches — driving the `ch1` / `ch2` signals into the `scopeFace` renderer which composites the waveform, grid, and trigger markers into RGB pixel values for the `hdmi_tx_0` serializer.

![hdmi](hdmi.jpg)

### AXI4-Lite Slave Wrapper

The AXI wrapper (`final_oscope_slave_lite_v1_0_S00_AXI`) implements a custom AXI4-Lite slave with 10 32-bit registers and two independent read/write state machines. The register map exposes the oscilloscope IP to the ARM:

| Register | Direction | Contents |
|---|---|---|
| slv_reg0 | Read | CH1 sample data (16-bit) |
| slv_reg1 | Read | CH2 sample data (16-bit) |
| slv_reg2 | Read | Status flags |
| slv_reg3 | Write | Control register |
| slv_reg4 | Write | Trigger voltage (signed 16-bit) |
| slv_reg5 | Write | Trigger time (pixel position) |

**Control register (slv_reg3) bit map:**

| Bit | Function |
|---|---|
| 0 | `single_mode` — pulse to acquire one frame |
| 1 | `forced_mode` — run continuously without trigger |
| 2 | `ch1enb` — enable channel 1 |
| 3 | `ch2enb` — enable channel 2 |
| 5:4 | `sampleRate_select` — 2-bit sample rate |
| 6 | Reset pulse |
| 7 | Flag clear — acknowledge sample-ready flag |

**Status register (slv_reg2) bit map:**

| Bit | Function |
|---|---|
| 0 | `triggerCh1` — CH1 threshold crossed |
| 1 | `triggerCh2` — CH2 threshold crossed |
| 2 | `conversionPlusReadoutTime` — ADC busy window |
| 3 | `sampleTimerRollover` — sample period elapsed |
| 4 | `flag_q` — new sample ready flag |

---

## Embedded Software — ARM Cortex-A9 (C)

The firmware runs on the PS under Xilinx Vitis (bare-metal, no OS) and provides a UART command-line interface for real-time oscilloscope control.

### TTC0 Timer ISR — Function Generator

A Triple Timer Counter (TTC0) is configured to fire at 10 kHz. On each interrupt, the ISR steps a 16-bit phase accumulator by a configurable `phaseIncrement` and indexes a 64-entry LUT to produce the next DAC output value via an enhanced PWM peripheral. Two waveforms are available:

- **Sine wave** — 64-point quantized sine LUT
- **Sinc wave** — 64-point sinc function LUT

Output frequency is set by entering a value in Hz over UART; the firmware computes the correct phase increment using a pre-characterized linear regression (`phaseIncrement = 6.5516 * frequency + 0.0062`).

### UART Command Interface

The main loop blocks on `XUartPs_RecvByte()` and dispatches on a single character:

| Key | Action |
|---|---|
| `t` | Toggle trigger / forced acquisition mode |
| `n` | Single-shot acquire (pulse `single_mode` bit high then low) |
| `+` / `-` | Increment / decrement trigger voltage by 1000 LSB |
| `v` | Reset trigger voltage to 0 |
| `a` / `b` | Toggle CH1 / CH2 enable |
| `s` | Toggle function generator on/off |
| `w` | Select sine or sinc waveform |
| `d` | Set PWM duty cycle manually |
| `u` | Read and print 64 sequential samples from CH1 |
| `r` | Universal reset (pulse reset bit in control register) |
| `?` | Print help menu |

### Register Access Pattern

All PS↔PL communication goes through the AXI register map using the generated `FINAL_OSCOPE_mReadReg` / `FINAL_OSCOPE_mWriteReg` macros. A typical read-modify-write on the control register looks like:

```c
// Toggle forced mode bit
u32 reg = FINAL_OSCOPE_mReadReg(XPAR_FINAL_OSCOPE_0_BASEADDR,
                                 FINAL_OSCOPE_S00_AXI_SLV_REG3_OFFSET);
reg ^= (1 << 1);  // flip forced_mode bit
FINAL_OSCOPE_mWriteReg(XPAR_FINAL_OSCOPE_0_BASEADDR,
                        FINAL_OSCOPE_S00_AXI_SLV_REG3_OFFSET, reg);
```

The trigger voltage is stored as a signed 16-bit value in the lower half of a 32-bit register, requiring a careful mask-and-cast on both read and write to preserve the upper 16 bits and handle two's complement correctly:

```c
// Read signed 16-bit trigger voltage
u32 full = FINAL_OSCOPE_mReadReg(BASEADDR, REG4_OFFSET);
int16_t voltage = (int16_t)(full & 0xFFFF);
voltage += 1000;
FINAL_OSCOPE_mWriteReg(BASEADDR, REG4_OFFSET,
    (full & 0xFFFF0000) | ((u32)voltage & 0xFFFF));
```

---

## Key Design Decisions

**Splitting acquisition and rendering into PL.** Doing the ADC sequencing and HDMI pixel rendering in VHDL keeps all hard real-time operations in the fabric and away from the ARM. The PS only needs to write configuration registers and poll status — it never has to meet a pixel clock deadline.

**AXI flag handshake for sample-ready.** Rather than polling the ADC busy signal directly from software, the PL sets a `flag_q` bit in the status register when a new sample is ready, and the PS acknowledges it by pulsing `flag_clear`. This decouples the ADC conversion timing from the software polling rate and avoids missed samples.

**Signed trigger voltage over AXI.** The AD7606 outputs signed 16-bit values, so the trigger threshold needs to be signed too. Passing it through a 32-bit AXI register required explicit masking to avoid sign extension corrupting the upper half of the register — a subtle bug that showed up during integration testing.

**Phase accumulator for waveform generation.** Instead of computing sine values in the ISR (too slow for bare-metal at 10 kHz), the ISR uses a 16-bit phase accumulator and a pre-computed 64-entry LUT. The upper 6 bits of the accumulator index into the table, giving smooth frequency control by just changing the increment value.

---

## Tools Used

- **Xilinx Vivado** — VHDL synthesis, implementation, AXI IP packaging
- **Xilinx Vitis** — ARM Cortex-A9 bare-metal C firmware
- **ModelSim** — VHDL functional simulation
- **Zynq-7010 SoC** (Digilent board) — target hardware

[View on GitHub →](https://github.com/scast3/oscilloscope)