---
name: rf-component-selection
description: Use this skill for a rigorous, manufacturer-agnostic process for selecting active RF/microwave components — LNAs, mixers, ADCs, VCOs, amplifiers — wherever noise figure, linearity, dynamic range, phase noise, ENOB, availability, or lead time is a system-level constraint. Trigger when the user (or the rf-pcb-engineer agent) needs to choose an RF/analogue part, compare candidate ICs at an operating frequency, run a multi-round component search, or build and justify a BOM line. Covers establishing requirements, the multi-pass search process, common search failure modes, price prioritisation by build type, and documentation requirements. NOT for laying out the copper around a chosen part — that is the rf-pcb-design-process skill, driven by the rf-pcb-engineer agent.
---

# RF Component Selection Methodology

**Purpose:** A rigorous process for selecting RF/microwave components for prototype and production radar systems, derived from the FMCW MIMO radar development process.

**Scope:** Applies to active RF/analogue components: LNAs, mixers, ADCs, VCOs, amplifiers, and any part where noise figure, linearity, dynamic range, or phase noise is a system-level constraint.

---

## 0. Instructions for the Search Agent

> **This section is for anyone — human or AI — executing the search.**

**Never stop at the first result.** The first search in a component selection will almost always surface the most-marketed part, not the best part. During the FMCW radar development, the mixer search required five distinct search rounds across different manufacturers before landing on the correct part. Parts found in earlier rounds were wrong in frequency range, IIP3, phase noise behaviour, or availability. Do not present the first plausible result as the answer.

**Search rules:**
- Search at least three separate manufacturer namespaces per component type (e.g. ADI/Hittite, Qorvo/Custom MMIC, MACOM, TI, Marki, Mini-Circuits)
- If a search returns no clearly superior part or yields broad results, **explicitly ask the engineer to clarify which performance parameter to optimise before running the next search**. Do not guess.
- Always verify specs at the **actual operating frequency**, not the headline figure. Many parts advertise their best-case frequency. A 2–14 GHz mixer's headline IIP3 may be at 5.8 GHz; the figure at 10.25 GHz may be 4 dB lower.
- Always check **availability and stock** at a standard distributor (Digikey, Mouser) before presenting a part as a candidate. A part unavailable in prototype quantities is not a candidate regardless of performance.
- Always check **lead time**. A 35-week factory lead time on a part with 455 units in stock changes procurement urgency fundamentally.
- Check **part status** — active, last-time-buy, discontinued, or not recommended for new designs. Do not recommend discontinued parts.
- When a part is unavailable but clearly superior, note it explicitly as a future revision option, not a current choice.

---

## 1. Before Searching — Establish the Requirements

Before running any searches, the following must be defined. If they are not defined, ask. Do not proceed with a search that will return irrelevant results because the requirements were ambiguous.

### 1.1 Functional Requirements

| Requirement | Questions to Answer |
|---|---|
| **Operating frequency** | What is the exact RF frequency? Is the part at RF, LO, or IF? What is the IF frequency range? (DC to X MHz for dechirp radar; never assume.) |
| **Signal bandwidth** | How wide must the part perform? Avoid parts whose rated bandwidth extends far beyond the system requirement — they pay a performance penalty at the operating frequency to cover the wider range. |
| **Primary performance metric** | What matters most? NF (for LNA), IIP3 (for mixer linearity), phase noise (for LO-connected parts), conversion loss, SFDR, ENOB? State this before searching. |
| **Secondary metrics** | What is the acceptable trade-off space? (e.g. 0.5 dB extra NF is acceptable if it saves $30/chip) |
| **Interface constraints** | What does the part connect to? Single-ended or differential? DC-coupled or AC-coupled? What drive levels does it see? What does it drive? |
| **Supply voltage** | Single positive supply preferred? Negative gate bias acceptable? High voltage (7V for some HMC parts) tolerable? |
| **Package** | SMD required? Max footprint? Does it need to be pin-compatible with another part for an upgrade path? |
| **Quantity and build type** | How many units are needed, and is this a prototype or production build? This determines price weighting (see Section 4). |

### 1.2 System Context Requirements

Before selecting any component, understand its position in the signal chain and what that implies:

