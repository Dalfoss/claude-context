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

## Alternative Band: S-band Amateur Allocation (~3.4 GHz)

### Allocation
- ITU amateur: 3.400–3.410 GHz (ITU Region 1) or 3.300–3.500 GHz (varies)
- Check national allocation — coverage varies significantly by country

### Why Consider This
- Falls within ZCU208 direct sampling range (2nd Nyquist zone)
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

## Frequency Planning (X-band Option)
- Centre: 10.25 GHz
- LO: 8 GHz (fixed)
- IF centre: 2.25 GHz
- IF bandwidth: 500 MHz (1.75–2.25 GHz to avoid DC)
- Image frequency: 10.25 + 2.25 = 12.5 GHz (must be rejected by RF BPF)
