# Signal Processing Pipeline

## Overview
ZCU208 performs ADC sampling and digital down-conversion on-chip.
Decimated IQ data is then streamed to a host for further processing.
**Host processing architecture and tools are not yet decided.**

---

## On-Chip Processing (ZCU208 / RFSoC fabric)
This section is settled — the ZCU208 DDC and MTS are definite parts of the design.

### DDC — Digital Down Converter
- Built into ZU48DR RFSoC fabric (hardened IP)
- Converts real ADC samples to complex IQ via NCO and mixer
- Applies programmable decimation filter
- Outputs: complex IQ at reduced sample rate

### Decimation
- Target: 8× decimation (or as required for 500 MHz IQ bandwidth)
- Input: 5 GSPS real samples
- Output: 500 MSPS complex IQ per channel
- Processing gain from decimation: ~4.5 dB (~3 dB per octave of oversampling)

### Data Rate After Decimation
- 8 channels × 500 MSPS × 24 bits (IQ at 12-bit — confirm word width) = 96 Gbps
- ZCU208 SFP28 (4× 25G) = 100 Gbps — sufficient for continuous streaming
- With PYNQ pre-built image this works without additional IP licensing

---

## Multi-Tile Synchronisation (MTS)
- ZCU208 MTS ensures all 8 ADC channels are phase-aligned to within
  one sample at startup
- Essential for coherent beamforming across channels
- Must be enabled in Vivado RF Data Converter IP configuration
- PYNQ supports MTS — verify in current PYNQ image release notes

---

## Host Processing (UNDECIDED)

### What Needs to Happen (regardless of implementation)
The radar processing pipeline requires operations across three dimensions:
1. **Fast time (range)** — matched filter / FFT across samples within a pulse
2. **Slow time (Doppler)** — FFT across pulses
3. **Spatial (angle/beamforming)** — phase-weighted summation or FFT across channels

### Candidate Approaches (explored, not decided)

**GPU with cuFFT:**
- cuFFT library handles large batched FFTs efficiently
- All three dimensions can be processed on GPU
- Data organised as [channels × pulses × samples] array
- High memory bandwidth available on modern GPUs
- Explored as a candidate but not committed to

**FPGA fabric (ZCU208 PL):**
- Lower latency than GPU
- Harder to modify once implemented
- Makes sense if real-time processing is a hard requirement
- Beamforming in particular is a candidate for FPGA implementation

**CPU / numpy / scipy:**
- Simplest to implement for initial testing and algorithm development
- Insufficient throughput for real-time operation at full data rate
- Valid for offline processing during bring-up and validation

---

## Pending Design Decisions
- [ ] Host processing architecture: GPU, FPGA fabric, or hybrid
- [ ] If GPU: which library/framework (cuFFT, PyTorch, custom CUDA)
- [ ] IQ word width: 12-bit, 16-bit, or 32-bit float — affects data rate and host memory
- [ ] Pulse repetition interval and waveform design
- [ ] Number of pulses per CPI (coherent processing interval)
- [ ] Whether beamforming runs on FPGA or host
- [ ] Real-time vs offline processing requirement
