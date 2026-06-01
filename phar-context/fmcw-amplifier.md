# FMCW Radar Amplifier Design: Performance Metrics, Component Recommendations, and Cost Summary

## 1. Critical PA Performance Metrics for FMCW Radar

### Phase Noise (Most Important)
- FMCW measures range via beat frequency between transmitted chirp and echo. Any phase noise on the PA output corrupts this directly.
- PA contributes additive phase noise and 1/f upconversion. Both raise the noise floor in the range profile.
- Mitigated partially by the **range correlation effect**: in monostatic FMCW the same chirp is used for TX and mixing reference, so phase noise partially cancels — most effective at short range.
- At W-band and above (and in multistatic systems), phase noise becomes severely limiting.

### Linearity (AM-AM, AM-PM)
- FMCW chirp is nominally constant-envelope, but wideband chirps have residual amplitude ripple.
- AM-AM compression and AM-PM distortion in the PA generate additional phase perturbations indistinguishable from chirp nonlinearity, raising sidelobe levels in the range profile.
- For high-resolution FMCW (wide chirp bandwidth, e.g. 10 GHz sweep at 90 GHz), gain flatness and group delay flatness across the full chirp bandwidth are critical.
- Spectral regrowth from a nonlinear PA is also a regulatory concern in dense radar environments (e.g. automotive 77–81 GHz).

### Gain Flatness Across Chirp Bandwidth
- Range resolution = c / (2 × BW). A 5% bandwidth at 5.8 GHz = ~290 MHz; at 10.5 GHz = ~525 MHz.
- PA gain must be flat (< ~0.5 dB variation) across this bandwidth to avoid range sidelobe degradation.
- Becomes harder at millimeter-wave due to reduced MAG/MSG and parasitic capacitances.

### Power-Added Efficiency (PAE)
- FMCW is CW operation — thermal dissipation is continuous, unlike pulsed radar.
- Low PAE = high heat = performance degradation and reliability risk.
- GaN-on-SiC achieves 50–57% PAE at C/X-band; CMOS/GaAs are typically 10–30%.

### Operating Point / Backoff
- For FMCW, PA should be operated 5–6 dB below Psat (class-AB) to minimise AM-PM and added phase noise.
- This reduces effective output power significantly — factor in when selecting Psat.

### Output P1dB and OIP3
- High OIP3 needed for spectral purity and suppression of intermodulation.
- P1dB should be well above actual operating power to stay in linear region.

---

## 2. Critical LNA Performance Metrics for FMCW Radar

### Noise Figure (Most Important)
- LNA noise figure dominates system noise figure (Friis formula — first stage is critical).
- Even 1 dB insertion loss before the LNA raises system NF by ~1 dB.
- Target: < 1.5 dB NF at 5.8 GHz; < 2.5 dB NF at 10.5 GHz.
- LNA must be the first element after the antenna/switch.

### Gain
- High LNA gain suppresses noise contributions from downstream stages.
- Typical target: > 15–20 dB.
- Tradeoff: too much gain drives subsequent stages into compression.

### OIP3 / Dynamic Range
- In FMCW receivers, strong nearby reflections can coexist with weak distant targets.
- High OIP3 (> 25–30 dBm) needed to avoid receiver desensitisation.
- HMC8410 achieves OIP3 ~33 dBm — good for radar.

### Input Ruggedness / Protection
- In monostatic or near-monostatic FMCW, TX leakage into RX is a concern.
- GaN LNAs (e.g. within QPM2637) are inherently more robust than GaAs and may not need a separate limiter.
- GaAs LNAs (HMC8410/8412) may benefit from a PIN limiter diode ahead of them in high-leakage scenarios.

---

## 3. Pulsed Radar vs FMCW — Key Differences (for context)
- Pulsed radar: peak power and pulse fidelity (droop, overshoot) dominate. PA can be saturated.
- FMCW: phase noise and linearity dominate. PA must be backed off.
- Pulse droop in GaN: ~-0.012 dB/°C thermal effect; 70°C rise → 0.84 dB droop per stage. Not relevant for CW FMCW.

---

## 4. Technology Summary
| Technology | Power Density | Freq. Range | PAE | NF | Best For |
|---|---|---|---|---|---|
| GaN-on-SiC | 5× GaAs | DC–100 GHz+ | 50–85% | ~1.5–2 dB | TX PA, robust LNA |
| GaAs pHEMT | Moderate | DC–40 GHz | 15–35% | 0.5–1.5 dB | LNA, driver stages |
| SiGe BiCMOS | Low | DC–100 GHz | 10–25% | 1.5–3 dB | Integrated transceivers |
| CMOS | Very Low | DC–100 GHz | 5–15% | 2–5 dB | Consumer mmWave radar |
| LDMOS | High (low freq.) | DC–6 GHz | 40–65% | N/A | L/S-band high-power |

