# Sampling Theory — Critical Distinctions

## [PERSIST] DDC Architecture Across Platforms

The real-ADC → digital-IQ distinction holds across all platforms considered,
but the location and configurability of the DDC differs meaningfully.

### RFSoC (ZCU208) and Versal RF Series
- Real ADC at hardware level
- DDC is **hardened IP in the FPGA silicon** — not soft logic, not in the ADC chip
- IQ produced inside the FPGA fabric after sampling
- Configured via PYNQ Python API at runtime (NCO freq, decimation, gain)
  or via Vivado RF Data Converter IP for structural changes
- Versal RF Series uses the same architecture — the [PERSIST] distinction
  applies identically to Versal

### ADI MxFE (AD9081, AD9082, AD9088)
- Real ADC at the converter core level — same fundamental physics
- DDC is **on-chip in the ADC device itself**, before the JESD204C output
- By the time data leaves the chip over JESD, it is already IQ
- FPGA receives IQ directly — no DDC needed in FPGA fabric
- Configured via SPI register writes using ADI software stack:
  ACE (desktop), no-OS (embedded C), or Linux IIO drivers
- Changing decimation = SPI register write, not a bitstream rebuild
- Trade-off: simpler FPGA design, less flexibility to change DDC strategy
  post-design without reconfiguring the ADC chip

### Comparison Table
| | RFSoC / Versal | MxFE |
|---|---|---|
| ADC core | Real samples | Real samples |
| DDC location | Hardened IP in FPGA silicon | Hardened in ADC chip |
| JESD output data | Raw real samples | Already IQ |
| DDC configuration | PYNQ API or Vivado | SPI registers via ADI stack |
| Bitstream needed to reconfigure? | Basic params: No. Structural: Yes | No |
| FPGA DDC work required | None (hardened) | None (done in ADC chip) |

### ZCU208 ADC is a Real ADC at Hardware Level
- The ZU48DR ADCs produce **real samples** at 5 GSPS
- The **IQ output** you work with is produced by the on-chip **DDC (Digital
  Down Converter)** running inside the RFSoC fabric after sampling
- This means the hardware Nyquist constraint applies to the **analog input**,
  not to the final IQ bandwidth

### IQ / Complex Sampling (the correct principle)
- In a true hardware IQ receiver: two ADCs sample I and Q in quadrature
- Required sample rate per channel: fs ≥ B (signal bandwidth only)
- Carrier frequency does NOT determine the required sample rate
- This IS the correct principle — but it applies to the **DDC output**, not
  the analog input of the ZCU208

### Where Each Rule Applies
| Stage | Constraint | Rule |
|---|---|---|
| Analog input → ZCU208 ADC | Real ADC, 5 GSPS | fs ≥ 2 × fmax of analog input signal |
| ZCU208 DDC output → your code | Digital IQ | Sample rate ≥ signal bandwidth only |

---

## [PERSIST] Analog Input Bandwidth vs Nyquist Frequency
Two separate and independent limits on the ZU48DR ADC:

| Parameter | Value | What it means |
|---|---|---|
| Sample rate | 5 GSPS | Digital sampling rate |
| Nyquist frequency | 2.5 GHz | fs/2. Signals above this alias |
| **Analog input bandwidth** | **6 GHz (-3 dB)** | Analog front-end limit. Signals above this are physically attenuated before reaching the sampler |

**Both limits apply simultaneously.** A signal must pass through the analog
front end AND satisfy Nyquist (or be deliberately aliased via Nyquist zones).

### Term: Analog Input Bandwidth (also: Full-Power Bandwidth)
The frequency at which the ADC's analog front end attenuates a full-scale
input signal by 3 dB. Property of the sample-and-hold circuit, input amplifier,
and ESD structures. Distinct from and independent of the digital sample rate.
Always check this spec in an ADC datasheet — it sets the practical ceiling
for any Nyquist zone approach.

---

## Nyquist Zones (ZU48DR at 5 GSPS)

| Zone | Frequency Range | Notes |
|---|---|---|
| 1st | 0 – 2.5 GHz | Clean. No aliasing. Full analog bandwidth. |
| 2nd | 2.5 – 5 GHz | Aliases into 1st zone. Predictable, recoverable. Good analog response. |
| 3rd | 5 – 6 GHz | Aliases into 1st zone. Analog front end rolling off. Marginal. |
| Above 6 GHz | — | Analog front end cannot pass signal. Not usable. |

### Using Higher Nyquist Zones Deliberately
- Signal aliases down into 1st zone in a mathematically predictable way
- Requires a tight bandpass filter before ADC to:
  1. Pass only the desired band
  2. Reject the mirror image frequency in the 1st zone (image rejection)
  3. Reject out-of-band noise and interference
- Filter requirements become more demanding in higher zones
- AMD documents and supports 2nd and 3rd zone operation for ZCU208

---

## Practical Implications for This Radar Project

### Direct Sampling (S-band, no mixing)
- Target band: 3.1–3.5 GHz → sits in **2nd Nyquist zone**
- Aliases cleanly into 1st zone
- Requires S-band bandpass filter before ADC
- ZCU208 DDC then digitally down-converts to baseband IQ

### Downconversion (X-band with fixed LO)
- RF at 10.0–10.5 GHz, downconverted to IF at ~2.25 GHz
- IF sits in **1st Nyquist zone** — clean, no aliasing needed
- Simpler from a sampling perspective; complexity moves to RF frontend
- See `rf-frontend/architecture.md`
