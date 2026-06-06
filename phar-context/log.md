# Design Decisions Log

Chronological record of major decisions and the reasoning behind them.
Load this file when reviewing why something was chosen or to avoid
re-litigating already-settled decisions.

---

## Decision 1: ZCU208 as Primary Platform  ⚠️ SUPERSEDED (see Decision 6)
**Chosen over:** ZCU216, VCU118 + AD9088, VCK190 + AD9088, AD9088 + PolarFire

> **Reversed June 2026.** The move to analogue dechirp at the antenna means the
> ADC samples a low-frequency beat signal, so the high-GSPS monolithic-RFSoC
> requirement that justified this choice disappeared. Replaced by distributed
> commodity ADCs + ECP5 edge FPGA — see Decision 6.

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

## Decision 4: Single-Conversion Architecture (if X-band path chosen)  ⚠️ SUPERSEDED (see Decision 7)
**Chosen over:** Double conversion, IQ mixing at RF

**Reasons:**
- Minimum component count for prototype
- One mixer stage, one LO, one set of filters per channel
- Image rejection achievable with good RF BPF at 10.25 GHz
- Double conversion adds complexity and second LO without clear benefit
  at prototype stage

> **Reversed June 2026.** Replaced by analogue dechirp at the antenna: the
> reference chirp drives the mixer LO directly and the ADC sees the baseband
> beat signal — no fixed LO, no IF stage. See Decision 7.

---

## Decision 5: Phased Two-Phase Prototyping Approach  ⚠️ SUPERSEDED (see Decision 9)
**Decided:** Phase 1 = single TX + single RX, modular SMA bench assembly.
Phase 2 = full 16 channels on custom Rogers 4350B PCB.

> **Reversed June 2026.** The bench-on-ZCU208 phasing is replaced by the Board A
> staging/scaling path (Stage 1 = one 8-RX board, tiling to 32 boards) and the
> Stage 1 → Stage 2 ADC progression — see Decision 9.

**Reasons:**
- No professional radar team goes directly to 16-channel custom PCB
- Phase 1 validates RF chain, gain budget, mixer performance, ZCU208 interface
  before committing to PCB design
- Modular bench assembly allows component swaps when something underperforms
- Phase 2 PCB design benefits from knowing exactly what works from Phase 1

---

---

# Architecture Pivot — June 2026

Decisions 6–10 record the move from the ZCU208-centred design to the settled
analogue-dechirp / digitise-at-antenna / fibre-transport architecture. Full
detail and the candidate analysis live in
`rf-frontend/radar-rx-frontend-edge-digitization.md`.

## Decision 6: Distributed Edge Digitisation (replaces Decision 1)
**Chosen over:** Central RFSoC (ZCU208) digitiser

**Reasons:**
- With dechirp at the antenna the ADC sees only a DC–~100 MHz beat signal, so a
  commodity 14-bit pipeline ADC (LTC2145-14 at 125 MSPS → ADC3668 at 250 MSPS)
  is more than adequate — the high-GSPS monolithic-RFSoC requirement vanishes
- Each self-contained "Board A" carries 8 RX, its RF chain, 4 dual ADCs, and a
  Lattice ECP5 edge FPGA; boards tile 1→32 (8→256 RX)
- Open-source FPGA toolchain (Yosys + nextpnr + openFPGALoader), no licence
- Far cheaper and more scalable than one expensive central RFSoC

## Decision 7: Analogue Dechirp at the Antenna (replaces Decision 4)
**Chosen over:** Direct RF sampling; downconvert-to-IF then sample

**Reasons:**
- Sampling the raw carrier wastes ENOB to aperture jitter; dechirping in the
  analogue domain so the ADC sees the baseband beat signal is the single
  highest-impact design choice
- The reference chirp drives the mixer LO directly — no fixed LO, no IF stage
- A passive GaAs mixer adds no phase noise (vs an active-LO-buffer part)

## Decision 8: Reject Analogue Channel Multiplexing
**Chosen over:** Subcarrier multiplexing (SCM) over coax; WDM / RF-over-fibre

**Reasons:**
- SCM third-order intermodulation lands on adjacent channels and floors weak
  returns against strong ones; the problem scales as N³ — fundamentally unsuited
  to radar's 60–80 dB near/far dynamic range
- WDM does not reduce ADC count
- Digitising per channel at the antenna and transporting digital samples over
  fibre eliminates intermodulation and scales cleanly

## Decision 9: Board A Staging and Scaling Path (replaces Decision 5)
**Decided:** Stage 1 = one 8-RX Board A (LTC2145-14, 125 MSPS); Stage 2 = board
respin to ADC3668 (250 MSPS + on-chip DDC), preserving the ECP5 module and RF
front-end. Scale by tiling Board A: 1 → 4 → 16 → 32 boards (8 → 256 RX).

**Reasons:**
- A single self-contained board validates the full RF + ADC + FPGA + fibre chain
  before committing to a multi-board aperture
- The ADC upgrade is the main performance lever and is isolated to a respin

## Decision 10: GPU/CPU Host Processing; No Central FPGA (resolves old open items)
**Decided:** Heavy DSP (range/Doppler/beamforming/CFAR) runs on a GPU/CPU host
over CUDA; the edge FPGA does only Hilbert + decimation (bandwidth reduction).
Beamforming runs on the host. A central FPGA is revisited only if very low
detection latency becomes a hard requirement at large scale.

**Reasons:**
- Raw data across 32 boards is ~448 Gbps — untenable to centralise before
  reduction, which is why decimation must live at the edge
- After edge reduction, per-board rates (~3.5–7 Gbps) fit a standard 10G NIC
- Host processing is maximally flexible for algorithm development

**Still open:** IQ word width on the fibre link; PRI / pulses-per-CPI; real-time
vs offline host requirement; Stage 4 (32-board) host aggregation approach.
