---
title: "Carrier Offset Tracking and Phase-Locking in a QPSK Digital Receiver using Extended Kalman Filter"
date: 2026-04-30
summary: "test"
tags: ["MATLAB", "Simulink", "Kalman Filter", "DSP", "Communications"]
github: "https://github.com/scast3/qpsk_receiver"
tools: ["Matlab", "Simulink"]
status: "in-progress"          # complete | in-progress
weight: 1                   # lower = appears first in listings
math: true
---

## Project Summarry

This project simulates the effects of receiving a wireless signal that has been altered by carrier offsets and channel noise. I modeled the channel effects as a nonlinear state space system and then implemented an Extended Kalman Filter (EKF) to estimate these offsets and correct them. I decided to apply this to a quadrature shift phase keying (QPSK) modulation scheme since it is widely used in satellite communications which regularly have to correct doppler shifts. The simulation and algorithms were developed with Matlab and Simulink.


## Background
In a digital communication system, accurate demodulation requires the receiver to maintain synchronization with the incoming RF carrier. In practice, several physical effects distort the received signal and make this challenging. 

One of the most prominent effects is carrier frequency offset (CFO) which occurs when the receiver's local oscillator is not perfectly synchronized with the transmitter's. This can occur due to oscillator inaccuracies or Doppler shifts. These frequency offsets accumulate over time and result in a rotating phase in the received baseband signal.

Additional distortions include signal amplitude variations due to channel fading and additive noise from receiver electronics.

Together, these distortions make accurate symbol detection difficult, and the receiver must estimate these carrier offsets in order to recover the transmitted symbol.

---

## System Model

### QPSK Signal

In QPSK, each transmitted symbol sits on one of four constellation points on the unit circle, each separated by 90°:

$$s_k \in \left\{ \frac{\pm 1 \pm j}{\sqrt{2}} \right\}$$

The received baseband signal after passing through the channel is modeled as:

$$
r_k=a_k s_k e^{j\theta_k}+v_k
$$

where the distortions are $\theta_k$, the carrier phase offset, $a_k$, the signal amplitude, and $v_k \sim \mathcal{CN}(0,\sigma^2)$, complex additive white Gaussian noise (AWGN).

Separating into real and imaginary components gives the in-phase (I) and quadrature (Q) components of the received symbol:
$$
\begin{aligned}
\Re(r_k) &= a_k \big( \Re(s_k)\cos\theta_k - \Im(s_k)\sin\theta_k \big) + \Re(v_k), \\
\Im(r_k) &= a_k \big( \Re(s_k)\sin\theta_k + \Im(s_k)\cos\theta_k \big) + \Im(v_k).
\end{aligned}
$$

### State Vector

Four internal states are tracked by the estimator:

| State | Symbol | Description |
|---|---|---|
| Carrier phase | $\theta_k$ | Instantaneous phase offset |
| Frequency offset | $\omega_k$ | CFO from oscillator mismatch |
| Frequency drift | $\alpha_k$ | Rate of change of frequency offset |
| Signal amplitude | $a_k$ | Channel fading amplitude |

This gives a state vector $x_k$:
$$
x_k =
\begin{bmatrix}
\theta_k \\
\omega_k \\
\alpha_k \\
a_k
\end{bmatrix}.
$$

### State Equations

The state equations were derived from the integrator relationships between $\theta_k$ and $\omega_k$. For this project, the integration is expanded to include frequency drift $\alpha_k$ which is the discrete-time change in $\omega_k$.

The input $u_k$ is applied to the receiver oscillator which allows adjustment of the frequency offset. This input influences both phase and frequency due to their integrator relationship. In this model, $u_k$ appears in both the phase and frequency equations with a gain of $T$ in the phase equation reflecting the integration of frequency over one symbol period, and a gain of $1$ in the frequency equation as a direct offset correction.
    
Furthermore, for model simplicity, the amplitude $a_k$ is modeled as a random walk driven by process noise. 

Lastly, for each state, a random noise variable in the vector $n_k \sim \mathcal{N}(0,Q)$ is added to finalize the state equation for this system:

$$
\begin{aligned}
\theta_{k+1} &= \theta_k + T\omega_k + \frac{T^2}{2}\alpha_k- T u_k + n_{\theta,k} \\
\omega_{k+1} &= \omega_k + T\alpha_k - u_k + n_{\omega,k} \\
\alpha_{k+1} &= \alpha_k + n_{\alpha,k} \\
a_{k+1} &= a_k + n_{a,k}
\end{aligned}
$$

The full discrete-time state equation in matrix form is:


$$
x_{k+1} =
\underbrace{
\begin{bmatrix}
1 & T & \frac{T^2}{2} & 0 \\
0 & 1 & T & 0 \\
0 & 0 & 1 & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
}_{\Phi}
x_k
+
\underbrace{
\begin{bmatrix}
-T \\
-1 \\
0 \\
0
\end{bmatrix}
}_{\Gamma}
u_k + n_k
$$