- **LNA:** Noise figure dominates the system by Friis. Every 0.1 dB of LNA NF is 0.1 dB of system NF. Linearity (P1dB, IIP3) matters only for strong-signal scenarios. Gain must be enough to suppress downstream noise contributions.
- **Mixer after a high-gain LNA:** The mixer's noise figure is divided by the LNA gain in Friis. IIP3 matters more than NF. Phase noise added by any active element in the LO path is not divided — it appears directly in the IF output.
- **ADC:** ENOB is limited by the worst of SNR and SFDR at the actual IF input frequency, not the headline frequency. Oversampling and decimation improve effective ENOB. Interface complexity (parallel LVDS vs JESD204B) has downstream consequences for FPGA selection.

Run a Friis calculation for the full chain before finalising any part selection. A component that looks inferior in isolation may be adequate once placed in system context.

---

## 2. The Search Process

### 2.1 First Pass — Breadth

The first search should be **broad**. The goal is to identify the space of possible parts, not to select one. Use generic terms: technology type, frequency range, primary metric.

Example for a dechirp radar mixer: *"X-band double balanced mixer 8-12 GHz high IIP3 SMT available 2025"*

Collect all plausible candidates without eliminating any. Record manufacturer, part number, headline specs, package, and apparent availability.

**If the first search returns more than 6–8 distinct plausible parts**, the requirements are not specific enough. Stop. Ask which parameter to prioritise before narrowing.

### 2.2 Second Pass — Frequency Verification

For every candidate from the first pass:

1. Find the **datasheet spec at your specific operating frequency**, not the headline figure or the spec at the part's optimum frequency. Many datasheets include performance vs frequency plots — read them.
2. Flag any part where the operating frequency is near the rated band edge. Performance degrades toward band edges. A 2–14 GHz part operating at 10.25 GHz is near its upper limit and will underperform its mid-band headline specs.
3. Eliminate any part whose rated band does not comfortably cover the operating frequency with margin.

### 2.3 Third Pass — Competitive Comparison

Build a comparison table of surviving candidates with specs **at the actual operating frequency**:

- Include: primary metric, secondary metrics, supply requirements, package, price at prototype quantity, price at production quantity, stock at Digikey/Mouser, lead time, part status
- Note explicitly whether each spec is measured at your operating frequency or extrapolated
- Note whether the part's NF/IIP3/SFDR figure comes from the datasheet spec table or from a performance graph (graphs are more accurate; spec tables may be worst-case or best-case depending on manufacturer)

### 2.4 Fourth Pass — Availability and Procurement

For each remaining candidate:

- Confirm **in-stock availability** at Digikey and/or Mouser for prototype quantities
- Confirm the **price at prototype quantity** (typically 10–100 units) — this is almost never the 1ku list price shown on manufacturer websites
- Confirm the **price break quantities** and whether the prototype order qualifies
- Check the **factory lead time** — if a part has long lead time, note whether current stock is adequate for the full prototype run including potential rework
- Check whether the part is available in **cut tape** (individual units) or only in full reels — some RF MMICs are reel-only with minimum orders of 100–500 units
- For any part only available through specialty distributors (RFMW, Richardson RFPD, Marki direct), note this explicitly — it affects procurement complexity

### 2.5 Final Selection

Present candidates ranked by primary performance metric at actual operating frequency, with price and availability clearly noted. State explicitly:

- Which part is recommended and why
- What tradeoffs were made
- Which rejected parts would be superior in a different cost or performance context
- Whether any part should be noted as a future revision option

---

## 3. Common Failure Modes in RF Component Searches

These are errors that occurred in practice during the FMCW radar development. Each one caused a part to be selected, then reversed.

### 3.1 Wrong Frequency Range

**What happened:** The QPL9547 was specified as an 8–12 GHz LNA. It is a 0.1–6 GHz LNA. The HMC213BMS8E was specified as an 8–18 GHz mixer. It is a 1.5–4.5 GHz mixer.

**How to avoid:** Always verify the rated frequency range from the actual datasheet, not from search result snippets or memory. Check both the lower and upper rated limits. If a part number is familiar, verify it anyway — part numbers in the same family can cover very different frequencies.

### 3.2 Headline Spec Not at Operating Frequency

**What happened:** The LTC5548 was cited as having +24.4 dBm IIP3. That figure is at 5.8 GHz. At our 10.25 GHz operating frequency the IIP3 is approximately +20 dBm — and conversion loss degrades from 7 dB to ~10 dB.

