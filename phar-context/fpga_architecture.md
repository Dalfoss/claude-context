# FPGA Front-End: DDC, Decimation, and Resource Budget

Multi-band digital downconversion architecture for an 8-ADC, 7-subband-per-ADC
receiver, targeting an AMD Artix-7 XC7A200T. This document captures the
front-end DSP architecture and the resource budget behind it.

---

## 1. System context

- **8 ADCs per board**, sharing one on-board sample clock (single synchronous domain).
- **7 subbands per ADC** → **56 DDCs per board** total.
- ADC: 14-bit, 125 MSPS (LTC2145-14 baseline).
- Bands come from **FDMA** — 7 simultaneous transmit carriers. The 7 subbands are
  therefore 7 *frequency-diverse looks at the same range slice*, not 7 range gates.
- Each subband is mixed to baseband and output as **complex IQ**.
- Output per band ~1 MHz wide (the foreseeable-future value; 2.5 MHz was an
  aspirational "future-proof" figure that does **not** fit the link budget — see
  the clock/link notes and §6).

The choice of an independent DDC per band (rather than a polyphase channelizer)
is driven by the bands being **sparse in frequency** and not on a clean uniform
bin grid. A channelizer would compute ~25 channels to keep 7; independent DDCs
cost roughly linearly and give arbitrary center/width freedom.

---

## 2. Per-band DDC chain

```
  ADC (real, 125 MSPS)
        │
        ▼
  Complex NCO mixer        ← only full-rate stage (2 real mults/band)
        │
        ▼
  CIC decimator            ← multiplierless; this is the LUT cost driver
        │
        ▼
  CIC droop-comp FIR       ← inverts CIC passband sinc rolloff
        │
        ▼
  Halfband / cleanup FIR   ← final shaping + decimate
        │
        ▼
  Complex IQ baseband out
```

Because every band is the **same width and same decimation**, the chain is
designed **once** and replicated/folded; the only per-band difference is the
**NCO phase increment** (which carrier it pulls to DC). Keeping the chains
bit-identical is not just cheaper — it guarantees matched group delay across
bands, which is a hard requirement for the coherent cross-carrier (synthetic
wideband) combine downstream.

### Decimation factor

- For a ÷32 → 3.9 MHz complex case (the early 2.5 MHz-band sizing), a clean split
  is CIC ÷8 → comp FIR → halfband ÷2 → halfband ÷2. Halfband-friendly.
- For the current **~1 MHz band**, decimation is around **÷125** (125 MSPS → 1 MSPS
  complex). Note 125 = 5³ is **not** halfband-friendly, so this leans harder on the
  CIC (R large) plus a comp FIR, with fewer/no halfband stages. The 800 kHz output
  variant (see §6) is the chosen tuning lever and sits at a similar ratio.
- Avoid critically-sampled outputs (e.g. ÷50 → 2.5 MHz exactly): leaves no
  transition band. Keep ~20–25% margin for the filter skirt.

### Things baked into the chain that are actually requirements

- **CIC bit growth**: DC gain is (R·M)^N. Internal word growth must be managed by
  **Hogenauer pruning** stage-by-stage, not carried at full width. This is the
  single biggest LUT lever in the whole design.
- **CIC droop**: causes a passband amplitude taper; the comp FIR flattens it.
- **Stopband attenuation = inter-carrier isolation**: because the bands are FDMA
  carriers, a loose DDC stopband lets a strong return on carrier A leak into
  carrier B's band. Target on the order of **70–80 dB**; this is an architectural
  spec, not a tuning detail. The CIC alone won't deliver it — the comp/cleanup FIR
  does.
- **NCO spurs**: size the DDS phase-to-amplitude path so phase-truncation spurs sit
  below ~ **-85 dBc** (≈ 6 dB SFDR per phase bit into the LUT, plus dither/Taylor
  correction as needed). With 14-bit data you want spurs under the noise floor.
- **Complex (not real) NCO** is mandatory: it preserves the sign of the beat
  offset, which is what lets the host resolve range unambiguously within a gate.

---

## 3. Replicate vs. time-multiplex (TDM)

