# Vivado Licensing (ZCU208)

> **⚠️ SUPERSEDED — June 2026.** Vivado licensing is moot under the settled
> architecture: the Lattice **ECP5** edge FPGA uses a fully open-source
> toolchain (Yosys + nextpnr + openFPGALoader) with no licence of any kind, and
> there is no RFSoC/Versal device in the design. See
> [`radar-rx-frontend-edge-digitization.md`](radar-rx-frontend-edge-digitization.md).
> Retained only as a record of the ZCU208-era licensing analysis.

* Pre-loaded bitstream on ZCU208 → no Vivado license needed at all
* Custom bitstream for that specific device → device-locked license included in the kit
* PYNQ overlay path → no FPGA development, no licensing cost
* Production devices (different chips) → requires appropriate Vivado Enterprise and per-device licensing
