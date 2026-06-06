# RF Frontend Architecture

> **⚠️ SUPERSEDED — June 2026.** This document describes the original
> ZCU208-centred RF frontend (the Option A direct-S-band-sampling vs.
> Option B downconvert-to-IF decision, with the RFSoC as the central
> digitiser). The project has since settled on **analogue dechirp at the
> antenna, a commodity ADC per channel digitising the beat signal at the
> edge, an ECP5 edge FPGA, and fibre transport to a GPU/CPU host** — see
> [`radar-rx-frontend-edge-digitization.md`](radar-rx-frontend-edge-digitization.md).
> The band is now fixed at X-band 10.0–10.5 GHz (Option A is dead). Kept as a
> historical record of the platform decision and its rationale; do not treat
> the design below as current.

## System Overview
- 8 TX channels + 8 RX channels (bistatic phased array)
- Separate TX and RX antennas — no circulator or T/R switch needed
- All channels share a common LO (X-band path only)
- Array geometry is a signal processing concern, not an RF frontend concern

---

## Two Architecture Options (Decision Pending)

### Option A: Direct Sampling — S-band (~3.1–3.5 GHz)
No mixing. RF signal sampled directly by ZCU208 ADC via 2nd Nyquist zone.

**RX chain (×8):**
```
Antenna → LNA → BPF (S-band) → ZCU208 ADC
```
**TX chain (×8):**
```
ZCU208 DAC → BPF (S-band) → PA → Antenna
```
**No LO required.**

Advantages:
- Dramatically simpler frontend — no mixers, no LO, no LO distribution
- Fewer components = fewer failure modes during bring-up
- Phase coherence trivially maintained — all channels share same ADC clock
- S-band components are cheaper and more available than X-band

Disadvantages:
- Lower carrier frequency → larger antenna aperture for same angular resolution
- Less range resolution for a given fractional bandwidth
- 3.1–3.5 GHz is more congested than 10 GHz amateur band

---

### Option B: Single-Conversion — X-band (10.0–10.5 GHz)
Fixed LO downconverts X-band RF to IF, which ZCU208 samples directly.

**Frequency plan:**
- Target band: 10.0–10.5 GHz (ITU amateur allocation)
- LO: 8 GHz fixed (OCXO-based, ultra-low phase noise)
- IF centre: 10.25 − 8 = 2.25 GHz
- IF bandwidth: 500 MHz (1.75–2.25 GHz)
- IF sits comfortably in ZCU208 1st Nyquist zone

**RX chain (×8):**
```
Antenna → LNA → BPF (RF, 10.25 GHz) → Mixer → BPF (IF, 2.25 GHz) → ZCU208 ADC
```
**TX chain (×8):**
```
ZCU208 DAC → BPF (IF, 2.25 GHz) → Mixer → BPF (RF, 10.25 GHz) → PA → Antenna
```
**LO chain:**
```
Reference oscillator → 8 GHz LO source → LO amplifier → 16-way splitter → 16× mixers
```

Advantages:
- Higher carrier frequency → smaller antenna elements, better angular resolution
- 10 GHz amateur band is relatively clean and well-allocated
- X-band is the classic radar band with excellent target scattering properties

Disadvantages:
- Significant additional hardware: 16 mixers, LO source, splitter, RF filters
- Phase coherence requires careful LO distribution (matched cable lengths)
- Bring-up complexity much higher
- Higher component cost (~$5,000 more for Phase 1)

---

## Phase Coherence

### Direct Sampling (Option A)
Excellent by default. All channels clocked from same ZCU208 reference via MTS.
No inter-channel phase management needed beyond standard ZCU208 MTS setup.

### Single-Conversion (Option B)
**LO phase coherence is critical for phased array operation.**
- Fixed LO derived from same reference as ZCU208 CLK104 → inherently coherent
- Pulse-to-pulse phase stability determined by LO oscillator quality
- Static mixer LO-to-RF phase offset is fixed per channel — calibrate once
- LO distribution tree must have matched path lengths to all 16 mixers
- At 8 GHz, 1mm cable length difference ≈ 10° phase error
- Use corporate feed network (binary tree of power splitters) with
  equal-length SMA cables cut to matching physical length

---

## LO Distribution (Option B only)

**The 16-way splitting problem:**
- Passive 16-way split = ~12 dB intrinsic insertion loss + connector loss ≈ 13–15 dB total
- Each mixer needs approximately +7 to +13 dBm LO drive
- LO source must provide ~+25 dBm before splitter, OR use amplifier stage

**Recommended approach:**
- LO source at ~+13 dBm (Wenzel MXO-PLMX)
- Buffer amplifier to reach ~+25 dBm
- 1× 2-way splitter → 2× 8-way splitters (binary tree, better phase matching
  than single 16-way unit)
- All SMA cables from splitter outputs to mixer LO ports cut to equal length

---

## Add-On Cards Required
| Card | Option A | Option B Phase 1 | Option B Phase 2a (standalone PCB) | Option B Phase 2b (custom RFMC card) |
|---|---|---|---|---|
| XM655 | Yes | Yes | Yes | No — replaced by custom RFMC card |
| CLK104 | Yes | Yes | Yes | Yes |
| XM650 | No | No | No | No |

---

## Commercial Daughter Cards for X-band — Does Not Exist

Searched thoroughly. No commercial off-the-shelf RFMC 2.0 daughter card
exists that performs X-band (10 GHz) upconversion/downconversion for the ZCU208.

The only commercial RFMC RF frontend card is the **Otava DTRX2** (20–30 GHz, K/Ka band).
The **PFE system** from intermod.pro (pfe-wide, pfe-inlna) covers 10 MHz–10 GHz
but is a wideband balun/LNA card only — no frequency conversion.

For X-band, external hardware is required regardless of approach.

---

## Prototype Approach

### Phase 1 (both options)
Single TX + single RX channel. Modular bench assembly with SMA components
connected to ZCU208 via XM655. Validate chain end-to-end before committing
to any PCB design. See `components.md` for full BOM.

### Phase 2 Implementation Options (explored, not decided)

**Option 2a: Standalone RF frontend PCB**
Custom Rogers 4350B PCB carrying all 16 channels. Connects to ZCU208 via
XM655 SMA cables. Simplest to design independently of ZCU208 mechanical
constraints. Requires cable runs between boards.

**Option 2b: Custom RFMC daughter card** [explored — not designed]
A custom PCB that plugs directly into the ZCU208's RFMC 2.0 connectors,
replacing the XM655. Carries mixers, LO, filters, LNA, and PA on-board.
The RFMC connectors expose all 8 ADC and 8 DAC differential pairs, so a
single card could handle all 16 channels simultaneously.

Precedent: An ADI EngineerZone user documented exactly this approach for
a 24 GHz 8×8 phased array RFSoC system — designing a custom RFMC board
carrying ADI upconverter/downconverter chips directly mated to the RFSoC
eval board RFMC connectors.

Advantages over 2a: No SMA cable runs, cleaner integration, lower IF path loss.
Disadvantages: PCB must conform to RFMC 2.0 mechanical and electrical spec
(Samtec LPAM connector, 4mm mated height), adding constraint to layout.
Design complexity is similar to 2a.

Note: Option 2b is structurally similar to what Otava did for the DTRX2 at
20–30 GHz — but for 10 GHz. No one appears to have commercialised this yet.
