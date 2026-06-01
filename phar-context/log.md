# Design Decisions Log

Chronological record of major decisions and the reasoning behind them.
Load this file when reviewing why something was chosen or to avoid
re-litigating already-settled decisions.

---

## Decision 1: ZCU208 as Primary Platform
**Chosen over:** ZCU216, VCU118 + AD9088, VCK190 + AD9088, AD9088 + PolarFire

**Reasons:**
- 14-bit ADC resolution — only platform achieving this at ≥5 GSPS
- Single board — no JESD204C bring-up, no FMC+ mating complexity
- 8 ADC + 8 DAC channels maps exactly to 8 TX + 8 RX requirement
- Multi-Tile Synchronisation (MTS) for phase-coherent phased array
- PYNQ pre-built images — can stream data within days of hardware receipt
- Vivado license included with kit
- Monolithic integration advantage explains why 14-bit is achievable
  (no PCB interface noise, shared clock, no JESD serialisation loss)

**Why not VCU118 + AD9088:**
- AD9088 is 12-bit (vs 14-bit) — 12 dB dynamic range penalty
- AD9088 is preliminary silicon (Feb 2026 datasheet), uncharacterised
- Two-board system requires weeks of JESD204C bring-up
- AD9088 availability uncertain

**Why not VCK190:**
- Also would require discrete ADC (12-bit ceiling on available parts)
- More expensive than ZCU208 at $13,000

**Why not PolarFire:**
- Icicle Kit (MPFS250T, FCVG484 package) has only 4 SerDes lanes
- Insufficient for AD9088's 24 JESD204C lanes
- Larger PolarFire devices have no suitable off-the-shelf eval board
- Hard PCIe limited to Gen2 x4 (~10 Gbps) — inadequate for 96 Gbps streaming

---

## Decision 2: Bistatic Architecture (8 TX + 8 RX)
**Chosen over:** Monostatic (shared TX/RX antenna with circulator)

**Reasons:**
- Eliminates circulator requirement at X-band
- Better TX/RX isolation
- Enables simultaneous transmit and receive (no dead time)
- ZCU208 has exactly 8 ADC + 8 DAC channels — natural fit

---

## Decision 3: Fixed LO (if X-band path chosen)
**Chosen over:** Swept LO, agile frequency hopping

**Reasons:**
- Pulse-to-pulse phase coherence is automatic — every pulse sees identical LO phase
- No frequency settling time between pulses
- Simpler hardware — no VCO, no fast switching PLL
- Fixed LO phase offset per mixer is calibratable once

---

## Decision 4: Single-Conversion Architecture (if X-band path chosen)
**Chosen over:** Double conversion, IQ mixing at RF

**Reasons:**
- Minimum component count for prototype
- One mixer stage, one LO, one set of filters per channel
- Image rejection achievable with good RF BPF at 10.25 GHz
- Double conversion adds complexity and second LO without clear benefit
  at prototype stage

---

## Decision 5: Phased Two-Phase Prototyping Approach
**Decided:** Phase 1 = single TX + single RX, modular SMA bench assembly.
Phase 2 = full 16 channels on custom Rogers 4350B PCB.

**Reasons:**
- No professional radar team goes directly to 16-channel custom PCB
- Phase 1 validates RF chain, gain budget, mixer performance, ZCU208 interface
  before committing to PCB design
- Modular bench assembly allows component swaps when something underperforms
- Phase 2 PCB design benefits from knowing exactly what works from Phase 1

---

## Open Decisions (Unresolved)

### Frequency Band
Two viable paths remain open:
- **Option A:** Direct sampling, S-band (~3.1–3.5 GHz), no mixing
- **Option B:** Single conversion, X-band (10.0–10.5 GHz), 8 GHz fixed LO

Key trade-off: Option A is simpler and cheaper (~$5,000 less); Option B gives
better radar performance (higher carrier frequency, smaller antennas, better
angular resolution) but is significantly more complex to build.

### Host Processing Architecture
Explored candidates include GPU with cuFFT, FPGA fabric, and CPU/numpy for
offline use. No decision made. Key questions: real-time vs offline requirement,
latency constraints, development complexity.
See `signal-processing/pipeline.md` for full breakdown.

### IQ Word Width for Streaming
12-bit, 16-bit, or 32-bit float — affects data rate and host memory bandwidth.
Decision depends partly on host processing architecture (also undecided).

### Beamforming Location
FPGA fabric (lower latency, harder to modify) vs host (more flexible).