The 8-ADC scale-up (56 DDCs) is what forces discipline. Rule:

> **Replicate only the full-rate stage. Time-multiplex everything downstream.**

- **Full-rate mixers** are the only stage at 125 MSPS. 56 bands × 2 real mults.
  Run the DSP fabric at a multiple of the ADC clock (e.g. **250 MHz = 2×** or
  **375 MHz = 3×**) and fold 2–3 bands onto each physical DSP. DSP48E1 Fmax is
  comfortably north of 400 MHz when fully pipelined on this part.
- **Post-CIC stages** run at ≤15.6 MHz (after a ÷8) so they fold *hard*. One
  physical comp-FIR engine and one halfband engine, TDM'd at the fabric clock,
  serve many bands each.
- **The CIC must be per-band** — each band is mixed to a different DC before
  decimation, so the integrators can't be shared (though the CIC Compiler can TDM
  the *adders* if the feedback path closes timing on the wide accumulate).

Vendor IP does the folding: the **AMD FIR Compiler "number of channels"** parameter
generates the shared datapath and slot scheduling automatically. Feed it the
channel count (56, or 7×8) for the TDM back-end.

---

## 4. Resource budget — Artix-7 XC7A200T

Part resources (approx): **~740 DSP48E1**, **~134k LUT**, **~269k FF**, **~365 BRAM**.

### DSP

| Stage | DSP (approx) |
|---|---|
| Full-rate mixers (folded 2–3×) | ~40 |
| Post-CIC comp-FIR + halfbands (TDM) | ~60–100 |
| CICs | 0 (multiplierless) |
| **Total** | **~150 of 740** |

DSP is **comfortable** — roughly a fifth of the part.

### LUT (the gating resource)

The CIC bank dominates. Worked for N=4, R=8, M=1, 16-bit mixer product, 56 complex
bands (= 112 real CICs):

- Bit growth N·log₂(RM) = 12 bits → 28-bit full output.
- Un-pruned: ~206 LUT/real CIC (integrators ~94, combs ~112).
- Hogenauer-pruned: ~150 LUT/real CIC.
- **CIC bank**: 112 × ~150–206 ≈ **17–23k LUT** replicated; **~12–15k** if TDM-folded.

Everything else (rough adders):

| Block | LUT (approx) |
|---|---|
| CIC bank | ~12–23k |
| NCO phase accumulators + SFDR/correction logic | ~3–5k |
| Folded comp-FIR / halfband control, coeff/data RAM addressing, scheduling | ~5–10k |
| TDM control, sample alignment, AXI-Stream plumbing | ~3–5k |
| Output packing / FIFOs | ~2–4k |
| **Total** | **~30–45k of 134k (≈ 25–33%)** |

FF count runs comparable to or a bit above LUTs (~25–40k of 269k) — never the
constraint.

### What swings the LUT number, by leverage

1. **Mixer-output width into the CIC** — 28-bit growth scales ~linearly; 16→18 bits
   adds ~12% to the CIC bank.
2. **Prune depth** — full vs Hogenauer ≈ 25%.
3. **Fold vs replicate** on the integrators.
4. **N=4 vs N=5** — a 5th stage adds another integrator+comb per rail across all
   112 rails, ≈ +25% to the bank. Use N=4 unless alias rejection at band edges
   demands N=5.

The decimation *split* barely moves LUTs (combs and downstream run slow).

**Net:** the design is **DSP-comfortable and LUT-comfortable**, sitting around a
quarter to a third of the part, with headroom for N=5 or wider words if rejection
needs it. Optimization effort belongs on **CIC pruning and stage count**, not on
multiplier sharing.

---

## 5. What stays on the FPGA vs. host

Beamforming is across **multiple receiver boards**, so it cannot happen on any one
FPGA — the spatial combine must occur where all element streams meet (the host /
GPU). This collapses the FPGA's job to a clean one:

> **FPGA: DDC + sync capture + timestamp + packetize.
> Host: range / Doppler / beamforming.**

Consequences:
- No spatial FFT, no irregular-array matmul, no Doppler corner-turn BRAM on the
  fabric — all of that moves off-chip.
