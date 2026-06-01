# Radar Theory

## ADC Resolution and Dynamic Range

### Why Resolution Matters for Radar
Dynamic range determines ability to simultaneously detect strong and weak
targets — e.g. a nearby building and a distant aircraft in the same beam.

Each additional ADC bit ≈ 6 dB of dynamic range (SFDR).
ZCU208 (14-bit) vs AD9088 (12-bit) = **12 dB difference** — significant.

### ZCU208 (14-bit ADC)
- Theoretical SNR: ~86 dB
- Practical SFDR: 70–80 dBc (RFSoC Gen 3, input-frequency dependent)

### AD9088 (12-bit ADC, rejected alternative)
- Theoretical SNR: ~74 dB
- Practical SFDR: 60–70 dBc (preliminary silicon, not fully characterised)

### ENOB — Effective Number of Bits
Real-world ADC performance metric accounting for noise and distortion.
A "14-bit" ADC at 5 GSPS typically achieves 11–12 ENOB in practice.
ENOB is always lower than nominal resolution. Check datasheet SNR plots
vs input frequency — performance degrades at higher input frequencies.

---

## The 14-bit Wall at High Sample Rates
**No discrete ADC from any vendor achieves 14-bit resolution at ≥8 GSPS.**
This is a physics constraint, not a vendor limitation:
- Thermal noise sets a floor at room temperature
- Aperture jitter worsens with speed
- PCB interface losses degrade ENOB on discrete parts

The ZCU208 achieves 14-bit at 5 GSPS only because:
- ADC and digital fabric share the same die (monolithic)
- No PCB serialisation interface between ADC and logic
- Shared ultra-low-jitter clock tree on-chip
- TSMC 16nm FinFET+ process enables dense, low-noise integration

ADI's discrete ADC lineup (for reference):
| Part | Resolution | Max GSPS |
|---|---|---|
| AD9689 | 14-bit | 2.6 |
| AD9208 | 14-bit | 3.0 |
| AD9081 | 12-bit | 4.0 |
| AD9082 | 12-bit | 6.0 |
| AD9088 | 12-bit | 8.0 |

---

## Dynamic Range Mitigation Techniques
The 12 dB gap between 14-bit and 12-bit ADCs can be partially recovered:

| Technique | Typical Gain |
|---|---|
| Oversampling + decimation (8×) | ~4.5 dB |
| Coherent pulse integration (N pulses) | 10×log10(N) dB |
| Pulse compression (matched filter) | 10×log10(BW×T) dB |
| Front-end LNA + filtering | Reduces required ADC dynamic range |

None fully recover 12 dB, but system-level dynamic range of 80–100 dB
is achievable with 12-bit ADCs through good system design.

---

## Coherent Processing
Radar requires pulse-to-pulse phase coherence for:
- Doppler processing (moving target indication)
- SAR imaging
- Phased array beamforming

### Phase Coherence in This System
**Option A (direct sampling):** Inherent — all channels share same ADC clock.
**Option B (X-band mixing):** Requires fixed LO derived from same reference
as ADC clock. Every pulse sees identical LO phase. Static mixer phase offset
calibrated once at startup and removed digitally.

---

## Processing Gain Summary
For a radar with bandwidth B, pulse width T, N integrated pulses:
- Range resolution: c/(2B)
- Doppler resolution: 1/(N×PRI)
- SNR gain from pulse compression: 10×log10(B×T) dB
- SNR gain from coherent integration: 10×log10(N) dB
