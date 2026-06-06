# FDM Processing in FMCW MIMO Radar

## Standard Approach: Coherent Sub-Band Combining

Each TX is assigned a contiguous, non-overlapping sub-band of the total chirp bandwidth. All TXs transmit simultaneously, and all RX channels receive the superposition of all TX returns.

**Separation** is done by dechirping the composite RX signal against each TX's reference chirp in turn. The matched TX lands near DC; unmatched TXs land at integer multiples of Δf, allowing isolation via bandpass filtering (or equivalently, bin extraction after FFT).

**Range resolution recovery** is achieved by coherently stitching the sub-bands across all TX-RX pairs in post-processing. Since the sub-bands are contiguous and phase-coherent (requires a common clock/reference — in the settled architecture, the shared OCXO master clock plus startup-tone phase calibration; see `rf-frontend/radar-rx-frontend-edge-digitization.md` §2.4), the combined bandwidth equals the full system bandwidth, recovering the full range resolution.

- Per-channel resolution: `c / (2 × B_sub)` where `B_sub = B_total / N_TX`
- After combining: `c / (2 × B_total)`

**Requirements:**
- Contiguous sub-bands
- Phase coherence across all channels (shared LO / NCO-based frequency plan)
- Exact, stable sub-band spacing

---

## Advanced Approach: Sub-Nyquist / Compressed Sensing Recovery (Cohen/Eldar)

Developed by the Weizmann Institute on an 8TX/10RX X-band hardware prototype. TXs transmit in true non-overlapping sub-bands, but sub-band placement can be irregular or randomised. All TX-RX pairs are processed **jointly** using compressed sensing or sub-Nyquist recovery algorithms to reconstruct the full-bandwidth range profile.

The irregular array geometry (random antenna placement) is a deliberate design choice — it avoids spatial aliasing and reduces range-azimuth coupling that arises in regular FDM arrays.

**Advantages over standard combining:**
- Works without strictly contiguous sub-bands
- Can decouple range and azimuth resolution more cleanly
- Exploits sparsity of the scene for recovery

**Drawbacks:**
- Significantly more complex signal processing pipeline
- Computationally expensive — challenging for real-time operation
- Requires careful antenna placement design
- Recovery algorithms must be built from scratch (no off-the-shelf implementation)

---

## Summary Comparison

| | Standard Coherent Combining | Cohen/Eldar Sub-Nyquist |
|---|---|---|
| Sub-band layout | Contiguous, regular | Arbitrary / irregular |
| Processing complexity | Moderate | High |
| Real-time feasibility | Yes (FPGA-friendly) | Challenging |
| Array geometry constraint | Flexible | Random/irregular preferred |
| Full bandwidth recovery | Yes | Yes |
| Off-the-shelf support | Partial | No |
