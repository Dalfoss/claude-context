# FMCW MIMO Radar — System Architecture

**Band:** 10.0 – 10.5 GHz  
**Architecture:** FDMA/CDM · Analogue Dechirp · RF/Digitizer Board Split (SMA) · Fibre Transport  
**Version:** 0.6 · June 2026

---

## Table of Contents

1. [Introduction and System Goals](#1-introduction-and-system-goals)
2. [Architecture Decision Trail](#2-architecture-decision-trail)
3. [System Architecture Overview](#3-system-architecture-overview)
4. [Front-End: RF Board and Digitizer Board](#4-front-end-rf-board-and-digitizer-board)
5. [Waveform Strategy](#5-waveform-strategy)
6. [TX Board](#6-tx-board)
7. [Central Unit](#7-central-unit)
8. [Scaling Path](#8-scaling-path)
9. [TX/RX Element Placement and FDMA Compatibility](#9-txrx-element-placement-and-fdma-compatibility)
10. [Stage 2 ADC — TI ADC3668](#10-stage-2-adc--ti-adc3668)
11. [Bill of Materials and Cost Estimate — RF + Digitizer Boards, Stage 1](#11-bill-of-materials-and-cost-estimate--rf--digitizer-boards-stage-1)
12. [Open Items and Next Steps](#12-open-items-and-next-steps)

---

## 1. Introduction and System Goals

This document describes the hardware and signal-chain architecture for a software-defined FMCW MIMO radar operating in the 10.0–10.5 GHz band. The primary application is small UAV (drone) detection and tracking.

> **On the 1.5 km figure.** The ~1.5 km range used throughout this document is the **v1 prototype goal**, not the end goal. The design's first objective (§1.1) is maximum receive channel count, and the scaling path (§8) tiles to 256 RX / 2,048 virtual elements — range grows from there through the larger MIMO aperture and additional coherent-integration gain as boards are added. The ultimate range target is not yet fixed; treat 1.5 km as a v1 milestone for sizing the prototype, not the destination.

### 1.1 Primary Objectives

- **Maximum receive channel count.** Scale to 256 RX elements (8 per board initially) to maximise the MIMO virtual aperture.
- **High dynamic range.** Detect weak distant targets simultaneously with strong nearby returns.
- **Software-defined waveform.** All waveform parameters are controlled in firmware; hardware is a transparent analogue front-end.
- **Modular, scalable design.** Each board pair (RF Board + Digitizer Board) is self-contained and tiles cleanly. Multiple pairs feed a central processing unit over fibre.
- **Physical separation.** Central processing can be located remotely via fibre.

### 1.2 Key Specifications

| Parameter | Value / Target |
|---|---|
| RF band | 10.0 – 10.5 GHz |
| Chirp bandwidth | Up to 500 MHz (waveform-defined) |
| Range resolution | ≈ 1.5 m at 100 MHz total BW via joint processing |
| Max target range | ~1.5 km (v1 prototype goal, not the end goal; scales with channel count — see §8) |
| TX channels | 8 (initial) |
| RX channels per board | 8 |
| Stage 1 ADC | 14-bit, 125 MSPS → ~12.0 ENOB |
| Stage 2 ADC | 16-bit, 250 MSPS + DDC → ~13.8–14.3 ENOB |
| RF substrate | Rogers 4003C, 0.508 mm core |

---

## 2. Architecture Decision Trail

### 2.1 Subcarrier Multiplexing — Rejected

The initial architecture used SCM to transport multiple beat signals over a single coaxial cable to a central ADC. Third-order intermodulation products from the SCM chain land exactly on adjacent channel frequency slots for evenly spaced subcarriers. With 60–80 dB near/far dynamic range, this makes SCM fundamentally unsuited to radar. The problem scales as N³ with channel count. SCM was eliminated.

> **Post-mixer LPF constraint:** The dechirp mixer produces a beat signal (DC to ~500 MHz) and a sum term at ~20.5 GHz. No mixing product other than the fundamental difference term falls below 500 MHz, so the anti-alias LPF cutoff is fixed by the RF band choice, not the waveform design. The only front-end hardware parameter that changes with the channel plan is the anti-aliasing cutoff — now on the Digitizer Board (§4.4.1) — which tracks the ADC sample rate.

### 2.2 Settled Architecture: Digitise at the Front-End, Transport Digital

Each beat signal is digitised at the front-end — within a short SMA-coax hop of the dechirp mixer — and the digital samples are transported over fibre. This eliminates intermodulation (no analogue multiplexing of beat signals onto a shared medium), scales cleanly, and is maximally software-defined. The digitiser is implemented on a board adjacent to the RF board rather than on the antenna PCB itself; see §2.5.

### 2.3 Processing Architecture — No Central FPGA

All heavy processing (Hilbert transform, decimation, FFTs, beamforming) runs on a host GPU/CPU workstation. A central FPGA would need to receive raw samples from all boards before any reduction. At full scale (32 boards):

```
32 boards × 8 channels × 14 bits × 125 MSPS = 448 Gbps raw
```

No central FPGA handles this. Decimation and Hilbert must live at the edge — they are the bandwidth reduction step. After edge processing, data rates are tractable for a standard 10G NIC into a GPU workstation.

### 2.4 ADC Phase Coherence

Coherence is achieved by: (1) a shared OCXO master clock distributed to every Digitizer Board, eliminating frequency variation while leaving a fixed power-up phase offset per board; and (2) startup calibration — the central processor injects a known tone, each Digitizer Board measures its per-channel phase offset, and fixed correction coefficients are stored in firmware. With the two-board split (§2.5), the SMA-coax run between the RF Board and Digitizer Board adds a fixed per-channel delay; this is absorbed into the same startup-tone calibration provided the eight beat cables are length-matched to within the per-channel phase budget.

### 2.5 Board Split — Digitise on a Separate FR4 Board

Earlier revisions placed the antennas, RF chain, ADC array, and an FPGA mezzanine module all on a single Rogers board. This forced a 381-ball BGA and its 6–8 layer escape routing onto a 4-layer Rogers RF substrate (or an awkward Rogers/FR4 hybrid stackup, or a DF40 mezzanine workaround), with the BGA fanout vias threatening the RF ground planes.

The settled design splits the front-end into two boards joined by SMA coax:

- **RF Board** (Rogers 4003C, 4-layer) — antennas + full analogue receive chain through the dechirp mixer. Outputs one beat signal per channel as a baseband (≤~60 MHz) analogue signal over SMA. Pure RF/analogue; no ADCs, no FPGA, no BGA, no LVDS.
- **Digitizer Board** (FR4, multilayer) — 8× SMA beat inputs, anti-alias LPFs, ADC array, ECP5 FPGA, SFP+ output, ADC SPI, and power. All LVDS runs between the ADCs and the FPGA are on this one board, so trace-length matching, impedance, and stackup are fully controlled — a solved PCB-layout problem with no inter-board interface uncertainty.

**Why this is the right call:**

- The inter-board interface is a ~62.5 MHz baseband signal over 50 Ω SMA coax — completely standard, low-risk, any cable length. No connector compromises, no inter-board LVDS, no PMODs.
- The Rogers RF Board stays a clean 4-layer board; the ECP5 BGA escapes normally on standard multilayer FR4. The mezzanine module (DF40 connectors, separate FR4 sub-module) is eliminated entirely.
- Each board is fabbed where it is natural and cheap: Rogers at Würth, FR4 at JLCPCB Advanced.
- RF bring-up and digital bring-up proceed fully independently; risk is isolated.
- A Stage 2 ADC upgrade (§10) respins **only** the Digitizer Board — the RF Board is ADC-agnostic.

---

## 3. System Architecture Overview

### 3.1 Signal Flow

```
OCXO (100 MHz) ──┬── Chirp generator ── TX board ── TX antennas
                 │
                 ├── Reference chirp ── RF Board LO SMA (see note)
                 │
                 └── Master clock ── Digitizer Board ADC clock input

RF Board (per channel):
  [Patch] → [BPF] → [Limiter] → [LNA] → [BPF] → [Dechirp mixer] → [IF term.] → SMA out
                                                       ↑
                                                ref chirp (LO)
                                                                          │ beat signal
                                                                          │ (≤~60 MHz)
                                                                          ▼ SMA coax
Digitizer Board (per channel):
  SMA in → [Anti-alias LPF] → [balun] → [ADC] → [FPGA: Hilbert + Decimate] → [SerDes]
                                                                                  │
                                                                    Fibre → GPU workstation
```

The anti-alias LPF sits on the Digitizer Board immediately ahead of the ADC, not at the mixer — see §4.3.6. The mixer IF port is terminated locally on the RF Board (`IF term.`); the termination scheme is an open decision (pad vs. diplexer, §4.3.6, §12).

### 3.2 Reference Chirp as LO

The reference chirp is a copy of the transmitted chirp at 10.0–10.5 GHz. When it drives the dechirp mixer LO port, the output is the beat signal `f_beat = kτ`. The RF Board routes it directly to the mixer LO port. Required LO level depends on mixer choice (see Section 4.3.5).

### 3.3 Interface Summary

| Interface | Signal | Direction | Connector |
|---|---|---|---|
| Reference chirp LO | 10.0–10.5 GHz; +15 dBm (MAMX-011035) or 0 dBm (LTC5548) | TX → RF Board | SMA edge-launch |
| Beat signal (×8) | Baseband ≤~60 MHz analogue, 50 Ω | RF Board → Digitizer Board | SMA + coax |
| ADC master clock | 100 MHz OCXO-derived | Central → Digitizer Board | SMA |
| Fibre data link | Decimated complex IQ | Digitizer Board → Central | SFP+ cage |
| DC power (RF Board) | 3.3 V | PSU → RF Board | Molex or header |
| DC power (Digitizer Board) | 1.1 V / 1.8 V / 1.8 V / 3.3 V | PSU → Digitizer Board | Molex or header |

---

## 4. Front-End: RF Board and Digitizer Board

The front-end is two boards joined by SMA coax (§2.5):

- **RF Board** (Rogers 4003C, 4-layer) — 8 RX antenna elements and the full analogue receive chain through the dechirp mixer and IF termination. Covered by §4.1–§4.3.
- **Digitizer Board** (FR4, multilayer) — anti-alias filtering, ADC array, ECP5 FPGA, fibre output, and power. Covered by §4.4–§4.6.

Section §4.7 gives the physical layout of both boards.

### 4.1 RF Board — Substrate and Fabrication

| Parameter | Value |
|---|---|
| Substrate | Rogers 4003C |
| Core thickness | 0.508 mm |
| Copper weight | 35 µm (1 oz) |
| Dielectric constant | 3.55 @ 10 GHz |
| Loss tangent | 0.0027 @ 10 GHz |
| 50 Ω trace width | ≈ 1.09 mm |
| Fabrication | Würth Elektronik (controlled impedance, Rogers capability) |

### 4.2 RF Board — Antenna Array

- Type: microstrip patch, linear 1×8
- Centre frequency: 10.25 GHz
- Element spacing: λ/2 = 14.6 mm
- Total aperture: 116.8 mm
- Feed: 8 independent 50 Ω microstrip feed lines — one per element, each running directly to its own RF chain input (BPF → Limiter → LNA). No combining network. Keep each feed trace under ~3 mm to limit NF penalty (§4.3.3).
- Substrate: Rogers 4003C (same board as RF chain)

### 4.3 RF Board — RF Signal Chain (per RX channel)

The chain runs from the patch element through to the dechirp mixer IF port and its local termination, ending at the per-channel SMA beat output. The anti-alias LPF that used to follow the mixer has moved to the Digitizer Board (§4.3.6, §4.4.1); what remains at the mixer IF on the RF Board is a broadband termination (§4.3.6).

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

> **Part not yet settled.** An earlier draft listed an "ADI ADL8111" here. That part number was confabulated — it is not a real PIN limiter (and "ADL" is an ADI active-device prefix, i.e. it would be an amplifier, not a limiter) — and has been removed. Packaged SOT-sized X-band PIN limiters are scarce; at 10 GHz most off-the-shelf limiters are large connectorised modules rather than drop-in SMDs. The realistic candidates are below, and final selection is an open item (§12) because it affects the Rogers layout.

| Parameter | Value |
|---|---|
| Candidate family | MACOM MA4L / MADL-0110-series PIN limiter diodes (≈10 MHz–18 GHz, designed to protect LNAs/mixers/detectors) |
| Lead X-band candidate | MACOM MADL-011088 — 6–12 GHz, insertion loss ~0.35–0.7 dB, CW incident power handling to ~+39 dBm. **Bare die**, not a packaged SMD. |
| Insertion loss (assumed) | ~0.5–0.8 dB, candidate-dependent (adds directly to system NF ahead of LNA) |
| Recovery time | < 1 µs (typical for PIN limiters; confirm on the selected part) |
| Package | **TBD** — bare die assembled into an on-board shunt limiter circuit on the Rogers substrate, or a packaged limiter if a suitable SMD is found |

**Two honest paths, to be resolved in evaluation:**
- **MADL die + on-board shunt limiter circuit** on the Rogers board — standard practice at X-band, but adds RF design work and a layout dependency.
- **Leave the limiter as a TBD line** in the BOM pending evaluation, since the choice affects the Rogers layout.

Assuming ~0.5–0.8 dB insertion loss until the part is fixed, the limiter loss adds directly to the system noise figure ahead of the LNA. At 0.95 dB LNA NF, the combined front-end NF lands on the order of ~1.5–1.75 dB — acceptable for the v1 prototype's sensitivity budget. Pin down the exact figure once the part is chosen.

#### 4.3.3 LNA

**Selected part: Qorvo CMD319C3.** Selected together with the CMD253C3 mixer (§4.3.5) as a matched Qorvo Custom MMIC pair — shared design ecosystem, compatible supply voltages, and consistent X-band performance characterisation.

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

The 50Ω internal matching eliminates external matching networks — a meaningful simplification for an 8-channel RF Board layout. Single positive 2–5 V supply.

Every millimetre of PCB trace between antenna feed and LNA input adds ~0.05 dB to NF. Keep under 3 mm.

**Alternative candidate: Qorvo QPM1000 — integrated limiter/LNA.**

The QPM1000 is not a direct LNA swap; it combines a PIN limiter and LNA in a single package, which would replace both the §4.3.2 limiter and the §4.3.3 LNA with one part.

| Parameter | Value |
|---|---|
| Function | Integrated limiter + LNA |
| Frequency | 2 – 20 GHz |
| Noise figure | **1.7 – 4.0 dB across band** (headline 2.8 dB; ~2.5–3.5 dB expected at 10 GHz) |
| Gain | 17 dB |
| P1dB (output) | +18 dBm |
| Limiter survivability | Up to **4 W** incident power without degradation |
| Supply | **5 V, 100 mA** + **−0.6 V gate bias** (needs negative supply rail) |
| Package | 6×5 mm QFN, 18-pin |
| Price | $163.90 (1–24 units) · **$111.14 (25+)** — RFMW (not on Digikey) |

**Tradeoff vs CMD319C3 + separate limiter:**

The integration benefit is real — one fewer part per channel (×8), no bare-die limiter assembly risk, and 4 W limiter survivability is conservative headroom. However the NF penalty is significant:

```
QPM1000 alone @ 10 GHz:         ~2.8 dB NF (estimated)
CMD319C3 + 0.7 dB limiter loss:  ~1.52 dB system NF (§4.3.3)
NF penalty for QPM1000:          ~1.3 dB
```

A 1.3 dB system NF increase reduces maximum detection range by roughly 15–20% — meaningful for the v1 prototype's 1.5 km target. CMD319C3 remains the primary choice. QPM1000 is worth reconsidering only if the PIN limiter (§4.3.2) proves unresolvable — in which case QPM1000 eliminates that problem at a known sensitivity cost. Also note the negative gate bias requirement adds a supply rail that CMD319C3 does not need.

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

**Primary choice — Qorvo CMD253C3**

Selected together with CMD319C3 LNA (§4.3.3) as a matched Qorvo Custom MMIC pair. Both parts share the same design ecosystem, supply voltage range, and X-band performance characterisation methodology. The pairing reduces integration risk compared to cross-manufacturer combinations.

| Parameter | Value |
|---|---|
| Technology | GaAs passive double-balanced Schottky diode |
| RF/LO range | 6 – 20 GHz (10.25 GHz is mid-band) |
| IF bandwidth | DC-coupled (verify lower cutoff from datasheet) |
| Conversion loss | Verify @ 10.25 GHz from datasheet |
| IIP3 | Verify @ 10.25 GHz from datasheet |
| LO drive required | Verify from datasheet |
| DC power | **Zero** (fully passive) |
| Phase noise added | **Zero** (no active elements in LO path) |
| Package | 3×3 mm QFN (SMT) |
| Stock | **998 units at Digikey — ships same day** |
| Price | **$84.94 (1 unit) · $58.14 (25+) · $47.50 (100+)** — Digikey |

At 40 units (5 boards × 8 channels), the 25-unit price break applies: **$58.14/unit, $465 total** — slightly cheaper than MAMX-011035 at the same quantity.

> **Verify IIP3, conversion loss, and LO drive level from the Qorvo CMD253C3 datasheet before locking the RF Board layout.** LO drive level determines TX board LO PA requirements; IIP3 at 10.25 GHz sets the near/far dynamic range limit; IF port DC coupling is required for the beat signal (DC to ~60 MHz).

> **IF port:** On the RF Board it drives a local broadband termination (§4.3.6) and then the per-channel SMA beat output. The balun / single-ended-to-differential conversion into the ADC lives on the Digitizer Board after the coax run (§4.4.1).

---

**Alternative — MACOM MAMX-011035-TR0100**

Previously the primary choice. Retained as documented alternative with fully verified specs.

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
| Phase noise added | **Zero** |
| Package | 12-QFN, 3×3 mm |
| Price | ~$61.33 (Mouser) |

Revert to MAMX-011035 if CMD253C3 datasheet verification reveals a significant performance disadvantage or availability problem.

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

---

**Future upgrade — Marki MM1-0115HPSM-2**

+24 dBm IIP3 mid-band (10.25 GHz is mid-band for 1–15 GHz part), 54 dB LO-RF isolation, 8 dB conversion loss, passive GaAs, zero phase noise degradation. ~$80–150 via RFMW or Marki direct. Recommended if a future revision must push the Doppler floor lower.

#### 4.3.6 Mixer IF Termination (and Relocated Anti-Alias LPF)

**The anti-alias LPF has moved off the RF Board to the Digitizer Board**, immediately ahead of each ADC (§4.4.1). Rationale:

- **Anti-aliasing is a property of what reaches the sampler, not where you filter.** As long as the LPF precedes the ADC, Nyquist protection is identical; the filter location only changes what travels on the coax.
- **The coax does not distort the wanted band.** Over DC–~60 MHz the short 50 Ω run is essentially lossless and linear, and passive coax generates no new mixing products — the beat signal arrives intact.
- **The only significant out-of-band product is harmless in transit.** Nothing falls between the beat band and the ~20.5 GHz sum term (§2.1); that term is heavily attenuated by the coax itself and removed by the LPF at the ADC.
- **Better RFI rejection.** Filtering at the sampler also rejects out-of-band interference the cable picks up in transit, which a mixer-side filter would let the downstream cable re-acquire.
- **Modularity.** The LPF and ADC are a matched pair (cutoff tracks sample rate); colocating them means a Stage 2 ADC upgrade (§10) swaps both on the Digitizer Board and leaves the RF Board untouched.

**What remains at the mixer IF on the RF Board is a broadband termination.** A passive double-balanced diode mixer (MAMX-011035) needs a clean, broadband IF termination to hold its conversion linearity and IIP3. With the anti-alias filter now remote — and reflective in its stopband — that reflection returns down the coax and can ripple the IF termination. A local absorptive element at the mixer IF restores the match.

> **Open decision — IF termination scheme (§12). Not settled; to be resolved before production.**
> - **Resistive pad (~3 dB)** — broadband, absorptive, sets a defined 50 Ω source for the cable. The ~3 dB beat-signal loss is negligible against the ADC's ~73 dB SNR floor. Pragmatic for prototype bring-up.
> - **IF diplexer** — routes the wanted low band to the cable and dumps the high-band products (incl. the sum term) into an on-board 50 Ω load. Cleaner termination and removes the 20.5 GHz energy before it reaches the cable, at the cost of more components per channel on the RF Board. Stronger candidate for a production revision where mixer linearity and radiated-EMI margin matter.
>
> Both are per-channel (×8). Evaluate the pad first for the prototype, then characterise the diplexer before committing to a production build.

> **Relocated LPF part constraint (Digitizer Board):** For LTC2145-14 at 125 MSPS, the anti-aliasing cutoff is ~55–60 MHz. Suitable part: Mini-Circuits LFCN-80+ (DC–80 MHz, SMD). When the ADC is upgraded for Stage 2, this component is swapped as a matched pair on the Digitizer Board — same PCB footprint.

### 4.4 Digitizer Board — IF Input Conditioning

Each beat signal arrives at the Digitizer Board on its own SMA. The per-channel input chain is:

#### 4.4.1 Per-Channel Input Chain

```
SMA in → [anti-alias LPF] → [balun / single-ended drive] → ADC
```

- **Anti-alias LPF** — relocated here from the RF Board (§4.3.6). Mini-Circuits LFCN-80+ (DC–80 MHz, SMD) for the LTC2145-14 at 125 MSPS; ~55–60 MHz effective anti-alias cutoff. Swapped as a matched pair with the ADC at Stage 2 — same footprint.
- **Balun / single-ended drive** — converts the single-ended beat signal to the ADC differential input. The MAMX-011035 IF is DC-coupled and the beat band reaches near-DC, so preserve the DC path (or apply the defined DC bias the LTC2145-14 input common-mode spec requires) rather than AC-coupling.
- **Grounding note** — each coax shield ties the RF Board and Digitizer Board grounds together (×8 paths). This is generally desirable (shared reference), but with a DC-coupled IF, watch board-to-board ground-potential offset in the power/grounding scheme.

### 4.5 Digitizer Board — ADC Array

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

- 4 chips per Digitizer Board (dual-channel each) → 8 RX channels
- Supply: 1.8 V (ADC core) + 1.8 V (ADC output/LVDS)
- Total ADC power: 4 × 190 mW = **760 mW**

### 4.6 Digitizer Board — FPGA

#### 4.6.1 Selected Part: XC7A100T (Xilinx Artix-7 100T)

| Parameter | Value |
|---|---|
| Logic cells | ~101,000 |
| DSP slices | **240** (DSP48E1) |
| Block RAM | 4,860 Kbits |
| GTP transceivers | 4 lanes at up to **6.6 Gbps** each |
| User I/O (FBG484) | up to ~300 signal pins |
| Toolchain | **openXC7** (Yosys + nextpnr-xilinx) — 100% open-source |
| Package | FBG484 (required for GTP transceiver access) |
| Price | ~$80–150 (check Digikey/Mouser at time of order) |

**Why Artix-7 over ECP5-85F:**

The ECP5-85F SerDes tops out at **5.0 Gbps per lane**. Streaming all 64 DDC channels at the target subband rates produces ~4.1–6.1 Gbps aggregate (see §4.6.2), which presses against or exceeds the ECP5 ceiling — forcing either aggressive clock margining or reduced subband bandwidth. The XC7A100T's GTP transceivers run natively at **up to 6.6 Gbps**, providing clean headroom on a single lane without overclocking. Additionally, the 240 DSP48E1 slices (vs 156 ECP5 DSP blocks) handle the 64-channel CORDIC DDC matrix more comfortably.

**Toolchain: openXC7.** Yosys + nextpnr-xilinx synthesises and places Artix-7 (7-series) designs fully open-source, fulfilling the rigid open-source toolchain constraint. openXC7 has demonstrated working hardware implementations on XC7A100T.

**FPGA tasks on the Digitizer Board:**

| Task | Implementation |
|---|---|
| ADC LVDS DDR capture | IDDR primitives on differential I/O; DCO as capture clock; FCO for channel de-interleaving |
| DDC per channel (×64) | CORDIC complex mixer → CIC decimation filter; 8 DDC instances per physical ADC channel |
| Byte-interleaving packer | Lines up IQ samples from all 64 DDC channels into structured data frames; no FPGA-side FFT |
| ADC SPI configuration | One-time startup |
| Data framing | Frame DDC IQ samples with channel ID, subband ID, and timestamp |
| GTP TX | Native 10G-class SFP+ output at up to 6.6 Gbps; one fiber per board |

#### 4.6.2 DSP Architecture — Frequency-Selective Decimation

Each of the 8 physical ADC channels feeds a parallel bank of 8 DDC (Digital Down-Converter) channels, giving **64 DDC channels total**. Each DDC:

1. Applies a CORDIC-based complex mixer to translate the assigned 2–3 MHz subband to baseband
2. Follows with a CIC decimation filter to isolate the subband and drop the sample rate
3. Produces complex (I+Q) baseband samples at the decimated output rate

**No FPGA-side FFT.** Instead of up-converting and stitching subbands into a spectrum on the FPGA, a parallel-to-serial **byte-interleaving packer** lines up the raw time-domain IQ samples from all 64 DDC channels back-to-back into structured data frames and streams them raw. The host workstation's AMD CPU unpacks and computes FFTs in software. This eliminates a significant FPGA firmware complexity and reduces estimated firmware difficulty from ~8/10 to ~6/10.

**Data rate budget:**
```
64 DDC channels × 2–3 MSPS (CIC decimated) × 32 bits (16-bit I + 16-bit Q)
  Low end (2 MSPS subbands):  64 × 2e6 × 32 = 4.1 Gbps payload
  High end (3 MSPS subbands): 64 × 3e6 × 32 = 6.1 Gbps payload
  → Both fit within a single 6.6 Gbps GTP lane with framing overhead  ✓
  → One SFP+ fiber per board                                           ✓
```

CIC decimation factor is firmware-configurable; target is the 2–3 MHz subband bandwidth set by the FDMA waveform plan.

#### 4.6.3 FPGA Integration: Monolithic Digitizer Board (FR4)

ADCs and FPGA share one FR4 PCB — no mezzanine connectors, no daughter cards. This was already the case with the ECP5 design (§2.5); the Artix-7 upgrade maintains it. All 100 Ω differential LVDS DDR traces between the ADCs and the FPGA are fully controlled and length-matched on-board — a solved PCB layout problem.

**Reference Design: ScopeFun.** The ScopeFun open-source oscilloscope (KiCad schematic + PCB, open-source licence) targets the Artix-7 and provides a verified starting baseline for:

- **Artix-7 power sequencing** — the 7-series power-up sequence (VCCINT → VCCAUX → VCCIO) is critical and easy to get wrong; ScopeFun has this verified
- **High-speed SMA input stages** — verified front-end circuitry
- **Differential LVDS ADC routing** — confirmed trace geometry and termination
- **GTP transceiver routing and SFP+ integration** — the ScopeFun schematic/layout is used as a starting baseline; expand to 8 ADC channels from this verified foundation

**FPGA support circuitry:**

| Component | Part | Cost |
|---|---|---|
| FPGA | XC7A100T-FBG484 | ~$80–150 |
| Core regulator 1.0 V | TPS62130 or ScopeFun equivalent | ~$3 |
| I/O LDO 1.8 V | TLV75801 | ~$2 |
| Config flash | W25Q128 | $1 |
| Passives / decoupling | — | ~$5 |

> **Power sequencing is strict.** Follow the ScopeFun power tree exactly: VCCINT (1.0 V) must ramp before VCCAUX (1.8 V); VCCIO must ramp last. Deviating from this sequence can latch-up or damage the FPGA.

#### 4.6.4 IQ Generation via CORDIC DDC

Each DDC channel applies a CORDIC-based complex rotation to translate its assigned subband to baseband, producing I and Q components directly. This replaces the Hilbert FIR approach used in the prior ECP5 plan. Advantages:

- IQ is produced directly from CORDIC rotation — no Hilbert approximation; image rejection is limited only by CORDIC iteration depth and quantisation noise (theoretically superior to the 60 dB of a 64-tap Hilbert FIR)
- Each DDC independently tunes to any subband via its NCO frequency word — firmware-reconfigurable without hardware changes
- The 64 DDC channels pipeline in parallel across the DSP48E1 fabric; time-multiplexed CORDIC engines can reduce resource usage if needed

The GPU workstation receives raw IQ from each DDC and applies FFT and beamforming processing entirely in software.

#### 4.6.5 Fibre Output — 10G-Class UDP Ethernet

The Artix-7 GTP transceiver drives a single SFP+ cage at up to 6.6 Gbps. At 4.1–6.1 Gbps aggregate DDC output, one GTP lane comfortably carries the full 64-channel stream over a single OM3/OM4 multimode fiber to the host workstation's 10G NIC.

| Parameter | Value |
|---|---|
| Framing | UDP/IP Ethernet |
| Physical | GTP transceiver → SFP+ cage → multimode fiber |
| Max GTP line rate | 6.6 Gbps |
| Aggregate DDC payload | ~4.1–6.1 Gbps (decimation-dependent) |
| Host interface | Standard SFP+ port on 10G NIC |

> **10GbE line-rate note.** Standard 10GBASE-SR uses a 10.3125 Gbps line rate. Artix-7 GTPs top out at 6.6 Gbps and cannot reach 10.3125 Gbps natively. The design runs a custom/non-standard line rate over 10G-class SFP+ hardware (fiber and transceiver tolerate it; the optical layer is passive). The host NIC must accept this custom line rate — verify host NIC and driver compatibility before committing to a specific GTP baud rate. An alternative is an external 10G MAC/PHY chip between FPGA and SFP+ cage to produce a standards-compliant 10.3125 Gbps signal, at the cost of an additional component. Confirm the approach during Digitizer Board schematic capture.

### 4.7 Physical Layout

**RF Board** (Rogers 4003C). Signal flows top-to-bottom: antennas at the top edge, beat SMA outputs along the bottom edge.

```
┌─────────────────────────────────────────────────┐
│  ANTENNA SECTION                                │
│  8× patch elements + corporate microstrip feed  │
├─────────────────────────────────────────────────┤
│  X-BAND RF SECTION                              │
│  BPF → Limiter → LNA → BPF → Dechirp mixer (×8)  │
│  Via fence every λ/20 ≈ 1.5 mm along all edges  │
│  Vias under all MMICs; LNA input trace < 3 mm   │
├─────────────────────────────────────────────────┤  ← LO SMA (one side)
│  IF TERMINATION + OUTPUT                         │
│  Mixer IF → termination (pad/diplexer, §4.3.6)  │
│  → 8× SMA beat outputs along bottom edge         │
└─────────────────────────────────────────────────┘
        │ │ │ │ │ │ │ │   8× SMA beat out
        ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼   coax to Digitizer Board
```

**Digitizer Board** (FR4). Beat SMA inputs at the top edge, fibre at the bottom edge.

```
        8× SMA beat in            1× SMA ADC clock in
        │ │ │ │ │ │ │ │                  │
┌─────────────────────────────────────────────────┐
│  IF INPUT SECTION                               │
│  Anti-alias LPF → balun (×8)                    │
├─────────────────────────────────────────────────┤
│  ADC SECTION                                    │
│  LTC2145-14 array (×4) · matched LVDS to FPGA   │
├─────────────────────────────────────────────────┤
│  DIGITAL SECTION                                │
│  ECP5-85F (direct BGA) · LVDS routed on-board   │
│  SFP+ cage · SPI · Power 1.1 / 1.8 / 1.8 / 3.3V │
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

All TX channels transmit the full chirp bandwidth simultaneously, distinguished by Hadamard phase codes across chirp periods. Zero hardware changes on the RF Board or Digitizer Board. Limitation: narrowband assumption can be violated at large apertures and wide bandwidths.

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
- Distribute reference chirp to all RF Board instances

**LO distribution depends on mixer choice:**
- MAMX-011035 (primary): +15 dBm at each RF Board LO SMA. For 8 boards: ~+25 dBm into 1:8 splitter — requires GaAs PA stage on TX board.
- LTC5548 (budget): 0 dBm at each RF Board LO SMA. Passive splitter from modest source suffices.

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
| Distribution | Passive 1:N coaxial splitter to all Digitizer Board ADC clock inputs and the TX board |

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

A "board pair" is one RF Board + one Digitizer Board joined by 8 beat SMA coax.

| Stage | Board pairs | RX channels | TX config | Virtual elements | Fibre links |
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

The 8 RX patches within each RF Board must stay at λ/2 = 14.6 mm spacing. Changing this introduces grating lobes and breaks the equal-path-length corporate feed. No modification is needed or permitted.

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
| RX within RF Board | λ/2 uniform — fixed | None | Not relevant (coupling is TX-side) |
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

### 10.5 Stage 2 Respins Only the Digitizer Board

ADC3668 is not pin-compatible with LTC2145-14. Because of the board split (§2.5), **the RF Board is untouched at Stage 2** — it never saw the ADC. Stage 2 is a new Digitizer Board only; the FPGA (ECP5-85F), the SMA beat interface, and the FPGA firmware structure are all preserved. Changes, all confined to the Digitizer Board:

- ADC footprint and 1.8 V power routing
- Serial LVDS capture firmware
- DDC configuration and NCO programming
- Anti-alias LPF cutoff updated to match the ADC3668 sample rate (swapped with the ADC as a matched pair, §4.4.1)
- Lower total ADC power (500 mW vs 760 mW Stage 1)

---

## 11. Bill of Materials and Cost Estimate — RF + Digitizer Boards, Stage 1

Costs are given per **board pair** (one RF Board + one Digitizer Board) and for a 5-pair build. SMA/coax, baluns, IF termination, and the second board's fab/assembly are new line items versus the single-board v0.4 estimate; several are marked as estimates pending vendor quotes.

### 11.1 Part Correction History

| Original Part | Error / Reason for Change | Replacement |
|---|---|---|
| QPL9547 | 0.1–6 GHz LNA, not X-band | Qorvo CMD319C3 (8–12 GHz) |
| HMC213BMS8E | 1.5–4.5 GHz mixer, not X-band | Qorvo CMD253C3 (primary); MACOM MAMX-011035 (documented alternative) |
| MACOM MAMX-011035 | Replaced with matched Qorvo pair — CMD253C3 + CMD319C3 from same design ecosystem; verify CMD253C3 specs before finalising | Qorvo CMD253C3 |
| AD9628 | Being discontinued; misidentified as 14-bit | Removed |
| AD9648-125 | $87 was 1ku ADI list price; real price ~$140 | Replaced by LTC2145-14 at same real price with better SFDR and lower power |
| ADL8111 | Confabulated part number — not a real PIN limiter (ADL is an ADI active-device/amplifier prefix) | MACOM MADL-0110-series PIN limiter; selection pending — see §4.3.2 and §12 |
| LFE5UM5G-85F-8BG381C (ECP5-85F) | 5 Gbps/lane SerDes ceiling insufficient for 64 DDC channels at full bandwidth; replaced with higher-transceiver-speed part | Xilinx XC7A100T (Artix-7); 6.6 Gbps GTP; openXC7 toolchain |

### 11.2 Correct Active RF Parts

| Function | Part | Band | Key spec | Package | Price |
|---|---|---|---|---|---|
| LNA | Qorvo CMD319C3 | 8–12 GHz | 0.95 dB NF, 20–24 dB gain, +16 dBm P1dB | 16-QFN 3×3 mm | $48.97 (25+) |
| LNA (alt.) | Qorvo QPM1000 | 2–20 GHz | Integrated limiter+LNA; 2.8 dB NF (headline), 17 dB gain; +5 V / −0.6 V supply | 6×5 mm QFN | $111.14 (25+, RFMW) |
| Mixer (primary) | Qorvo CMD253C3 | 6–20 GHz | Passive GaAs DBM; verify IIP3, CL, LO drive @ 10.25 GHz | 3×3 mm QFN | **$58.14 (25+, Digikey)** — 998 in stock |
| Mixer (alt.) | MACOM MAMX-011035 | 5.5–19 GHz | +20 dBm IIP3, 6–7 dB CL, +15 dBm LO | 12-QFN 3×3 mm | $61.33 |
| Mixer (budget) | ADI LTC5548IUDB | 2–14 GHz | ~+20 dBm IIP3, 0 dBm LO | 12-QFN 3×2 mm | $24.01 |
| ADC Stage 1 | ADI LTC2145IUP-14 | — | 73.1 dBFS SNR, 90 dBc SFDR | 64-QFN 9×9 mm | ~$125 (10+) |
| ADC Stage 2 | TI ADC3668IRTD | — | 16-bit, 250 MSPS, DDC, LVDS | 64-VQFN 9×9 mm | ~$200 |

### 11.3 RF Board Component BOM (Stage 1, CMD253C3 mixer)

| Ref | Component | Part | Qty | Unit price | Line total |
|---|---|---|---|---|---|
| U1–U8 | LNA | Qorvo CMD319C3 | 8 | $48.97† | $392 |
| U9–U16 | PIN Limiter | **TBD** — MACOM MADL-011088 die (+ on-board shunt circuit) lead candidate | 8 | ~$4 (est.) | ~$32 (est.) |
| U17–U24 | Dechirp mixer | Qorvo CMD253C3 | 8 | $58.14 (25+ Digikey) | **$465** |
| *(alt)* | *(Alt. mixer)* | *MACOM MAMX-011035* | *8* | *$61.33* | *$491* |
| *(alt)* | *(Budget mixer)* | *ADI LTC5548IUDB* | *8* | *$24.01* | *$192* |
| T1–T8 | IF termination | **TBD** — 3 dB resistive pad *or* IF diplexer (§4.3.6, §12) | 8 | ~$1–4 (est.) | ~$8–32 (est.) |
| J1–J8 | Beat SMA out | Edge-launch SMA | 8 | ~$4 | $32 |
| J9 | LO SMA | Edge-launch SMA | 1 | $5 | $5 |
| U25 | Power regulator | 3.3 V LDO | 1 | $3 | $3 |
| Passives | R, C, bias networks | SMD 0402/0201 | ~160 | — | ~$20 |
| **RF Board subtotal (CMD253C3)** | | | | | **~$957** |
| *(Alt. build with MAMX-011035)* | | | | | *~$983* |
| *(Budget build with LTC5548 mixer)* | | | | | *~$684* |

No ADC, anti-alias LPF, FPGA, or SFP+ on the RF Board — those moved to the Digitizer Board (§2.5).

†CMD319C3 at 25+ unit price break ($48.97/unit); 40 units across 5 boards.

### 11.4 Digitizer Board Component BOM (Stage 1)

| Ref | Component | Part | Qty | Unit price | Line total |
|---|---|---|---|---|---|
| J1–J8 | Beat SMA in | Edge-launch SMA | 8 | ~$4 | $32 |
| J9 | ADC clock SMA | Edge-launch SMA | 1 | $3 | $3 |
| FL1–FL8 | Anti-alias LPF | Mini-Circuits LFCN-80+ | 8 | ~$8 | $64 |
| T1–T8 | Input balun | Wideband balun / single-ended drive | 8 | ~$2 | $16 |
| U1–U4 | ADC Stage 1 | ADI LTC2145IUP-14#PBF | 4 | $125‡ | $500 |
| U5 | FPGA | XC7A100T-FBG484 (Artix-7) | 1 | ~$80–150 | ~$100 |
| U6 | Core reg 1.0 V | TPS62130 | 1 | ~$3 | $3 |
| U7 | FPGA I/O LDO 1.8 V | TLV75801 | 1 | ~$2 | $2 |
| U8 | ADC supply 1.8 V | LDO/buck | 1 | ~$3 | $3 |
| U9 | 3.3 V rail | LDO | 1 | ~$3 | $3 |
| U10 | Config flash | W25Q128 | 1 | $1 | $1 |
| J10 | SFP+ cage | Single-port SMD | 1 | ~$10 | $10 |
| J11 | SFP+ transceiver | 10G SR multimode | 1 | ~$35 | $35 |
| Passives | R, C, decoupling | SMD 0402/0201 | ~200 | — | ~$25 |
| **Digitizer Board subtotal** | | | | | **~$797** |

‡LTC2145IUP-14 at 10+ unit price break ($125/unit confirmed at Digikey).

### 11.5 PCB Fabrication and Assembly

**RF Board** — Rogers 4003C, 4-layer, controlled impedance, ~170 mm × 110 mm (antenna array + RF chain + IF termination/output). Würth (formal quote required; no online Rogers pricing).

| Item | Per board | For 5 boards |
|---|---|---|
| Rogers 4003C, 4-layer, controlled impedance | ~$200 | ~$1,000 |
| SMT assembly (~80 placements) | ~$160 | ~$800 |

**Digitizer Board** — FR4, 6–8 layer (381-ball BGA escape + controlled-impedance LVDS), ~150 mm × 100 mm. JLCPCB Advanced.

| Item | Per board | For 5 boards |
|---|---|---|
| FR4 6–8 layer, controlled impedance | ~$30 (est.) | ~$150 (est.) |
| SMT assembly incl. BGA (~150 placements) | ~$80 (est.) | ~$400 (est.) |

### 11.6 Total Cost Summary — 5 Board Pairs

| Item | Per pair | For 5 pairs |
|---|---|---|
| RF Board components (CMD253C3) | ~$957 | ~$4,785 |
| *(Alt. build with MAMX-011035)* | *~$983* | *~$4,915* |
| *(Budget build with LTC5548 mixer)* | *~$684* | *~$3,420* |
| Digitizer Board components (incl. ADC $500 + FPGA ~$100) | ~$797 | ~$3,985 |
| RF Board PCB + assembly (Würth) | ~$360 | ~$1,800 |
| Digitizer Board PCB + assembly (JLCPCB Advanced) | ~$110 | ~$550 |
| Misc (SMA coax cables, shipping, consumables) | ~$40 | ~$200 |
| **Total (CMD253C3 primary)** | **~$2,264** | **~$11,320** |
| **Total (MAMX-011035 alt.)** | **~$2,290** | **~$11,450** |
| **Total (LTC5548 budget mixer)** | **~$1,991** | **~$9,955** |

Primary cost drivers per board pair: ADC on Digitizer Board ($500, 22%), mixer on RF Board ($465, 21%), LNA on RF Board ($392, 17%). Würth Rogers fabrication ($360) is the largest non-component cost. All component prices confirmed at prototype quantities from Digikey/Mouser except XC7A100T (reconfirm before ordering).

---

## 12. Open Items and Next Steps

| Item | Description | Priority |
|---|---|---|
| PIN limiter part selection | Select an X-band PIN limiter. MACOM MADL-011088 (6–12 GHz) is the lead candidate but is a bare die needing an on-board shunt limiter circuit; evaluate any packaged SMD alternatives. Affects the Rogers layout. Verify insertion loss, incident-power handling, and recovery time against the worst-case TX-leakage scenario | Medium |
| CMD319C3 stock | **Order immediately** — 35-week factory lead time; 455 in stock at Digikey now; need 40 units for 5-board run | High |
| QPM1000 evaluation | Evaluate QPM1000 vs CMD319C3 as LNA: confirm NF @ 10 GHz, gain, P1dB, supply, package, price. If QPM1000 NF is meaningfully lower, it displaces CMD319C3 and updates the Friis budget | High |
| CMD253C3 datasheet verification | Price confirmed ($58.14/25+ Digikey, 998 in stock). Confirm remaining specs from Qorvo datasheet: LO drive level @ 10.25 GHz, IIP3 @ 10.25 GHz, conversion loss @ 10.25 GHz, IF port DC coupling lower cutoff. Update §4.3.5 with verified specs | High |
| IF termination scheme | **Decide before production:** 3 dB resistive pad (prototype) vs. IF diplexer (cleaner mixer termination, removes 20.5 GHz before the coax, more parts). Characterise mixer IIP3 / spurious under each (§4.3.6) | High |
| LTC2145-14 stock | Confirm LTC2145IUP-14#PBF stock at Digikey for 20 units (5 Digitizer Boards × 4 chips) | High |
| ADC3668 price | Confirm ADC3668IRTD exact price at Digikey before Stage 2 BOM lock | High |
| LTC2145-14 interface | Verify DDR LVDS pinout; confirm IF input range matches the beat-signal level arriving over coax (post-termination) | High |
| MAMX-011035 stock | Order 40 units from Mouser (5 RF Boards × 8 channels) | High |
| Inter-board SMA + coax | Select edge-launch SMA and coax type; specify and length-match the 8 beat cables to the per-channel phase budget (§2.4); confirm DC-coupled path and shield-grounding scheme (§4.4.1) | High |
| TX board design | Chirp generator, PA chain, reference chirp splitter and driver | High |
| Antenna simulation | Simulate 10.25 GHz patch element and corporate feed in openEMS | High |
| Pre-LNA filter | Design 2-pole coupled-line BPF on Rogers 4003C; run tolerance corners | Medium |
| Post-LNA filter | Design 5-pole interdigital BPF on Rogers 4003C | Medium |
| Anti-alias LPF (Digitizer Board) | Select Mini-Circuits LFCN part matched to LTC2145-14 Nyquist (~55–60 MHz); place ahead of the ADC, swap as a matched pair at Stage 2 | Medium |
| Digitizer Board schematic + layout | XC7A100T-FBG484 on 6–8 layer FR4: start from ScopeFun KiCad reference design; expand to 8 ADC channels; verify BGA escape, ADC LVDS routing/length-match, SFP+, Artix-7 power sequencing (1.0/1.8/VCCIO V); confirm openXC7 toolchain compatibility with target package | Medium |
| Input balun | Select wideband single-ended→differential balun (or DC-coupled drive) for the ADC input on the Digitizer Board (§4.4.1) | Medium |
| Fibre output — GTP line rate and host NIC compatibility | Artix-7 GTP max is 6.6 Gbps; standard 10GBASE-SR is 10.3125 Gbps. Decide: (a) custom line rate at ≤6.6 Gbps over SFP+ hardware — verify host NIC driver accepts non-standard rate; or (b) external 10G MAC/PHY chip for standards-compliant 10.3125 Gbps. Resolve before Digitizer Board schematic is locked (§4.6.5) | High |
| FPGA firmware skeleton | ADC LVDS DDR capture → 64 CORDIC DDC channels (CORDIC mixer + CIC filter) → byte-interleaving packer → GTP TX; openXC7 (Yosys + nextpnr-xilinx) for XC7A100T | Medium |
| Clock distribution | OCXO splitter skew budget; max allowable skew to Digitizer Board ADC clock inputs | Medium |
| Calibration procedure | Startup tone injection and per-channel phase offset measurement algorithm (now also absorbs SMA cable-length delta) | Medium |
| Würth quote | Obtain RF Board Rogers 4003C 4-layer PCB and assembly quote | Medium |
| JLCPCB quote | Obtain Digitizer Board 6–8 layer FR4 fab + BGA assembly quote (Advanced) | Medium |
| KiCAD schematic capture | Begin RF Board and Digitizer Board schematics; all footprints known | Low |
| TX antenna placement | Optimise TX positions for FDMA compatibility; simulate range-azimuth coupling | Low |
| Stage 4 aggregation | Evaluate multi-NIC vs SmartNIC vs central FPGA for 32-board aggregation | Low |
| ADC3669 relevance | Revisit ADC3669 (500 MSPS) if Stage 3 FDMA plan exceeds 125 MHz beat bandwidth | Low |
| Mixer upgrade pricing | Obtain Marki MM1-0115HPSM-2 price from RFMW for future Doppler-optimised revision | Low |
