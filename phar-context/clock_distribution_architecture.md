# Clock Distribution & Synchronization Architecture

Timing backbone for a phase-coherent radar receiver: one standalone synthesizer /
fan-out board, 5 digitizer boards (8 ADCs each), across nearby chassis (≤ a couple
of metres). The host performs beamforming, so **everything depends on the boards
being phase-coherent** — coherence is the property this whole architecture exists
to guarantee.

---

## 1. The core principle: same *clock*, not same *frequency*

The most common and most fatal clocking mistake is "set every board to 125 MHz and
they'll follow each other." They won't.

- Two oscillators nominally at 125 MHz are never identical — each has a ppm error
  and drifts with temperature/age. A 20 Hz difference means their phase relationship
  **slides continuously** and slips full cycles forever.
- Every coherence claim in this system (NCO phase across bands and boards, the
  synthetic-wideband combine, beamforming) is a **phase** property. Independent
  oscillators at the "same frequency" drift without bound and destroy it.

The fix is a **synchronous hierarchy**: exactly **one** source of time, with every
clock on every board *derived from it*. No second opinion about what a second is →
no relative drift. (Synchronous = everyone derives from one reference. The broken
"same frequency" model is *plesiochronous* = nominally equal but independent.)

---

## 2. The hierarchy, master → sample

```
  Master reference (e.g. 10 MHz OCXO/TCXO, on synth/fan-out board)
        │   low-frequency reference survives cable runs well
        ▼   phase-matched fan-out (equal-length lines, low-skew buffer)
  ┌───────────────── per digitizer board ─────────────────┐
  │  Jitter-cleaning PLL / clock IC                        │
  │   • locks local VCO to the reference  → synchronous    │
  │   • narrow loop BW filters cable jitter → clean clock  │
  │   • synthesizes 125 MHz ADC clock + FPGA clocks        │
  │   • SYSREF-aligned dividers → deterministic phase      │
  │        │                                               │
  │        ▼                                               │
  │  ADCs sample on aligned 125 MHz edges                  │
  └────────────────────────────────────────────────────────┘
```

Each layer removes one specific disagreement:

| Layer | Removes |
|---|---|
| Shared reference | frequency drift between boards |
| Jitter-cleaning PLL | cable-induced jitter; establishes lock |
| SYSREF / divider alignment | divider power-up phase ambiguity |
| Ramp-start strobe (§5) | ramp-phase ambiguity in the data stream |

Miss any layer and the corresponding error reappears downstream. "Same frequency"
handles **none** of these; a shared reference handles only the first.

### Why distribute a *low* reference
You fan out ~10 MHz, not 125 MHz: low frequency suffers less cable loss and skew
sensitivity, and each board multiplies up locally. Distribution must be
**phase-matched** — equal-length lines and a clock-distribution-grade fan-out
buffer (low additive jitter, matched output skew).

### What the jitter-cleaner IC actually does
A jitter-cleaning PLL (TI LMK0482x / LMK04828, ADI AD9528 / HMC7044 class) does
three jobs at once:
1. **Locks** its local VCO to the incoming reference — this is what makes the board
   synchronous (frequency *and* phase track the master forever).
2. **Cleans jitter** — the PLL loop bandwidth "hands off" between reference stability
   (inside the loop BW, low offsets) and local-VCO spectral purity (outside it, high
   offsets). Cable-induced high-frequency jitter on the reference is filtered out and
   replaced by the clean local VCO. **This matters directly for the 14-bit ADC**:
   clock jitter → sampling-instant uncertainty → voltage error → lost ENOB.
3. **Generates** the actual 125 MHz ADC clock and FPGA clocks with **deterministic
   phase** (via SYSREF-aligned dividers).

### SYSREF — turning "synchronous" into "phase-aligned"
Even with every board locked to the same reference, when each PLL *divides down* to
125 MHz the divider can power up in any of several phase states. Two boards can both
be perfect, locked 125 MHz yet sit a cycle or two apart. SYSREF is a slower signal
(from the same clock chip) that **resets every divider to a known phase on a common
edge**, edge-aligning all boards' 125 MHz. This is the JESD204B mechanism, but the
*concept* is general (see §4). SYSREF and the ramp-start strobe must be mutually
consistent — both define "the same instant" across boards.

---

## 3. Cross-chassis specifics (≤ a couple of metres)

This distance is still firmly "matched coax + lock locally" territory — no need for
optical / White Rabbit / PTP-style distance-tolerant schemes (those come in at much
larger separations).

Recommended topology: **distribute reference + SYSREF (+ strobes) from the master**,
re-time SYSREF locally on each board's clock IC. One source of truth for both
frequency and alignment instant; each board's chip just re-clocks the incoming
SYSREF to its local clock to kill cable jitter.

- The master/fan-out board outputs on the harness: **reference clock**, **SYSREF/SYNC
  pulse**, and the **ramp strobes** (§5) — all travelling together so they share
  propagation characteristics.
