# Spectrum and Regulatory

## Primary Target Band: 10 GHz Amateur Allocation

### Allocation
- ITU amateur radio: **10.000–10.500 GHz**
- Amateur satellite: 10.450–10.500 GHz (subset)
- Known as the "3-centimetre band" in amateur radio

### Why This Band
- Only freely available X-band allocation for experimental transmission
- 500 MHz of bandwidth — sufficient for good range resolution
- Amateur radar community actively uses this band
- Relatively clean (not congested like 2.4 GHz or 5.8 GHz)

### License Required
An amateur radio license is required to transmit in this band.
The user will obtain this. Exam preparation is handled separately.

#### By Country
| Country | License Class Needed | Notes |
|---|---|---|
| USA (FCC) | Technician or higher | Technician covers 10 GHz. All classes permitted. Up to 100W. |
| UK (Ofcom/RSGB) | Full license | Foundation and Intermediate do not cover 10 GHz |
| EU | Typically full/advanced | Varies by national regulator |

### Transmission Notes
- Amateur rules require periodic station identification (callsign)
- Pulse radar waveforms are permitted as experimental technical operation
- Confirm specific mode and power rules with national regulator before transmitting

---

## Alternative Band: S-band Amateur Allocation (~3.4 GHz) *(no longer pursued)*

> The band is now settled on X-band 10.0–10.5 GHz. S-band was attractive only
> because it fell in the ZCU208 direct-sampling range — that platform and the
> direct-RF-sampling approach are abandoned, so this section is historical.

### Allocation
- ITU amateur: 3.400–3.410 GHz (ITU Region 1) or 3.300–3.500 GHz (varies)
- Check national allocation — coverage varies significantly by country

### Why It Was Considered (no longer applies)
- Fell within ZCU208 direct sampling range (2nd Nyquist zone)
- No downconversion hardware required
- Components cheaper and more available than X-band
- S-band propagation better in rain than X-band

### Disadvantages vs 10 GHz
- Larger antenna elements for same angular resolution (λ ∝ 1/f)
- Less range resolution for same fractional bandwidth
- More congested spectrum environment in some regions

---

## ISM Bands (No License Required)

### 2.4 GHz (2.400–2.4835 GHz)
- Unlicensed worldwide
- Extremely congested — WiFi, Bluetooth, ZigBee, microwave ovens
- Suitable for indoor bench testing only
- Not recommended for field radar testing

### 5.8 GHz (5.725–5.875 GHz)
- Unlicensed in most countries
- Just outside ZCU208 1st Nyquist zone (2.5 GHz)
- Would require 3rd Nyquist zone sampling — marginal
- Not recommended

---

## Frequency Planning (current — dechirp at the antenna)
- Centre: 10.25 GHz; band 10.0–10.5 GHz
- **LO = the reference chirp itself** (10.0–10.5 GHz), driving the dechirp mixer
  per channel — there is no fixed 8 GHz LO and no 2.25 GHz IF.
- ADC sees the **beat signal at baseband** (DC–~100 MHz, set by the FDMA plan),
  not an IF.
- Image rejection is handled by the post-LNA BPF before the mixer plus the
  digital Hilbert transform; the old 12.5 GHz image-frequency concern does not
  arise. See `rf-frontend/radar-rx-frontend-edge-digitization.md` §3, §4.3.

> *(Superseded plan: fixed 8 GHz LO → 2.25 GHz IF, image at 12.5 GHz. This
> belonged to the ZCU208 downconvert-to-IF architecture and is no longer used.)*
