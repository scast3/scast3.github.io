---
title: "Carrier Offset Tracking and Phase-Locking in a QPSK Digital Receiver"
date: 2026-04-01
summary: "test"
tags: ["MATLAB", "Simulink", "Kalman Filter", "DSP", "Communications"]
github: "https://github.com/scast3/qpsk_receiver"
tools: ["Matlab", "Simulink"]
status: "in-progress"          # complete | in-progress
weight: 1                   # lower = appears first in listings
---

## Overview

Accurate demodulation in a digital communication system requires the receiver to stay synchronized with the incoming RF carrier. In practice, several physical effects make this difficult: carrier frequency offset (CFO) from oscillator inaccuracies or Doppler shifts causes a continuously rotating phase in the received baseband signal, channel fading varies the signal amplitude, and additive noise corrupts every measurement.

This project models a QPSK receiver in MATLAB and Simulink and implements an Extended Kalman Filter (EKF) to jointly estimate all four distortion states in real time, enabling correction of the received IQ samples and accurate symbol recovery.

---

## System Model

### QPSK Signal

In QPSK, each transmitted symbol sits on one of four constellation points on the unit circle, each separated by 90°:

```
s_k ∈ { (±1 ± j) / sqrt(2) }
```

The received baseband signal after passing through the channel is modeled as:

```
r_k = a_k * s_k * exp(j*theta_k) + v_k
```

where `theta_k` is the carrier phase offset, `a_k` is the signal amplitude, and `v_k` is complex AWGN. The trigonometric dependence of the IQ components on `theta_k` introduces the nonlinearity that prevents a standard linear Kalman filter from being used directly.

### State Vector

Four internal states are tracked by the estimator:

| State | Symbol | Description |
|---|---|---|
| Carrier phase | θₖ | Instantaneous phase offset |
| Frequency offset | ωₖ | CFO from oscillator mismatch |
| Frequency drift | αₖ | Rate of change of frequency offset |
| Signal amplitude | aₖ | Channel fading amplitude |

### State Equations

Phase, frequency, and drift are coupled through integrator relationships. The amplitude is modeled as a random walk. The full discrete-time state equation in matrix form is:

```
x_{k+1} = Φ * x_k + Γ * u_k + n_k
```

where `u_k` is the oscillator correction input and `n_k` is zero-mean Gaussian process noise with covariance Q. The state transition and input matrices are:

```
Φ = | 1   T   T²/2   0 |       Γ = | -T |
    | 0   1   T      0 |           | -1 |
    | 0   0   1      0 |           |  0 |
    | 0   0   0      1 |           |  0 |
```

### Measurement Equation

The measurement is the received IQ sample, split into in-phase and quadrature components:

```
y_k = h(x_k, s_k) + v_k

h = | a_k * ( Re(s_k)*cos(θ_k) - Im(s_k)*sin(θ_k) ) |
    | a_k * ( Re(s_k)*sin(θ_k) + Im(s_k)*cos(θ_k) ) |
```

The transmitted symbol `s_k` is assumed known at the receiver via pilot symbols. The process model is linear; the measurement equation is nonlinear — specifically requiring an EKF rather than a standard Kalman filter.

---

## Simulation Parameters

| Parameter | Value |
|---|---|
| Number of symbols N | 500 |
| Symbol period T | 1×10⁻⁴ s |
| Symbol rate | 10,000 symbols/s |
| Initial phase offset θ₀ | π/3 rad |
| Initial frequency offset ω₀ | 2π×200 rad/s |
| Initial frequency drift α₀ | 2π×0.5 rad/s² |
| Initial amplitude a₀ | 1.0 |
| SNR | 10 dB |

---

## Open-Loop Characterization

Before applying the EKF, two open-loop scenarios were simulated in Simulink to understand the effect of the carrier offsets.

### No Synchronization (u_k = 0)

With no correction applied, the carrier phase accumulates linearly over time due to the constant frequency offset of ω₀ = 200 Hz. Over 500 symbols the phase accumulates roughly 20π radians — approximately 10 full rotations of the constellation. The received IQ plot shows a ring of samples rather than four clusters, because the rotating phase smears all four constellation points uniformly around the unit circle.

### Ideal Impulse Correction