**How to avoid:** Always find the spec **at the operating frequency**. If only a graph is available, read the value from the graph at the operating point. State explicitly whether a spec is measured or extrapolated.

### 3.3 Active vs Passive in the LO Path

**What happened:** The LTC5548's integrated active LO buffer was initially treated as a pure benefit (0 dBm LO drive). The noise added by that active buffer to the LO path — directly degrading Doppler floor — was only identified after the part was already selected and partially documented.

**How to avoid:** For any component in the LO or reference signal path of a radar or other phase-noise-sensitive system, explicitly ask: does this part add any active gain stage? If so, what is the noise figure of that stage, and how does it affect the phase noise budget?

### 3.4 Obsolete or End-of-Life Parts

**What happened:** The AD9628 was initially in the ADC staging plan. It is being discontinued. The ADS5485 appeared as a candidate for Stage 2; it is discontinued at Digikey.

**How to avoid:** Check part status at Digikey explicitly. "Not recommended for new designs", "last-time-buy", and "discontinued" are all disqualifying unless the part is for short-term stock buying only.

### 3.5 Stopping at the First Plausible Result

**What happened:** The mixer search initially settled on HMC774ALC3B (too wide band), then CMD177C3 (+16 dBm IIP3, too low), then LTC5548 (active LO concerns). Only after multiple search rounds did the MAMX-011035 emerge as the correct part.

**How to avoid:** If the first search returns a plausible part, run at least one more search from a different manufacturer or with more specific terms before presenting a recommendation. The first result is rarely the best result in a competitive component space.

### 3.6 Distributor List Price vs Real Prototype Price

**What happened:** The AD9648-125 was cited at $87/chip based on the ADI 1ku list price. The actual Digikey price at small quantities was $140/chip — 60% higher. The BOM was wrong for multiple iterations.

**How to avoid:** Always look up the price at Digikey or Mouser for the actual quantity being ordered. Manufacturer list prices (1ku, MSRP) and actual small-quantity distributor prices are systematically different. State both where relevant, and use the distributor price for BOM calculations.

---

## 4. Price Prioritisation Framework

Price weighs differently depending on build type. Be explicit about which context applies before the search.

### Prototype / First Build

**Price is a significant constraint.** A prototype may be redesigned. Parts that cost 3× as much for marginal performance gains are hard to justify when the design may be respun. Price should be a first-order filter on candidates — parts over a clear cost threshold require explicit justification.

*Example decision: LTC5548 at $24 vs MAMX-011035 at $61. The MAMX-011035 is better, but at prototype stage the 0 dBm LO simplification and $37 saving per chip were relevant factors. The final decision to use MAMX-011035 came because the performance gap (phase noise, conversion loss) was large enough to justify it even at prototype stage.*

**Price break quantities matter.** Check whether ordering the full prototype run (e.g. 5 boards × 8 chips = 40 units) crosses a price break. The difference between 1-unit and 25-unit pricing can be 30–40%. Always state both prices.

### Second Build / Validated Design

**Price is a medium constraint.** The design is proven. Better parts can be introduced for specific performance improvements without full respin risk. A $50 premium per chip across 8 channels is $400 per board — meaningful but not prohibitive if the performance gain is documented and justified.

### Production Build

**Price is a lower constraint.** Volume pricing, alternative approved sources, and alternative parts with equivalent performance become relevant. The focus shifts to availability security, second-sourcing, and long-term supply agreements. Performance that was marginal at prototype stage can be re-evaluated. Parts that were "good enough" may be replaced with optimal parts now that the design is stable.

*Note: Even in production, do not sacrifice system-critical specs (LNA noise figure, mixer phase noise in a Doppler radar, ADC ENOB) for cost unless a system-level analysis shows the degradation is acceptable.*

### Price Summary Table

| Build Type | Price priority | Primary filter | Secondary filter |
|---|---|---|---|
| Prototype / first build | High | Performance at operating frequency | Availability and lead time |
| Second build | Medium | Performance; consider upgrades | Cost delta vs prototype part |
| Production | Lower | Long-term supply security | Volume pricing and second sources |

---

## 5. Documentation Requirements

Every part selection must be documented with the following, in the architecture document:

