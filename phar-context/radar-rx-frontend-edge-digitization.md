# FMCW MIMO Radar — System Architecture

**Band:** 10.0 – 10.5 GHz  
**Architecture:** FDMA · Analogue Dechirp · RF/Digitizer Board Split · Fibre Transport  
**Version:** 0.8 · 2026-06-23 (IQ-mixer / 16-channel reconciliation)

> **Reconciliation note (2026-06-23) — DECHIRP MIXER → IQ, DIGITIZER 8→16 CHANNELS.** The dechirp
> mixer changed to the **Qorvo CMD258C4 I/Q mixer**, run as a **complex-IQ downconverter** (each RX
> channel emits I = IF1 + Q = IF2). Consequences are authoritative in `radar-project.md` → *Dechirp
> mixer → IQ mixer; digitizer 8 → 16 channels*:
> - **8 RX → 16 IF lines (8 I/Q pairs); digitizer = 16 ADC channels** (2× octal serial-LVDS ADC, or
>   the **OHWR FMC ADC 100M 14b 16cha** card — 16 ch = 8 I/Q pairs).
> - **IF egress: the 8-coax 8W8 D-sub is superseded → 16 coax** (connector class TBD).
> - **I/Q is now ANALOG** (from the mixer), not generated digitally: the DDC processes a **complex**
>   input; image rejection = analog I/Q balance (~29 dB) + digital correction; the composite-channel
>   cal extends to **per-channel I/Q gain/phase imbalance**. **DDC channel count (56) and the decimated
>   payload / 1G–2.5GbE budget are unchanged.**
> - **LO drive +21 dBm/port nominal**; **LO-IF isolation 15 dB min** (worse → more LO leakage per IF
>   pin; if-termination now on both IF pins).
>
> Sections below that still say "single IF / octal ADC / 8 channels / digital-IQ-via-CORDIC / 8W8"
> predate this; where they conflict, this note and `radar-project.md` are authoritative.

> **Reconciliation note (2026-06-16).** This document predated several firmed system decisions
> now recorded in the PCB workspace's **`radar-project.md`** (the project-level architecture of
> record). It has been updated to match them; where any older passage still conflicts,
> **`radar-project.md` is authoritative.** Superseding decisions folded in here:
> - **Digitizer FPGA is a bought SoM, not a direct BGA:** Trenz **TE0712** (Artix-7 **XC7A200T**)
>   on a carrier via Samtec **LSHM** board-to-board — no FMC, no BGA escape, no ScopeFun power
>   tree (the SoM owns DDR3 / config flash / power sequencing). This **reopens the open-source
>   toolchain constraint** (openXC7/nextpnr-xilinx on A200T is unverified; the TE0712 vendor flow
>   is Vivado) — see §4.6.
> - **ADC baseline is octal serial-LVDS** (AD9681 lean; LTC2145-14 dual-parallel kept as the
>   characterized alternative) — pin count is what makes the SoM B2B close (§4.5).
> - **No 10GbE.** Edge-decimate harder (~0.8–1 MHz subbands) → ~1.4–1.8 Gbps/board → **1G/2.5GbE
>   over SFP, GTP-as-PHY** (no Ethernet chip), aggregated by a **switch** to a 100G-class uplink —
>   not point-to-point to a NIC. 10.3125 Gbd is in-band for the 10 GHz aperture (§2.3, §4.6, §7.2).
> - **Clock backbone:** distribute a **low (~10 MHz) master reference** + phase-matched fan-out +
>   a **per-board jitter-cleaning PLL / network-synchronizer** → local 125 MHz, plus **SYSREF and
>   a ramp-start strobe**. Replaces "100 MHz OCXO on a passive splitter + one-time startup-tone
>   phase cal." See `clock_distribution_architecture.md` and §2.4 / §7.1.
> - **RF board is passive at 10 GHz** (no on-board PA), fed **~+30–31 dBm** LO at one SMA and split
>   1:8 on-board; LO amplification lives on the distribution bricks. The exciter is **three boards**
>   (Synthesizer → BPF → PA+Mixer), not one "TX board" (§6).
> - **RX→digitizer egress is a corner 8W8 combination-D-sub bulkhead** (8× shielded coax + a
>   filtered bias/control entry), not 8 loose edge-launch SMAs (§3.3, §4.7).

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
| RX channels per board | 8 (each → I/Q → **16 IF / 16 ADC ch**, 2026-06-23 IQ mixer) |
| Stage 1 ADC | 14-bit, 125 MSPS, serial-LVDS — **16 channels (8 I/Q pairs)**: 2× octal AD9681 or OHWR FMC-16cha (2026-06-23 IQ mixer) → ~12.0 ENOB |
| Stage 2 ADC | 16-bit, 250 MSPS + DDC → ~13.8–14.3 ENOB |
| Digitizer FPGA | Trenz **TE0712** SoM (Artix-7 **XC7A200T**) on a carrier via Samtec LSHM B2B (no FMC) |
| Backhaul | **1G/2.5GbE over SFP** (GTP-as-PHY, no 10GbE) → switch → 100G-class uplink |
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