---

## 5. Component Recommendations

### Target Bands
- **Band 1:** 5.51–6.09 GHz (5.8 GHz centre, ~5% BW)
- **Band 2:** 9.98–11.03 GHz (10.5 GHz centre, ~5% BW)

### 5.8 GHz — Power Amplifiers
| Part | Type | Freq | Psat | PAE | Notes |
|---|---|---|---|---|---|
| **Qorvo QPA2309** | GaN-on-SiC MMIC | 5–6 GHz | 100 W (50 dBm) | 52% | Best-in-class; 7×7mm QFN; quote-only; ~$500–800/unit est. |
| **MACOM WSA4501S** | GaN-on-SiC MMIC | 5.2–5.9 GHz | 50 W | 57% | Radar-array optimised; check upper freq edge at 5.9 GHz; quote-only |
| **Qorvo TQP7M9102** | GaAs HBT | 0.4–4 GHz (usable to ~5 GHz) | ~0.5 W | — | Driver/low-power only; OIP3 +44 dBm; ~$5–8/unit |

### 5.8 GHz — Low Noise Amplifiers
| Part | Type | Freq | NF | Gain | OIP3 | Notes |
|---|---|---|---|---|---|---|
| **ADI HMC8410** | GaAs pHEMT | 0.01–10 GHz | 1.1 dB | 19.5 dB | 33 dBm | Top pick; no external matching; ~$17–22/unit est. |
| **ADI HMC8412** | GaAs pHEMT | 0.4–11 GHz | 1.4 dB | 15.5 dB | 33 dBm | Integrated bias choke; easier design-in; ~$17–22/unit est. |
| **Skyworks SKY65981-11** | GaAs pHEMT | 5.15–5.85 GHz | ~1.0 dB | ~14 dB | — | Purpose-built for 5.8 GHz ISM; check upper edge at 6.09 GHz; ~$5–10/unit est. |
| **Qorvo QPL9547** | GaAs E-pHEMT | 0.1–6 GHz | 0.3 dB (@ 1.9 GHz) | 19.5 dB | +39 dBm | Ultra-low NF; rises toward 6 GHz; single supply 3.3–5V |

### 10.5 GHz — Power Amplifiers / Front-End Modules
| Part | Type | Freq | Psat | Gain | NF (RX) | Notes |
|---|---|---|---|---|---|---|
| **Qorvo QPM2637** | GaN FEM (PA+LNA+Switch) | 9–10.5 GHz | 4 W (36 dBm) | 23 dB TX | 2.7 dB | Best match to band; integrates PA+LNA+T/R switch; no limiter needed; **$130.81/unit** (1–24 qty, RFMW) |
| **ADI ADTR1107** | GaN FEM (PA+LNA+Switch) | 6–18 GHz | ~316 mW (25 dBm) | 22 dB TX | 2.5 dB | Covers both bands if needed; pairs with ADAR1000 beamformer; **$228.31/unit** (DigiKey) |

### 10.5 GHz — Standalone LNAs
| Part | Type | Freq | NF | Gain | Notes |
|---|---|---|---|---|---|
| **ADI HMC8410** | GaAs pHEMT | 0.01–10 GHz | 1.1 dB | 19.5 dB | Works at 10 GHz edge of band |
| **ADI HMC8412** | GaAs pHEMT | 0.4–11 GHz | 1.4 dB | 15.5 dB | Better for 10.5 GHz; covers to 11 GHz |
| **ADI HMC1049** | GaAs pHEMT | 0.3–20 GHz | ~1.5–2 dB at 10.5 GHz | ~20 dB | Broadest coverage; good if full X-band needed |

---

## 6. Evaluation / Modular Boards
| Board | Part | Freq | Price | Notes |
|---|---|---|---|---|
| **QPA2309EVB** | Qorvo | 5–6 GHz PA | **$3,000** (RFMW) | Rogers 4350B; TX only |
| **QPM2637EVB** | Qorvo | 9–10.5 GHz FEM | **$1,000** (RFMW) | Rogers 4350B; PA+LNA+switch |
| **ADTR1107-EVALZ** | ADI | 6–18 GHz FEM | **$1,029** (DigiKey) | Rogers 4350B; PA+LNA+switch; 2.9mm connectors |

