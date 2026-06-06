# Signal Processing Pipeline

## Overview
Processing is split between the **edge FPGA on each Board A** and a **central
GPU/CPU host**. The edge FPGA does the bandwidth-reduction work (Hilbert IQ
generation + decimation) immediately after the ADC; decimated complex IQ is
sent over fibre to the host, which does range/Doppler/beamforming/CFAR. There
is no central FPGA and no RFSoC. See
`rf-frontend/radar-rx-frontend-edge-digitization.md` §2.3, §4.6, §7.

---

## Edge Processing (Lattice ECP5 on Board A)
This is the bandwidth-reduction stage — it must live at the edge because raw
samples across all boards are untenable to centralise (~448 Gbps at 32 boards).

### Hilbert IQ generation
- The dechirp mixer is real; a ~64-tap Hilbert FIR generates complex IQ from
  the real ADC samples, with >60 dB image rejection.

### Decimation
- CIC or half-band FIR, ~4× nominal (the beat bandwidth is far below the ADC
  Nyquist, so decimation costs little and recovers ENOB).
- Processing gain: ~3 dB per octave of oversampling.

### Data rate after edge processing
- 8 ch × 14-bit × 2 (I+Q) × ~31 MSPS ≈ **7 Gbps → 10G SFP+** (at 4× decimation)
- ~3.5 Gbps at 8× decimation → 5G SFP+
- One fibre per Board A into a multi-port 10G/25G NIC on the host.

---

## Phase Coherence (no MTS)
Coherence comes from a shared **OCXO master clock** to every Board A plus a
**startup tone calibration**: the host injects a known tone, each board measures
its per-channel phase offset, and fixed correction coefficients are stored in
firmware. See `rf-frontend/radar-rx-frontend-edge-digitization.md` §2.4.

---

## Host Processing (GPU/CPU workstation — settled)
The host receives decimated complex IQ over fibre and runs (CUDA):

1. **Per-channel phase calibration** — apply the stored startup-tone coefficients
2. **TX channel separation** — digital bandpass per FDM slot; slow-time
   correlation against Hadamard codes for CDM
3. **Fast time (range)** — range FFT per channel
4. **Slow time (Doppler)** — Doppler FFT per channel
5. **Spatial (beamforming)** — digital beamforming across the full virtual aperture
6. **CFAR** — detection

A central FPGA is only revisited if very low detection latency becomes a hard
requirement at large scale.

---

## Open / Pending
- [ ] IQ word width on the fibre link (affects per-board data rate)
- [ ] PRI and waveform parameters; pulses per CPI
- [ ] Real-time vs offline host processing requirement
- [ ] Stage 4 (32-board) host aggregation: multi-NIC vs SmartNIC vs central FPGA