- **Cable skew** between boards is a real but small phase term. Handle it with
  matched cable lengths and the clock IC's per-output fine delay adjustment.
- Anything **static and stable** (fixed cable-length differences) folds into the
  **per-board phase calibration** you already need for the synthetic-wideband
  combine. The enemy is **drift** (temperature-dependent group delay, clock phase
  noise), **not** fixed offset. Calibration *cadence* — how often per-carrier /
  per-board phases are re-measured — is therefore a real system parameter.

The synth board needs the **same discipline** as a digitizer: its chirp generator
must be locked to the master reference, not free-running, or the chirp drifts
relative to sampling and the per-carrier phase anchor slides. Since the master
typically lives on the synth board, this is natural — the chirp DDS/DAC clock is
derived from the same reference it fans out.

---

## 4. Why SYSREF is NOT needed for the chosen ADC (LTC2145-14)

**SYSREF/SYNC exists to resolve phase ambiguity created *inside* the ADC.** That
ambiguity only arises if the part has internal phase state — primarily an **internal
clock divider** (fast clock in, divided down internally) or internal NCO/decimation
state. If the conversion happens **directly on the supplied clock edge** with no
internal division, there is no internal phase state, and **nothing for a SYNC pin to
do.**

The **LTC2145-14 is clock-direct**: sample on the encode edge, source-synchronous
DDR LVDS out. So:

- It has **no SYNC pin and needs none.**
- Cross-board alignment lives entirely **one component upstream** — in the **clock
  IC's** SYSREF-aligned 125 MHz outputs feeding each ADC's encode input. If every
  board's encode edge is aligned, every ADC converts coincidentally *by
  construction*; the ADC is a passive beneficiary and is told nothing.
- The LVDS **data capture** on the FPGA (ISERDES + IDELAY eye alignment) is a
  separate, **per-board, local** concern. It is "a phase thing" but **downstream of
  the aligned sample instant**, so it does not affect array coherence — once data is
  latched into each board's 125 MHz domain, all boards are back on the common
  timebase.

**Bottom line: SYSREF capability is required on the clock IC, not on this ADC.**

---

## 5. The ramp-start / chirp strobe (framing + phase anchor)

The ADC does **not** need a chirp-start signal — sampling is stationary w.r.t. the
ramp, and a beat tone is a range profile regardless of when sampling began. But the
**processing frame** does need ramp timing:

- **Doppler** is sweep-to-sweep phase evolution; sweeps must be cut at a consistent
  ramp point (a constant cut offset cancels in the slow-time difference — so Doppler
  alone is forgiving).
- **Synthetic-wideband (FDMA carrier) combine** reads the **per-carrier phase**
  directly. The beat phase at ramp-start, φ₀,k, encodes range in the carrier-to-
  carrier progression. All 7 carriers must be referenced to **the same instant in
  the ramp** (ramp-start is the natural, physically defined choice). An unknown
  offset τ from ramp-start adds 2π·f_beat,k·τ, which differs per carrier → a
  range error/sidelobe. This does **not** cancel the way Doppler's common offset
  does.

So the requirement is: **anchor the per-carrier NCO phases to a consistent ramp
phase.** Two ways:
1. **Explicit ramp-start strobe** (robust, recommended): a digital pulse, one per
   ramp, edge-aligned to ramp-start, from the synthesizer. Immune to ramp-period /
   sample-clock ratio messiness.
2. **Calibrate τ out** (viable only if the NCO reset is at a *stable, repeatable*
   offset from ramp-start — i.e. the ramp period is an exact, repeatable integer of
   sample clocks). If the ratio isn't clean, τ walks and no static calibration saves
   you. This fragility is exactly why the explicit strobe is preferred.

Since a shared clock harness already exists between synth and digitizers, the strobe
is one more line on a connector already being run — cheap insurance.

### Source the strobe from the chirp *generator*, not the chirp *signal*
Do **not** recover ramp-start by detecting the flyback/apex in the analog chirp
(noisy, scene-dependent). The standalone synthesizer's ramp generator (DDS / ramp
DAC) has an **explicit digital ramp-start event** — tap that. It is exact,
jitter-free, and **inherently phase-locked to the transmitted ramp**, which is
precisely the φ₀ anchor needed. Route this digital event from the synth/fan-out
board to the digitizers on the shared harness (the synth box is already the central
fan-out point, so this is one added digital output, not a new generator).

### What the strobe hooks into on the FPGA
Bring it in as **LVDS → 2–3 FF synchronizer → edge detect** in the 125 MHz domain,
producing a one-clock `ramp_start` tick. That tick drives three consumers:
1. **NCO phase-accumulator reset/reload** — all 7 NCOs per ADC (and across all 8 ADCs
   × 5 boards, since the strobe is common) reload to a known phase on the same edge.
   This is the array-wide phase origin.
2. **Sample-counter timestamp** — latch a free-running sample count into the outgoing
   stream as a frame marker so the host can slice sweeps and align across boards.
   Latch-and-continue (monotonic) also lets the host detect dropped sweeps.
