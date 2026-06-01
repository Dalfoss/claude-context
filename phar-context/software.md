# Software and Toolchain

## PYNQ
- Pre-built PYNQ images available for ZCU208
- Supports 25GbE out of the box (pre-compiled bitstream — no license needed)
- Python + Jupyter Notebook interface for ADC/DAC control
- Best path for rapid prototyping without FPGA expertise
- PYNQ framework: https://www.pynq.io
- RFSoC-specific resources: https://www.rfsoc-pynq.io

## Vivado
- Node-locked, device-locked license for ZU48DR **included with ZCU208 kit**
- Required for custom bitstream generation
- Not required if using pre-built PYNQ images
- Version compatibility: check AMD/Xilinx release notes — use the version
  recommended for your PYNQ image version

## Key IP Blocks (Vivado)
| IP | Notes |
|---|---|
| RF Data Converter | Core IP for ADC/DAC tile configuration. Free with device license. |
| XXV Ethernet (10G/25G MAC) | Required for custom 25GbE bitstreams. **Separate license needed** if not using pre-built PYNQ image. Included in pre-built images. |
| AXI DMA | For streaming data to PS/host. Free. |

## Host Interface
- Pre-built PYNQ: 25GbE works, stream to GPU host via 100G (4×25G)
- Custom design: PCIe endpoint or 25GbE — both require Vivado IP

## RF Data Converter Evaluation Tool
- AMD GUI tool for ZCU208/ZCU216
- Configures ADC/DAC tiles, evaluates performance
- Windows only
- Useful for initial hardware bring-up and characterisation

## Development Approach (Recommended Order)
1. Flash PYNQ pre-built image to SD card
2. Validate ADC/DAC operation using RF Data Converter Evaluation Tool
3. Develop signal processing in Python/Jupyter using PYNQ framework
4. Migrate to custom Vivado bitstream only when PYNQ constraints are hit

## GPU Host (cuFFT)
- 3D FFT pipeline runs on GPU host connected via 25GbE
- cuFFT library handles FFT computation
- Data arrives as IQ samples from ZCU208 DDC output
- See `signal-processing/pipeline.md` for pipeline details