1. **What was searched** — manufacturer families covered, search terms used
2. **What was found** — full candidate table with specs at operating frequency
3. **What was rejected and why** — explicit rejection rationale for each candidate
4. **What was selected and why** — primary reasoning, tradeoffs accepted
5. **What is the future upgrade path** — if a better part exists but was ruled out on price or availability, document it explicitly as the future revision target
6. **Confirmed specs** — verify from datasheet, not from search snippets. Note explicitly if any spec is extrapolated rather than directly measured
7. **Price and availability** — at actual prototype quantity from Digikey/Mouser, not manufacturer list price
8. **Lead time and stock** — critical for ordering decisions

---

## 6. Component-Specific Considerations

### LNA

- **NF at operating frequency dominates everything.** Use Friis to verify the LNA NF contributes more than all other sources combined. If it does not, the LNA is not the bottleneck and a cheaper part may be appropriate.
- **50Ω internal matching is strongly preferred** for X-band — external matching at 10+ GHz requires simulation and board area, and varies with tolerance.
- **Single positive supply** strongly preferred — negative gate bias requires additional circuitry and introduces tuning variability.
- **Input trace length matters** — every millimetre of PCB trace between the antenna and the LNA input adds approximately 0.05 dB at X-band. Quantify this in the layout.

### Mixer (Dechirp)

- **IF frequency range first** — a dechirp radar produces a beat signal from near-DC to a few hundred MHz. Many microwave mixers have AC-coupled IF ports that cut off below 500 MHz. Verify the IF port works from DC.
- **Passive core strongly preferred** — any active element in the LO path (including integrated LO buffers) adds phase noise that directly degrades the Doppler floor. For Doppler-critical applications, passive GaAs is preferred.
- **IIP3 at operating frequency** — not headline, not at best-case frequency. The near/far dynamic range directly maps to mixer IIP3.
- **LO drive level** — passive mixers typically need +13 to +17 dBm. This drives the TX board reference chirp distribution design. Lower LO drive (e.g. active LO buffer parts) simplifies distribution but adds noise.

### ADC

- **Beat signal frequency, not RF frequency** — the ADC sees the beat signal (DC to hundreds of MHz), not the 10 GHz RF carrier. Aperture jitter requirements are orders of magnitude more relaxed than for direct RF sampling.
- **SNR and SFDR at the IF frequency** — not at Nyquist. Pipeline ADC SNR is relatively flat at low input frequencies and degrades toward Nyquist.
- **ENOB = min(SNR-limited ENOB, SFDR-limited ENOB)** — check both. At low input frequencies, SNR usually dominates.
- **Oversampling gain** — if the beat bandwidth is much less than the ADC Nyquist, oversampling + decimation recovers meaningful ENOB. Calculate this explicitly.
- **Interface complexity** — parallel LVDS, serial LVDS, and JESD204B have very different FPGA resource and I/O pin implications. Confirm FPGA compatibility before locking in the ADC.
- **Upgrade path** — if the ADC family has a higher-resolution pin-compatible variant, document it. The ability to swap resolution tiers without a board respin is valuable.
- **Temperature grade** — C grade (0°C to 70°C) is inadequate for outdoor systems. I grade (-40°C to 85°C) is the minimum for any outdoor radar application.

---

## 7. Quick Reference Checklist

Before finalising any component selection, verify every item:

- [ ] Frequency range verified from datasheet — operating point is NOT near the band edge
- [ ] Primary spec (NF / IIP3 / SNR) confirmed at actual operating frequency, not headline frequency
- [ ] Spec source identified — datasheet table or graph? Measured or extrapolated?
- [ ] Part status confirmed — active, not discontinued, not NRND
- [ ] In-stock quantity confirmed at Digikey or Mouser for full prototype run
- [ ] Factory lead time confirmed — if > 8 weeks, current stock must cover the order
- [ ] Price confirmed at actual prototype quantity from distributor, not manufacturer list price
- [ ] Price break quantities identified
- [ ] Temperature grade appropriate for deployment environment
- [ ] Supply voltage requirements compatible with board power architecture
- [ ] Package footprint confirmed — SMD, solderable by Würth or equivalent
- [ ] Friis calculation updated with selected part
- [ ] Rejected alternatives documented with rationale
- [ ] Future revision upgrade path identified and noted
- [ ] BOM updated with confirmed price and part number including full suffix