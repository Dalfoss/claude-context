# FMCW MIMO Radar — System Architecture

**Band:** 10.0 – 10.5 GHz  
**Architecture:** FDMA/CDM · Analogue Dechirp · Digitise at Antenna · Fibre Transport  
**Version:** 0.4 · June 2026

---

## Table of Contents

1. [Introduction and System Goals](#1-introduction-and-system-goals)
2. [Architecture Decision Trail](#2-architecture-decision-trail)
3. [System Architecture Overview](#3-system-architecture-overview)
4. [Board A — RF Front-End](#4-board-a--rf-front-end)
5. [Waveform Strategy](#5-waveform-strategy)
6. [TX Board](#6-tx-board)
7. [Central Unit](#7-central-unit)
8. [Scaling Path](#8-scaling-path)
9. [TX/RX Element Placement and FDMA Compatibility](#9-txrx-element-placement-and-fdma-compatibility)
10. [Stage 2 ADC — TI ADC3668](#10-stage-2-adc--ti-adc3668)
11. [Bill of Materials and Cost Estimate — Board A Stage 1](#11-bill-of-materials-and-cost-estimate--board-a-stage-1)
12. [Open Items and Next Steps](#12-open-items-and-next-steps)

---

## 1. Introduction and System Goals

This document describes the hardware and signal-chain architecture for a software-defined FMCW MIMO radar operating in the 10.0–10.5 GHz band. The primary application is small UAV (drone) detection and tracking at ranges up to 500 m.

### 1.1 Primary Objectives

- **Maximum receive channel count.** Scale to 256 RX elements (8 per board initially) to maximise the MIMO virtual aperture.
- **High dynamic range.** Detect weak distant targets simultaneously with strong nearby returns.
- **Software-defined waveform.** All waveform parameters are controlled in firmware; hardware is a transparent analogue front-end.
- **Modular, scalable design.** Board A is self-contained and tiles cleanly. Multiple boards feed a central processing unit over fibre.
- **Physical separation.** Central processing can be located remotely via fibre.

### 1.2 Key Specifications

| Parameter | Value / Target |
|---|---|
| RF band | 10.0 – 10.5 GHz |
| Chirp bandwidth | Up to 500 MHz (waveform-defined) |
| Range resolution | ≈ 1.5 m at 100 MHz total BW via joint processing |
| Max target range | 500 m |
| TX channels | 8 (initial) |
| RX channels per board | 8 |
| Stage 1 ADC | 14-bit, 125 MSPS → ~12.0 ENOB |
| Stage 2 ADC | 16-bit, 250 MSPS + DDC → ~13.8–14.3 ENOB |
| RF substrate | Rogers 4003C, 0.508 mm core |

---

## 2. Architecture Decision Trail

### 2.1 Subcarrier Multiplexing — Rejected

The initial architecture used SCM to transport multiple beat signals over a single coaxial cable to a central ADC. Third-order intermodulation products from the SCM chain land exactly on adjacent channel frequency slots for evenly spaced subcarriers. With 60–80 dB near/far dynamic range, this makes SCM fundamentally unsuited to radar. The problem scales as N³ with channel count. SCM was eliminated.

> **Post-mixer LPF constraint:** The dechirp mixer produces a beat signal (DC to ~500 MHz) and a sum term at ~20.5 GHz. No mixing product other than the fundamental difference term falls below 500 MHz, so the post-mixer LPF cutoff is fixed by the RF band choice, not the waveform design. The only Board A hardware parameter that changes with the channel plan is the anti-aliasing cutoff, which tracks the ADC sample rate.

### 2.2 Settled Architecture: Digitise at the Antenna

Beat signals are digitised at the antenna board immediately after the dechirp mixer and low-pass filter. Digital samples are transported over fibre. This eliminates intermodulation, scales cleanly, and is maximally software-defined.

### 2.3 Processing Architecture — No Central FPGA

All heavy processing (Hilbert transform, decimation, FFTs, beamforming) runs on a host GPU/CPU workstation. A central FPGA would need to receive raw samples from all boards before any reduction. At full scale (32 boards):

```
32 boards × 8 channels × 14 bits × 125 MSPS = 448 Gbps raw
```

No central FPGA handles this. Decimation and Hilbert must live at the edge — they are the bandwidth reduction step. After edge processing, data rates are tractable for a standard 10G NIC into a GPU workstation.

### 2.4 ADC Phase Coherence

Coherence is achieved by: (1) a shared OCXO master clock distributed to every Board A, eliminating frequency variation while leaving a fixed power-up phase offset per board; and (2) startup calibration — the central processor injects a known tone, each Board A measures its per-channel phase offset, and fixed correction coefficients are stored in firmware.

---

## 3. System Architecture Overview

### 3.1 Signal Flow

```
OCXO (100 MHz) ──┬── Chirp generator ── TX board ── TX antennas
                 │
                 ├── Reference chirp ── Board A LO SMA (see note)
                 │
                 └── Master clock ── Board A ADC clock input

Board A (per channel):
  [Patch] → [BPF] → [Limiter] → [LNA] → [BPF] → [Dechirp mixer] → [LPF] → [ADC]
                                                         ↑
                                                  ref chirp (LO)
  [ADC] → [FPGA: Hilbert + Decimate] → [SerDes] → Fibre → GPU workstation
```

### 3.2 Reference Chirp as LO

The reference chirp is a copy of the transmitted chirp at 10.0–10.5 GHz. When it drives the dechirp mixer LO port, the output is the beat signal `f_beat = kτ`. Board A routes it directly to the mixer LO port. Required LO level depends on mixer choice (see Section 4.3.4).

### 3.3 Interface Summary

| Interface | Signal | Direction | Connector |
|---|---|---|---|
| Reference chirp LO | 10.0–10.5 GHz; +15 dBm (MAMX-011035) or 0 dBm (LTC5548) | TX → Board A | SMA edge-launch |
| ADC master clock | 100 MHz OCXO-derived | Central → Board A | SMA or MCX |
| Fibre data link | Decimated complex IQ | Board A → Central | SFP+ cage |
| DC power | 3.3 V + 1.8 V | PSU → Board A | Molex or header |

---

## 4. Board A — RF Front-End

Board A is the RF front-end PCB. It houses 8 RX antenna elements, the full analogue receive chain, ADC array, and an FPGA module for data processing and fibre output.

### 4.1 Substrate and Fabrication

| Parameter | Value |
|---|---|
| Substrate | Rogers 4003C |
| Core thickness | 0.508 mm |
| Copper weight | 35 µm (1 oz) |
| Dielectric constant | 3.55 @ 10 GHz |
| Loss tangent | 0.0027 @ 10 GHz |
| 50 Ω trace width | ≈ 1.09 mm |
| Fabrication | Würth Elektronik (controlled impedance, Rogers capability) |

### 4.2 Antenna Array

- Type: microstrip patch, linear 1×8
- Centre frequency: 10.25 GHz
- Element spacing: λ/2 = 14.6 mm
- Total aperture: 116.8 mm
- Feed: corporate microstrip (equal-path-length)
- Substrate: Rogers 4003C (same board as RF chain)

### 4.3 RF Signal Chain (per RX channel)

#### 4.3.1 Pre-LNA Bandpass Filter

2-pole parallel-coupled microstrip BPF etched on Rogers 4003C. Rejects far-out-of-band interference before the LNA.

| Parameter | Value |
|---|---|
| Type | 2-pole parallel-coupled microstrip |
| Passband | 10.0 – 10.5 GHz |
| Insertion loss | ~0.5 – 0.8 dB |
| NF penalty | ~0.1 dB (negligible) |
| Implementation | PCB copper — no discrete components |

#### 4.3.2 PIN Limiter

Placed between the pre-LNA BPF and the LNA to protect the LNA from transmit leakage, nearby high-power interferers, or accidental close-range illumination. The BPF is upstream of the limiter — passive distributed filters are inherently robust to high power and benefit from seeing a reduced out-of-band spectrum before the limiter.

| Parameter | Value |
|---|---|
| Part | ADI ADL8111 (or equivalent PIN limiter) |
| Limiting threshold | +13 dBm |
| Insertion loss | ~0.8 dB |
| Recovery time | < 1 µs |
| NF penalty | ~0.8 dB (directly adds to system NF ahead of LNA) |
| Package | SOT-363 |

The 0.8 dB insertion loss adds directly to the system noise figure ahead of the LNA. At 0.95 dB LNA NF, the combined front-end NF becomes approximately 1.75 dB — acceptable for a 500 m drone detection system.

#### 4.3.3 LNA

| Parameter | Value |
|---|---|
| Part | Qorvo CMD319C3 |
| Frequency | 8 – 12 GHz |
| Noise figure | 0.95 dB (0.92 dB per Qorvo datasheet @ 10 GHz) |
| Gain | 20–24 dB (band-dependent; 20 dB @ 10 GHz typical) |
| P1dB (output) | +16 dBm |
| Supply | 3.3 V, 30 mA |
| Package | 16-QFN, 3×3 mm |
| Lead time | **35 weeks factory** — order from current Digikey stock (455 units in stock) |

> **Stock warning:** Factory lead time is 35 weeks. 40 units are needed for the 5-board run (5 × 8 channels). Order from current Digikey stock immediately; do not assume reorder is possible on short notice.

**Why CMD319C3 over alternatives:** Every competitive X-band LNA in SMT packaging accepts a noise figure penalty to cover a wider frequency range. The CMD328K3 (6–18 GHz) achieves 1.4 dB NF at 10 GHz; CMD264P3 (6–18 GHz) achieves 1.6 dB; HMC1049 (0.3–20 GHz) achieves 1.7 dB at 7 V / 70 mA. The CMD319C3, optimised specifically for 8–12 GHz, achieves 0.95 dB — nearly half a dB better than the next best option. Since the LNA noise figure dominates the system by Friis, this directly sets the radar sensitivity floor.

**System NF estimate:**
```
Pre-LNA BPF (0.7 dB IL) + CMD319C3 (0.95 dB NF, 20 dB gain):
  F_system ≈ 1.175 + (1.245 − 1) = 1.42 → NF ≈ 1.52 dB
All downstream stages contribute < 0.1 dB total after 20 dB LNA gain.
```

The 50Ω internal matching eliminates external matching networks — a meaningful simplification for an 8-channel Board A layout. Single positive 2–5 V supply.

Every millimetre of PCB trace between antenna feed and LNA input adds ~0.05 dB to NF. Keep under 3 mm.

#### 4.3.4 Post-LNA Bandpass Filter

5-pole interdigital BPF on Rogers 4003C. Provides image rejection before the dechirp mixer. Insertion loss here contributes less than 0.05 dB to system NF (divided by ~251 by Friis at 24 dB LNA gain).

| Parameter | Value |
|---|---|
| Type | 5-pole interdigital microstrip |
| Passband | 10.0 – 10.5 GHz |
| Insertion loss | 1.5 – 2.5 dB (irrelevant to NF) |
| Image rejection | ~40 dB |

#### 4.3.5 Dechirp Mixer

A passive GaAs mixer is used. RF port receives the filtered signal; LO port receives the pre-amplified reference chirp; IF output is the beat signal.

**Primary choice — MACOM MAMX-011035-TR0100**

| Parameter | Value |
|---|---|
| Technology | GaAs passive double-balanced diode |
| RF/LO range | 5.5 – 19 GHz |
| IF bandwidth | DC – 6 GHz (DC-coupled) |
| Conversion loss | ~6–7 dB @ 10.25 GHz (measured, mid-band) |
| IIP3 | **+20 dBm** @ 10.25 GHz (datasheet, 10–19 GHz sub-band) |
| LO drive required | **+15 dBm** |
| LO-IF isolation | **45 dB** (10–19 GHz sub-band) |
| DC power | **Zero** (fully passive) |
| Phase noise added | **Zero** (no active elements in LO path) |
| Package | 12-QFN, 3×3 mm |
| Price | ~$61.33 (Mouser, stock confirmed) |

10.25 GHz is mid-band for this part. All specs are measured datasheet values at our operating sub-band, not extrapolated.

> **LO drive:** The MAMX-011035 requires +15 dBm at its LO SMA. For N Board A instances, a passive 1:N splitter loses ~10 log(N) dB, requiring approximately +25 dBm into the splitter for 8 boards. A dedicated LO driver PA stage on the TX board is required.

> **IF port:** Single-ended, DC-coupled. Connects to the ADC differential input via a wideband balun or DC-coupled single-ended drive.

---

**Budget option — ADI LTC5548IUDB#TRMPBF**

| Parameter | Value |
|---|---|
| Technology | SiGe BiCMOS — passive mixer core + integrated active LO buffer |
| RF/LO range | 2 – 14 GHz |
| IF bandwidth | DC – 6 GHz (differential, DC-coupled) |
| Conversion loss | ~9.5–10 dB @ 10.25 GHz (extrapolated, upper band) |
| IIP3 | ~+20 dBm @ 10.25 GHz (extrapolated) |
| LO drive required | **0 dBm** (on-chip LO buffer) |
| DC power | **~400 mW/chip** (3.3 V, 120 mA) |
| Phase noise added | Yes — active SiGe LO buffer degrades reference chirp phase noise |
| Package | 12-QFN, 3×2 mm |
| Price | $24.01 (Digikey, ships today) |

LTC5548 is cheaper and eliminates the TX board LO PA. Tradeoffs: ~3 dB worse conversion loss at 10.25 GHz, active LO buffer degrades Doppler floor, 3.2 W extra heat across 8 channels. Acceptable for a prototype where Doppler sensitivity is not the primary concern.

> **LO drive summary:** MAMX-011035 needs +15 dBm at each LO SMA (TX board PA required). LTC5548 needs 0 dBm (passive splitter only). Switching between them does not affect Board A layout — the LO SMA footprint is identical.

---

**Future upgrade — Marki MM1-0115HPSM-2**

+24 dBm IIP3 mid-band (10.25 GHz is mid-band for 1–15 GHz part), 54 dB LO-RF isolation, 8 dB conversion loss, passive GaAs, zero phase noise degradation. ~$80–150 via RFMW or Marki direct. Recommended if a future revision must push the Doppler floor lower. The LTC5548 active LO buffer is the binding limit on Doppler sensitivity.

#### 4.3.6 Post-Mixer Low-Pass Filter

Lumped-element LC LPF immediately after the mixer IF port. Anti-aliasing cutoff set to approximately half the ADC sample rate.

> **Part constraint:** For LTC2145-14 at 125 MSPS, the anti-aliasing cutoff is ~55–60 MHz. Suitable part: Mini-Circuits LFCN-80+ (DC–80 MHz, SMD). When the ADC is upgraded for Stage 2, this component is swapped as a matched pair — same PCB footprint.

### 4.4 IQ Sampling via Hilbert Transform

The dechirp mixer is a real (non-IQ) mixer. The FPGA implements a ~64-tap Hilbert FIR filter to generate complex IQ from real ADC samples. Image rejection exceeds 60 dB — superior to hardware IQ balance (~25–40 dB). The GPU workstation receives complex IQ and applies FFTs directly.

### 4.5 ADC Array

#### 4.5.1 Selection Rationale

All 14-bit pipeline ADCs at 100–125 MSPS cluster around 73–75 dBFS SNR — this is a physics constraint on the technology tier, not a design differentiator. The key discriminators are SFDR, power consumption, and upgrade path.

For our DC to 40 MHz beat signal, ENOB is SNR-limited at approximately 12.0 bits. SFDR is not the binding ENOB constraint at these frequencies, but better SFDR reduces ADC harmonic spurs that appear as false targets.

#### 4.5.2 Stage 1 Part: ADI LTC2145IUP-14#PBF

| Parameter | Value |
|---|---|
| Part | LTC2145IUP-14#PBF |
| Architecture | Dual-channel 14-bit pipeline |
| Sample rate | 125 MSPS |
| SNR | 73.1 dBFS |
| SFDR | **90 dBc** |
| ENOB (raw, 40 MHz IF) | ~12.0 bits |
| ENOB (with 5× oversampling + decimation) | ~13.1 bits |
| Aperture jitter | 0.08 ps RMS |
| Power | **95 mW/channel** (380 mW total, 4 chips) |
| Output interface | DDR LVDS |
| Package | 64-QFN, 9×9 mm |
| Supply | 1.8 V |
| Price | ~$125/chip (10+ units) |

**Why LTC2145-14 over AD9648-125 (same price ~$140):**

The AD9648 was the initial choice for its pin-compatible 16-bit upgrade path (AD9268/AD9650). However, Stage 2 uses the TI ADC3668 which requires a board respin regardless. Footprint continuity within the ADI family therefore no longer justifies the trade-off. At equal price the LTC2145-14 wins on:

- **SFDR: 90 dBc vs ~88 dBc** — fewer ADC harmonic spurs as false targets
- **Power: 95 vs ~175 mW/channel** — 320 mW less across 4 chips; important for Rogers board thermal management
- **Aperture jitter: 0.08 ps RMS** — effectively eliminates jitter noise at 40 MHz beat frequency

SNR is 1 dBFS lower (73.1 vs ~74 dBFS) — negligible ENOB difference.

#### 4.5.3 Stage 1 vs Stage 2 ADC Comparison

| Parameter | LTC2145-14 (Stage 1) | ADC3668 (Stage 2) |
|---|---|---|
| Rate | 125 MSPS | 250 MSPS |
| Bits | 14 | 16 |
| SNR raw | 73.1 dBFS | ~79 dBFS |
| ENOB with DDC/decimation | ~13.1 bits | ~13.8–14.3 bits |
| Interface | Parallel DDR LVDS | Serial LVDS (with DDC) |
| Phase noise | Passive (none added) | Passive (none added) |
| Price | ~$140 | ~$200 |

#### 4.5.4 LVDS Interface

The LTC2145-14 uses DDR LVDS output. Both channels share one set of output pairs via time-multiplexing (DDR clocking), identical in concept to the AD9648 interface. Per chip:

```
14 LVDS pairs (channels A+B multiplexed, DDR)  = 28 pins
DCO (data clock) + FCO (frame clock)           =  4 pins
Total per chip:                                = 32 pins
4 chips × 32 pins = 128 ADC data pins
```

The FPGA captures both channels using DDR input registers; FCO identifies which edge carries which channel.

#### 4.5.5 Board Configuration

- 4 chips per Board A (dual-channel each) → 8 RX channels
- Supply: 1.8 V
- Total ADC power: 4 × 190 mW = **760 mW**

### 4.6 Board A FPGA

#### 4.6.1 Selected Part: LFE5UM5G-85F-8BG381C (Lattice ECP5-85F)

| Parameter | Value |
|---|---|
| LUTs | 84,000 |
| DSP blocks | 156 |
| User I/O (CABGA381) | 365 |
| Hard SerDes | 4 channels at 5 Gbps |
| Toolchain | Yosys + nextpnr + openFPGALoader (open source) |
| Package | CABGA381, 21×21 mm, 0.8 mm ball pitch |
| Price | ~$80–120 |

**I/O budget:** 128 ADC data pins + 4 SFP+ SerDes + 4 SPI + 2 clock + 15 power = ~153 signal pins. CABGA381 provides 365 user I/O — fits with ~210 spare.

**FPGA tasks on Board A:**

| Task | Implementation |
|---|---|
| ADC LVDS DDR capture | IDDRX1F primitives on LVDS I/O; DCO as capture clock; FCO for channel de-interleaving |
| Hilbert FIR (per channel) | ~64-tap FIR, time-multiplexed across DSP blocks; generates IQ from real ADC samples |
| Decimation | CIC or half-band FIR; 4× nominal |
| ADC SPI configuration | One-time startup; trivial resource cost |
| Data framing | Frame IQ samples with channel ID and timestamp |
| SerDes TX | ECP5 hard SerDes; 5 Gbps to SFP+ cage |

#### 4.6.2 Data Rate Budget

```
After Hilbert + 4× decimation on Board A:
  8 ch × 14 bit × 2 (I+Q) × ~31 MSPS = 7 Gbps → 10G SFP+

After Hilbert + 8× decimation:
  8 ch × 14 bit × 2 (I+Q) × ~16 MSPS = 3.5 Gbps → 5G SFP+
```

#### 4.6.3 FPGA Integration: Custom Mezzanine Module

**Why not direct BGA on Board A:** The 381-ball BGA at 0.8 mm pitch requires 6–8 PCB layers for signal escape. Rogers 4003C in standard RF configuration is 4 layers. BGA fanout vias pierce the RF ground planes, disrupting isolation. Mixed Rogers/FR4 stackups are expensive speciality fabrication.

**Module approach:** A separate FR4 FPGA module eliminates these problems. Board A stays a clean 4-layer Rogers board. The FPGA module is FR4, assembled independently at JLCPCB Advanced. Risk is isolated — RF bring-up and FPGA bring-up proceed independently.

**Connector:** Hirose DF40 mezzanine (0.4 mm pitch, characterised to several GHz — well above 125 MHz LVDS requirements). Two 80-position DF40 connectors per interface provide 160 signal positions covering all 153 required pins with 7 spare. Price: ~$6–10 per mated pair. Stack height: 1.5 mm.

**Commercial SoM survey result — no suitable off-the-shelf option:**

| Board | Chip | SERDES | Embeddable | Pin count | Price |
|---|---|---|---|---|---|
| ButterStick | LFE5UM5G-85F ✓ | ✓ | No (80×49 mm dev board) | ~20 diff pairs via SYZYGY | $180 |
| OrangeCrab 85F | LFE5U-85F (no SERDES) | ✗ | Partly (castellated) | ~40 | ~$60 |
| SoldierCrab | LFE5U-25K/45K | ✗ | Yes (M.2) | ~47 | $89 |
| **Custom module** | **LFE5UM5G-85F ✓** | **✓** | **Yes (DF40)** | **160** | **~$150–200** |

**Custom module BOM:**

| Component | Part | Cost |
|---|---|---|
| FPGA | LFE5UM5G-85F-8BG381C | ~$100 |
| Core regulator 1.1 V | TPS62130 | ~$3 |
| I/O LDO 1.8 V | TLV75801 | ~$2 |
| Config flash | W25Q128 | $1 |
| DF40 plugs ×2 | DF40C-80DP-0.4V | $8 |
| Passives | — | $5 |
| FR4 PCB (JLCPCB, 5×) | — | ~$3/module |
| Assembly (JLCPCB Advanced, 5×) | — | ~$20/module |
| **Module subtotal** | | **~$142** |

ButterStick KiCad open-source files serve as reference for the FPGA power, decoupling, and configuration flash sections.

### 4.7 Board A Physical Layout

Signal flows top-to-bottom. Antennas at top edge, fibre at bottom edge.

```
┌─────────────────────────────────────────────────┐
│  ANTENNA SECTION                                │
│  8× patch elements + corporate microstrip feed  │
├─────────────────────────────────────────────────┤
│  X-BAND RF SECTION                              │
│  BPF → LNA → BPF → Dechirp mixer (×8)          │
│  Via fence every λ/20 ≈ 1.5 mm along all edges  │
│  Vias under all MMICs; LNA input trace < 3 mm   │
├─────────────────────────────────────────────────┤  ← LO SMA (one side)
│  IF / ADC SECTION                               │
│  Post-mixer LPF → LTC2145-14 array (×4)         │
│  Separate from X-band by continuous ground wall  │
├─────────────────────────────────────────────────┤
│  DIGITAL SECTION                                │
│  FPGA module (DF40 connectors)                  │
│  SFP+ cage · Power · SPI lines far from LNA     │
└─────────────────────────────────────────────────┘
                       │
                  Fibre out (SFP+)
```

---

## 5. Waveform Strategy

All waveform parameters are controlled in firmware. Hardware is waveform-agnostic.

### 5.1 FDMA with Joint Processing

Cohen & Eldar ("High Resolution FDMA MIMO Radar", 2017) showed that FDMA with joint processing achieves range resolution corresponding to the **total** bandwidth, while relaxing the narrowband assumption to the per-channel bandwidth. With 8 TX channels each at 12.5 MHz/channel, joint processing recovers 1.5 m range resolution (equivalent to 100 MHz total bandwidth), while the ADC captures a beat spectrum to ~95 MHz.

**FDMA beat bandwidth by channel plan:**

| Scheme | B/channel | N_TX | Beat bandwidth | ADC needed |
|---|---|---|---|---|
| 4-FDM × 2-CDM | 12.5 MHz | 8 | ~50 MHz | 125 MSPS ✓ |
| 8-FDM | 12.5 MHz | 8 | ~100 MHz | 250 MSPS |
| 8-FDM wide | 62.5 MHz | 8 | ~500 MHz | ~1 GSPS |

> **Note:** The 500 MSPS ADC3669 has a 250 MHz Nyquist bandwidth. It cannot capture an 8-TX × 62.5 MHz FDMA beat spectrum (~500 MHz) — that requires a ~1 GSPS ADC, which is a Stage 3+ hardware class.

### 5.2 CDM

All TX channels transmit the full chirp bandwidth simultaneously, distinguished by Hadamard phase codes across chirp periods. Zero hardware changes on Board A. Limitation: narrowband assumption can be violated at large apertures and wide bandwidths.

### 5.3 Hybrid 4-FDM × 2-CDM — Recommended Stage 1

Four FDM frequency slots, each with 2 CDM-separated TX channels (length-2 Hadamard codes). Velocity ambiguity limit:

```
v_max = λ / (2 × N_CDM × T_chirp) ≈ 73 m/s   (well above drone speeds)
```

Total beat bandwidth ~50 MHz — comfortable for LTC2145-14 at 125 MSPS. Range resolution 1.5 m via joint processing.

---

## 6. TX Board

Not fully specified in this revision. Responsibilities:

- Generate 8 TX chirp signals in assigned frequency slots
- Amplify each TX channel to target radiated power
- Distribute reference chirp to all Board A instances

**LO distribution depends on mixer choice:**
- MAMX-011035 (primary): +15 dBm at each Board A SMA. For 8 boards: ~+25 dBm into 1:8 splitter — requires GaAs PA stage on TX board.
- LTC5548 (budget): 0 dBm at each Board A SMA. Passive splitter from modest source suffices.

**Chirp generator candidates:**
- PLL-based: ADF4159 (FMCW ramp controller) + HMC733 VCO
- DDS-based: AD9914 at IF + HMC-series upconverter

CDM encoding applied digitally at DAC output — TX board is waveform-agnostic.

---

## 7. Central Unit

### 7.1 Reference Oscillator

| Parameter | Value |
|---|---|
| Type | OCXO |
| Frequency | 100 MHz |
| Candidate | Abracon ABLJO-AT-100.000MHz or equivalent |
| Distribution | Passive 1:N coaxial splitter to all Board A instances and TX board |

OCXO preferred over TCXO — phase noise on the master clock degrades ADC aperture jitter and limits sensitivity.

### 7.2 GPU/CPU Workstation

The central unit is a GPU/CPU workstation receiving fibre links over a multi-port 10G or 25G NIC.

| Function | Implementation |
|---|---|
| Per-channel phase calibration | Startup tone correlation; store complex correction coefficients |
| TX channel separation (FDM) | Digital bandpass filter per TX slot |
| TX channel separation (CDM) | Slow-time correlation against Hadamard codes |
| Range FFT | Fast-time FFT per channel (CUDA) |
| Doppler FFT | Slow-time FFT per channel (CUDA) |
| Beamforming | Digital beamforming across full virtual aperture (CUDA) |
| CFAR | Standard CUDA kernel |

No dedicated central FPGA required. If very low detection latency becomes a hard requirement at large scale, a central FPGA can be added at that point.

---

## 8. Scaling Path

| Stage | Board A count | RX channels | TX config | Virtual elements | Fibre links |
|---|---|---|---|---|---|
| 1 | 1 | 8 | 8 TX co-located | 64 | 1 |
| 2 | 4 | 32 | 8 TX co-located | 256 | 4 |
| 3 | 16 | 128 | 8 TX distributed | 1,024 | 16 |
| 4 | 32 | 256 | 8 TX distributed | 2,048 | 32 |

---

## 9. TX/RX Element Placement and FDMA Compatibility

### 9.1 Range-Azimuth Coupling

FDMA range-azimuth coupling arises from the joint linear ordering of TX position and TX frequency. With TX element index *t*, position `x_t = t·d`, and frequency `f_t = f_0 + t·Δf`, the round-trip phase contains a cross term `∝ t² · Δf · d · sin(θ) / c` that couples range and angle. The fix is to break the linear relationship between TX index and position.

**This is a TX placement problem only.** The RX array being regular does not cause coupling.

### 9.2 RX Elements — Must Remain Uniform

The 8 RX patches within each Board A must stay at λ/2 = 14.6 mm spacing. Changing this introduces grating lobes and breaks the equal-path-length corporate feed. No modification is needed or permitted.

### 9.3 TX Elements — Full Placement Freedom

TX antennas are independent hardware. Positions are chosen by numerical optimisation (minimum redundancy array, coprime array, or simulated annealing) to break the `t²` coupling term. They must be known precisely for joint processing but do not need to be regular.

Example over a 1 m span:

```
Regular (causes coupling):
  0   125   250   375   500   625   750   875  mm

Irregular (breaks coupling):
  0    93   201   314   467   589   723   891  mm
```

### 9.4 Summary

| Level | Constraint | Flexibility | Cohen & Eldar relevance |
|---|---|---|---|
| RX within Board A | λ/2 uniform — fixed | None | Not relevant (coupling is TX-side) |
| RX between boards | Physical tiling | Can be irregular at Stage 3–4 | Minor benefit |
| TX elements | Independent hardware | Complete freedom | This is where irregularity is required |

Verify TX placement in simulation before committing to physical positions.

---

## 10. Stage 2 ADC — TI ADC3668

### 10.1 Part Selection

| Parameter | Value |
|---|---|
| Part | ADC3668IRTD |
| Architecture | Dual-channel 16-bit pipeline + on-chip DDC |
| Sample rate | 250 MSPS |
| Noise spectral density | −160 dBFS/Hz |
| SNR bypass | ~79 dBFS → 12.8 ENOB |
| SNR with 4× DDC | ~85 dBFS → **13.8 ENOB** |
| SNR with 8× DDC | ~88 dBFS → **14.3 ENOB** |
| Power | 250 mW/channel (500 mW total) |
| Output interface | Parallel DDR LVDS (bypass) or Serial LVDS (with decimation) |
| Package | 64-VQFN, 9×9 mm |
| Price | ~$200/chip |
| Availability | Digikey, ships today |

### 10.2 ADC3668 vs ADC3669 (500 MSPS)

Both share −160 dBFS/Hz NSD. At identical output bandwidth they produce identical ENOB:

```
ADC3668 (250 MSPS): SNR_bypass = 79 dBFS, Nyquist BW = 125 MHz
ADC3669 (500 MSPS): SNR_bypass = 76 dBFS, Nyquist BW = 250 MHz

At same output bandwidth via DDC:
ADC3668 (8× decimate):  79 + 9  = 88 dBFS → 14.3 ENOB
ADC3669 (16× decimate): 76 + 12 = 88 dBFS → 14.3 ENOB
```

ADC3669 burns 20% more power and costs more for zero ENOB benefit at Stage 2 beat bandwidths. ADC3669 becomes relevant only if a future FDMA scheme requires beat capture beyond 125 MHz (4-FDM × 2-CDM at 25 MHz per channel → 100 MHz beat). For Stage 2, **ADC3668 is correct.**

### 10.3 DDC as a Radar Processing Asset

The ADC3668 includes a quad-band DDC with a 48-bit NCO. In FDMA operation, the NCO tunes to each TX sub-band and the DDC shifts it to baseband before decimation — FDMA channel separation occurs on-chip. The FPGA receives pre-decimated complex IQ already separated by TX channel. This removes a significant processing stage from the FPGA firmware.

### 10.4 Serial LVDS Interface

With DDC and decimation enabled, the ADC3668 switches from parallel to serial LVDS:

```
At 4× decimation:
  Data rate per channel: ~1 Gbps per lane
  4 chips × 2 channels = 8 serial LVDS lanes
  (vs 128 parallel DDR pins in Stage 1)
```

8 serial lanes at 1 Gbps is well within ECP5 LVDS I/O capability. The Stage 2 board routes more easily than Stage 1 despite the faster ADC.

### 10.5 Stage 2 Requires a Board Respin

ADC3668 is not pin-compatible with LTC2145-14. Stage 2 is a new Board A. The FPGA module (ECP5-85F), RF front-end (LNA, mixer, filters), and FPGA firmware structure are all preserved. Changes:

- ADC footprint and 1.8 V power routing
- Serial LVDS capture firmware
- DDC configuration and NCO programming
- LPF cutoff updated to match ADC3668 sample rate
- Lower total ADC power (500 mW vs 760 mW Stage 1)

---

## 11. Bill of Materials and Cost Estimate — Board A Stage 1

### 11.1 Part Correction History

| Original Part | Error | Replacement |
|---|---|---|
| QPL9547 | 0.1–6 GHz LNA, not X-band | Qorvo CMD319C3 (8–12 GHz) |
| HMC213BMS8E | 1.5–4.5 GHz mixer, not X-band | MACOM MAMX-011035 (primary), ADI LTC5548 (budget) |
| AD9628 | Being discontinued; misidentified as 14-bit | Removed |
| AD9648-125 | $87 was 1ku ADI list price; real price ~$140 | Replaced by LTC2145-14 at same real price with better SFDR and lower power |

### 11.2 Correct Active RF Parts

| Function | Part | Band | Key spec | Package | Price |
|---|---|---|---|---|---|
| LNA | Qorvo CMD319C3 | 8–12 GHz | 0.95 dB NF, 20–24 dB gain, +16 dBm P1dB | 16-QFN 3×3 mm | $48.97 (25+) |
| Mixer (primary) | MACOM MAMX-011035 | 5.5–19 GHz | +20 dBm IIP3, 6–7 dB CL | 12-QFN 3×3 mm | $61.33 |
| Mixer (budget) | ADI LTC5548IUDB | 2–14 GHz | ~+20 dBm IIP3, 0 dBm LO | 12-QFN 3×2 mm | $24.01 |
| ADC Stage 1 | ADI LTC2145IUP-14 | — | 73.1 dBFS SNR, 90 dBc SFDR | 64-QFN 9×9 mm | ~$125 (10+) |
| ADC Stage 2 | TI ADC3668IRTD | — | 16-bit, 250 MSPS, DDC, LVDS | 64-VQFN 9×9 mm | ~$200 |

### 11.3 Component BOM Per Board A (Stage 1, MAMX-011035 mixer)

| Ref | Component | Part | Qty | Unit price | Line total |
|---|---|---|---|---|---|
| U1–U8 | LNA | Qorvo CMD319C3 | 8 | $48.97† | $392 |
| U9–U16 | PIN Limiter | ADI ADL8111 (or equiv) | 8 | ~$4 | ~$32 |
| U17–U24 | Dechirp mixer | MACOM MAMX-011035 | 8 | $61.33 | $491 |
| *(alt)* | *(Budget mixer)* | *ADI LTC5548IUDB* | *8* | *$24.01* | *$192* |
| FL1–FL8 | Post-mixer LPF | Mini-Circuits LFCN-80+ | 8 | ~$8 | $64 |
| U17–U20 | ADC Stage 1 | ADI LTC2145IUP-14#PBF | 4 | $125† | $500 |
| J1–J2 | Mezzanine socket | Hirose DF40C-80DS-0.4V | 2 | ~$5 | $10 |
| J3 | SFP+ cage | Single-port SMD | 1 | ~$10 | $10 |
| J4 | SFP+ transceiver | 10G SR multimode | 1 | ~$35 | $35 |
| J5 | LO SMA | Edge-launch | 1 | $5 | $5 |
| J6 | Clock input | MCX edge-launch | 1 | $3 | $3 |
| U21–U22 | Power regulators | 3.3 V + 1.8 V LDO | 2 | $3 | $6 |
| Passives | R, C, bias networks | SMD 0402/0201 | ~200 | — | ~$25 |
| **Board A component subtotal** | | | | | **~$1,573** |
| *(Budget build with LTC5548 mixer)* | | | | | *~$1,274* |

†CMD319C3 at 25+ unit price break ($48.97/unit); 40 units across 5 boards.
‡LTC2145IUP-14 at 10+ unit price break ($125/unit confirmed at Digikey).

### 11.4 FPGA Module Per Board

| Component | Part | Cost |
|---|---|---|
| FPGA | LFE5UM5G-85F-8BG381C | ~$100 |
| Core reg 1.1 V | TPS62130 | ~$3 |
| I/O LDO 1.8 V | TLV75801 | ~$2 |
| Config flash | W25Q128 | $1 |
| DF40 plugs ×2 | DF40C-80DP-0.4V | $8 |
| Passives | — | $5 |
| FR4 PCB (JLCPCB, 5×) | — | ~$3/module |
| Assembly (JLCPCB Advanced, 5×) | — | ~$20/module |
| **Module subtotal** | | **~$142** |

### 11.5 PCB Fabrication and Assembly at Würth

Estimated board dimensions: ~200 mm × 110 mm (antenna array + RF chain + digital section).

| Item | Per board | For 5 boards |
|---|---|---|
| Rogers 4003C, 4-layer, controlled impedance | ~€200 (~$220) | ~€1,000 (~$1,100) |
| SMT assembly, ~100 placements | ~€180 (~$200) | ~€900 (~$990) |

Würth does not publish online pricing for Rogers boards — a formal quote is required.

### 11.6 Total Cost Summary — 5 Boards

| Item | Per board | For 5 boards |
|---|---|---|
| Board A components (MAMX-011035) | ~$1,573 | ~$7,865 |
| *(Budget build with LTC5548 mixer)* | *~$1,274* | *~$6,370* |
| FPGA module | ~$142 | ~$710 |
| PCB fabrication (Würth) | ~$220 | ~$1,100 |
| Board A assembly (Würth) | ~$200 | ~$1,000 |
| FPGA module PCB + assembly (JLCPCB) | ~$23 | ~$115 |
| Misc (shipping, consumables) | ~$40 | ~$200 |
| **Total (MAMX-011035)** | **~$2,198** | **~$10,990** |
| **Total (LTC5548 budget mixer)** | **~$1,899** | **~$9,495** |

Primary cost drivers: ADC ($560, 25%), LNA ($392, 18%), mixer ($491, 22%). Würth fabrication is the second-largest cost item after components — get a quote early.

---

## 12. Open Items and Next Steps

| Item | Description | Priority |
|---|---|---|
| PIN limiter part confirm | Confirm ADL8111 availability and price; verify +13 dBm threshold is sufficient for worst-case TX leakage scenario | Medium |
| CMD319C3 stock | **Order immediately** — 35-week factory lead time; 455 in stock at Digikey now; need 40 units | High |
| LTC2145-14 stock | Confirm LTC2145IUP-14#PBF stock at Digikey for 20 units (5 boards × 4 chips) | High |
| ADC3668 price | Confirm ADC3668IRTD exact price at Digikey before Stage 2 BOM lock | High |
| LTC2145-14 interface | Verify DDR LVDS pinout; confirm IF input range matches MAMX-011035 IF output level | High |
| MAMX-011035 stock | Order 40 units from Mouser (5 boards × 8 channels) | High |
| TX board design | Chirp generator, PA chain, reference chirp splitter and driver | High |
| Antenna simulation | Simulate 10.25 GHz patch element and corporate feed in openEMS | High |
| Pre-LNA filter | Design 2-pole coupled-line BPF on Rogers 4003C; run tolerance corners | Medium |
| Post-LNA filter | Design 5-pole interdigital BPF on Rogers 4003C | Medium |
| LPF component | Select Mini-Circuits LFCN part matched to LTC2145-14 Nyquist (~55–60 MHz) | Medium |
| Custom FPGA module | ECP5-85F FR4 module: schematic, layout, DF40 footprints; ButterStick KiCad as reference | Medium |
| FPGA firmware skeleton | ADC LVDS DDR capture + Hilbert FIR + decimation + SerDes TX for ECP5-85F | Medium |
| Clock distribution | OCXO splitter skew budget; max allowable skew to Board A ADC clock inputs | Medium |
| Calibration procedure | Startup tone injection and per-channel phase offset measurement algorithm | Medium |
| Würth quote | Obtain Board A Rogers 4003C 4-layer PCB and assembly quote | Medium |
| KiCAD Board A schematic | Begin schematic capture; all RF component footprints known | Low |
| TX antenna placement | Optimise TX positions for FDMA compatibility; simulate range-azimuth coupling | Low |
| Stage 4 aggregation | Evaluate multi-NIC vs SmartNIC vs central FPGA for 32-board aggregation | Low |
| ADC3669 relevance | Revisit ADC3669 (500 MSPS) if Stage 3 FDMA plan exceeds 125 MHz beat bandwidth | Low |
| Mixer upgrade pricing | Obtain Marki MM1-0115HPSM-2 price from RFMW for future Doppler-optimised revision | Low |