A unit impulse equal to the true frequency offset ω₀ was applied at k=0 — an unrealizable but instructive scenario. The frequency offset drops to near zero immediately, stopping the linear phase accumulation. The constellation now shows four distinct clusters, but they are rotated by the initial phase offset θ₀ = π/3 from the correct QPSK positions. This demonstrates a key limitation: frequency-only correction is insufficient. The initial phase offset has no mechanism to be removed by the control input `u_k` alone, motivating the need for full state estimation.

---

## Extended Kalman Filter

### Why EKF

The measurement function `h` is nonlinear in `θ_k` — the received IQ samples depend on `cos(θ_k)` and `sin(θ_k)`. The EKF handles this by linearizing `h` around the current state estimate at each time step using a Jacobian matrix, then applying standard Kalman filter equations to the linearized model. The process model is already linear, so no linearization is needed in the prediction step.

### Measurement Jacobian

Defining shorthand terms evaluated at the predicted state estimate:

```
A_k = Re(s_k)*cos(θ̂_k) - Im(s_k)*sin(θ̂_k)
B_k = Re(s_k)*sin(θ̂_k) + Im(s_k)*cos(θ̂_k)
```

The Jacobian of the measurement function with respect to the state vector is:

```
H_k = | -a_k*B_k   0   0   A_k |
      |  a_k*A_k   0   0   B_k |
```

The zero columns for ω_k and α_k reflect that these states do not appear in the measurement equation directly — they are only observable indirectly through the accumulation of phase over time via the integrator chain in the process model.

### EKF Algorithm

The filter runs recursively at each symbol period, alternating between a measurement update and a time propagation step.

**Initialization** — The filter starts cold with no prior knowledge of phase or frequency:

```
x̂_0 = [0, 0, 0, a_nom]

P_0 = diag([(π/2)², (2π×500)², (2π×5)², 0.25])
```

**Step 1 — Measurement Update:**

```
K_k   = P⁻_k * H_kᵀ * (H_k * P⁻_k * H_kᵀ + R)⁻¹
x̂⁺_k = x̂⁻_k + K_k * (y_k - h(x̂⁻_k, s_k))
P⁺_k  = (I - K_k * H_k) * P⁻_k
```

**Step 2 — Time Propagation:**

```
x̂⁻_{k+1} = Φ * x̂⁺_k + Γ * u_k
P⁻_{k+1}  = Φ * P⁺_k * Φᵀ + Q
```

Because the process model is linear, no Jacobian is required in the prediction step. All linearization error in the EKF is confined to the measurement update.

### Convergence

The EKF estimates of phase and amplitude converge quickly to the true states after a short transient at startup. The frequency offset ω_k and drift α_k take longer to converge since they are not directly observed — the filter must infer them from the accumulated effect on phase over multiple symbol periods. Once converged, the estimation errors remain small relative to the noise floor.

---

## Signal Correction and Symbol Recovery

The EKF state estimate is used to undo the phase and amplitude distortions on each received sample:

```
r̃_k = (r_k * exp(-j*θ̂_k)) / â_k
```

As the EKF converges and `θ̂_k → θ_k` and `â_k → a_k`, the corrected sample approaches the ideal transmitted symbol plus residual noise only. A nearest-neighbor decision rule then maps each corrected sample to the closest QPSK constellation point:

```
ŝ_k = argmin over s in S of |r̃_k - s|
```

The corrected IQ constellation shows tight clustering around all four QPSK points, confirming successful synchronization.

---

## Observability

A necessary condition for EKF convergence is that all states are observable from the measurements. Only θ_k and a_k appear directly in `H_k`. The states ω_k and α_k are not directly observed but become observable over time through the integrator structure: ω_k drives the accumulation of θ_k, and α_k drives the drift of ω_k. The filter builds sufficient observability for these states after several symbol periods, which explains the longer initial transient in their estimates.

---

## Closed-Loop Extension (Future Work)

In the current implementation, the oscillator correction input `u_k` is held at zero throughout the simulation and correction is applied as a post-processing step. In a fully closed-loop receiver, the EKF frequency estimate would be fed back to drive the oscillator in real time:

```
u_k = ω̂⁺_k
```

This would actively reduce the frequency offset at each step, shrinking the residual the filter needs to track and improving both convergence speed and steady-state accuracy.

---

## Tools Used

- **MATLAB** — EKF implementation, signal correction, constellation plots
- **Simulink** — Open-loop state-space plant model

[View on GitHub →](https://github.com/scast3/qpsk_receiver)