No central FPGA handles this. DDC + decimation must live at the edge — they are the bandwidth-reduction step. After edge processing each board emits ~1.4–1.8 Gbps, which crosses a **standard 1G/2.5GbE link over SFP** (no 10GbE — see §4.6); a **commodity switch** aggregates the N boards into a single 100G-class uplink to the GPU workstation (switchable fabric, not point-to-point links — `radar-project.md` expansion rule #1).

### 2.4 ADC Phase Coherence

*(Reconciled 2026-06-16 — the earlier "fixed power-up offset absorbed by a one-time startup tone" model was too simple; full treatment in `clock_distribution_architecture.md` and `radar-project.md`.)*

Coherence is built in layers, each removing one disagreement:

1. **Shared low reference + per-board PLL.** A **low (~10 MHz) master reference** is fanned out
   phase-matched to every Digitizer Board; each board's **jitter-cleaning PLL / network-synchronizer**
   locks to it and multiplies up to the local 125 MHz ADC clock. Locking kills the inter-board
   **sample-RATE (ppm)** error — the term that actually breaks combining N boards (a free-running
   "set them all to 125 MHz" does not).
2. **Divider phase — SYSREF vs cal (open).** When each PLL divides down to 125 MHz the divider can
   power up in a different phase **each boot**, so two locked boards can sit ~1 sample period apart
   until that is resolved. A **SYSREF**-capable clock IC pins it deterministically; without SYSREF it
   re-randomises every power cycle and must be re-measured by calibration on each boot. **Which way to
   go is an open clock-IC decision** (`radar-project.md` → *The two clock-IC classes*).
3. **Ramp-start strobe** anchors the per-carrier NCO phase to a consistent ramp instant — the φ₀
   anchor the synthetic-wideband (FDMA) combine needs (§5; `clock_distribution_architecture.md` §5).
4. **Calibration of the residual.** Static terms — the inter-board cable delays, the SMA/D-sub egress
   run, ADC gain/timing spread — fold into a **composite-channel calibration** (keyed to the RF-board
   ⊕ digitizer ⊕ harness tuple), seeded on the bench and refined **over-the-air at join**, not a single
   startup tone. There are **no on-board cal-injection couplers** (adding them later is a respin).

### 2.5 Board Split — Digitise on a Separate FR4 Board

Earlier revisions placed the antennas, RF chain, ADC array, and an FPGA mezzanine module all on a single Rogers board. This forced a 381-ball BGA and its 6–8 layer escape routing onto a 4-layer Rogers RF substrate (or an awkward Rogers/FR4 hybrid stackup, or a DF40 mezzanine workaround), with the BGA fanout vias threatening the RF ground planes.

The settled design splits the front-end into two boards joined by SMA coax:

- **RF Board** (Rogers 4003C, 4-layer) — antennas + full analogue receive chain through the dechirp mixer. Outputs one beat signal per channel as a baseband (≤~60 MHz) analogue signal, egressing through a **corner 8W8 combination-D-sub bulkhead** (8× shielded coax contacts + a filtered bias/control entry). Pure RF/analogue; no ADCs, no FPGA, no BGA, no LVDS.
- **Digitizer Board** (FR4, multilayer) — octal beat inputs, anti-alias LPFs, octal serial-LVDS ADC, **Trenz TE0712 SoM (Artix-7 XC7A200T) on a carrier via Samtec LSHM B2B (no FMC)**, SFP output (1G/2.5GbE), ADC SPI, clock IC, and power. All LVDS runs between the ADCs and the SoM are on this one board, so trace-length matching, impedance, and stackup are fully controlled — a solved PCB-layout problem with no inter-board interface uncertainty.

**Why this is the right call:**

- The inter-board interface is a ~62.5 MHz baseband signal over 50 Ω shielded coax (8W8 D-sub) — completely standard, low-risk, any cable length. No high-speed inter-board LVDS, no PMODs.
- The Rogers RF Board stays a clean 4-layer board. **The SoM removes the BGA-escape problem entirely** — the A200T BGA, its DDR3, config flash, and the strict 7-series power sequencing all live on the bought module; the carrier only routes the LSHM B2B, the ADC LVDS, the SFP pair, and power. (The earlier direct-ECP5/A100T-BGA + ScopeFun-power-tree plan is retired — see §4.6.)
- Each board is fabbed where it is natural and cheap: Rogers at Würth, FR4 at JLCPCB Advanced.
- RF bring-up and digital bring-up proceed fully independently; risk is isolated.
- A Stage 2 ADC upgrade (§10) respins **only** the Digitizer Board — the RF Board is ADC-agnostic.

---

## 3. System Architecture Overview

### 3.1 Signal Flow

```
Exciter chain (Synthesizer → BPF → PA+Mixer, §6):
  ~10 MHz master ref ──┬── chirp DDS/DAC + FDMA carriers ── BPF (subband plan) ── PA+Mixer
                       │                                                            ├── TX antenna
                       │                                                            └── reference chirp (LO)
                       ├── phase-matched fan-out: ref clock + SYSREF + ramp strobe ──► every Digitizer Board
                       └── (ref chirp distributed via cascadable 1:8 fan-out bricks, per-tier re-amplified)

RF Board (per channel), passive at 10 GHz:
  [Patch] → [BPF] → [Limiter] → [LNA] → [BPF] → [Dechirp mixer] → [IF term.] → 8W8 D-sub bulkhead
                                                       ↑
                                          ref chirp (LO, ~+30 dBm at SMA → 1:8 Wilkinson on-board)
                                                                          │ beat signal
                                                                          │ (≤~60 MHz, shielded coax)
                                                                          ▼
Digitizer Board (per channel):
  coax in → [Anti-alias LPF] → [balun] → [octal serial-LVDS ADC] → [SoM/FPGA: DDC (NCO+CIC) + decimate]
                                              ▲                                       │
                       local jitter-cleaning PLL/clock IC ── 125 MHz, SYSREF          ▼ GTP → SFP (1G/2.5GbE)
                       (locked to the ~10 MHz ref; ramp strobe → NCO reset)           │
                                                                       switch (aggregate) → 100G uplink → GPU
```

The anti-alias LPF sits on the Digitizer Board immediately ahead of the ADC, not at the mixer — see §4.3.6. The mixer IF port is terminated locally on the RF Board (`IF term.`); the termination scheme is an open decision (pad vs. diplexer, §4.3.6, §12).

### 3.2 Reference Chirp as LO

The reference chirp is a copy of the transmitted chirp at 10.0–10.5 GHz. When it drives the dechirp mixer LO port, the output is the beat signal `f_beat = kτ`. The RF Board routes it directly to the mixer LO port. Required LO level depends on mixer choice (see Section 4.3.5).

### 3.3 Interface Summary

| Interface | Signal | Direction | Connector |
|---|---|---|---|
| Reference chirp LO | 10.0–10.5 GHz; **~+30–31 dBm** at the board SMA (passive 1:8 Wilkinson on-board → ~+19 dBm/mixer) | PA+Mixer / fan-out brick → RF Board | SMA edge-launch |
| Beat signals (×8) | Baseband ≤~60 MHz analogue, 50 Ω, per-channel shielded | RF Board → Digitizer Board | **8W8 combination D-sub** (8× coax contacts) |
| Timing harness | **~10 MHz master ref + SYSREF + ramp-start strobe** | Synth → Digitizer Board | coax / harness (travelling together) |
| Backhaul data link | Decimated complex IQ, **1G/2.5GbE** | Digitizer Board → switch | SFP cage (GTP-as-PHY) |
| DC power (RF Board) | bias rails (LNA LDO, etc.) | PSU → RF Board (filtered bias entry) | D-sub bias contacts / header |
| DC power (Digitizer Board) | SoM-defined rails + ADC/clock analog rails (TBD at carrier design) | PSU → Digitizer Board | Molex or header |

---

## 4. Front-End: RF Board and Digitizer Board

The front-end is two boards joined by SMA coax (§2.5):

- **RF Board** (Rogers 4003C, 4-layer) — 8 RX antenna elements and the full analogue receive chain through the dechirp mixer and IF termination. Covered by §4.1–§4.3.
- **Digitizer Board** (FR4, multilayer) — anti-alias filtering, octal serial-LVDS ADC array, Trenz TE0712 SoM (A200T), SFP fibre output, and power. Covered by §4.4–§4.6.

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

> **SUPERSEDED 2026-06-23 — the primary is now the Qorvo CMD258C4 I/Q mixer** (7.5–13 GHz, Ceramic QFN
> 4×4 mm 24-lead), run as a **complex-IQ downconverter**: each channel emits I (IF1) + Q (IF2), doubling
> to 16 IF / 16 ADC channels; image rejection is digital (analog I/Q balance ~29 dB + per-channel cal).
> LO drive +21 dBm/port nominal; LO-IF isolation 15 dB min. See `radar-project.md` → *Dechirp mixer → IQ
> mixer*. The CMD253C3 below is retained as the single-real-IF lineage.

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

- **Anti-alias LPF** — relocated here from the RF Board (§4.3.6). **General rule: the AA-LPF is tailored to whatever ADC the Digitizer Board carries — its corner tracks the chosen ADC's Nyquist (a matched pair), never a fixed number.** Set the passband to the maximum beat frequency of interest, place the stopband to reject everything above the ADC's Nyquist (fs/2) before it samples, and pick fs so fs/2 clears the max beat with ~20–25 % margin. For any 125 MSPS Stage-1 candidate (LTC2145-14 *or* the octal-serial AD9681, §4.5) the corner is ~55–60 MHz → Mini-Circuits LFCN-80+ (DC–80 MHz, SMD). Re-derive and swap the LFCN value as a matched pair with the ADC whenever the sample rate changes (e.g. Stage 2) — same footprint.
- **Balun / single-ended drive** — converts the single-ended beat signal to the ADC differential input. The MAMX-011035 IF is DC-coupled and the beat band reaches near-DC, so preserve the DC path (or apply the defined DC bias the LTC2145-14 input common-mode spec requires) rather than AC-coupling.
- **Grounding note** — each coax shield ties the RF Board and Digitizer Board grounds together (×8 paths). This is generally desirable (shared reference), but with a DC-coupled IF, watch board-to-board ground-potential offset in the power/grounding scheme.

### 4.5 Digitizer Board — ADC Array

#### 4.5.1 Selection Rationale

All 14-bit pipeline ADCs at 100–125 MSPS cluster around 73–75 dBFS SNR — this is a physics constraint on the technology tier, not a design differentiator. The key discriminators are SFDR, power consumption, and upgrade path.

For our DC to 40 MHz beat signal, ENOB is SNR-limited at approximately 12.0 bits. SFDR is not the binding ENOB constraint at these frequencies, but better SFDR reduces ADC harmonic spurs that appear as false targets.

> **Co-candidate added 2026-06-16 — octal-serial AD9681.** The current digitizer plan (no-FMC FPGA-SoM carrier, `radar-project.md`) prefers an **octal serial-LVDS** ADC over the dual parallel-DDR LTC2145-14 on **pin count**: octal-serial lands ~18–20 LVDS pairs for 8 channels vs the LTC2145's 128 parallel data pins (§4.5.4), which is what lets the ADC→FPGA fabric fit the Samtec LSHM board-to-board budget. The **AD9681** (octal, 14-bit, 125 MSPS, ~85 dBc SFDR) is the lean candidate — same SNR tier as LTC2145-14; it takes the Stage-2 serial interface (§10.4) at Stage 1 without the 16-bit/250 MSPS/DDC jump. Settle AD9681 vs LTC2145-14 side by side once the LSHM pin/bank map is in hand; the discriminator is interface/pin-count, not ENOB (a wash). The LTC2145-14 detail below remains the parallel-interface baseline.

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

#### 4.6.1 Selected Part: Trenz TE0712 SoM (Artix-7 XC7A200T)

The digitizer FPGA is now a **bought SoM**, not a direct BGA: the **Trenz TE0712** (Artix-7 **XC7A200T**, 1 GB DDR3, 32 MB config flash) on the carrier via **Samtec Razor Beam LSHM** board-to-board (0.50 mm) — **no FMC, no BGA escape**. The carrier routes only the LSHM B2B, the ADC serial-LVDS, the SFP pair, the clock IC, and power; the SoM owns the BGA, DDR3, config, and the strict 7-series power sequencing.

| Parameter | Value |
|---|---|
| Module | Trenz **TE0712** |
| FPGA | Artix-7 **XC7A200T** |
| DSP slices | **740** (DSP48E1) |
| GTP transceivers | 4 lanes ≤ **6.6 Gbps** (used for 1G/2.5GbE only — §4.6.5) |
| Carrier mount | Samtec **LSHM** B2B — JM1/JM2 = 2×100-pin (ADC LVDS), JM3 = 60-pin (4 GTP + 125 MHz MGT refclk → SFP) |
| On-module | 1 GB DDR3, 32 MB config flash, power sequencing |
| Toolchain | **OPEN** — TE0712 vendor flow is Vivado; openXC7/nextpnr-xilinx A200T support unverified (below) |

**Why a SoM (vs the earlier direct-BGA A100T / ECP5 plan).** Two things changed: (1) decimating to ~0.8–1 MHz subbands drops the per-board link to ~1.4–1.8 Gbps (§4.6.2), so the SerDes ceiling that drove "A100T-GTP over ECP5" is no longer the constraint — **the A200T is chosen for its 740 DSP48 (vs 240 on A100T)** to hold 56 streaming DDC channels comfortably, not for transceiver speed. (2) Buying the SoM **removes the 200k-BGA escape, DDR3 layout, and 7-series power-tree risk** from the board entirely — the dominant Stage-1 digital risk in the old plan.

> **Open — open-source toolchain constraint.** The earlier plan picked the A100T partly for **openXC7** (Yosys + nextpnr-xilinx, "100% open-source"). The TE0712 ships with a **Vivado** flow, and openXC7/nextpnr-xilinx support for the **A200T** is unverified. If the open-source toolchain is still a hard requirement, confirm A200T coverage (or reconsider the SoM/part); otherwise this is a deliberate relaxation to Vivado. Flagged for the user.

**FPGA tasks on the Digitizer Board:**

| Task | Implementation |
|---|---|
| ADC serial-LVDS capture | ISERDES + IDELAYE2 (+ IDELAYCTRL) deserialize the octal serial-LVDS; per-board eye alignment (local, downstream of the aligned sample instant — does not affect cross-board coherence) |
| DDC per subband (×56) | complex NCO mixer → CIC decimation → comp/cleanup FIR; **7 subbands per physical ADC channel × 8 = 56** |
| Strobe → NCO reset | ramp-start strobe (LVDS → synchronizer → edge detect) reloads all NCO phase accumulators on a common edge; sample-counter timestamp into the stream |
| Byte-interleaving packer | Lines up IQ samples from all 56 DDC channels into structured data frames; no FPGA-side FFT |
| ADC SPI configuration | One-time startup |
| Data framing | Frame DDC IQ samples with channel ID, subband ID, and timestamp |
| GTP TX | 1G/2.5GbE over SFP (GTP-as-PHY, no Ethernet chip); one fiber per board |

#### 4.6.2 DSP Architecture — Frequency-Selective Decimation

Each of the 8 physical ADC channels feeds a bank of **7 DDC (Digital Down-Converter) channels — one per FDMA subband — giving 56 DDC channels total** (the 7 subbands are 7 frequency-diverse looks at the same range slice, not range gates). Each DDC:

1. Applies a **complex NCO mixer** to translate its assigned subband to baseband (full-rate stage)
2. Follows with a **CIC decimator + droop-comp FIR + cleanup/halfband FIR** to isolate the subband and drop the rate (≈÷125 → ~1 MSPS complex; the comp/cleanup FIR — not the CIC — delivers the 70–80 dB inter-carrier stopband)
3. Produces complex (I+Q) baseband samples at the decimated output rate

Replicate only the full-rate mixer; time-multiplex everything downstream (vendor FIR-Compiler "channels" param). Sizing is **LUT/DSP-bound** on the A200T (740 DSP, ~150 used; the CIC bank is the LUT driver), not transceiver-bound — which is why the part is chosen on DSP, not SerDes.

**No FPGA-side FFT.** A **byte-interleaving packer** lines up the raw time-domain IQ samples from all 56 DDC channels into structured frames and streams them raw; the host computes FFTs/beamforming in software. (An on-chip range FFT earns nothing on bandwidth here — the decimation already *is* the range gate — so it stays on the host.)

**Data rate budget (the lever that kills 10GbE):**
```
56 DDC channels × f_out × 32 bits (16-bit I + 16-bit Q)
  1.0 MHz subbands:   56 × 1.0e6 × 32 = 1.79 Gbps payload
  0.8 MHz (chosen):   56 × 0.8e6 × 32 = 1.43 Gbps payload
  → fits 2.5GbE (~2.3 Gbps net after framing) comfortably           ✓
  → one SFP fiber per board, 1G or 2.5G                              ✓
```

The 16-bit IQ width is **not** shaved for the link — it carries the ~3 ENOB of decimation processing gain (10·log₁₀(62.5/0.8) ≈ 18.9 dB); dropping bits would throw that away. The **band width** (0.8 MHz) is the chosen tuning lever instead. The old 2–3 MHz / 4–6 Gbps / 10G-class sizing is retired.

#### 4.6.3 Carrier Integration: SoM + Octal ADC on One FR4 Board

The ADC array and the SoM share one FR4 carrier — no mezzanine, no daughter cards, **no FMC** (§2.5). All differential serial-LVDS traces between the octal ADC and the SoM's LSHM B2B are controlled and length-matched on-board — a solved PCB layout problem.

**The SoM subsumes the old BGA-integration risk.** With the TE0712, the carrier no longer carries the FPGA BGA, its DDR3, the config flash, or the strict 7-series power tree (VCCINT → VCCAUX → VCCIO) — all of that is on the module and verified by Trenz. The earlier **ScopeFun** reference (used to de-risk exactly the A100T BGA power-up and LVDS routing) is therefore retired as the baseline; the relevant references are now the **Trenz TE0712 TRM + carrier reference designs** — pull the exact LSHM dash-code, the JM1/JM2/JM3 pin/bank table, and the SoM input-rail spec before footprinting.

**Carrier support circuitry (what's left to design):**

| Component | Part | Notes |
|---|---|---|
| FPGA | Trenz TE0712 SoM (A200T) | module; mounts via LSHM B2B |
| SoM input rail | per TE0712 TRM | the module regulates its own VCCINT/VCCAUX internally |
| ADC analog / IO rails | LDO/buck (1.8 V class) | clean, separate from the digital corner |
| Clock IC | jitter-cleaner / network-synchronizer | own clean rail; see clock backbone (§2.4, §7.1) |
| ADC config | SPI from the SoM | one-time startup |
| Passives / decoupling | — | per SoM + ADC + clock |

#### 4.6.4 IQ Generation via CORDIC DDC

> **Updated 2026-06-23 (IQ mixer):** baseband **I/Q is now produced in ANALOG by the CMD258C4 mixer**
> (IF1/IF2 → two ADCs per channel). The DDC's NCO/CORDIC rotation now operates on a **complex** input
> (per-subband translation + decimation as before); it no longer manufactures I/Q from a real signal.
> Net: DDC channel count (56) and decimated payload unchanged; the analog IQ resolves beat **sign** (no
> negative-frequency fold) and the host adds **per-channel I/Q-imbalance correction**. See
> `radar-project.md` → *Dechirp mixer → IQ mixer*.

Each DDC channel applies a CORDIC-based complex rotation to translate its assigned subband to baseband, producing I and Q components directly. This replaces the Hilbert FIR approach used in the prior ECP5 plan. Advantages:

- IQ is produced directly from CORDIC rotation — no Hilbert approximation; image rejection is limited only by CORDIC iteration depth and quantisation noise (theoretically superior to the 60 dB of a 64-tap Hilbert FIR)
- Each DDC independently tunes to any subband via its NCO frequency word — firmware-reconfigurable without hardware changes
- The 56 DDC channels pipeline across the DSP48E1 fabric (A200T = 740 DSP48, ~150 used — comfortable); time-multiplexed engines reduce resource usage further if needed

The GPU workstation receives raw IQ from each DDC and applies FFT and beamforming processing entirely in software.

#### 4.6.5 Fibre Output — 1G/2.5GbE over SFP (GTP-as-PHY, no 10GbE)

The SoM's GTP transceiver drives a single **SFP** cage at a **standards-compliant** 1000BASE-X (1.25 Gbd) or 2.5GBASE-X (3.125 Gbd) line rate — both far inside the Artix-7 GTP ≤6.6 Gbps ceiling. At ~1.43–1.79 Gbps aggregate DDC payload (§4.6.2), **2.5GbE carries the full 56-channel stream** over one fiber; prefer **1G** if the decimated data fits (fewer in-band clock sources near the analog, smaller uplink).

| Parameter | Value |
|---|---|
| Framing | UDP/IP Ethernet over 1000BASE-X / 2.5GBASE-X |
| Physical | **GTP = the PHY** → SFP cage → fiber (SFP is a dumb optical SerDes; **no Ethernet chip on the carrier**) |
| Line rate | 1.25 Gbd (1G) or 3.125 Gbd (2.5G) |
| Aggregate DDC payload | ~1.43–1.79 Gbps (0.8–1.0 MHz subbands) |
| Host side | **commodity switch** aggregating N boards → 100G-class uplink to the GPU (not a per-board NIC) |
| Extra parts | 2 diff pairs + AC caps, slow I²C/MOD control; a 156.25 MHz osc **only if 2.5G** |

> **Why no 10GbE (hard datapoint).** Raw 8-ch (~14 Gbps) cannot cross the GTP — but rather than chase a custom rate or an external 10G MAC/PHY, the architecture **decimates on the FPGA first** (§4.6.2) so a standard 1G/2.5GbE link suffices. Beyond cost, **10.3125 Gbd is an in-band emitter for the 10.0–10.5 GHz aperture** — a SerDes reason to keep 10G away from the RF apertures in the enclosure. Coherence does **not** ride the Ethernet (no PTP): samples align by index off the shared ramp-start strobe; the switch only aggregates bandwidth.

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

**Digitizer Board** (FR4). Two domains in opposite corners (per `radar-project.md` floorplan/EMI); solid continuous ground — partition by placement + supply, do **not** slot the plane.

```
   octal beat in (8W8 D-sub)     timing harness in (~10 MHz ref + SYSREF + strobe)
        │ │ │ │ │ │ │ │                  │
┌─────────────────────────────────────────────────┐
│  ANALOG CORNER                                   │
│  Anti-alias LPF → balun (×8) → octal serial-LVDS │
│  ADC · jitter-cleaning clock IC (clean rail)     │
├─────────────────────────────────────────────────┤
│  ADC → SoM serial-LVDS (matched, on-board)       │
├─────────────────────────────────────────────────┤
│  DIGITAL CORNER                                  │
│  Trenz TE0712 SoM (A200T) on LSHM B2B · SPI ·    │
│  GTP → SFP cage (JM3 side) · power               │
└─────────────────────────────────────────────────┘
                       │
                  Fibre out (SFP, 1G/2.5GbE)
```

In-band aggressors to keep off the ADCs: the 125 MHz ADC clock + DSP fabric-clock harmonics and any GTP refclk (125/156 MHz land in DC–500 MHz); the SFP Gbaud fundamental is out of band (the AA-LPF rejects it).

---

## 5. Waveform Strategy

All waveform parameters are controlled in firmware. Hardware is waveform-agnostic.

> **Reconciliation note (2026-06-16).** The current direction is **≈7-subband FDMA with non-uniform (irregular) subband spacing** (Cohen, Cohen & Eldar), realized by the dedicated **BPF board** (§6) and matched by **56 = 7×8 DDCs** on the digitizer (§4.6.2). This supersedes the hybrid **4-FDM × 2-CDM** recommendation below as the baseline; the FDMA/CDM trade-off discussion is kept as the decision record. The per-channel chirp bandwidth and ~50 MHz beat span (→ AA-LPF ~55–60 MHz) are unchanged. **Confirm the final subband count + spacing plan with the user before BPF design.**

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

## 6. Exciter Chain (Synthesizer · BPF · PA+Mixer) and LO Distribution

*(Reconciled 2026-06-16 — the single "TX board" is now a three-board exciter chain; full detail in `radar-project.md` → *The boards* and *LO distribution at scale*.)*

The central exciter is **shared by the whole array** (range-correlated phase-noise cancellation requires every RX board and the TX to ride the **same** chirp instance). It is three boards:

1. **Synthesizer board** — FPGA + DAC generate the FMCW chirp + the FDMA carriers; holds the **master timing reference** (~10 MHz) and runs the management plane (Ethernet-over-SFP). Likely an FPGA SoM + carrier, as the digitizer.
2. **BPF board** — sets the FDMA **subband plan**: center freqs + bandwidths of the ≈7 carriers, **non-uniform spacing** (Cohen, Cohen & Eldar) so the synthetic-wideband combine is ambiguity-free. Its analog stopbands share the 70–80 dB inter-carrier isolation budget with the digitizer's per-band FIR (§4.6.2).
3. **PA + Mixer board** — upconverts to 10.0–10.5 GHz and provides power to **both** the **TX chirp** (→ transmit antenna) and the **reference chirp** (→ every RX RF board's LO). Head of the LO-distribution tree.

**LO distribution — RF boards are passive; amplification lives on the bricks.** Each RF board is fed **~+30–31 dBm** at its LO SMA and splits it 1:8 on-board (Wilkinson, −10.17 dB → ~+19 dBm/mixer); **no PA on the RF board** (keep power amplifiers off the µV-level LNA inputs). At scale a single source cannot feed N boards (+30 + 10·log₁₀N would be non-physical), so the **cascadable 1:8 fan-out bricks** carry **distributed amplification** — each heatsinkable brick re-amplifies its tier with a GaN X-band source-PA (QPA1003/1011 class). The brick/PA design is deferred (`radar-project.md`).

**Chirp generator candidates (Synthesizer board):** DDS-based (AD9914-class at IF + X-band upconverter) or PLL-based (ADF4159 ramp controller + VCO). The ramp generator's **explicit digital ramp-start event** is tapped as the array-wide strobe (§5; `clock_distribution_architecture.md` §5) — not recovered from the analog chirp.

---

## 7. Central Unit

### 7.1 Master Reference and Distribution

The master timing reference lives on the **Synthesizer board** and disciplines the whole array (§2.4; `clock_distribution_architecture.md`).

| Parameter | Value |
|---|---|
| Type | OCXO / TCXO (low-frequency master) |
| Frequency | **~10 MHz** (distribute LOW; each board multiplies up locally) |
| Distribution | **phase-matched fan-out** (equal-length lines, clock-distribution-grade low-skew buffer) carrying **ref clock + SYSREF + ramp-start strobe** together; cascaded via the 1:8 fan-out bricks |
| Per board | a **jitter-cleaning PLL / network-synchronizer** locks to the ref and synthesizes the local 125 MHz ADC clock (clock-IC class is an open decision — §2.4) |

Distribute ~10 MHz rather than 125 MHz: low frequency suffers less cable loss/skew, and the per-board PLL both multiplies up **and cleans** the cable-induced jitter (narrow loop BW → the clean local oscillator sets the ADC clock). Note clock jitter does **not** limit ENOB here — the ADC samples the low-frequency dechirped beat — so the reference need not be heroic; the PLL's job is lock + clean, and **SYSREF / divider alignment** is the property that actually matters for cross-board coherence (§2.4).

### 7.2 GPU/CPU Workstation

The central unit is a GPU/CPU workstation. The N per-board 1G/2.5GbE fibres land on a **commodity switch** that aggregates them into a **100G-class uplink** to the workstation (switchable fabric, not per-board NIC ports — `radar-project.md` expansion rule #1).

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

A "board pair" is one RF Board + one Digitizer Board joined by the **8W8 D-sub beat harness**. Per-board fibres **aggregate at a commodity switch** (100G-class uplink); the chirp reference, clock, and sync are distributed by **cascadable 1:8 fan-out bricks** (per-tier re-amplified) — 8 boards = one brick, 64 = two tiers.

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

8 serial lanes at 1 Gbps is well within the SoM's LVDS I/O. If Stage 1 used the parallel LTC2145-14, Stage 2's serial interface routes far more easily despite the faster ADC; with the **AD9681 octal-serial baseline**, Stage 1 is already serial and the two are comparable.

### 10.5 Stage 2 Respins Only the Digitizer Board

ADC3668 is not pin-compatible with the Stage-1 ADC. Because of the board split (§2.5), **the RF Board is untouched at Stage 2** — it never saw the ADC. Stage 2 is a new Digitizer Board only; the **SoM (TE0712 / A200T)**, the 8W8 D-sub beat interface, and the FPGA firmware structure are all preserved. Changes, all confined to the Digitizer Board:

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
| LFE5UM5G-85F-8BG381C (ECP5-85F) | 5 Gbps/lane SerDes ceiling insufficient for 64 DDC channels at full bandwidth; replaced with higher-transceiver-speed part | Xilinx XC7A100T (Artix-7) — **later superseded by the Trenz TE0712 SoM (A200T)** once decimation dropped the link below the SerDes constraint; see banner / §4.6 |

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

No ADC, anti-alias LPF, FPGA, or SFP on the RF Board — those moved to the Digitizer Board (§2.5).

†CMD319C3 at 25+ unit price break ($48.97/unit); 40 units across 5 boards.

### 11.4 Digitizer Board Component BOM (Stage 1)

> **Reconciled 2026-06-16 — re-estimate pending.** This BOM moved to a **bought SoM + octal-serial ADC + SFP (1G/2.5G)**; several Stage-1 line items (direct FPGA, its core/IO regulators, config flash, the four parallel LTC2145s, 10G optics) are replaced. Prices marked *(re-est.)* need fresh quotes; the SoM and the octal ADC are the swing items. Note the octal AD9681 (one chip) collapses the old 4× LTC2145 ($500) line, so the subtotal **drops** despite the SoM.

| Ref | Component | Part | Qty | Unit price | Line total |
|---|---|---|---|---|---|
| J1 | Beat egress | 8W8 combination D-sub (8× coax) bulkhead | 1 | ~$25 (re-est.) | ~$25 |
| J2 | Timing harness in | ref + SYSREF + strobe connector | 1 | ~$5 | ~$5 |
| FL1–FL8 | Anti-alias LPF | Mini-Circuits LFCN-80+ (tied to ADC Nyquist) | 8 | ~$8 | $64 |
| T1–T8 | Input balun | Wideband balun / single-ended drive | 8 | ~$2 | $16 |
| U1 | ADC Stage 1 | **AD9681** octal serial-LVDS (lean) — or LTC2145-14 ×4 (alt) | 1 (or 4) | ~$120 (re-est.) | ~$120 |
| U2 | FPGA | **Trenz TE0712 SoM (A200T)** | 1 | ~$250 (re-est.) | ~$250 |
| U3 | Clock IC | jitter-cleaner / network-synchronizer (class TBD, §2.4) | 1 | ~$15 (re-est.) | ~$15 |
| U4 | ADC analog / IO rails | LDO/buck (1.8 V class) | 1–2 | ~$3 | ~$5 |
| J3 | SFP cage | Single-port SMD | 1 | ~$8 | $8 |
| J4 | SFP transceiver | **1G/2.5G** SX/LX | 1 | ~$20 (re-est.) | ~$20 |
| Passives | R, C, decoupling | SMD 0402/0201 | ~200 | — | ~$25 |
| **Digitizer Board subtotal** | re-estimate pending SoM + ADC quotes | | | | **~$550 (re-est.)** |

Dropped vs the old BOM — now on the SoM: FPGA core 1.0 V reg, FPGA I/O 1.8 V LDO, config flash (the SoM input rail replaces them).

### 11.5 PCB Fabrication and Assembly

**RF Board** — Rogers 4003C, 4-layer, controlled impedance, ~170 mm × 110 mm (antenna array + RF chain + IF termination/output). Würth (formal quote required; no online Rogers pricing).

| Item | Per board | For 5 boards |
|---|---|---|
| Rogers 4003C, 4-layer, controlled impedance | ~$200 | ~$1,000 |
| SMT assembly (~80 placements) | ~$160 | ~$800 |

**Digitizer Board** — FR4, controlled-impedance LVDS, ~150 mm × 100 mm. **Layer count drops without the BGA escape** (the SoM mounts via LSHM B2B; the carrier routes the B2B + ADC serial-LVDS + SFP — likely ~6 layer; re-estimate). JLCPCB Advanced.

| Item | Per board | For 5 boards |
|---|---|---|
| FR4 ~6 layer, controlled impedance (re-est.) | ~$25 (est.) | ~$125 (est.) |
| SMT assembly (no BGA — octal ADC QFN + LSHM receptacles + passives) | ~$60 (est.) | ~$300 (est.) |

### 11.6 Total Cost Summary — 5 Board Pairs

| Item | Per pair | For 5 pairs |
|---|---|---|
| RF Board components (CMD253C3) | ~$957 | ~$4,785 |
| *(Alt. build with MAMX-011035)* | *~$983* | *~$4,915* |
| *(Budget build with LTC5548 mixer)* | *~$684* | *~$3,420* |
| Digitizer Board components (SoM + octal ADC, re-est.) | ~$550 (re-est.) | ~$2,750 (re-est.) |
| RF Board PCB + assembly (Würth) | ~$360 | ~$1,800 |
| Digitizer Board PCB + assembly (JLCPCB Advanced) | ~$110 | ~$550 |
| Misc (SMA coax cables, shipping, consumables) | ~$40 | ~$200 |
| **Total (CMD253C3 primary)** | **~$2,264** | **~$11,320** |
| **Total (MAMX-011035 alt.)** | **~$2,290** | **~$11,450** |
| **Total (LTC5548 budget mixer)** | **~$1,991** | **~$9,955** |

> **Totals predate the digitizer re-estimate (2026-06-16).** The rows above still carry the old ~$797 digitizer components; with the SoM + octal-ADC BOM (~$550 re-est.) each pair drops ~$250. Re-total once SoM / ADC / clock quotes land.

Primary cost drivers per board pair are now on the **RF Board**: mixer ($465, ~23%), LNA ($392, ~19%), and Würth Rogers fabrication ($360). On the digitizer the swing items are the **SoM (~$250)** and the **octal ADC (~$120)** — both to be reconfirmed before ordering (the 4× LTC2145 $500 line is retired). All RF-board component prices confirmed at prototype quantities from Digikey/Mouser.

---

## 12. Open Items and Next Steps

| Item | Description | Priority |
|---|---|---|
| PIN limiter part selection | Select an X-band PIN limiter. MACOM MADL-011088 (6–12 GHz) is the lead candidate but is a bare die needing an on-board shunt limiter circuit; evaluate any packaged SMD alternatives. Affects the Rogers layout. Verify insertion loss, incident-power handling, and recovery time against the worst-case TX-leakage scenario | Medium |
| CMD319C3 stock | **Order immediately** — 35-week factory lead time; 455 in stock at Digikey now; need 40 units for 5-board run | High |
| QPM1000 evaluation | Evaluate QPM1000 vs CMD319C3 as LNA: confirm NF @ 10 GHz, gain, P1dB, supply, package, price. If QPM1000 NF is meaningfully lower, it displaces CMD319C3 and updates the Friis budget | High |
| CMD253C3 datasheet verification | Price confirmed ($58.14/25+ Digikey, 998 in stock). Confirm remaining specs from Qorvo datasheet: LO drive level @ 10.25 GHz, IIP3 @ 10.25 GHz, conversion loss @ 10.25 GHz, IF port DC coupling lower cutoff. Update §4.3.5 with verified specs | High |
| IF termination scheme | **Decide before production:** 3 dB resistive pad (prototype) vs. IF diplexer (cleaner mixer termination, removes 20.5 GHz before the coax, more parts). Characterise mixer IIP3 / spurious under each (§4.3.6) | High |
| ADC stock | Confirm stock for the chosen ADC — AD9681 (1/board, lean) or LTC2145-14 (4/board, alt) | High |
| ADC3668 price | Confirm ADC3668IRTD exact price at Digikey before Stage 2 BOM lock | High |
| ADC interface | Verify the chosen octal-serial-LVDS pinout (AD9681) — or LTC2145-14 DDR if parallel; confirm IF input range matches the beat-signal level over coax (post-termination) | High |
| MAMX-011035 stock | Order 40 units from Mouser (5 RF Boards × 8 channels) | High |
| Inter-board beat harness (8W8 D-sub) | Select the 8W8 combination-D-sub bulkhead + coax contacts; specify and length-match the 8 beat runs to the per-channel phase budget (§2.4); confirm DC-coupled path and shield-grounding scheme (§4.4.1) | High |
| Exciter chain design | Synthesizer (FPGA + DAC + master ref), BPF (FDMA subband plan), PA+Mixer (upconvert + power TX & ref chirp); fan-out bricks (distributed LO amplification) — §6 | High |
| Antenna simulation | Simulate 10.25 GHz patch element and corporate feed in openEMS | High |
| Pre-LNA filter | Design 2-pole coupled-line BPF on Rogers 4003C; run tolerance corners | Medium |
| Post-LNA filter | Design 5-pole interdigital BPF on Rogers 4003C | Medium |
| Anti-alias LPF (Digitizer Board) | Select Mini-Circuits LFCN part matched to the chosen ADC's Nyquist (~55–60 MHz at 125 MSPS); place ahead of the ADC, swap as a matched pair at Stage 2 (§4.4.1) | Medium |
| Clock-IC class (digitizer) | **Narrowed (2026-06-16):** a **Class A network-synchronizer that ALSO provides deterministic, power-cycle-repeatable cross-board SYSREF/sync** (LMK05318B / FemtoClock 3 RC3 / ZL3073x class, no VCXO); discrete + VCXO (Class B) is fallback. Verify the *cross-device* determinism spec at BOM. See `radar-project.md` → *The two clock-IC classes* / *Middle path found* | High |
| Open-source toolchain vs SoM | Confirm whether openXC7/nextpnr-xilinx covers the **A200T**, or accept the TE0712 **Vivado** flow (relaxing the prior "100% open-source" constraint) — §4.6.1 | High |
| ADC settle: AD9681 vs LTC2145-14 | Decide octal-serial AD9681 (lean, ~18–20 pairs) vs dual-parallel LTC2145-14 (×4, ~64 pairs) once the LSHM pin/bank map is in hand — interface/pin-count is the discriminator, ENOB a wash (§4.5) | High |
| Digitizer Board schematic + layout | **TE0712 SoM (A200T) on a carrier via Samtec LSHM B2B** (no BGA escape): pull the LSHM dash-code + JM1/JM2/JM3 pin/bank table from the Trenz TRM; route ADC serial-LVDS (length-matched), SFP pair on JM3, clock IC on a clean rail; analog/digital corner split, solid ground | Medium |
| Input balun | Select wideband single-ended→differential balun (or DC-coupled drive) for the ADC input on the Digitizer Board (§4.4.1) | Medium |
| Backhaul — confirm 1G vs 2.5G | Pick 1000BASE-X or 2.5GBASE-X per the decimated rate (~1.43–1.79 Gbps fits 2.5G; prefer 1G if it fits — fewer in-band clocks). GTP-as-PHY, no 10G, no Ethernet chip (§4.6.5) | Medium |
| FPGA firmware skeleton | ADC serial-LVDS capture (ISERDES+IDELAY) → **56** DDC (NCO mixer + CIC + comp/cleanup FIR) → ramp-strobe NCO reset + timestamp → byte-interleaving packer → GTP / 1–2.5GbE; toolchain per the open-source-vs-Vivado decision | Medium |
| Clock & sync distribution | ~10 MHz master ref + phase-matched fan-out (ref + SYSREF + ramp strobe) + per-board jitter-cleaning PLL → 125 MHz; skew budget; loop-BW-as-register bench tuning (`clock_distribution_architecture.md`) | Medium |
| Calibration procedure | Composite-channel (RF ⊕ digitizer ⊕ harness) cal: bench H_dig once → OTA known-target cal at join → opportunistic refinement. No on-board cal couplers (§2.4) | Medium |
| Würth quote | Obtain RF Board Rogers 4003C 4-layer PCB and assembly quote | Medium |
| JLCPCB quote | Obtain Digitizer Board 6–8 layer FR4 fab + BGA assembly quote (Advanced) | Medium |
| KiCAD schematic capture | Begin RF Board and Digitizer Board schematics; all footprints known | Low |
| TX antenna placement | Optimise TX positions for FDMA compatibility; simulate range-azimuth coupling | Low |
| Stage 4 aggregation | Evaluate multi-NIC vs SmartNIC vs central FPGA for 32-board aggregation | Low |
| ADC3669 relevance | Revisit ADC3669 (500 MSPS) if Stage 3 FDMA plan exceeds 125 MHz beat bandwidth | Low |
| Mixer upgrade pricing | Obtain Marki MM1-0115HPSM-2 price from RFMW for future Doppler-optimised revision | Low |
