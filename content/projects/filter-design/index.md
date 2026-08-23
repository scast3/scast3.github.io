---
title: "FIR Filter Design/Implementation"
date: 2026-8-1
tags: ["Matlab", "C++", "VHDL"]
github: "https://github.com/scast3/deauth_detect"
tools: ["Vim", "Vivado"]
status: "complete"          # complete | in-progress
weight: 1                   # lower = appears first in listings
---

## Overview

Designing an FIR Filter in Matlab and implementing in C++ and VHDL to perform optimization. The filter parameters were:

- Sampling Rate: 100 MHz
- Passband: 10 MHz
- Stopband: 15 MHz

The filter was first designed in matlab to determine the minimum amount of taps, and to find the corresponding FIR coefficients.

```matlab
rp = 3;
rs = 40;
fs = 100e6;
Fpass = 10e6;
Fstop = 17e6;
dev = [(10^(rp/20)-1)/(10^(rp/20)+1) 10^(-rs/20)]; 
[n, fo, ao , w]=firpmord([Fpass Fstop],[1 0],dev,fs);
b = firpm(n,fo,ao,w);
freqz(b,1,1024,fs)
fprintf('b(%d) = %.6f\n', [1:length(b); b]);
fprintf('%.6f\n',b');
```
The response of the theoretical filter can be seen below:

![diagram](response.png)