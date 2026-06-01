# RF Signal Transport and Multiplexing for FMCW MIMO Radar

**Subject:** A technical comparison of traditional RF coaxial transport and photonic channelisation via Optical Frequency Comb (OFC) for multi-channel FMCW MIMO radar receivers, including the case for photonic dechirping and its ENOB implications.

**Platform context:** 8TX × 64RX MIMO FMCW radar at 5.8 GHz, processed on Xilinx ZCU208 (ZU48DR), targeting drone detection at 500m+ with swarm discrimination capability.

---

## Index

### Document Sections
- [1. The Transport Problem in MIMO Radar](#1-the-transport-problem-in-mimo-radar)
- [2. Traditional RF Transport](#2-traditional-rf-transport)
- [3. Intermediate Architecture — Analog Dechirp with Coaxial Transport](#3-intermediate-architecture--analog-dechirp-with-coaxial-transport)
- [4. Photonic Channelisation via Optical Frequency Comb](#4-photonic-channelisation-via-optical-frequency-comb)
- [5. ENOB Implications: Analog vs Digital Dechirping](#5-enob-implications-analog-vs-digital-dechirping)
- [6. Photonic Dechirping](#6-photonic-dechirping)
- [7. Dynamic Range in the Photonic Link](#7-dynamic-range-in-the-photonic-link)
- [8. Lower-Cost Alternative: Analog Dechirp + SCM over Fibre](#8-lower-cost-alternative-analog-dechirp--scm-over-fibre-stage-2)
- [9. Coaxial SCM — All Benefits of Stage 2 Without Photonics](#9-coaxial-scm--all-benefits-of-stage-2-without-photonics-stage-175)
- [10. Architecture Comparison Summary](#10-architecture-comparison-summary)
- [11. Staged Development Roadmap](#11-staged-development-roadmap)
- [12. Key References and Technology Suppliers](#12-key-references-and-technology-suppliers)

### Development Stages at a Glance

| Stage | Name | RX | ENOB | Key limitation | Section |
|---|---|---|---|---|---|
| 1 | Direct RF sampling, digital dechirp | 8 | ~7 bits | ADC jitter at 5.8 GHz limits dynamic range to ~44 dB | [§2](#2-traditional-rf-transport) |
| 1.5A | Analog dechirp, mixer at ZCU208 rack | 8 | ~11–12 bits | Antennas must be within ~10 m of ZCU208 | [§3](#3-intermediate-architecture--analog-dechirp-with-coaxial-transport) |
| 1.5B | Analog dechirp, mixer at antenna, coax return | 8 | ~11–12 bits | Antennas limited to ~50 m from ZCU208; one ADC per antenna | [§3](#3-intermediate-architecture--analog-dechirp-with-coaxial-transport) |
| 1.75 | Analog dechirp + coaxial SCM multiplexing | 64 | ~11–12 bits | Antennas limited to ~50 m; susceptible to EM interference on cables | [§9](#9-coaxial-scm--all-benefits-of-stage-2-without-photonics-stage-175) |
| 2 | Analog dechirp + SCM + Radio over Fibre | 64 | ~11–12 bits | Requires photonic components at antenna and central unit | [§8](#8-lower-cost-alternative-analog-dechirp--scm-over-fibre-stage-2) |
| 3 | OFC photonic channelisation | 64 | ~11–12 bits | High cost; EOM required at every antenna | [§4](#4-photonic-channelisation-via-optical-frequency-comb) |
| 3+ | OFC + TFLN integrated modulators | 64+ | ~11–12 bits | Dependent on TFLN commercial availability | [§4](#4-photonic-channelisation-via-optical-frequency-comb) |

---

## 1. The Transport Problem in MIMO Radar

In a MIMO FMCW radar, the received signal from each antenna element must reach the ADC coherently — meaning phase relationships between channels must be preserved. For a physically distributed antenna array (antennas separated from the processing unit), this creates a transport challenge that scales directly with channel count.

For an 8TX × 64RX system, 64 receive channels need to reach the ZCU208. The ZCU208 provides 8 ADC channels natively. Bridging the gap between 64 physical channels and 8 ADC channels is the central problem this document addresses.

Two architectures are compared:

- **Traditional with digital dechirp:** One coaxial cable per channel, ADC samples raw RF, dechirp in FPGA. Simple but ENOB-limited.
- **Traditional with analog dechirp:** Dechirp mixer moves to the antenna or ZCU208 rack; ADC samples beat frequency, recovering full dynamic range. No photonics required.
- **Coaxial SCM multiplexing:** Beat signals from 8 antennas are frequency-offset and combined onto one coax return cable per group. Scales to 64 channels with 8 ADCs. No photonics, no fibre. Distance-limited to ~50 m.
- **Photonic SCM over fibre:** Identical to coaxial SCM but with fibre replacing coax. Unlimited separation, full EM immunity.
- **Photonic OFC channelisation:** Full photonic link with no RF mixing at antenna. Maximum capability and elegance at the highest cost.

---

## 2. Traditional RF Transport

### Architecture

Each RX antenna is connected to the ZCU208 via a direct RF chain:

```
Antenna → LNA → [downconverter or direct] → coaxial cable → ADC (ZCU208)
```

For direct RF sampling at 5.8 GHz, the ZCU208 ADCs can sample directly in the third Nyquist zone (ADC sample rate: 5 GSPS, third Nyquist: 5.0–7.5 GHz). No external downconversion is strictly required.

The dechirp is performed digitally: the ADC captures the raw 5.8 GHz chirped return, and the ZCU208 FPGA performs a complex conjugate multiplication against a numerically generated reference chirp to produce the beat signal.

### Hardware (per channel)

| Component | Purpose | Unit cost (approx.) |
|---|---|---|
| LNA (X-band, ~5.8 GHz) | Low-noise amplification | $200–$500 |
| LMR-400 coaxial cable (100 m run) | RF transport | $8–$15/m → $800–$1,500 |
| SMA connectors + adapters | Interface | $20–$80 |
| Anti-alias / bandpass filter | Spectral cleanup | $100–$400 |

**Per channel total: ~$1,200–$2,500**

### ENOB at 5.8 GHz — The Core Limitation

This is the critical constraint of direct RF sampling. ADC effective resolution degrades with input frequency due to aperture jitter. The ZU48DR has an aperture jitter of approximately 100–150 fs RMS. The jitter-limited SNR is:

```
SNR_jitter = −20 · log₁₀(2π · f_in · t_j)
```

At 5.8 GHz with 150 fs jitter:

```
SNR_jitter = −20 · log₁₀(2π × 5.8×10⁹ × 150×10⁻¹⁵) ≈ 45 dB
```

Converting to ENOB:

```
ENOB = (SNR − 1.76) / 6.02 ≈ 7.2 bits
```

The 14-bit ADC delivers approximately **7 effective bits** at 5.8 GHz. This is a fundamental physical limit of the silicon, not a configuration issue. The 14-bit dynamic range of 86 dB collapses to ~45 dB at the operating frequency.

| Parameter | Theoretical | At 5.8 GHz (direct RF) |
|---|---|---|
| ADC bit depth | 14 bits | — |
| ENOB | 14 bits | ~7 bits |
| Dynamic range | 86 dB | ~45 dB |
| Clutter rejection headroom | Excellent | Marginal |

For a counter-drone radar operating in any non-trivial clutter environment, 45 dB of instantaneous dynamic range is insufficient. Strong ground return or infrastructure clutter can easily occupy 30–40 dB of this budget, leaving 5–15 dB of margin for weak target detection.

### Scaling to 64 Channels

The ZCU208 provides 8 ADC channels. To reach 64 channels with direct RF transport requires 7 additional ADC boards or RF data acquisition systems — each of comparable cost to the ZCU208 itself (~$15,000 each). The cable plant alone (64 × 100m of LMR-400) approaches $80,000–$100,000. Total cost for a 64-channel traditional system easily exceeds $150,000 before RF front-end costs, and introduces severe coherence challenges across physically separate ADC systems.

---

## 3. Intermediate Architecture — Analog Dechirp with Coaxial Transport

The single largest performance gain in this entire document — recovering 4–5 ENOB bits — does not require photonics at all. It requires only moving the dechirp operation from the ZCU208 FPGA into the analog domain before the ADC. This section evaluates how best to do this using only coaxial cable and the ZCU208's own capabilities, with no fibre optics involved.

### The Core Insight

The ADC's ENOB limitation at 5.8 GHz is caused by aperture jitter acting on a rapidly-changing input signal. The fix is to ensure the ADC only ever sees a slowly-changing signal — specifically, the beat frequency output of the dechirp mixer, which at 500 m maximum range is at most 5 MHz. At 5 MHz, the same 150 fs jitter produces an SNR of ~130 dB — far above the thermal noise floor. The ADC recovers its full dynamic range.

The question is not *whether* to dechirp before the ADC. The question is *where* to dechirp, and how to distribute the reference chirp needed to do it.

### Architecture Options Evaluated

Three placement options for the dechirp mixer were considered:

**Option A — Mixer at the ZCU208 rack (received signal transported at RF)**

The received 5.8 GHz signal travels from each antenna to the ZCU208 via coaxial cable. At the ZCU208 rack, a bank of 8 RF mixers — driven by the ZCU208 DAC — performs the dechirp. The beat signals then enter the ADC inputs directly.

Strengths: No RF hardware at the antenna beyond the LNA. The reference chirp never needs to leave the ZCU208 — the DAC output connects to the mixer bank over a few centimetres of cable. Simple, cheap, no distribution problem.

Weakness: The received signal must travel at 5.8 GHz over coaxial cable. LMR-400 attenuation at 5.8 GHz is approximately 10 dB per 100 m (1 dB per 10 m). Beyond approximately 5–10 m, cable loss begins to degrade the system noise figure meaningfully. This option is therefore only viable for compact arrays where antennas are co-located with the ZCU208.

**Option B — Mixer at the antenna (beat signal transported, reference distributed)**

The dechirp mixer is placed at each antenna. The received signal never travels over coaxial cable at 5.8 GHz — instead, the beat signal (0–5 MHz) is transported back to the ZCU208. At these frequencies, LMR-400 loss is approximately 0.5 dB per 100 m — effectively zero for any practical run length. Even thin cable such as RG-174 (~3 dB/100 m at 10 MHz) is entirely adequate.

The reference chirp must be distributed from the ZCU208 DAC to each antenna. This is the key challenge of this option and is addressed in detail below.

Strengths: Beat signal transport is trivial — any cable works at any length. Sets up naturally for the fibre stages (Stage 2 and 3 simply replace the reference coax and beat return coax with fibre, with no change to the antenna hardware). Antennas can be physically separated from the ZCU208.

Weakness: Reference chirp distribution at 5.8 GHz over coaxial cable has practical distance limits. Beyond approximately 30–50 m, amplification of the reference at each antenna is required.

**Option C — IF downconversion at the antenna, digital dechirp at ZCU208**

An LO signal is distributed to each antenna, downconverting the received signal to an intermediate frequency (e.g. 500 MHz). The IF signal travels to the ZCU208 over coax. Digital dechirp is performed after sampling.

This option was rejected as the primary recommendation because it provides only partial ENOB recovery. At 500 MHz, ENOB is approximately 9–10 bits — better than 7 bits at 5.8 GHz, but significantly worse than the 11–12 bits achievable with a full analog dechirp. It also introduces a separate LO distribution problem without eliminating the digital dechirp. The complexity is not justified given that full ENOB recovery is achievable with Options A or B.

### Recommended Architecture

**For compact arrays (antennas within 5–10 m of ZCU208): Option A**

```
ZCU208 DAC → TX chirp → [power divider]
                               ├──→ TX chain
                               └──→ Mixer bank (8× RF mixer, at ZCU208 rack)
                                           ↑ LO port
Antenna → LNA → coax (5.8 GHz, ≤10 m) → mixer RF port
                                           ↓ IF port
                                   beat signal → ZCU208 ADC
```

Hardware at each antenna: LNA only. All active processing at the ZCU208 rack. This is the absolute minimum hardware configuration and the natural starting point for a lab prototype.

**For distributed arrays (antennas 10–100 m from ZCU208): Option B**

```
ZCU208 DAC → TX chirp → [power divider]
                               ├──→ TX chain
                               └──→ [1:8 RF splitter] → reference coax to each antenna

At each antenna:
  Reference coax (5.8 GHz) → [RF amp if needed] → mixer LO port
  Antenna → LNA → mixer RF port
  Mixer IF port → LPF → beat signal → thin coax (0–5 MHz) → ZCU208 ADC
```

This is the recommended architecture for any system where physical separation is needed, and it is the direct precursor to Stage 2. The only change going from Stage 1.5 (Option B) to Stage 2 is replacing the reference coax with an optical fibre link and the beat return coax with an optical SCM return link.

### Reference Chirp Distribution Over Coaxial Cable

This is the critical feasibility question for Option B. The ZCU208 DAC generates the TX chirp at 5.8 GHz. A copy of this signal must reach every antenna element.

**Coaxial cable loss at 5.8 GHz:**

| Cable type | Loss at 5.8 GHz | Usable range (≤6 dB loss) |
|---|---|---|
| LMR-100A | ~33 dB/100 m | ~18 m |
| LMR-195 | ~16 dB/100 m | ~37 m |
| LMR-400 | ~10 dB/100 m | ~60 m |
| LMR-600 | ~6.5 dB/100 m | ~92 m |
| 7/8" Heliax | ~3.5 dB/100 m | ~170 m |

For reference distribution, 6 dB of cable loss is easily compensated with a small MMIC amplifier at the antenna (e.g. Mini-Circuits GALI-74+ gain ~17 dB, NF ~3 dB, cost ~$5). This sets a practical usable range of approximately 60 m on LMR-400, or ~100 m if a more substantial run is needed using Heliax or lower-loss cable.

An important consideration: the reference chirp for dechirping does not need to be a high-power signal. The mixer LO port typically requires only +7 dBm. A DAC output of +10 dBm at the ZCU208, through a 1:8 splitter (−9 dB) and 10 m of LMR-400 (−1 dB), arrives at each antenna at 0 dBm — adequate for mixer drive with a small booster amp.

**For runs beyond ~50 m without fiber:** Distribute the reference at a lower intermediate frequency from the ZCU208 DAC, and upconvert at each antenna using a fixed local oscillator derived from a PLL locked to a common low-frequency reference distributed over the same coax. This adds a PLL per antenna group (~$50–$200 each) but enables essentially unlimited coax run length. Beyond this complexity threshold, transitioning to fibre (Stage 2) is almost always the better engineering decision.

### ZCU208 Features That Enable This Architecture

The ZCU208 is unusually well-suited to this architecture — more so than a generic software-defined radio platform.

**DAC as reference chirp generator:** The ZCU208's 8 DACs at 10 GSPS can synthesize the TX chirp at 5.8 GHz directly in the third Nyquist zone, using the RF mixed mode available in the DUC (Digital Up Converter) chain. No external synthesizer or VCO is needed for the reference. The chirp is generated numerically, controlled entirely in software, and is phase-coherent with all ADC channels by construction. A single DAC can drive an 8-way power splitter to supply the reference to all 8 mixer LO ports simultaneously.

**Multi-tile synchronization (MTS):** The ZU48DR supports deterministic latency across all ADC and DAC tiles through MTS. When MTS is enabled, all ADC channels sample with a known, fixed phase relationship to each other — critical for MIMO coherent processing. Without MTS, phase offsets between ADC tiles are random at startup, making coherent aperture synthesis impossible. With MTS, the phase relationship is deterministic and calibratable. This is a hardware feature of the ZU48DR that is not available on lower-end RFSoC devices.

**DDC (Digital Down Converter) hard blocks:** After the ADC samples the beat signal (0–5 MHz), the on-chip DDC handles decimation and filtering in dedicated hardware. At 5 GSPS sampling a 5 MHz signal, a decimation ratio of 1000:1 is appropriate. The DDC hard blocks perform this efficiently without consuming FPGA fabric resources, producing a complex IQ output at a manageable sample rate (~10 kSPS per channel) for downstream range/Doppler FFT processing.

**NCO-based IQ demodulation:** The DDC includes a numerically controlled oscillator (NCO) that can mix the beat signal to complex baseband, producing the in-phase (I) and quadrature (Q) components needed for Doppler processing. This eliminates the need for analog IQ demodulation hardware.

**RFDC IP block:** Xilinx provides the RF Data Converter IP as a unified software interface to all ADC/DAC configuration, DUC/DDC chains, NCOs, and MTS. This significantly reduces the FPGA development effort compared to managing these blocks individually.

**CLK104 ultra-low-jitter clock module:** The ZCU208 includes the CLK104 add-on card, which provides a low-phase-noise sample clock to all converters using an LMX2594 synthesizer driven by an external 10 MHz reference. This is the clock source responsible for the ADC aperture jitter figure quoted throughout this document. Running the CLK104 from a high-quality OCXO (oven-controlled crystal oscillator) reference can further reduce jitter, though the improvement beyond 100 fs is marginal for this application.

### Hardware Bill of Materials

#### Option A — Compact (mixer at rack)

| Component | Quantity | Unit cost | Total |
|---|---|---|---|
| LNA at antenna (X-band) | 8 | $150–$400 | $1,200–$3,200 |
| Coaxial cable, LMR-195, 5 m/run | 8 | $30–$60 | $240–$480 |
| RF mixer bank (8× MMIC, e.g. Mini-Circuits ZX05-43MH+) | 8 | $25–$80 | $200–$640 |
| 1:8 RF power divider (reference distribution, very short cable) | 1 | $50–$150 | $50–$150 |
| Low-pass filters (beat signal, 5–10 MHz) | 8 | $15–$40 | $120–$320 |
| SMA connectors, adapters | — | — | $50–$150 |
| **Total (Option A, 8 channels)** | | | **~$1,860–$4,940** |

#### Option B — Distributed (mixer at antenna)

| Component | Quantity | Unit cost | Total |
|---|---|---|---|
| LNA at antenna (X-band) | 8 | $150–$400 | $1,200–$3,200 |
| RF mixer at antenna (dechirp, 5.8 GHz) | 8 | $25–$80 | $200–$640 |
| Low-pass filter at antenna (beat signal) | 8 | $15–$40 | $120–$320 |
| MMIC amplifier (reference recovery, if >20 m run) | 8 | $10–$30 | $80–$240 |
| Coaxial cable, LMR-400, reference distribution, 30 m/run | 8 | $40–$80 | $320–$640 |
| Coaxial cable, RG-174, beat return, 30 m/run | 8 | $15–$30 | $120–$240 |
| 1:8 RF splitter (reference, at ZCU208 end) | 1 | $50–$150 | $50–$150 |
| SMA connectors, weatherproof enclosures | — | — | $100–$300 |
| **Total (Option B, 8 channels)** | | | **~$2,190–$5,730** |

Both options are dramatically cheaper than any photonic stage, and in both cases the ZCU208 ADC is now sampling beat signals rather than raw RF, recovering the full ~11–12 ENOB performance.

### Limitations of This Stage

**Channel count is bounded by ADC count.** This stage provides 8 receive channels — one per ZCU208 ADC. Scaling to 64 channels requires either 7 additional ZCU208 boards (impractical and expensive) or a multiplexing scheme. This is the fundamental motivation for the SCM and OFC stages that follow.

**Coaxial reference distribution limits physical separation.** Option B works cleanly up to ~30–50 m with LMR-400. Beyond this, amplification is required and noise figure is progressively degraded. Option A is limited to ~5–10 m. For deployments where the array must be physically remote from the processing unit, this stage is insufficient and transition to fibre (Stage 2) is necessary.

**Natural transition path:** Option B is designed so that moving to Stage 2 requires no changes to the antenna hardware. The dechirp mixer, LPF, and LNA stay in place. The reference coax is replaced with a fibre-based optical reference link, and the beat return coax is replaced with an SCM optical return. The antenna simply gains a wideband PD for reference recovery.

---

## 4. Photonic Channelisation via Optical Frequency Comb

### Concept

An Optical Frequency Comb (OFC) is a light source producing a spectrum of discrete, equally-spaced frequency lines — effectively, N simultaneous optical carriers from a single source. In the proposed architecture, each comb tooth (wavelength) is assigned to one antenna element. The received RF signal at each antenna directly modulates its assigned wavelength via a small electro-optic modulator (EOM). All modulated wavelengths are then combined onto a single return fibre.

At the central processing unit, a second reference OFC — with a tooth spacing slightly different from the signal OFC — is combined with the returning signal on a single photodetector. The optical beating between corresponding teeth of the two combs produces electrical signals at different frequencies for each channel, without any RF mixing at the antenna. This is the core insight: **the channel frequency separation is generated at the receiver via optical heterodyning, not at the antenna.**

The dechirped (beat-frequency) signals for all 64 RX channels then sit in distinct frequency slots within the ADC's bandwidth, separated by digital bandpass filtering in the ZCU208.

### System Architecture (8 antennas per fibre group, 8 groups total)

```
CENTRAL UNIT
─────────────────────────────────────────────────────────────────────
  [CW Pump Laser]
       │
  [EO-OFC Generator]  ← driven by stable microwave reference
       │ (8 wavelengths: λ₁...λ₈)
  [EDFA optical amplifier]
       │
  [8-ch DWDM Demux] → λ₁ ─┐
                      → λ₂ ─┤
                      ...    ├──→ (8 individual fibres out to antenna array)
                      → λ₈ ─┘

  [Reference OFC]  ← slightly offset tooth spacing Δf_r ≠ Δf_s
       │
  [Balanced Photodetector + 90° optical hybrid]
       ↑
  [8-ch DWDM Mux] ← (1 return fibre carrying all 8 modulated wavelengths)
       │
  [ADC channel N on ZCU208]

─────────────────────────────────────────────────────────────────────
ANTENNA UNIT (×8 per group, ×8 groups = 64 total elements)
─────────────────────────────────────────────────────────────────────
  [λᵢ fibre in]
        │
  [VGA / step attenuator]    ← dynamic range protection
        │
  [EOM / MZM]  ← RF drive from LNA
        ↑
  [LNA]
        ↑
  [Antenna element]
        │
  [λᵢ modulated fibre out]
```

**Total fibres per group:** 9 (8 outbound individual, 1 combined return) — or 2 if WDM distribution is used on the outbound side as well (1 out, 1 back).

### Channel Separation at the Receiver

Signal OFC teeth: $\nu_n = \nu_0 + n \cdot \Delta f_s$, for $n = 0, 1, ..., 7$

Reference OFC teeth: $\nu_n^{ref} = \nu_0 + n \cdot \Delta f_r$, where $\Delta f_r = \Delta f_s - \delta f$

Beat frequency for channel $n$: $f_{beat,n} = n \cdot \delta f$

With $\delta f = 10$ MHz, the 8 channels land at: **0, 10, 20, 30, 40, 50, 60, 70 MHz**

Each channel occupies a slot of width equal to the radar beat signal bandwidth (~5 MHz for 500 m range). Total composite ADC bandwidth needed: ~80–100 MHz. This is trivially achievable at any sample rate the ZCU208 supports — no high-frequency ADC headroom is consumed.

### Hardware Bill of Materials

#### Central Unit (per 8-channel group; × 8 groups for full 64-channel system)

| Component | Specification | Unit cost (approx.) |
|---|---|---|
| Low-RIN CW pump laser | 1550 nm, RIN < −165 dBc/Hz, linewidth < 10 kHz (e.g. NKT Koheras Basik, Orbits Lightwave) | $3,000–$8,000 |
| EO-OFC generator | EOM driven by stable microwave reference at desired tooth spacing; or EO-comb module | $4,000–$10,000 |
| EDFA (optical amplifier) | Boosts OFC power before distribution; Erbium-doped fibre amplifier | $2,000–$5,000 |
| 8-ch DWDM demux (outbound) | Passive athermal AWG, 100 GHz spacing, C-band; e.g. from DK Photonics, FS.com | $300–$800 |
| 8-ch DWDM mux (return) | Same technology, return path | $300–$800 |
| Reference OFC | Second EO-comb, phase-locked, offset tooth spacing | $4,000–$10,000 |
| Balanced photodetector + 90° hybrid | InGaAs balanced PD pair, 1–10 GHz, e.g. Thorlabs RXM series | $3,000–$6,000 |
| MZM automatic bias controller | Maintains quadrature operating point for all antenna EOMs | $2,000–$4,000 |
| Optical power monitor | Tracks link health | $500–$1,000 |

**Central unit subtotal per group: ~$19,000–$45,600**
**8 groups total: ~$152,000–$365,000**

**Important cost reduction — shared CW infrastructure:** The CW pump laser and reference OFC do not need to be duplicated per group. A single high-power low-RIN CW laser can feed all 8 EO-OFC generators via a 1×8 fibre splitter and per-branch EDFAs. Similarly, a single reference OFC can be distributed to all 8 balanced receivers. This eliminates 7 of the 8 CW lasers and 7 of the 8 reference OFCs from the bill of materials, reducing the central unit cost to approximately **$35,000–$80,000 total** across all 8 groups rather than the per-group sum. The EDFA and splitter hardware required for distribution adds approximately $5,000–$12,000.

#### Antenna Unit (per element; × 64 total)

| Component | Specification | Unit cost (approx.) |
|---|---|---|
| LNA (X-band) | Low noise figure at 5.8 GHz; e.g. Mini-Circuits PMA2-33LN+ | $150–$400 |
| EOM / MZM | 10 GHz LiNbO₃ intensity modulator, fibre-coupled; e.g. Thorlabs LN series | $1,500–$3,000 |
| VGA / step attenuator | Dynamic range protection before EOM | $200–$500 |
| Fibre patch cable + connectors | PM fibre, FC/PC | $50–$200 |

**Per antenna unit total: ~$1,900–$4,100**
**64 antenna units total: ~$122,000–$262,000**

**The dominant cost driver and the path to reduction:** The EOM at each antenna is the single largest cost item — $1,500–$3,000 × 64 = $96,000–$192,000. Traditional LiNbO₃ modulators carry this price because they are discrete, fibre-pigtailed, precision-assembled devices. Thin-film lithium niobate (TFLN) integrated modulators are a fundamentally different technology: the modulator is fabricated on a wafer-scale photonic integrated circuit (PIC), enabling batch production at dramatically lower per-unit cost. Companies such as HyperLight (Cambridge, MA) and Ligentec are producing TFLN devices with >100 GHz bandwidth, sub-1 V half-wave voltage (Vπ), and a cost trajectory that is expected to reach $200–$500 per unit at modest volume within the next 2–3 years. A 64-element TFLN-based antenna array would reduce the antenna unit hardware cost to approximately **$30,000–$55,000** total — roughly a 4× reduction. Silicon photonics integration (combining the WDM filter, EOM, and monitor PD onto a single chip per antenna) would reduce this further still and is an active area of development at companies including Luxtera and Intel Silicon Photonics.

#### Fibre plant

| Item | Cost |
|---|---|
| SMF-28 fibre, 100 m per run, 9 runs per group × 8 groups | ~$3,000–$8,000 |
| Fibre connectors, enclosures, cable management | ~$2,000–$5,000 |

**Fibre total: ~$5,000–$13,000**

#### System Total Estimate

| Subsystem | Low estimate (current) | High estimate (current) | TFLN projection |
|---|---|---|---|
| Central units (8 groups, shared laser/OFC) | $35,000 | $80,000 | $35,000–$80,000 |
| Antenna units (64 elements) | $122,000 | $262,000 | $30,000–$55,000 |
| Fibre plant | $5,000 | $13,000 | $5,000–$13,000 |
| **Total** | **~$162,000** | **~$355,000** | **~$70,000–$148,000** |

Current pricing is based on commercial-off-the-shelf discrete photonic components. The TFLN projection assumes near-term availability of integrated modulator PICs at volume pricing.

---

## 5. ENOB Implications: Analog vs Digital Dechirping

This is perhaps the most important system-level argument for the photonic architecture.

### The Digital Dechirp Path (traditional)

In digital dechirping, the raw RF return at 5.8 GHz is digitised first, and the conjugate multiplication is performed in the FPGA. This means the ADC must digitise the full 5.8 GHz carrier. As shown in Section 2, aperture jitter limits the ZU48DR to approximately **7 ENOB** at this frequency — regardless of bit depth.

```
Dynamic range (digital dechirp) ≈ 6.02 × 7 + 1.76 ≈ 44 dB
```

### The Analog Dechirp Path (electronic or photonic)

If dechirping occurs before the ADC — whether at the ZCU208 rack (Stage 1.5A), at the antenna (Stage 1.5B, Stage 2), or photonically (Stage 3) — the signal reaching the ADC is a low-frequency beat signal (0–5 MHz for a 500 m maximum range). The location of the mixer does not affect the ENOB benefit: what matters is that the ADC input frequency is the beat frequency, not the carrier. At these frequencies, the ZU48DR operates near its thermal noise floor, delivering close to its full rated performance.

At 1 MHz input, aperture jitter contribution:
```
SNR_jitter = −20 · log₁₀(2π × 1×10⁶ × 150×10⁻¹⁵) ≈ 130 dB
```

This is far above the ADC's thermal noise floor. ENOB is now limited by quantisation noise and thermal noise, not jitter — returning approximately **11–12 ENOB** at beat frequencies.

```
Dynamic range (analog/photonic dechirp) ≈ 6.02 × 11.5 + 1.76 ≈ 71 dB
```

### Comparison

| Dechirp method | Location | Signal at ADC | ENOB | Dynamic range |
|---|---|---|---|---|
| Digital conjugate multiply | ZCU208 FPGA | 5.8 GHz RF | ~7 bits | ~44 dB |
| Analog mixer (Stage 1.5A) | ZCU208 rack | ~1–5 MHz beat | ~11–12 bits | ~68–74 dB |
| Analog mixer (Stage 1.5B) | At antenna | ~1–5 MHz beat | ~11–12 bits | ~68–74 dB |
| Analog mixer + fibre return (Stage 2) | At antenna | ~1–5 MHz beat | ~11–12 bits | ~68–74 dB |
| Photonic (Stage 3, see Section 6) | Optical / at antenna | ~1–5 MHz beat | ~11–12 bits | ~68–75 dB |

The gain from any form of pre-ADC dechirping is approximately **4–5 bits**, or **25–30 dB** of additional dynamic range — regardless of whether the mixer is at the rack or at the antenna. The location affects deployment flexibility and sets up the transition to fibre stages, but does not change the ENOB outcome.

In the photonic OFC architecture, the dechirp can occur either:

- **Electronically at the antenna:** A mixer dechirps the signal before the EOM. The EOM carries the beat signal, not the full RF carrier. Most practical given current technology.
- **Photonically (see Section 6):** The dechirp happens in the optical domain, requiring no RF hardware at the antenna at all.

---

## 6. Photonic Dechirping

### Concept

Photonic dechirping performs the chirp correlation function in the optical domain rather than the RF domain. It eliminates the need for an RF dechirp mixer at the antenna while preserving the full ENOB advantage of sampling at beat frequencies.

### Architecture

The TX reference chirp is modulated onto a separate optical carrier using a second MZM at the central unit. This chirp-modulated optical signal is distributed to the antenna along the same fibre as the CW OFC wavelength for that element. At the antenna, the received RF signal and the optical reference chirp are combined optically on a single photodetector. The photodetector output is the dechirped beat signal — arising from the optical beating between the reference chirp carrier and the modulated return signal.

```
Central unit:
  [CW OFC tooth λᵢ] ─────────────────────────────┐
  [TX reference chirp → MZM → chirped optical ref] ─┤→ WDM combiner → fibre → antenna
                                                     │
At antenna:
  fibre → [WDM tap: separates λᵢ and chirp ref] 
         │                          │
  [EOM] ← [LNA] ← [Antenna]    [chirp ref optical]
         │                          │
         └──→ [2×1 coupler] ←───────┘
                    │
              [Photodetector]
                    │
              [beat signal output] → fibre → central unit → ADC
```

The photodetector performs the mixing: its square-law response to the combined optical field produces the product of the two signals, which includes the dechirped beat tone.

### Advantages

- No RF mixer at the antenna whatsoever — antenna unit is reduced to: **LNA + EOM + PD**
- Full ENOB benefit: beat signal goes directly to ADC
- Dechirp quality is set by optical component phase stability, which can be excellent
- Reference chirp distribution over fibre is immune to EM interference

### Current Maturity

Photonic dechirping for FMCW radar has been demonstrated in research settings (e.g., Pozar group, and various microwave photonics groups in Europe and China). It is not yet a turnkey commercial product. Key engineering challenges include:

- Precise alignment of optical path lengths to avoid range offset
- Phase noise of the chirp-modulated reference across the fibre round trip
- Calibration of the dechirp reference delay vs. antenna position

For a first-generation system, analog electronic dechirping is recommended as the lower-risk approach that captures the full ENOB benefit. If antennas are within ~10 m of the ZCU208, Stage 1.5A (mixer at rack, no hardware change at antenna) is the simplest possible starting point. For longer distances, Stage 1.5B (mixer at antenna, coax transport) or Stage 2 (fibre transport) follow naturally. Photonic dechirping is the longer-term optimisation target for deployments where any RF hardware at the antenna is undesirable.

---

## 7. Dynamic Range in the Photonic Link

The OFC photonic architecture introduces its own dynamic range constraints from the analogue optical link. The dominant mechanisms are MZM nonlinearity (producing third-order intermodulation distortion, IMD3) and laser relative intensity noise (RIN).

Spurious-free dynamic range (SFDR) for a photonic link is typically quoted in dB·Hz²/³ and must be converted to instantaneous dynamic range at the operating bandwidth. For beat signals at ~5 MHz bandwidth:

```
SFDR_inst = SFDR [dB·Hz²/³] − (2/3) · 10 · log₁₀(BW)
           = SFDR − (2/3) · 67 = SFDR − 44.5 dB
```

| Link configuration | SFDR (typical) | SFDR_inst at 5 MHz | ENOB equivalent |
|---|---|---|---|
| Basic MZM, no optimisation | ~105 dB·Hz²/³ | ~60 dB | ~9.7 bits |
| + Low-RIN laser (< −165 dBc/Hz) | ~112 dB·Hz²/³ | ~68 dB | ~11 bits |
| + Balanced detection | ~120 dB·Hz²/³ | ~75 dB | ~12.2 bits |
| + Dual-parallel MZM (linearisation) | ~128 dB·Hz²/³ | ~83 dB | ~13.5 bits |

An optimised photonic link (low-RIN laser + balanced detection) provides **~68–75 dB** instantaneous dynamic range, matching or exceeding the ZCU208's real-world performance with analog dechirping (~71 dB). The photonic link therefore does not introduce a dynamic range penalty relative to the best achievable ADC performance — it is not the limiting element.

The **VGA (variable gain amplifier)** at each antenna input is essential: it prevents strong clutter signals from compressing the MZM, protecting the operating point and preserving SFDR under high-power receive conditions.

---

## 8. Lower-Cost Alternative: Analog Dechirp + SCM over Fibre (Stage 2)

The OFC channelisation architecture is the most elegant solution, but its cost is dominated by the 64 high-quality EOMs at each antenna. A substantially cheaper path to 64 channels, full ENOB recovery, and physical separation exists — at the cost of accepting one simple RF mixer per antenna for dechirping.

### Architecture Overview

The principle is: dechirp at the antenna using a distributed optical reference, transport the resulting low-frequency beat signals back over fibre using subcarrier multiplexing (SCM), and separate channels digitally at the ZCU208.

```
CENTRAL UNIT
─────────────────────────────────────────────────────────────────────
  ZCU208 DAC → TX chirp (5.8 GHz)
                   │
                   ├──→ TX chain (to transmit antennas)
                   │
                   └──→ [Reference distribution MZM]
                               │
                         [DFB reference laser]
                               │
                         [1×64 passive fibre splitter]
                               │
                    (one reference fibre to each antenna)

  Return path (one fibre per group of 8 antennas):
  [PD] → [ZCU208 ADC] × 8 channels
─────────────────────────────────────────────────────────────────────
ANTENNA UNIT (×64)
─────────────────────────────────────────────────────────────────────
  [Reference fibre in]
         │
  [Wideband PD]        ← converts optical chirp reference back to RF
         │
  [RF amplifier]       ← brings recovered chirp to mixer LO level
         │
  ┌──────┘
  │  [LO port]
  [Dechirp mixer]  ←── [RF port] ← [LNA] ← [Antenna]
  │  [IF port]
  └──→ [beat signal: 0–5 MHz]
         │
  [SCM offset mixer]   ← shifts beat to unique frequency slot
         │             ← driven by cheap crystal-derived tone (n × 10 MHz)
  [SCM signal: e.g. 30–35 MHz for antenna 3]
         │
  ──────→ [passive RF combiner] (shared with 7 other antennas in group)
                   │
           [composite SCM signal: 0–80 MHz, 8 channels]
                   │
           [cheap VCSEL/DFB + basic modulator]
                   │
           [return fibre to central unit]
─────────────────────────────────────────────────────────────────────
```

### Reference Chirp Distribution — Explicit Hardware Detail

This is the component of the architecture that requires careful explanation. The ZCU208 DAC generates the TX chirp as an RF signal at 5.8 GHz. A copy of this chirp must reach every antenna element to serve as the dechirp reference. Distributing a 5.8 GHz signal over 100 m of coaxial cable is impractical — LMR-400 has approximately 10 dB/100 m loss at 6 GHz, and the signal would require significant amplification at each antenna, defeating the purpose. Fibre is the correct medium: loss is ~0.3 dB/km regardless of frequency, making 100 m essentially lossless.

**At the central unit — converting chirp to optical:**

| Component | Role | Cost |
|---|---|---|
| DFB reference laser (1310 or 1550 nm, basic telecom grade) | Optical carrier for reference; does not need low RIN since it carries a strong reference tone, not a weak radar return | $300–$800 |
| 10 GHz MZM (one shared unit) | Imprints the 5.8 GHz RF chirp onto the optical carrier | $1,500–$2,500 |
| 1×64 passive single-mode fibre splitter | Distributes the chirp-modulated optical signal to all 64 antennas simultaneously; passive, no power required | $200–$600 |
| EDFA (optional) | Compensates the ~18 dB splitting loss of a 1×64 splitter; keeps adequate optical power at each antenna PD | $1,500–$3,000 |

The splitter loss is: 10·log₁₀(64) ≈ 18 dB. With a launch power of ~10 dBm from the MZM and EDFA, each antenna receives approximately −8 dBm — sufficient for a wideband PD to recover the reference chirp cleanly.

**At each antenna — converting optical reference back to RF:**

| Component | Role | Cost |
|---|---|---|
| Wideband InGaAs photodetector (≥8 GHz bandwidth) | Converts the optical chirp reference back to a 5.8 GHz RF signal; square-law detection recovers the RF modulation directly | $300–$800 |
| RF amplifier / driver (e.g. Mini-Circuits GALI series) | Boosts the recovered chirp to adequate LO drive level for the dechirp mixer (typically −5 to +5 dBm) | $50–$150 |

The recovered RF signal at the PD output is an accurate replica of the TX chirp, delayed only by the optical propagation time (constant, ~5 ns/m of fibre — equivalent to a fixed range offset that is calibrated out in software). This recovered chirp drives the LO port of the dechirp mixer.

**Dechirp mixer at the antenna:**

| Component | Role | Cost |
|---|---|---|
| Wideband RF mixer (e.g. Mini-Circuits ZX05-43MH+) | Multiplies received signal by reference chirp; output is the dechirped beat tone at 0–5 MHz | $20–$80 |
| Low-pass filter (5–10 MHz cutoff) | Removes the sum frequency component from the mixer output | $10–$30 |

The combined hardware for reference chirp recovery and dechirping at each antenna is therefore: **wideband PD + RF amp + RF mixer + LPF ≈ $380–$1,060 per antenna.** This is the only meaningful addition over the OFC architecture's LNA-only antenna unit in terms of active hardware.

### SCM Return Path

The 8 dechirped beat signals within a group are shifted to non-overlapping frequency slots using cheap baseband MMIC mixers driven by crystal-derived tones:

| Antenna | SCM offset tone | Beat slot |
|---|---|---|
| 1 | 7.5 MHz | 5–10 MHz |
| 2 | 17.5 MHz | 15–20 MHz |
| 3 | 27.5 MHz | 25–30 MHz |
| ... | ... | ... |
| 8 | 77.5 MHz | 75–80 MHz |

The composite 0–80 MHz signal modulates a simple, cheap intensity modulator (a VCSEL-based transmitter or basic DFB + modulator combination). At the central unit, a single PD and one ZCU208 ADC channel captures all 8 channels, which are separated by digital bandpass filtering.

Cost of SCM offset mixer per antenna: $10–$30 (MMIC baseband mixer). Cost of the SCM tone generator (shared for 8 antennas): $50–$150 (crystal oscillator + divider chain).

### Stage 2 Bill of Materials

#### Central Unit (shared across all 64 channels)

| Component | Cost |
|---|---|
| DFB reference laser | $300–$800 |
| 10 GHz MZM for reference distribution | $1,500–$2,500 |
| 1×64 passive fibre splitter | $200–$600 |
| EDFA (optional, for splitter loss) | $1,500–$3,000 |
| 8× return PD (simple, 100 MHz bandwidth) | $800–$2,400 |
| SCM tone reference generator | $200–$500 |
| **Central unit total** | **$4,500–$9,800** |

#### Per Antenna (×64)

| Component | Cost |
|---|---|
| LNA (X-band, 5.8 GHz) | $150–$400 |
| Wideband PD for chirp reference recovery (≥8 GHz) | $300–$800 |
| RF amplifier (chirp reference drive) | $50–$150 |
| Dechirp mixer | $20–$80 |
| Low-pass filter | $10–$30 |
| SCM offset mixer (baseband) | $10–$30 |
| VCSEL/DFB return modulator (shared ÷ 8) | $25–$75 |
| Passives, filters, cable | $30–$80 |
| **Per antenna total** | **$595–$1,645** |

**64 antennas total: ~$38,000–$105,000**

#### Stage 2 System Total

| Subsystem | Low | High |
|---|---|---|
| Central unit | $4,500 | $9,800 |
| 64 antenna units | $38,000 | $105,000 |
| Fibre plant (reference + return) | $5,000 | $13,000 |
| **Total** | **~$47,500** | **~$128,000** |

This is approximately **3–4× cheaper** than the OFC channelisation architecture at current component prices, and achieves the same ENOB (~11–12 bits), the same 64-channel count, and the same physical separation. The sole tradeoff is the addition of a wideband PD and RF dechirp mixer at each antenna.

---

## 9. Coaxial SCM — All Benefits of Stage 2 Without Photonics (Stage 1.75)

Stage 2 multiplexes 8 antenna channels onto one fibre using SCM. The identical multiplexing principle works over coaxial cable — because at beat frequencies (0–80 MHz), coax is as capable a transport medium as fibre. This stage achieves 64 RX channels and full ENOB using nothing but coaxial cable, passive combiners, and cheap MMIC offset mixers.

### Why Coax Works at Beat Frequencies

After dechirping and SCM offsetting, the 8 channels from one antenna group sit in distinct frequency slots between 0 and 80 MHz. Coaxial cable loss at these frequencies is negligible:

| Cable type | Loss at 80 MHz / 100 m |
|---|---|
| RG-174 (thin, flexible) | ~1.5 dB |
| RG-58 | ~0.8 dB |
| LMR-195 | ~0.5 dB |
| LMR-400 | ~0.3 dB |

Even the thinnest, cheapest coax carries 8 multiplexed beat signal channels over 100 m with negligible degradation. This is the same principle used in cable television infrastructure — dozens of channels multiplexed between 5 MHz and 1 GHz over standard coax. The composite signal from one group of 8 antennas is directly sampled by one ZCU208 ADC channel, with digital bandpass filters separating the 8 channels in the FPGA.

### Architecture

```
ZCU208 DAC → TX chirp (5.8 GHz)
                  │
                  ├──→ TX chain
                  │
                  └──→ [1:8 RF splitter per group]
                              │
              (one reference coax per antenna, ~30–50 m max)

At each antenna (×8 per group, ×8 groups = 64 total):
  [ref coax] → [RF amp, if needed] → [LNA+mixer MMIC] ← [Antenna]
                                            │
                                       [LPF]
                                            │
                                    [SCM offset mixer] ← [crystal tone: n × 10 MHz]
                                            │
                               ─────────────────────────────
                               shared with 7 others in group ↓
                              [8-way passive RF combiner]
                                            │
                          ←─ one RG-58 return coax per group ─→ ZCU208 ADC
```

Eight groups × one return coax each = **8 coax return cables total** for 64 channels, directly into the 8 ZCU208 ADC inputs.

### SCM Offset Tone Generation

The 8 frequency offsets needed per group (e.g. 7.5, 17.5, 27.5 ... 77.5 MHz) are derived from a single cheap oscillator shared across the group. Options:

- **Crystal oscillator + divider chain:** A 70 MHz TCXO divided down provides all 8 tones. Passive, cheap, no configuration required. ~$5–$15 for the oscillator, ~$2–$5 per divider stage.
- **ADF4351 wideband PLL synthesiser:** Generates any frequency from 35 MHz to 4.4 GHz from a reference input. Configurable over SPI. ~$10 per chip. One chip per group can produce all 8 tones sequentially if time-division is acceptable, or 8 chips produce them simultaneously.
- **ZCU208 DAC auxiliary output:** The ZCU208 has 8 DACs. With one DAC used for TX chirp generation, remaining DAC channels can generate the SCM offset tones digitally. No additional oscillator hardware required at all — tones are generated in the FPGA and output over short coax to the antenna groups. This is the most elegant option if DAC channels are available.

### Reference Chirp Distribution

Identical to Stage 1.5B — one coax per antenna, amplified at the antenna if runs exceed ~30 m. The only change from Stage 1.5B is that the beat signal return is now combined with 7 other channels before entering a single coax, rather than returning individually. The reference distribution problem is unchanged.

If 8 separate reference coax runs per group (one per antenna) are undesirable, a local 1:8 splitter at the antenna group distributes from a single group-level reference coax run. This reduces the cable count from the ZCU208 from 8+8 = 16 per group down to 1+1 = 2 per group (one reference in, one return out), at the cost of ~9 dB splitting loss compensated by amplifiers at each antenna.

### Bill of Materials (per 8-antenna group)

| Component | Quantity | Unit cost | Total |
|---|---|---|---|
| LNA+mixer MMIC (e.g. HMC1190) | 8 | $30–$60 | $240–$480 |
| LPF, 5–10 MHz | 8 | $5–$15 | $40–$120 |
| SCM offset mixer (baseband MMIC) | 8 | $5–$15 | $40–$120 |
| SCM tone oscillator (shared, per group) | 1 | $10–$30 | $10–$30 |
| 8-way passive RF combiner (return) | 1 | $20–$60 | $20–$60 |
| 1:8 RF splitter (reference, if group-level) | 1 | $20–$50 | $20–$50 |
| RF amplifier for reference recovery | 8 | $5–$15 | $40–$120 |
| RG-58 reference coax, 50 m | 1 | $30–$60 | $30–$60 |
| RG-58 return coax, 50 m | 1 | $20–$40 | $20–$40 |
| SMA connectors, passives, enclosure | — | — | $50–$150 |
| **Per group total** | | | **$510–$1,230** |

**8 groups total: ~$4,000–$10,000**

Compare to Stage 2 (fibre SCM) which costs ~$48,000–$128,000 for the same 64 channels. Stage 1.75 is approximately **10–12× cheaper** for identical ENOB and channel count.

### Limitations vs Stage 2

**Physical separation is bounded.** The reference chirp at 5.8 GHz limits coax run length to ~50 m before amplification becomes necessary. Stage 2 removes this limit entirely with fibre.

**EM susceptibility.** Coaxial cables pick up interference and radiate at the composite SCM frequencies (0–80 MHz). In a high-EMI environment (near high-power TX antennas, motor drives, or power electronics) this can degrade dynamic range. Good shielding and braided coax mitigate this but cannot eliminate it. Fibre is immune.

**Cable bulk.** 8 reference runs + 8 return runs = 16 coaxial cables between ZCU208 and antenna groups, even with group-level reference splitting. Manageable in a lab or controlled deployment; less attractive for a field system.

**Natural transition to Stage 2.** Moving from Stage 1.75 to Stage 2 requires no changes to the antenna hardware whatsoever. The LNA+mixer MMIC, LPF, and SCM offset mixer stay in place. The coax runs are replaced with fibre, and a VCSEL return modulator is added at the antenna group output. The ZCU208-side coax inputs are replaced with photodetectors. The antenna PCBs do not change.

---

## 10. Architecture Comparison Summary

| Property | Stage 1 | Stage 1.5A | Stage 1.5B | Stage 1.75 | Stage 2 | Stage 3 |
|---|---|---|---|---|---|---|
| RX channels | 8 | 8 | 8 | 64 | 64 | 64 |
| ADC count | 8 | 8 | 8 | 8 | 8 | 8 |
| Link medium | Coax (RF) | Coax (RF+beat) | Coax (ref+beat) | Coax (ref+SCM) | Fibre | Fibre |
| Max separation | ~5 m | ~10 m | ~50 m | ~50 m | Unlimited | Unlimited |
| EM interference | High | High | High | Medium | Zero | Zero |
| Dechirp location | ZCU208 FPGA | ZCU208 rack | Antenna | Antenna | Antenna | Antenna/optical |
| Mixing at antenna | None | None | Yes | Yes | Yes | No |
| ENOB | ~7 bits | ~11–12 bits | ~11–12 bits | ~11–12 bits | ~11–12 bits | ~11–12 bits |
| Dynamic range | ~44 dB | ~68–74 dB | ~68–74 dB | ~68–74 dB | ~68–74 dB | ~68–75 dB |
| Virtual aperture | 64 el. | 64 el. | 64 el. | 512 el. | 512 el. | 512 el. |
| Angular resolution | ~1.8° | ~1.8° | ~1.8° | ~0.22° | ~0.22° | ~0.22° |
| Swarm discrim. | <100 m | <100 m | <100 m | <780 m | <780 m | <780 m |
| System cost | ~$10k–$20k | +~$2k–$5k | +~$2k–$6k | +~$4k–$10k | ~$48k–$128k | ~$162k–$355k |

---

## 11. Staged Development Roadmap

| Stage | Architecture | Channels | Key action | Cost estimate | ENOB |
|---|---|---|---|---|---|
| 1 | Direct RF sampling, digital dechirp | 8TX × 8RX | Validate signal processing chain on ZCU208 using DAC-generated chirp and digital conjugate multiply. Accept 7-bit dynamic range as known constraint during algorithm development. | ~$10k–$20k | ~7 bits |
| 1.5A | Analog dechirp, mixer at ZCU208 rack | 8TX × 8RX | Add 8× RF mixer bank at ZCU208. Reference from ZCU208 DAC via short power divider. Immediately recovers 4–5 ENOB bits. Antennas must remain within ~10 m. | +~$2k–$5k | ~11–12 bits |
| 1.5B | Analog dechirp, mixer at antenna, coax transport | 8TX × 8RX | Move mixers to antenna. Distribute reference chirp over coax. Beat signals return over thin coax. Enables antennas up to ~50 m from ZCU208. Antenna hardware is identical to all subsequent stages. | +~$2k–$6k | ~11–12 bits |
| 1.75 | Analog dechirp + coaxial SCM multiplexing | 8TX × 64RX | Add SCM offset mixers and passive combiner at each antenna group. 8 antenna channels share one return coax per group. Scale to 64 RX channels using 8 ZCU208 ADCs. No photonics required. | +~$4k–$10k | ~11–12 bits |
| 2 | Analog dechirp + SCM + Radio over Fibre | 8TX × 64RX | Replace reference coax with optical fibre link. Replace return coax with VCSEL + fibre. Enables unlimited physical separation and full EM immunity. Antenna hardware unchanged from Stage 1.75. | ~$48k–$128k | ~11–12 bits |
| 3 | OFC photonic channelisation | 8TX × 64RX | Replace SCM mixer chain with EOM per antenna. Eliminates all RF mixing at antenna. Full photonic link from antenna to ADC. | ~$162k–$355k | ~11–12 bits |
| 3+ | OFC + TFLN integrated modulators | 8TX × 64RX+ | TFLN photonic integrated circuits replace discrete EOMs. Cost approaches Stage 2. Architecture extendable beyond 64 RX. | ~$70k–$148k | ~11–12 bits |

The most important single step is Stage 1 → Stage 1.5, which recovers ~25 dB of dynamic range for minimal cost. Stage 1.75 is the most cost-effective path to 512-element virtual aperture and swarm discrimination capability, with no photonics required. Stage 2 is the transition point when physical separation beyond 50 m or EM cleanliness becomes a hard requirement.

---

## 12. Key References and Technology Suppliers

**Photonic channelisation:**
- Cohen, I. and Eldar, Y.C. — Sub-Nyquist MIMO radar (Weizmann Institute X-band prototype, 8TX/10RX, FDM + compressed sensing)
- Pan, S. et al. — Microwave photonic radars (comprehensive review)

**OFC sources:**
- NKT Photonics (Koheras Basik — low-RIN CW laser for OFC pumping)
- Toptica Photonics (commercial OFC modules)
- Menlo Systems (femtosecond OFC)

**EOM/MZM suppliers:**
- Thorlabs (LNA/LNX series, 10 GHz LiNbO₃, ~$1,500–$3,000)
- iXblue / Exail (broadband modulators, research grade)
- HyperLight (TFLN integrated modulators, next-generation cost point)

**Wideband photodetectors (reference chirp recovery, ≥8 GHz):**
- Thorlabs DET08CFC (5 GHz, InGaAs, fibre-coupled, ~$400)
- Discovery Semiconductors DSC-R series (10–20 GHz, ~$500–$1,200)
- Finisar/II-VI XPDV series (40 GHz+, research grade)

**RF dechirp mixers:**
- Mini-Circuits ZX05-43MH+ (5–4,300 MHz, ~$30)
- Marki Microwave M1 series (wideband, low conversion loss)

**WDM passive components:**
- FS.com, DK Photonics, Thorlabs (8-ch DWDM mux/demux, $300–$800)

**Cheap return-path optical transmitters (SCM):**
- Thorlabs S1FC series VCSEL modules (~$200–$400)
- Standard SFP+ DWDM transceivers (if analog bandwidth is sufficient)
