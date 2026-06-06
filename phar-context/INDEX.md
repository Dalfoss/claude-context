# Radar Project — Documentation Index

## How to Use This Index
Load this file first in any new conversation. It maps the full project.
For any given question, identify the relevant section(s) below and load
those files into context. You do not need to load all files at once.

---

## Project Summary
Bistatic phased array FMCW MIMO radar at X-band 10.0–10.5 GHz, for small-UAV
detection (~1.5 km is the v1 prototype goal, not the end goal — range scales
with channel count, see the scaling path). Settled architecture:
**analogue dechirp at the antenna →
commodity ADC per channel → Lattice ECP5 edge FPGA (Hilbert + decimation +
SerDes) → fibre → GPU/CPU host** (CUDA range/Doppler/beamforming/CFAR).
Self-contained "Board A" (8 RX) tiles 1→32 boards. No ZCU208/RFSoC, no analogue
channel multiplexing. See `rf-frontend/radar-rx-frontend-edge-digitization.md`.
Status: Design phase. No hardware purchased yet.

> The earlier ZCU208-centred design (direct-RF / downconvert-to-IF, RFSoC
> digitiser, S-band vs X-band band decision) is **superseded**. The files
> marked *(legacy)* below are retained for historical rationale only.

---

## File Map

### platform/ — *(legacy: ZCU208 era, superseded)*
| File | Contents | Load when... |
|---|---|---|
| `zcu208.md` *(legacy)* | ZCU208 specs, add-on cards, ADC/DAC details, analog input bandwidth, Nyquist zone behaviour, PYNQ | Reviewing the abandoned RFSoC platform decision — **not the current platform** |
| `software.md` *(legacy)* | PYNQ, Vivado licensing, toolchain, host interface (ECP5 open-source flow + GPU host now replace this; GPU/cuFFT section still relevant) | Reviewing the old ZCU208 toolchain |
| `vivado-licencing.md` *(legacy)* | Vivado license requirements for ZCU208 — moot under the ECP5 open-source flow | Historical only |

### rf-frontend/
| File | Contents | Load when... |
|---|---|---|
| `architecture.md` *(legacy)* | Old ZCU208 frontend: S-band-direct vs X-band-to-IF decision, LO distribution, phase coherence — **superseded by `radar-rx-frontend-edge-digitization.md`** | Reviewing the original frontend decision trail |
| `components.md` *(legacy)* | Old BOM for ZCU208 Phase 1/Phase 2 — **superseded by the BOM in `radar-rx-frontend-edge-digitization.md` §11** | Reviewing the original cost analysis |
| `sampling-theory.md` | Nyquist zones, analog input bandwidth, IQ vs real sampling — critical distinctions | Discussing sampling, bandwidth, frequency planning |
| `fmcw-amplifier.md` | FMCW radar amplifier design at 5.8 GHz and 10.5 GHz — covering critical PA/LNA performance metrics, recommended ICs and PCB modules, and 8TX/8RX channel cost breakdowns for both discrete and modular approaches | Discussing PA/LNA selection at 5.8/10.5 GHz, amplifier ICs vs modules, transmit/receive chain cost |
| `radar-rx-frontend-edge-digitization.md` | **Settled** RF front-end architecture for 10.0–10.5 GHz FMCW MIMO: analogue dechirp + digitise-at-antenna + fibre transport. Self-contained Board A (8 RX patch array, RF chain, 4× LTC2145-14 ADCs in Stage 1 → TI ADC3668 in Stage 2, Lattice ECP5 FPGA mezzanine doing Hilbert/decimation/SerDes), TX board, and GPU/CPU central unit. Documents the decision trail rejecting SCM and WDM, ADC phase coherence, waveform strategy (FDMA/CDM), and the 1→32 board scaling path | Discussing the RX front-end, dechirp-at-antenna, ADC selection/staging, edge FPGA processing, fibre transport, phase coherence, waveform multiplexing, channel scaling |

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

1. **Band is settled: X-band 10.0–10.5 GHz** (amateur allocation). The earlier
   open choice between S-band direct sampling and X-band is closed — there is no
   S-band path. See `regulatory/spectrum.md`.

2. **No ZCU208 / no central RFSoC** — Digitisation is distributed to the edge:
   a commodity ADC per channel on each Board A (LTC2145-14 → ADC3668). Because
   each ADC only ever sees the dechirped DC–~100 MHz beat signal, the
   "14-bit-at-5-GSPS only achievable monolithically" argument that drove the
   RFSoC choice no longer applies. The `platform/*` files are legacy.
   See `rf-frontend/radar-rx-frontend-edge-digitization.md`.

3. **IQ is synthesised digitally in the edge FPGA** — the dechirp mixer is a
   real (non-IQ) mixer; a ~64-tap Hilbert FIR in the ECP5 generates complex IQ
   from real ADC samples, with >60 dB image rejection (better than hardware IQ
   balance). See `rf-frontend/radar-rx-frontend-edge-digitization.md` §4.4.

4. **Phase coherence: shared OCXO + startup tone calibration**, not RFSoC MTS —
   one OCXO master clock is distributed to every Board A (fixing frequency,
   leaving a per-board power-up phase offset), and a startup tone injected by
   the host lets each board measure and store fixed per-channel phase
   corrections. See `rf-frontend/radar-rx-frontend-edge-digitization.md` §2.4.

5. **Bandwidth reduction lives at the edge; heavy DSP on a GPU/CPU host** — the
   ECP5 does Hilbert + decimation (the data-rate reduction step) before fibre;
   range/Doppler/beamforming/CFAR run on the host. No central FPGA: at 32 boards
   raw data is ~448 Gbps, untenable to centralise before reduction.
   See `rf-frontend/radar-rx-frontend-edge-digitization.md` §2.3, §7.

6. **Dechirp happens in the analog domain, at the antenna** — Sampling the
   raw carrier wastes ENOB to aperture jitter. The settled architecture
   dechirps each channel with a passive mixer driven by the reference chirp
   so the ADC only ever sees the low-frequency beat signal, then digitises
   immediately at the antenna board. This is the single highest-impact design
   choice. See `rf-frontend/radar-rx-frontend-edge-digitization.md`.

7. **No analog channel multiplexing — digitise at the antenna instead** —
   Analog multiplexing schemes (SCM, WDM/RF-over-fibre) were evaluated and
   rejected: SCM intermodulation floors weak channels and scales as N³, while
   WDM does not reduce ADC count. The settled approach puts an ADC per channel
   on each Board A and transports digital samples over fibre, scaling by
   tiling Board A instances (1→32 boards). See
   `rf-frontend/radar-rx-frontend-edge-digitization.md`.
