# PlutoSDR Complete Architecture Guide

## Table of Contents
1. [System Overview](#system-overview)
2. [Hardware Architecture](#hardware-architecture)
3. [Firmware Components](#firmware-components)
4. [Gateware (FPGA) Architecture](#gateware-fpga-architecture)
5. [Software Stack](#software-stack)
6. [Boot Process](#boot-process)
7. [Signal Processing Chain](#signal-processing-chain)

---

## 1. System Overview

### PlutoSDR High-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ADALM-PLUTO (PlutoSDR)                      │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │              Xilinx Zynq-7000 SoC (XC7Z010)                   │ │
│  │                                                               │ │
│  │  ┌─────────────────┐         ┌──────────────────────────┐   │ │
│  │  │  Processing     │         │     Programmable Logic   │   │ │
│  │  │  System (PS)    │◄───────►│        (PL/FPGA)         │   │ │
│  │  │                 │  AXI    │                          │   │ │
│  │  │  • Dual ARM     │         │  • DMA Engines           │   │ │
│  │  │    Cortex-A9    │         │  • ADC/DAC Interfaces    │   │ │
│  │  │    @ 666 MHz    │         │  • Digital Filters       │   │ │
│  │  │  • 512 MB DDR3  │         │  • AXI Interconnect      │   │ │
│  │  │  • USB 2.0      │         │  • HDL Components        │   │ │
│  │  └─────────────────┘         └──────────────────────────┘   │ │
│  │           │                              │                   │ │
│  └───────────┼──────────────────────────────┼───────────────────┘ │
│              │                              │                     │
│              │                              │ Digital I/Q         │
│              │ Control/Config               ▼                     │
│  ┌───────────▼──────────────────────────────────────────────────┐ │
│  │              AD9363 RF Agile Transceiver                     │ │
│  │                                                              │ │
│  │  RX Path:                    TX Path:                       │ │
│  │  ┌──────┐  ┌─────┐          ┌─────┐  ┌──────┐             │ │
│  │  │ LNA  │→│ Mixer│→ADC      DAC→│ Mixer│→│  PA  │            │ │
│  │  └──────┘  └─────┘          └─────┘  └──────┘             │ │
│  │                                                              │ │
│  │  • Frequency: 325 MHz - 3.8 GHz                             │ │
│  │  • Bandwidth: 200 kHz - 20 MHz (56 MHz capable)             │ │
│  │  • Sample Rate: up to 61.44 MSPS                            │ │
│  │  • 12-bit ADC/DAC                                           │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                      │                                             │
│                      ▼                                             │
│              ┌──────────────┐                                      │
│              │  SMA/u.FL    │                                      │
│              │  RF Ports    │                                      │
│              └──────────────┘                                      │
└─────────────────────────────────────────────────────────────────────┘
         │
         ▼
    USB 2.0 Host Connection
```

### Key Specifications

| Component | Specification |
|-----------|---------------|
| **SoC** | Xilinx Zynq XC7Z010 (Dual-core ARM Cortex-A9 + FPGA) |
| **RF Transceiver** | Analog Devices AD9363 |
| **Frequency Range** | 325 MHz - 3.8 GHz |
| **RF Bandwidth** | 200 kHz - 20 MHz (configurable, 56 MHz max) |
| **Sample Rate** | Up to 61.44 MSPS |
| **ADC/DAC Resolution** | 12-bit |
| **RAM** | 512 MB DDR3L |
| **Interface** | USB 2.0 (480 Mbps) |
| **Channels** | 1 TX + 1 RX (full duplex) |

---

## 2. Hardware Architecture

### 2.1 Zynq-7000 SoC Architecture

The Zynq-7000 combines:
- **Processing System (PS)**: ARM-based application processor
- **Programmable Logic (PL)**: FPGA fabric

```
┌─────────────────────────────────────────────────────────────────┐
│                  Xilinx Zynq-7000 (XC7Z010)                     │
│                                                                 │
│ ┌─────────────────────────────────────────────────────────────┐│
│ │               Processing System (PS)                        ││
│ │                                                             ││
│ │  ┌──────────────────┐      ┌──────────────────┐            ││
│ │  │  Application     │      │  Interconnect    │            ││
│ │  │  Processing Unit │      │  & Memory        │            ││
│ │  │                  │      │  Controllers     │            ││
│ │  │  • CPU0 (A9)     │◄────►│                  │            ││
│ │  │  • CPU1 (A9)     │      │  • DDR Controller│            ││
│ │  │  • NEON/FPU      │      │  • OCM           │            ││
│ │  │  • L1/L2 Cache   │      │  • AXI Ports     │            ││
│ │  └──────────────────┘      └──────────────────┘            ││
│ │           │                         │                       ││
│ │  ┌────────▼─────────────────────────▼──────────┐           ││
│ │  │           I/O Peripherals                   │           ││
│ │  │  • USB OTG (2.0)     • SPI                  │           ││
│ │  │  • UART              • I2C                  │           ││
│ │  │  • GPIO              • CAN                  │           ││
│ │  └─────────────────────────────────────────────┘           ││
│ └─────────────────────────────────────────────────────────────┘│
│                             │                                  │
│                         AXI HP/GP Ports                        │
│                             │                                  │
│ ┌───────────────────────────▼─────────────────────────────────┐│
│ │            Programmable Logic (PL/FPGA)                     ││
│ │                                                             ││
│ │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     ││
│ │  │   AXI        │  │   DMA        │  │  RF Data     │     ││
│ │  │   ADI IP     │◄─┤   Engines    │◄─┤  Path        │     ││
│ │  │   Cores      │  │              │  │              │     ││
│ │  └──────────────┘  └──────────────┘  └──────────────┘     ││
│ │         │                  │                  │            ││
│ │         └──────────────────┴──────────────────┘            ││
│ │                      AXI Interconnect                       ││
│ │                                                             ││
│ │  Logic Resources (XC7Z010):                                ││
│ │  • 28,000 Logic Cells                                      ││
│ │  • 80 DSP48E1 Slices                                       ││
│ │  • 2.1 Mb Block RAM                                        ││
│ └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 AD9363 RF Transceiver Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                    AD9363 RF Agile Transceiver                     │
│                                                                    │
│  RX CHAIN                                                          │
│  ┌────┐   ┌─────────┐   ┌──────┐   ┌─────┐   ┌─────┐            │
│  │Ant │──►│ RF      │──►│Mixer │──►│ LPF │──►│ ADC │──► I/Q Out  │
│  │    │   │ Input   │   │  LO  │   │ VGA │   │12bit│             │
│  └────┘   │ LNA     │   └──────┘   └─────┘   └─────┘             │
│           └─────────┘        │                                     │
│                              │                                     │
│                        ┌─────▼─────┐                               │
│                        │    PLL    │                               │
│                        │Synthesizer│                               │
│                        │(LO Gen)   │                               │
│                        └─────┬─────┘                               │
│  TX CHAIN                    │                                     │
│  ┌────┐   ┌─────────┐   ┌──┴───┐   ┌─────┐   ┌─────┐            │
│  │Ant │◄──│   PA    │◄──│Mixer │◄──│ LPF │◄──│ DAC │◄── I/Q In  │
│  │    │   │         │   │  LO  │   │ VGA │   │12bit│             │
│  └────┘   └─────────┘   └──────┘   └─────┘   └─────┘             │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │              Digital Interface (to FPGA)                     │ │
│  │                                                              │ │
│  │  • LVDS Data Interface (12 lanes)                           │ │
│  │  • SPI Control Interface                                    │ │
│  │  • Clock Management (Tx/Rx Frame Clocks)                    │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  Internal Blocks:                                                  │
│  • AGC (Automatic Gain Control)                                   │
│  • DC Offset Correction                                           │
│  • Quadrature Correction                                          │
│  • FIR Filters (programmable)                                     │
│  • RSSI Measurement                                               │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. Firmware Components

### 3.1 Complete Firmware Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                    PlutoSDR Firmware Stack                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 7: User Applications                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • libiio network daemon (iiod)                         │   │
│  │  • User custom applications                             │   │
│  │  • Shell utilities                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│  Layer 6: Userspace Libraries                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • libiio - Industrial I/O library                      │   │
│  │  • libad9361-iio - AD9361/AD9363 control                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│  Layer 5: Linux Kernel (4.x/5.x)                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Subsystems:                                            │   │
│  │  • IIO (Industrial I/O) Framework                       │   │
│  │  • Network Stack (USB Ethernet gadget)                  │   │
│  │  • USB OTG Driver                                       │   │
│  │  • Device Drivers                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│  Layer 4: IIO Drivers                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • ad9361-phy - RF transceiver driver                   │   │
│  │  • cf_axi_adc - ADI ADC DMA driver                      │   │
│  │  • cf_axi_dac - ADI DAC DMA driver                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│  Layer 3: Hardware Abstraction                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • AXI Bus Interface                                    │   │
│  │  • DMA Memory Mappings                                  │   │
│  │  • Device Tree Bindings                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│  Layer 2: FPGA Gateware (HDL)                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • AXI AD9361 Interface IP                              │   │
│  │  • AXI DMA IP Cores                                     │   │
│  │  • AXI Interconnect                                     │   │
│  │  • Custom Processing Blocks (optional)                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│  Layer 1: Bootloader (U-Boot + FSBL)                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • FSBL - First Stage Bootloader (loads from QSPI)      │   │
│  │  • U-Boot - Configures hardware, loads Linux           │   │
│  │  • Device Tree Loading                                  │   │
│  │  • FPGA Bitstream Loading                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│  Layer 0: Hardware                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • Zynq SoC (PS + PL)                                   │   │
│  │  • AD9363 RF Transceiver                                │   │
│  │  • DDR3 Memory                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Firmware Build Components

The PlutoSDR firmware is built from these repositories:

| Component | Repository | Branch | Description |
|-----------|-----------|--------|-------------|
| **HDL (Gateware)** | analogdevicesinc/hdl | hdl_2018_r1 | FPGA design (Vivado projects) |
| **Linux Kernel** | analogdevicesinc/linux | 2018_R1 | Modified Linux with IIO drivers |
| **U-Boot** | analogdevicesinc/u-boot-xlnx | pluto | Bootloader |
| **Buildroot** | analogdevicesinc/buildroot | master | Root filesystem builder |
| **plutosdr-fw** | analogdevicesinc/plutosdr-fw | master | Build orchestration |

### 3.3 Build Artifacts

```
build/
├── boot.bin           # FSBL + U-Boot (QSPI bootloader)
├── boot.dfu           # Bootloader in DFU format
├── boot.frm           # Bootloader for USB mass storage update
├── pluto.itb          # FIT image (Kernel + DTB + Rootfs + FPGA)
├── pluto.dfu          # Main firmware in DFU format
├── pluto.frm          # Main firmware for USB mass storage update
├── system_top.bit     # FPGA bitstream
├── system_top.xsa     # Vivado hardware export
├── zImage             # Compressed Linux kernel
├── zynq-pluto-sdr*.dtb # Device tree blobs
└── rootfs.cpio.gz     # Root filesystem
```

---

## 4. Gateware (FPGA) Architecture

### 4.1 FPGA Block Design

The PlutoSDR FPGA design is based on Analog Devices' HDL reference designs:

```
┌──────────────────────────────────────────────────────────────────┐
│               FPGA (Programmable Logic) Block Design             │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                   Processing System (PS)                   │ │
│  │                  (ARM Cortex-A9 CPU)                       │ │
│  └────────────┬───────────────────────┬──────────────────────┘ │
│               │                       │                         │
│          GP0 Master              HP0/HP1 Slave                  │
│          (Control)               (High-perf data)               │
│               │                       │                         │
│  ┌────────────▼───────────────────────▼──────────────────────┐ │
│  │              AXI Interconnect (AXI4 / AXI4-Stream)        │ │
│  └────┬─────────────┬─────────────┬──────────────┬───────────┘ │
│       │             │             │              │              │
│  ┌────▼─────┐  ┌───▼────┐   ┌───▼────┐    ┌────▼─────┐       │
│  │ AXI AD9361│  │AXI DMA │   │AXI DMA │    │  Other   │       │
│  │  Control  │  │ ADC    │   │ DAC    │    │  AXI     │       │
│  │    IP     │  │  IP    │   │  IP    │    │  Slaves  │       │
│  └────┬──────┘  └───┬────┘   └───┬────┘    └──────────┘       │
│       │             │            │                              │
│       │         ┌───▼────────────▼───┐                          │
│       │         │  AXI4-Stream FIFO  │                          │
│       │         │  (Data Buffering)  │                          │
│       │         └───┬────────────┬───┘                          │
│  ┌────▼─────────────▼──┐    ┌────▼──────────────┐              │
│  │    AD9361 Digital   │    │  Optional Custom  │              │
│  │    Interface        │    │  DSP Blocks       │              │
│  │  • RX Data Path     │    │  • FIR Filters    │              │
│  │  • TX Data Path     │    │  • FFT            │              │
│  │  • Clock Management │    │  • Custom logic   │              │
│  └──────┬──────────────┘    └───────────────────┘              │
│         │ LVDS/CMOS                                             │
│  ┌──────▼──────────────────────────────────────────────────┐   │
│  │          I/O Pins to AD9363 Transceiver                 │   │
│  │  • DATA_CLK      • FB_CLK                               │   │
│  │  • RX_FRAME      • TX_FRAME                             │   │
│  │  • RX_D[11:0]    • TX_D[11:0]                           │   │
│  │  • SPI (Control) • Enable/TXNRX                         │   │
│  └─────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 Key FPGA IP Cores

#### AXI AD9361 IP Core
- **Function**: Interface between FPGA and AD9361 transceiver
- **Features**:
  - LVDS/CMOS data interface
  - Clock domain crossing
  - Data format conversion (12-bit to 16-bit I/Q)
  - Control register interface via AXI4-Lite

#### AXI DMAC (DMA Controller)
- **RX Path**: Transfers samples from ADC to PS memory (DDR)
- **TX Path**: Transfers samples from PS memory to DAC
- **Features**:
  - Scatter-gather DMA
  - Cyclic buffer support
  - AXI4-Stream to AXI4 memory-mapped conversion

### 4.3 Data Flow Through FPGA

**Receive Path (RF → CPU):**
```
AD9363 → I/Q Data (LVDS) → AXI_AD9361 IP → AXI-Stream →
DMA Controller → HP Port → DDR3 Memory → CPU Access via IIO
```

**Transmit Path (CPU → RF):**
```
CPU writes to IIO → DDR3 Memory → HP Port → DMA Controller →
AXI-Stream → AXI_AD9361 IP → I/Q Data (LVDS) → AD9363
```

---

## 5. Software Stack

### 5.1 Linux IIO (Industrial I/O) Framework

The PlutoSDR uses the **IIO framework** for data acquisition and control:

```
┌──────────────────────────────────────────────────────────────┐
│                IIO Framework Architecture                    │
│                                                              │
│  User Space:                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Applications (Python, C++, GNU Radio, MATLAB)         │ │
│  └────────────────┬───────────────────────────────────────┘ │
│                   │                                          │
│  ┌────────────────▼───────────────────────────────────────┐ │
│  │            libiio (Userspace Library)                  │ │
│  │  • Local backend (direct kernel access)               │ │
│  │  • Network backend (iiod TCP/IP)                      │ │
│  │  • USB backend                                        │ │
│  └────────────────┬───────────────────────────────────────┘ │
│                   │                                          │
│  ══════════════════╪══════════════════════════════════════  │
│  Kernel Space:     │                                         │
│                   │                                          │
│  ┌────────────────▼───────────────────────────────────────┐ │
│  │          IIO Core (/sys/bus/iio/devices/)             │ │
│  └────────────────┬───────────────────────────────────────┘ │
│                   │                                          │
│  ┌────────────────▼────────┬──────────────┬──────────────┐  │
│  │   IIO Device Drivers    │              │              │  │
│  │  ┌──────────────────┐   │  ┌────────┐  │  ┌────────┐ │  │
│  │  │  ad9361-phy      │   │  │cf_axi_ │  │  │cf_axi_ │ │  │
│  │  │  (RF control)    │   │  │adc     │  │  │dac     │ │  │
│  │  └──────────────────┘   │  └────────┘  │  └────────┘ │  │
│  └─────────────────────────┴──────────────┴──────────────┘  │
│                   │                                          │
│  ┌────────────────▼───────────────────────────────────────┐ │
│  │            Hardware (FPGA + AD9363)                    │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 IIO Device Structure

PlutoSDR exposes these IIO devices:

```bash
/sys/bus/iio/devices/
├── iio:device0          # AD9361 PHY (Physical layer control)
│   ├── in_voltage0_gain_control_mode    # RX gain mode
│   ├── in_voltage0_hardwaregain         # RX gain value
│   ├── out_voltage0_hardwaregain        # TX attenuation
│   ├── out_altvoltage0_RX_LO_frequency  # RX LO freq
│   ├── out_altvoltage1_TX_LO_frequency  # TX LO freq
│   ├── in_voltage_sampling_frequency    # RX sample rate
│   └── out_voltage_sampling_frequency   # TX sample rate
│
├── iio:device1          # RX DMA (cf-ad9361-lpc)
│   ├── buffer/           # Buffer configuration
│   │   ├── enable
│   │   └── length
│   ├── scan_elements/    # Channel configuration
│   └── in_voltage0_i     # I channel
│       in_voltage0_q     # Q channel
│
└── iio:device2          # TX DMA (cf-ad9361-dds-core-lpc)
    ├── buffer/
    ├── scan_elements/
    ├── out_voltage0_i    # I channel
    └── out_voltage0_q    # Q channel
```

### 5.3 Network Access via iiod

The **iiod** (IIO daemon) provides network access:

```
┌────────────────────────────────────────────────────────────┐
│                    Network Access                          │
│                                                            │
│  Host PC (Windows/Linux/Mac)                              │
│  ┌──────────────────────────────────────────────┐         │
│  │  Application (Python/MATLAB/GNU Radio)       │         │
│  │                                              │         │
│  │  ┌───────────────────────────────────────┐  │         │
│  │  │  libiio (network backend)             │  │         │
│  │  └───────────┬───────────────────────────┘  │         │
│  └──────────────┼──────────────────────────────┘         │
│                 │                                          │
│            TCP/IP over USB Ethernet                        │
│            (IP: 192.168.2.1 or pluto.local)               │
│                 │                                          │
│  ┌──────────────▼──────────────────────────────┐         │
│  │  PlutoSDR (running iiod on port 30431)      │         │
│  │                                              │         │
│  │  ┌───────────────────────────────────────┐  │         │
│  │  │  iiod daemon                          │  │         │
│  │  │  • Accepts network connections        │  │         │
│  │  │  • Translates to local IIO calls      │  │         │
│  │  │  • Streams I/Q data                   │  │         │
│  │  └───────────┬───────────────────────────┘  │         │
│  │              │                                │         │
│  │  ┌───────────▼───────────────────────────┐  │         │
│  │  │  IIO Kernel Drivers                   │  │         │
│  │  └───────────────────────────────────────┘  │         │
│  └──────────────────────────────────────────────┘         │
└────────────────────────────────────────────────────────────┘
```

---

## 6. Boot Process

### 6.1 Complete Boot Sequence

```
Power On
   │
   ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Zynq Boot ROM (hardcoded in silicon)               │
│ • Reads boot mode pins → QSPI mode                         │
│ • Loads FSBL from QSPI flash offset 0x0                    │
│ • Executes FSBL                                            │
└─────────────────────┬───────────────────────────────────────┘
                      │
   ┌──────────────────▼────────────────────────────────────┐
   │ Step 2: FSBL (First Stage Bootloader)                 │
   │ • Initializes PS (DDR, clocks, MIO)                   │
   │ • Runs ps7_init.tcl generated by Vivado               │
   │ • Programs FPGA with bitstream from boot.bin          │
   │ • Loads U-Boot into DDR                               │
   │ • Jumps to U-Boot                                     │
   └─────────────────┬─────────────────────────────────────┘
                     │
   ┌─────────────────▼─────────────────────────────────────┐
   │ Step 3: U-Boot (Second Stage Bootloader)              │
   │ • Configures additional hardware                      │
   │ • Sets up USB gadget (Ethernet, mass storage)         │
   │ • Loads FIT image (pluto.itb) from QSPI or DFU        │
   │   - Contains: Kernel + Device Tree + Rootfs + FPGA    │
   │ • Extracts and loads components:                      │
   │   1. FPGA bitstream → Programs PL                     │
   │   2. Device Tree → Passes to kernel                   │
   │   3. Linux kernel → DDR address 0x8000                │
   │   4. Rootfs → RAM disk                                │
   │ • Jumps to kernel entry point                         │
   └─────────────────┬─────────────────────────────────────┘
                     │
   ┌─────────────────▼─────────────────────────────────────┐
   │ Step 4: Linux Kernel Boot                             │
   │ • Decompresses zImage                                 │
   │ • Parses device tree                                  │
   │ • Initializes hardware based on DT                    │
   │ • Mounts rootfs (cpio.gz ramdisk)                     │
   │ • Loads kernel modules                                │
   │ • Probes IIO drivers (ad9361, axi_adc, axi_dac)       │
   │ • Starts init process                                 │
   └─────────────────┬─────────────────────────────────────┘
                     │
   ┌─────────────────▼─────────────────────────────────────┐
   │ Step 5: User Space Init                               │
   │ • Runs /init scripts                                  │
   │ • Mounts filesystems (/dev, /proc, /sys)              │
   │ • Configures network (USB Ethernet gadget)            │
   │ • Starts iiod daemon                                  │
   │ • System ready for operation                          │
   └───────────────────────────────────────────────────────┘
```

### 6.2 Memory Map

```
┌─────────────────────────────────────────────────────────────┐
│              Zynq Memory Map                                │
├─────────────────────────────────────────────────────────────┤
│  0x0000_0000 - 0x0003_FFFF  │ On-Chip Memory (OCM) 256KB    │
│  0x0010_0000 - 0x3FFF_FFFF  │ Reserved                      │
│  0x4000_0000 - 0x7FFF_FFFF  │ PL (FPGA) AXI Slaves          │
│  0x8000_0000 - 0xBFFF_FFFF  │ PL (FPGA) AXI Slaves          │
│  0xE000_0000 - 0xE02F_FFFF  │ IOP (PS Peripherals)          │
│  0xF800_0000 - 0xF8F0_1FFF  │ SLCR, SMC, QSPI, etc.         │
│  0xFC00_0000 - 0xFDFF_FFFF  │ QSPI Flash (32MB)             │
│  0xFFFC_0000 - 0xFFFF_FFFF  │ OCM (high alias)              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              DDR3 Memory Map (512MB)                        │
├─────────────────────────────────────────────────────────────┤
│  0x0000_0000 - 0x0000_7FFF  │ Kernel entry (32KB)           │
│  0x0000_8000 - 0x00??_????  │ Linux Kernel Image            │
│  0x????_???? - 0x????_????  │ Device Tree Blob              │
│  0x????_???? - 0x0FFF_FFFF  │ RAM Disk (rootfs)             │
│  0x1000_0000 - 0x1FFF_FFFF  │ Kernel space                  │
│  0x2000_0000 - 0x3FFF_FFFF  │ DMA buffers / User space      │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Signal Processing Chain

### 7.1 Complete RX Chain (RF to I/Q Samples)

```
┌────────────────────────────────────────────────────────────────────┐
│                      RECEIVE SIGNAL PATH                           │
│                                                                    │
│  RF Input (Antenna)                                                │
│       │                                                            │
│  ┌────▼────────────────────────────────────────────────────────┐  │
│  │                    AD9363 Analog RX                          │  │
│  │                                                              │  │
│  │  ┌──────┐   ┌──────────┐   ┌─────────┐   ┌──────────┐      │  │
│  │  │ LNA  │──►│RF Filter │──►│ Mixer 1 │──►│IF Filter │      │  │
│  │  │      │   │(tunable) │   │(RF→IF)  │   │  (BPF)   │      │  │
│  │  └──────┘   └──────────┘   └────┬────┘   └─────┬────┘      │  │
│  │                                  │              │           │  │
│  │                            ┌─────▼──────┐       │           │  │
│  │                            │  RX LO     │       │           │  │
│  │                            │ (PLL/VCO)  │       │           │  │
│  │                            └────────────┘       │           │  │
│  │                                                 │           │  │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐   │           │  │
│  │  │ Mixer 2  │◄──│ 90° Phase│◄──┤Local Osc.│◄──┘           │  │
│  │  │(IF→BB I) │   │  Shift   │   │          │               │  │
│  │  └────┬─────┘   └──────────┘   └──────────┘               │  │
│  │       │                                │                   │  │
│  │  ┌────▼─────┐   ┌─────────┐      ┌────▼─────┐             │  │
│  │  │ LPF + VGA│──►│12-bit   │      │ LPF + VGA│             │  │
│  │  │ (I-path) │   │ADC (I)  │      │ (Q-path) │             │  │
│  │  └──────────┘   └────┬────┘      └─────┬────┘             │  │
│  │                      │                  │                  │  │
│  │  ┌─────────────┐     │   ┌──────────┐   │                 │  │
│  │  │ Mixer 2     │─────┘   │12-bit    │◄──┘                 │  │
│  │  │ (IF→BB Q)   │         │ADC (Q)   │                     │  │
│  │  └─────────────┘         └────┬─────┘                     │  │
│  │                               │                           │  │
│  └───────────────────────────────┼───────────────────────────┘  │
│                                  │                               │
│  ┌───────────────────────────────▼───────────────────────────┐  │
│  │              AD9363 Digital Processing                    │  │
│  │                                                            │  │
│  │  • DC Offset Removal                                      │  │
│  │  • I/Q Imbalance Correction                               │  │
│  │  • FIR Filter 1 (programmable decimation)                 │  │
│  │  • FIR Filter 2 (programmable decimation)                 │  │
│  │  • AGC (if enabled)                                       │  │
│  │                                                            │  │
│  │  Output: 12-bit I/Q at Fs (up to 61.44 MSPS)             │  │
│  └───────────────────────┬────────────────────────────────────┘  │
│                          │ LVDS Interface                        │
│                          │ (DATA_CLK, RX_FRAME, RX_D[11:0])      │
│  ┌───────────────────────▼────────────────────────────────────┐  │
│  │                 FPGA (Zynq PL)                             │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────────┐ │  │
│  │  │  AXI AD9361 IP Core                                  │ │  │
│  │  │  • LVDS deserializer                                 │ │  │
│  │  │  • Clock domain crossing                             │ │  │
│  │  │  • Format conversion: 12-bit → 16-bit signed         │ │  │
│  │  │  • I/Q packing                                       │ │  │
│  │  └──────────────────┬───────────────────────────────────┘ │  │
│  │                     │ AXI4-Stream                          │  │
│  │  ┌──────────────────▼───────────────────────────────────┐ │  │
│  │  │  Optional Custom DSP (user-added)                    │ │  │
│  │  │  • Digital filtering                                 │ │  │
│  │  │  • Decimation/Interpolation                          │ │  │
│  │  │  • FFT processing                                    │ │  │
│  │  └──────────────────┬───────────────────────────────────┘ │  │
│  │                     │ AXI4-Stream                          │  │
│  │  ┌──────────────────▼───────────────────────────────────┐ │  │
│  │  │  AXI DMA Controller (ADC DMA)                        │ │  │
│  │  │  • Packs I/Q into memory format                      │ │  │
│  │  │  • Writes to DDR via HP0 port                        │ │  │
│  │  │  • Generates interrupts when buffers fill            │ │  │
│  │  └──────────────────┬───────────────────────────────────┘ │  │
│  └────────────────────┬┴───────────────────────────────────────┘  │
│                       │ AXI HP Port                               │
│  ┌────────────────────▼───────────────────────────────────────┐  │
│  │                 DDR3 Memory                                │  │
│  │                                                            │  │
│  │  Circular DMA Buffers containing I/Q samples:             │  │
│  │  Format: [I0, Q0, I1, Q1, I2, Q2, ...]                    │  │
│  │  Each sample: 16-bit signed integer                       │  │
│  │  Sample rate: configurable (up to 61.44 MSPS)             │  │
│  └────────────────────┬───────────────────────────────────────┘  │
│                       │                                           │
│  ┌────────────────────▼───────────────────────────────────────┐  │
│  │              Linux IIO Driver (cf_axi_adc)                 │  │
│  │                                                            │  │
│  │  • Manages DMA buffers                                    │  │
│  │  • Provides /dev interface                                │  │
│  │  • Exposes sysfs attributes                               │  │
│  └────────────────────┬───────────────────────────────────────┘  │
│                       │                                           │
│  ┌────────────────────▼───────────────────────────────────────┐  │
│  │                 libiio Library                             │  │
│  │                                                            │  │
│  │  • Read I/Q samples via buffer API                        │  │
│  │  • Configure channels (enable I/Q)                        │  │
│  │  • Set buffer sizes                                       │  │
│  └────────────────────┬───────────────────────────────────────┘  │
│                       │                                           │
│  ┌────────────────────▼───────────────────────────────────────┐  │
│  │          User Application                                  │  │
│  │  (Python, C++, GNU Radio, MATLAB, etc.)                    │  │
│  │                                                            │  │
│  │  Receives complex I/Q samples: I + jQ                      │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

### 7.2 TX Chain (I/Q Samples to RF)

The TX chain is essentially the reverse:

```
Application → libiio → IIO Driver → DMA → FPGA → AD9363 DAC →
Mixer → PA → RF Output
```

### 7.3 Sample Format

**Memory format for I/Q samples:**
```
Offset  | Data
--------|--------
0x0000  | I[0] (16-bit signed, little-endian)
0x0002  | Q[0] (16-bit signed, little-endian)
0x0004  | I[1]
0x0006  | Q[1]
...
```

**Sample rate relationship:**
- ADC/DAC operates at internal rate (up to 122.88 MSPS)
- Decimation/Interpolation filters reduce to Fs (programmable)
- Typical Fs: 2.084 MSPS to 61.44 MSPS

---

## Summary

The PlutoSDR architecture consists of:

1. **Hardware**: Zynq SoC (dual ARM + FPGA) + AD9363 RF transceiver
2. **Gateware**: FPGA design with ADI IP cores for ADC/DAC DMA
3. **Firmware**: Linux with IIO framework for device control
4. **Software**: libiio library for application access

**Key Data Path:**
```
RF Signal ↔ AD9363 (Analog) ↔ AD9363 (Digital) ↔ FPGA (LVDS) ↔
DMA ↔ DDR Memory ↔ CPU ↔ USB ↔ Host Application
```

This architecture enables:
- Full software-defined radio functionality
- High-speed I/Q sample streaming
- Flexible RF configuration
- Open-source development
- Custom FPGA processing (advanced users)

---

## Next Steps

- **Custom Applications**: See `PLUTOSDR_CUSTOM_APPS.md`
- **Training Labs**: See `PLUTOSDR_TRAINING_LABS.md`
- **API Reference**: ADI IIO Documentation
