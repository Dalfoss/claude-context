# RF Frontend Components and BOM

## Phase 1 — Single TX + Single RX, Bench Modular

### Common to Both Architecture Options

| Item | Part | Supplier | Est. Cost |
|---|---|---|---|
| SDR/FPGA platform | ZCU208 Evaluation Kit | AMD/Xilinx distributors | ~$9,500 |
| RF breakout | XM655 add-on card | AMD/Xilinx | ~$400 |
| Reference clock | CLK104 add-on card | AMD/Xilinx | ~$200 |
| SMA cable assemblies (×10) | Phase-matched, equal length | Pasternack, Fairview | ~$200 |
| SMA terminations (×5) | 50Ω loads | Mini-Circuits | ~$50 |
| SMA torque wrench | Essential at 10 GHz | Pasternack | ~$100 |
| DC bench power supplies (×2) | For LNA, PA bias | Any | ~$300 |

### Option A Only: Direct Sampling, S-band (~3.1–3.5 GHz)

| Item | Part | Notes | Est. Cost |
|---|---|---|---|
| S-band LNA (×1) | TBD — Fairview or Mini-Circuits connectorized module | 3–4 GHz, NF < 2 dB, SMA | ~$150 |
| S-band PA (×1) | TBD — Fairview connectorized module | 3–4 GHz, ~1W, SMA | ~$200 |
| BPF at target frequency (×2) | TBD — Mini-Circuits or K&L | S-band, ~400 MHz BW, SMA | ~$100 each |
| **Option A Phase 1 Total** | | | **~$11,200** |

### Option B Only: X-band with Fixed LO

| Item | Part | Notes | Est. Cost |
|---|---|---|---|
| LO source | Wenzel MXO-PLMX (8 GHz) | OCXO-multiplied, ultra-low phase noise, +13 dBm, SMA. Also outputs 100 MHz reference for CLK104 | ~$1,500 |
| LO buffer amplifier | Mini-Circuits ZX60-V83+ or similar | Wideband SMA, boost to ~+25 dBm | ~$80 |
| LO splitter (Phase 1, 2-way) | Mini-Circuits — 2-way, 8 GHz rated, SMA | One output TX mixer, one RX mixer | ~$50 |
| RX LNA | Fairview SLNA-120-38-22-SMA | 8–12 GHz, 2.2 dB NF, 38 dB gain, SMA, hermetic | ~$450 |
| Mixers (×2) | Marki MM1-0312SS | Passive double-balanced GaAs MMIC, 3–12 GHz, SMA connectorized | ~$500 each |
| RF BPF 10.25 GHz (×2) | K&L Microwave or Wainwright Instruments | Cavity filter, ~500 MHz passband, SMA. Quote required | ~$400 each |
| IF BPF 2.25 GHz (×2) | Mini-Circuits or K&L | 2.25 GHz centre, 500 MHz BW, SMA | ~$150 each |
| TX PA | Fairview X-band 1W SMA module | GaAs pHEMT, hermetic, built-in bias sequencing | ~$500 |
| **Option B Phase 1 Total** | | | **~$15,500** |

---

## Phase 2 — Full 16 Channels, Custom PCB

### PCB

| Item | Notes | Est. Cost |
|---|---|---|
| PCB substrate | Rogers 4350B or Megtron-6. FR4 is NOT acceptable at 10 GHz | — |
| PCB fabrication | 4-layer, prototype qty. Suppliers: PCBWay, Sierra Circuits, Bittele | ~$2,000–4,000 |
| PCB assembly (SMT) | RF-specialist assembly house. Self-assembly saves ~$3,000–6,000 | ~$3,000–6,000 |
| KiCAD design notes | Use microstrip calculator for 50Ω trace widths. Supplement with Sonnet Lite (free EM sim) for critical RF structures. All RF on top layer, solid ground plane on bottom. | — |

### Option A Phase 2 — S-band SMT Components (×8 channels)

| Item | Part | Est. Cost |
|---|---|---|
| S-band LNA MMIC (×8) | TBD — Qorvo or MACOM S-band MMIC | ~$30 each |
| PA MMIC (×8) | TBD — GaN S-band MMIC | ~$80 each |
| BPF SMT (×16) | TDK or Murata S-band SMT filter | ~$20 each |
| Passives, decoupling | 0402/0603 | ~$200 |
| IF interface board to ZCU208 | Simple 4-layer FR4 board | ~$500 |
| **Option A Phase 2 Total** | | **~$8,000–12,000** |

### Option B Phase 2 — X-band SMT Components (×8 channels)

| Item | Part | Est. Cost |
|---|---|---|
| RX LNA MMIC (×8) | ADL5723 (10.1–11.7 GHz SiGe, 2mm×2mm LFCSP) — validate with ADL5723-EVALZ in Phase 1 first | ~$40 each |
| Mixer SMT (×16) | Marki MM1-0312S SMT QFN | ~$200 each |
| PA MMIC (×8) | Qorvo QPA0023D (6–18 GHz, 30 dBm P1dB) or MACOM equivalent | ~$100 each |
| RF BPF PCB-integrated | Coupled-line or interdigital filter designed into PCB layout | design cost only |
| IF BPF SMT (×16) | AVX, TDK or custom, 2.25 GHz | ~$30 each |
| LO amplifier | MACOM or Qorvo X-band driver, ~+25 dBm | ~$300 |
| LO splitter tree | Mini-Circuits: 1× 2-way + 2× 8-way (MP0208-8), all SMA | ~$400 |
| LO equal-length PCB routing | Microstrip corporate feed tree, matched within PCB | design cost only |
| IF interface board to ZCU208 | Simple 4-layer FR4 board | ~$500 |
| Passives, decoupling, connectors | 0402/0603 | ~$400 |
| **Option B Phase 2 Total** | | **~$12,000–16,000** |

---

## Total Estimated Budget

| Path | Phase 1 | Phase 2 | Total |
|---|---|---|---|
| Option A (S-band direct) | ~$11,200 | ~$8,000–12,000 | **~$19,000–23,000** |
| Option B (X-band mixing) | ~$15,500 | ~$12,000–16,000 | **~$27,000–32,000** |

*Excludes: test equipment, antennas, enclosures, PCB design labour.*

---

## Key Suppliers
- **Mini-Circuits**: splitters, filters, amplifiers — minicircuits.com
- **Fairview Microwave**: connectorized X-band modules — fairviewmicrowave.com
- **Marki Microwave**: mixers — markimicrowave.com
- **Wenzel/Quantic**: LO source — quanticwenzel.com
- **K&L Microwave**: cavity filters — klmicrowave.com (quote required)
- **Pasternack**: cables, adapters — pasternack.com
- **PCBWay / Bittele**: Rogers substrate PCB fabrication