3. **Internal sweep state** — any per-sweep bookkeeping / where an on-chip range FFT
   would hook in later.

**Critical:** the strobe→NCO-reset **latency must be identical across boards** (same
bitstream, fixed synchronizer depth → true by construction; just don't let that path
be retimed per board). Any per-board difference is a per-board phase offset
(calibratable if stable, but avoidable).

### Strobe cadence
**Strobe every ramp** is most robust (re-anchors each sweep; a missed sync self-heals
next ramp) — use it unless your host model wants the NCO to run *freely* across many
sweeps with only an initial anchor, in which case strobe once at start-of-acquisition
and timestamp thereafter. Both are trivially supported by the same hardware.

### Triangle-chirp note
A triangle gives up- and down-ramp beats (f↑ = f_R + f_D, f↓ = f_R − f_D) that
**separate range and Doppler in one sweep pair** — a feature, not something to
discard. Harvest a strobe at **every turning point** (with a direction bit / two
lines), again from the synthesizer's known reversals, not from the beat signal. It
does triple duty: NCO anchor, timestamp, and **up/down half-sweep labelling** (so
the host pairs the right halves). Caveat: each turning point is a slope
discontinuity → a filter transient; with heavy decimation and short sweeps
(~50 samples/half-sweep at 0.1 ms, 1 MSPS) the ~5–10 settling samples are a
non-trivial 10–20%. The strobe tells the host exactly which samples to trim. Check
sweep timing against filter group delay before committing to triangle + this
decimation.

---

## 6. Full chain summary

```
master 10 MHz ref (synth board)
   → phase-matched fan-out over harness (ref + SYSREF + ramp strobes)
   → each board's jitter-cleaning PLL: lock, clean, ×→125 MHz
   → SYSREF aligns every board's dividers to a common phase
   → ADCs (clock-direct, no SYNC) sample on aligned edges
   → ramp-start/turning-point strobes land in the common 125 MHz domain
   → FPGA: reset NCO phase, timestamp stream, label up/down half-sweeps
   → host: phase-coherent, frame-aligned, up/down-labelled IQ across all 5 boards
   → host forms range / Doppler / beams
```

Removing each disagreement in turn: shared reference → frequency drift; PLL →
jitter + lock; SYSREF → divider phase; strobe → ramp phase.

---

## 7. How this ties to a future ADC choice (e.g. ADC3668/3669)

The "no SYSREF needed at the ADC" conclusion is **specific to clock-direct
converters** like the LTC2145. It flips for smarter parts, and the rule for
evaluating any future ADC is:

> **Does the part have internal phase state (clock divider, on-chip NCO/DDC,
> interleaving)? If yes, it needs a SYNC/SYSREF input and you must align it. If it's
> clock-direct, all alignment stays in the clock IC.**

The **ADC3668/3669** (16-bit, 250/500 MSPS) illustrates both sides:

- It has an **on-chip quad-band DDC** with **48-bit phase-coherent NCOs** and
  decimation /2…/32768, and the datasheet block diagram shows **SYSREF**,
  **TIMESTAMP**, and **NCO SWITCH** clustered around the DDC. SYSREF exists here
  precisely **to align those internal NCOs across devices** — i.e. it is needed
  *because* the chip has internal NCO/decimation state, the converse of the LTC2145.
- **On-chip DDC path is ruled out for this system anyway**: the DDC is **quad-band
  (4 subbands), and the requirement is ≥7.** It also surrenders filter/stopband
  control (the 70–80 dB inter-carrier isolation spec would become "whatever TI's DDC
  delivers" rather than a hand-designed FIR).
- **In DDC-bypass mode**, the ADC3668 is functionally an LTC2145 with more bits,
  more sample-rate headroom, and lower aperture jitter (75 fs): clock-direct, raw
  16-bit DDR LVDS out, all 7+ DDCs done in the FPGA. Band count is then set by FPGA
  DDC instantiation, not the chip — so the ≥7 requirement is satisfied. In bypass,
  the chip's SYSREF/NCO machinery is for features you aren't using; coherence again
  comes from the aligned sample clock + FPGA NCOs, exactly as with the LTC2145.
  Its **TIMESTAMP** block is a potential *hardware* path to mark ramp-start at the
  converter — lining up neatly with the strobe plan in §5 if ever adopted.

**Practical selection rule for the clock IC regardless of ADC:** choose a
"clock generator / jitter cleaner with **JESD204B SYSREF support**" (LMK04828 /
HMC7044 / AD9528 class) even though JESD isn't used — that phrase is the marketing
label for "has SYSREF-capable aligned dividers + fine per-output delay." A plain
jitter cleaner without aligned dividers will lock and clean but won't give
deterministic cross-board phase. The **reference oscillator** (OCXO) does not
generate SYSREF; the **clock IC** does. This way the timing backbone is unchanged if
a future converter (clock-direct *or* internal-divider) is dropped in — only whether
you wire the ADC's SYNC/SYSREF input changes.