- An **on-chip range FFT is viable but not currently worthwhile**: an FFT preserves
  data rate (N samples → N bins), so it only cuts the link if you *range-gate* by
  discarding bins. Your decimation already *is* the range gate (you only ever pull
  the wanted slice of beat spectrum), so the FFT earns nothing on bandwidth and
  would freeze sweep length/window into the fabric and risk eating ENOB if
  fixed-point growth isn't carefully managed. Keep it on the host until you're
  range-gating to cut the link or host compute becomes the wall.

---

## 6. Link budget (front-end output rate)

Per board: `rate = (ADCs × bands) × f_out × bytes_per_IQ × 8`.

The trap is that all factors multiply: 8×7 = 56 streams, ×2 for I+Q, ×2 bytes each
→ a "1 MHz band" is really 1.25 MHz complex × 4 B = 5 MB/s, and ×56 ≈ 280 MB/s.

At 16-bit I + 16-bit Q (4 B/sample), per board:

| f_out | Per board | Fits 2.5GbE? |
|---|---|---|
| 1.25 MHz (÷100, margined) | 2.24 Gbps | yes, but ~93–95% utilization — too tight |
| 1.0 MHz (critical) | 1.79 Gbps | yes |
| **0.8 MHz (chosen lever)** | **~1.43 Gbps** | **yes, comfortable** |

2.5GBASE-X gives ~2.5 Gbps after line coding; after Ethernet/UDP/IP framing
you net ~2.3 Gbps (≈2.4 with jumbo frames). So the 2.24 Gbps row is real but
unsafe. **Bits are not shaved** (full dynamic range is needed — see §7), so the
chosen lever is **band width**: dropping to **800 kHz** sub-channels gives clean
2.5GbE headroom. The 2.5 MHz "future-proof" case is a ~7 Gbps monster that needs
10G and is explicitly out of scope.

Host ingest scales the same way: per-board Gbps ÷ ~0.75 Gbps/core (benchmarked) →
~2–3 cores per board just to ingest+process.

---

## 7. Processing gain from decimation (ENOB)

Filtering to an 800 kHz band and decimating discards out-of-band quantization
noise:

```
SNR gain = 10·log10( (fs/2) / B ) = 10·log10( 62.5 / 0.8 ) ≈ 18.9 dB ≈ 3.1 ENOB
```

So a ~11.5-ENOB ADC pushes toward ~14–14.5 effective in-band bits. Caveats:

- Recovers **out-of-band** noise only (quantization, broadband front-end). Noise
  already inside the 800 kHz (thermal after band-limiting) is **not** improved.
- Requires quantization to be **white** (true for busy/dithered RF; a lone low-level
  tone won't give the full gain).
- **You must carry the bits** through the CIC/FIR chain and out the wire. The
  16-bit IQ output is sized exactly to hold ~3 bits of processing gain on top of
  the ADC. **This is why bits are not shaved for the link** — dropping to 12-bit
  transport would throw the entire decimation gain away. Link budget and
  dynamic-range decision are consistent: the width being protected *is* the
  processing gain being paid for with ÷125 of decimation.

---

## 8. ADC interface (capture)

LTC2145-14 is **clock-direct**: it samples on the supplied encode-clock edge and
emits DDR LVDS with a data output clock (CLKOUT), even/odd bits multiplexed per
pin. There is no internal sample-rate divider, hence **no SYNC pin needed** (see
the clock-distribution doc for why this is fine and how it changes for smarter
ADCs).

FPGA capture: deserialize DDR LVDS with **ISERDES**, align the data eye with
**IDELAYE2** (+ IDELAYCTRL). The ADC's programmable CLKOUT phase delay gives an
extra coarse eye-centering knob. This capture alignment is **per-board and local**
— it does not affect cross-board coherence, because once data is correctly latched
into each board's 125 MHz domain everything is back on the common timebase.

Artix-7 LVDS capability (ISERDES + IDELAY, rated ~1.25 Gb/s per pair depending on
speed grade) comfortably covers a 14-bit/125 MSPS DDR-LVDS converter — more capable
here than the originally-considered ECP5, with margin at 125 MSPS.
