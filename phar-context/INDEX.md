# Radar Project — Documentation Index

## How to Use This Index
Load this file first in any new conversation. It maps the full project.
For any given question, identify the relevant section(s) below and load
those files into context. You do not need to load all files at once.

---

## Project Summary
8-channel bistatic phased array radar (8 TX + 8 RX antennas).
Direct RF sampling or single-conversion architecture depending on target band.
FPGA/SDR platform: ZCU208. Host processing architecture: undecided (GPU/cuFFT
explored as one candidate — see signal-processing/pipeline.md).
Status: Design phase. No hardware purchased yet.

---

## File Map

### platform/
| File | Contents | Load when... |
|---|---|---|
| `zcu208.md` | ZCU208 specs, add-on cards, ADC/DAC details, analog input bandwidth, Nyquist zone behaviour, PYNQ | Discussing hardware platform, sampling, ADC limits |
| `software.md` | PYNQ, Vivado licensing, toolchain, host interface | Discussing software, bring-up, development tools |
| `vivado-licencing.md` | Vivado license requirements for ZCU208: pre-loaded bitstream, custom bitstream, PYNQ overlay, and production device paths | Discussing Vivado licensing costs and options |

### rf-frontend/
| File | Contents | Load when... |
|---|---|---|
| `architecture.md` | Block diagram, frequency plan options, LO strategy, direct sampling vs downconversion, phase coherence | Discussing RF chain design |
| `components.md` | BOM for Phase 1 and Phase 2, part numbers, suppliers, pricing | Discussing parts, procurement, cost |
| `sampling-theory.md` | Nyquist zones, analog input bandwidth, IQ vs real sampling — critical distinctions | Discussing sampling, bandwidth, frequency planning |
| `fmcw-amplifier.md` | FMCW radar amplifier design at 5.8 GHz and 10.5 GHz — covering critical PA/LNA performance metrics, recommended ICs and PCB modules, and 8TX/8RX channel cost breakdowns for both discrete and modular approaches | Discussing PA/LNA selection at 5.8/10.5 GHz, amplifier ICs vs modules, transmit/receive chain cost |
| `rf-signal-transport-and-multiplexing.md` | RF transport and channel multiplexing for 8TX × 64RX MIMO FMCW: traditional coax, analog dechirp variants (mixer at rack vs. at antenna), photonic SCM-over-fibre, and OFC channelisation. Includes ENOB/aperture-jitter analysis at 5.8 GHz and a staged cost roadmap from $10k → $355k | Discussing how RX signals reach the ADC, scaling beyond 8 channels, ENOB/aperture jitter, dechirp location, fibre transport, photonic channelisation, system cost staging |

### signal-processing/
| File | Contents | Load when... |
|---|---|---|
| `pipeline.md` | DDC/DUC, decimation, 3D FFT, cuFFT, GPU streaming | Discussing DSP pipeline, processing chain |
| `radar-theory.md` | Dynamic range, ADC resolution, coherent processing gain, ENOB, radar waveforms | Discussing radar performance, dynamic range |
| `fdm-processing.md` | Waveform orthogonality and range resolution recovery via Frequency Division Multiplexing in MIMO FMCW radar systems | Discussing MIMO waveform design, channel orthogonality, FDM, range resolution recovery |

### regulatory/
| File | Contents | Load when... |
|---|---|---|
| `spectrum.md` | Amateur 10 GHz band (10.0–10.5 GHz), licensing, power limits, other candidate bands | Discussing frequency selection, legal transmission |

### decisions/
| File | Contents | Load when... |
|---|---|---|
| `log.md` | Chronological record of all major design decisions and the reasoning behind them | Reviewing why something was chosen, avoiding re-litigating settled decisions |

---

## Critical Project-Wide Facts
These apply globally and should always be kept in mind:

1. **ZCU208 ADCs are real ADCs** — IQ output is synthesised digitally via the
   on-chip DDC after sampling. The hardware constraint is real-ADC Nyquist,
   not IQ bandwidth. Do not conflate these. See `rf-frontend/sampling-theory.md`.

2. **Analog input bandwidth ≠ Nyquist frequency** — ZU48DR ADC samples at
   5 GSPS (Nyquist = 2.5 GHz) but analog input bandwidth extends to 6 GHz.
   Both limits apply simultaneously. See `rf-frontend/sampling-theory.md`.

3. **14-bit resolution is a hard constraint** — No discrete ADC from any vendor
   achieves 14-bit at ≥8 GSPS. The ZCU208 achieves this only through monolithic
   integration. This drove the platform decision. See `decisions/log.md`.

4. **mmWave support on ZCU208 is external** — "Extended mmWave support" refers
   to the RFMC 2.0 daughter card connector, not native ADC capability.
   The ADC ceiling is 6 GHz regardless. See `platform/zcu208.md`.

5. **Target frequency band is undecided** — Two viable paths remain open:
   - Direct sampling at S-band (3.1–3.5 GHz), no mixing required
   - Single-conversion at X-band (10.0–10.5 GHz amateur band), fixed LO
   See `rf-frontend/architecture.md` and `regulatory/spectrum.md`.

6. **Aperture jitter caps direct-RF ENOB at ~7 bits at 5.8 GHz** — The 14-bit
   ADC delivers only ~45 dB dynamic range when sampling the raw carrier.
   Moving the dechirp into the analog domain (so the ADC sees the 0–5 MHz
   beat signal, not the 5.8 GHz carrier) recovers 4–5 ENOB bits and is the
   single highest-impact design step in the project. Location of the mixer
   (rack vs. antenna vs. photonic) does not affect the ENOB outcome — only
   deployment flexibility. See `rf-frontend/rf-signal-transport-and-multiplexing.md`.

7. **64 RX channels with 8 ADCs requires multiplexing** — Scaling to the
   full 8TX × 64RX MIMO aperture cannot be done with one coax per channel.
   Photonic SCM-over-fibre (Stage 2) or OFC channelisation (Stage 3) are
   the two paths under evaluation. See
   `rf-frontend/rf-signal-transport-and-multiplexing.md`.
