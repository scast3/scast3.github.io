---
title: "Oscilloscope FPGA Design in VHDL"
date: 2025-12-10
summary: "Designed an oscilloscope on an fpga in vhdl"
tags: ["FPGA", "VHDL", "Verification", "Xilinx"]
github: "https://github.com/scast3/oscilloscope"
tools: ["Vivado", "Vitis"]
status: "complete"          # complete | in-progress
weight: 1                   # lower = appears first in listings
---

## Overview

Digital filtering is a core DSP operation, but software implementations on a CPU can struggle to keep up with high-bandwidth signals in real time. This project implements a **32-tap FIR low-pass filter** entirely in VHDL on a Xilinx Artix-7 FPGA (Basys 3 board), targeting a cutoff frequency of 10 kHz with a 100 kHz input sample rate.

The goal was to explore pipelined fixed-point arithmetic on an FPGA and compare resource utilization against an equivalent DSP48E1-optimized design.

---

## System architecture

The filter is structured as a fully pipelined direct-form transposed FIR, with one multiply-accumulate (MAC) stage per tap running in parallel. This trades logic resources for throughput — the filter produces one valid output per clock cycle after an initial latency of 32 cycles.

```
Input (16-bit signed)
        |
        v
 +-------------+     +-------------+
 |  Shift Reg  |---->| Coefficient |
 |  (32 taps)  |     |  ROM (32x16)|
 +-------------+     +------+------+
                            |
                     +------v------+
                     |  32x MAC    |
                     +------+------+
                            |
                     +------v------+
                     | Adder tree  |
                     +------+------+
                            |
                      Output (32-bit)
```

Coefficients were generated in MATLAB using the `fir1()` function with a Hann window, then quantized to Q1.15 fixed-point format and stored in a ROM initialized from a `.coe` file.

---

## Key design decisions

**Fixed-point word length.** Input samples are 16-bit signed integers. Coefficients are Q1.15 (16-bit). Each product is 32-bit; the accumulator grows to 40-bit to prevent overflow across 32 additions. The final output is rounded and truncated back to 16-bit.

**Pipelining vs. resource sharing.** A resource-shared design would use a single multiplier cycling through all 32 taps — lower LUT count but only 1/32 the throughput. Since the target application (audio-rate decimation) needed full-rate output, the pipelined approach was chosen. Vivado inferred DSP48 blocks for all 32 multipliers automatically.

**Timing closure.** At 200 MHz the critical path ran through the adder tree. Breaking the tree into two registered stages (16→4→1) resolved the timing violation with no functional change.

---

## Results

| Metric | Value |
|---|---|
| Clock frequency | 200 MHz |
| Latency | 32 clock cycles |
| LUT utilization | 412 / 20,800 (2%) |
| DSP48 blocks used | 32 / 90 (36%) |
| BRAM | 1 (coefficient ROM) |
| Stopband attenuation | −52 dB |
| Passband ripple | < 0.1 dB |

---

## Verification

Functional simulation was done in ModelSim with a MATLAB-generated test bench. A 5 kHz sine (passband) and a 40 kHz sine (stopband) were fed simultaneously; the output FFT confirmed correct attenuation. Post-implementation timing simulation verified no setup violations at 200 MHz.

The filter was also validated on hardware by connecting a signal generator to the PMOD header and observing the output on an oscilloscope.

---

## What I'd do differently

The coefficient ROM is currently hard-coded at synthesis time. A more flexible design would load coefficients over SPI or AXI-Lite at runtime, allowing filter reconfiguration without re-synthesizing. I prototyped this in a branch but didn't integrate it into the main design.

---

## Repository

[View on GitHub →](https://github.com/scast3/oscilloscope)

Full source, testbenches, and the MATLAB coefficient generation script are in the repo.