or simply:

$$x_{k+1}=\Phi x_k + \Gamma u_k + n_k$$

### Measurement Equation

The measurement is the received IQ sample, split into in-phase and quadrature components with added measurement noise $v_k \sim \mathcal{N}(0,R)$:

$$
y_k = h(x_k, s_k) + v_k
$$

$$
h =
\begin{bmatrix}
a_k * ( Re(s_k)*cos(θ_k) - Im(s_k)*sin(θ_k) ) \\
a_k * ( Re(s_k)*sin(θ_k) + Im(s_k)*cos(θ_k) )
\end{bmatrix}
$$

The transmitted symbol $s_k$ is assumed known at the receiver from pilot symbols. The process model is linear; the measurement equation is nonlinear, which justifies the use of the EKF rather over the standard Kalman filter.

---

## Simulation Parameters

| Parameter | Value |
|---|---|
| Number of symbols N | 500 |
| Symbol period T | 1×10⁻⁴ s |
| Symbol rate | 10,000 symbols/s |
| Initial phase offset $\theta_0$ | π/3 rad |
| Initial frequency offset $\omega_0$ | 2π×200 rad/s |
| Initial frequency drift $\alpha_0$ | 2π×0.5 rad/s² |
| Initial amplitude $a_0$ | 1.0 |
| SNR | 10 dB |

---

## Channel Distortion Simulations

Before applying the EKF, two open-loop scenarios were simulated in Simulink to understand the effect of the carrier offsets.

### No Synchronization ($u_k = 0$)

To establish a baseline, the system was first simulated with no synchronization applied, meaning $u_k=0$ for all $k$. In this scenario, no correction is applied to the receiver oscillator. This allows the carrier phase and frequency offset evolve freely according to the model.

![QPSK no sync](s11_single.png)

The received constellation shown in the figure depicts a ring of samples rather than clustering. This shape occurs because the CFO causes the constellation to rotate continuously, essentially "smearing" the samples uniformly around the unit circle. This behavior is expected when there is no synchronization since the receiver has no way of accounting for the phase rotation. Furthermore, the radial thickness of the ring reflects the amplitude variation in $a_k$ over the simulation.

### Ideal Impulse Correction

A unit impulse equal to the true frequency offset ω₀ was applied at k=0 — an unrealizable but instructive scenario. The frequency offset drops to near zero immediately, stopping the linear phase accumulation. The constellation now shows four distinct clusters, but they are rotated by the initial phase offset θ₀ = π/3 from the correct QPSK positions. This demonstrates a key limitation: frequency-only correction is insufficient. The initial phase offset has no mechanism to be removed by the control input `u_k` alone, motivating the need for full state estimation.

---

## Extended Kalman Filter

### Why EKF

The measurement function `h` is nonlinear in `θ_k` — the received IQ samples depend on `cos(θ_k)` and `sin(θ_k)`. The EKF handles this by linearizing `h` around the current state estimate at each time step using a Jacobian matrix, then applying standard Kalman filter equations to the linearized model. The process model is already linear, so no linearization is needed in the prediction step.

### Measurement Jacobian

Defining shorthand terms evaluated at the predicted state estimate:

$$
A_k = Re(s_k)*cos(θ̂_k) - Im(s_k)*sin(θ̂_k)
$$
$$
B_k = Re(s_k)*sin(θ̂_k) + Im(s_k)*cos(θ̂_k)
$$

The Jacobian of the measurement function with respect to the state vector is:

$$
H_k = \begin{bmatrix}
-a_k*B_k & 0 & 0 &  A_k \\\\
 a_k*A_k & 0 & 0 & B_k
\end{bmatrix}
$$

The zero columns for $\omega_k$ and $\alpha_k$ reflect that these states do not appear in the measurement equation directly — they are only observable indirectly through the accumulation of phase over time via the integrator chain in the process model.

### EKF Algorithm

The filter runs recursively at each symbol period, alternating between a measurement update and a time propagation step.

**Initialization** — The filter starts cold with no prior knowledge of phase or frequency:

$$
\hat{x}_0 = [0, 0, 0, a_{nom}]
$$
$$
P_0 = diag([(\pi/2)^2, (2 \pi * 500)^2, (2 \pi  * 5)^2, 0.25])
$$

**Step 1 — Measurement Update:**

$$
K_k   = P⁻_k * H_kᵀ * (H_k * P⁻_k * H_kᵀ + R)⁻¹
$$
$$
x̂⁺_k = x̂⁻_k + K_k * (y_k - h(x̂⁻_k, s_k))
$$
$$
P⁺_k  = (I - K_k * H_k) * P⁻_k
$$

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