Note: The original ADTR1107-EVAL (older version) is now listed as obsolete at DigiKey; use ADTR1107-EVALZ.

---

## 7. Cost Summary — 8TX + 8RX Channels

Architecture assumption: separate TX and RX antenna arrays (bistatic/quasi-bistatic FMCW). 8 PAs needed for TX, 8 LNAs for RX = 16 amplifier positions per band.

### IC-Only (Bare Components)
| Configuration | Band | TX (×8) | RX (×8) | Total ICs |
|---|---|---|---|---|
| QPA2309 + HMC8410 | 5.8 GHz | ~$4,000–6,400 | ~$136–176 | **~$4,200–6,600** |
| TQP7M9102 + HMC8410 (low power) | 5.8 GHz | ~$40–65 | ~$136 | **~$180–200** |
| QPM2637 (TX path) + HMC8412 (RX) | 10.5 GHz | ~$1,047 | ~$144 | **~$1,191** |
| QPM2637 × 16 (all channels) | 10.5 GHz | \$1,047 | \$1,047 | **~$2,094** |
| ADTR1107 × 16 (all channels) | 10.5 GHz | \$1,826 | \$1,826 | **~$3,652** |
| ADTR1107 (TX) + HMC8412 (RX) | 10.5 GHz | ~$1,826 | ~$144 | **~$1,970** |

### Evaluation/Modular Boards (×8 TX + ×8 RX boards)
| Configuration | Band | ×8 TX Boards | ×8 RX Boards | Total |
|---|---|---|---|---|
| QPA2309EVB + LNA boards | 5.8 GHz | $24,000 | ~$3,000–4,000 est. | **~$27,000–28,000** |
| QPM2637EVB × 16 | 10.5 GHz | $8,000 | $8,000 | **~$16,000** |
| ADTR1107-EVALZ × 16 | 10.5 GHz | $8,232 | $8,232 | **~$16,464** |

### Custom Rogers RO4350B PCB (Prototype Qty, 1–25 boards)
| Configuration | Band | PCB Manuf. (×16) | Components | Assembled Total |
|---|---|---|---|---|
| High-power PA + LNA boards | 5.8 GHz | $1,600–3,200 | $4,200–6,600 | **~$6,000–10,000** |
| QPM2637 / ADTR1107 FEM boards | 10.5 GHz | $1,600–3,200 | $1,200–2,100 | **~$3,000–5,500** |

Assembly (hand solder or PCB house): add ~$20–60/board at prototype quantities.

---

## 8. Design Notes and Recommendations

- **Substrate:** Rogers RO4350B (Dk 3.48, tan δ ~0.004) mandatory above ~3 GHz. FR4 is not acceptable at 5.8 GHz or 10.5 GHz.
- **PA backoff:** Run FMCW PAs at ≥5–6 dB below Psat for acceptable AM-PM and phase noise.
- **LNA placement:** LNA must be first active element after antenna. Pre-LNA losses directly add to system noise figure.
- **QPM2637 advantage:** Integrates PA + LNA + T/R switch in one part; no external limiter needed; exact upper frequency match to 10.5 GHz band.
- **ADTR1107 advantage:** Covers 6–18 GHz so spans both target bands; pairs with ADI ADAR1000 beamformer IC for phased array use.
- **Open-source reference:** GitHub project `NawfalMotii79/PLFM_RADAR` is a 10.5 GHz phased-array FMCW radar using ADTR1107 FEM chips and QPA2962 PA, with full schematics, PCB layouts, and firmware available.
- **Cost driver at 5.8 GHz:** The PA utterly dominates cost. QPA2309 at ~$500–800/unit × 8 = $4–6k just for PA chips. If power budget allows < 1W/channel, driver-class parts reduce IC cost to under $250 total.
- **Cost driver at 10.5 GHz:** QPM2637 at $130.81/unit is accessible and integrates PA+LNA. Full 16-channel IC cost under $1,200 using QPM2637 (TX) + HMC8412 (RX).
- **Eval boards are prototype tools, not deployment hardware.** Use them for system validation, then move to custom Rogers PCB for deployment.

---

*Document generated from conversation on FMCW radar power amplification theory, component selection, and costing. Covers 5.8 GHz (C-band) and 10.5 GHz (X-band) at 8TX + 8RX channel scale. Prices verified April 2026 from RFMW and DigiKey where available; defence-grade GaN PA prices are estimates.*
