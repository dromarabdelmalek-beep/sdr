# PlutoSDR Custom Application Development Guide

## Table of Contents
1. [Overview](#overview)
2. [Development Approaches](#development-approaches)
3. [Hosted Applications (On-PlutoSDR)](#hosted-applications-on-plutosdr)
4. [External Applications](#external-applications)
5. [Interface Options](#interface-options)
6. [Custom FPGA Gateware](#custom-fpga-gateware)
7. [Example Projects](#example-projects)

---

## 1. Overview

PlutoSDR supports multiple application development approaches:

| Approach | Execution Location | Use Case | Complexity |
|----------|-------------------|----------|------------|
| **Hosted Apps** | On PlutoSDR Linux | Standalone operation, low latency | Medium |
| **External libiio** | Host PC | Development, visualization, GUIs | Low |
| **GNU Radio** | Host PC | Flowgraph-based DSP | Low |
| **SoapySDR** | Host PC | SDR abstraction, portability | Low |
| **UHD** | Host PC (via SoapyUHD) | USRP compatibility | Low |
| **Custom FPGA** | PlutoSDR FPGA | Real-time DSP, hardware acceleration | High |

---

## 2. Development Approaches

### 2.1 Application Architecture Options

```
┌─────────────────────────────────────────────────────────────────┐
│                    Approach 1: Hosted on PlutoSDR               │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │               PlutoSDR Device                           │   │
│  │                                                         │   │
│  │  ┌───────────────────────────────────────────────────┐ │   │
│  │  │  Custom Application (C/C++/Python)                │ │   │
│  │  │  • Runs directly on ARM CPU                       │ │   │
│  │  │  • Direct IIO access (no network overhead)        │ │   │
│  │  │  • Can control external hardware via GPIO/SPI     │ │   │
│  │  └─────────────────┬─────────────────────────────────┘ │   │
│  │                    │                                     │   │
│  │  ┌─────────────────▼─────────────────────────────────┐ │   │
│  │  │  libiio (local backend)                           │ │   │
│  │  └─────────────────┬─────────────────────────────────┘ │   │
│  │                    │                                     │   │
│  │  ┌─────────────────▼─────────────────────────────────┐ │   │
│  │  │  IIO Kernel Drivers                               │ │   │
│  │  └─────────────────┬─────────────────────────────────┘ │   │
│  │                    │                                     │   │
│  │  ┌─────────────────▼─────────────────────────────────┐ │   │
│  │  │  Hardware (FPGA + AD9363)                         │ │   │
│  │  └───────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Pros: Low latency, no network, standalone                     │
│  Cons: Limited CPU power, limited storage                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                 Approach 2: External Application                │
│                                                                 │
│  ┌─────────────────────────────────────────┐                   │
│  │      Host PC (Windows/Linux/Mac)        │                   │
│  │                                         │                   │
│  │  ┌───────────────────────────────────┐ │                   │
│  │  │ Your Application                  │ │                   │
│  │  │ • Python/C++/MATLAB/GNU Radio     │ │                   │
│  │  │ • Full PC resources               │ │                   │
│  │  │ • Rich visualization              │ │                   │
│  │  └─────────┬─────────────────────────┘ │                   │
│  │            │                             │                   │
│  │  ┌─────────▼─────────────────────────┐ │                   │
│  │  │ SDR Abstraction Layer             │ │                   │
│  │  │ • libiio (network backend)        │ │                   │
│  │  │ • SoapySDR                        │ │                   │
│  │  │ • GNU Radio (gr-iio)              │ │                   │
│  │  └─────────┬─────────────────────────┘ │                   │
│  └────────────┼───────────────────────────┘                   │
│               │                                                 │
│        USB Ethernet / Network                                   │
│        (IP: 192.168.2.1)                                        │
│               │                                                 │
│  ┌────────────▼───────────────────────────┐                   │
│  │      PlutoSDR (iiod server)            │                   │
│  │                                         │                   │
│  │  ┌───────────────────────────────────┐ │                   │
│  │  │ iiod - IIO Network Daemon         │ │                   │
│  │  └─────────┬─────────────────────────┘ │                   │
│  │            │                             │                   │
│  │  ┌─────────▼─────────────────────────┐ │                   │
│  │  │ IIO Drivers + Hardware            │ │                   │
│  │  └───────────────────────────────────┘ │                   │
│  └─────────────────────────────────────────┘                   │
│                                                                 │
│  Pros: Full PC power, easy development                         │
│  Cons: Network latency, requires host connection               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Hosted Applications (On-PlutoSDR)

### 3.1 When to Use Hosted Applications

- **Standalone operation** without PC connection
- **Low-latency** real-time processing
- **Embedded deployment** (battery-powered, remote)
- **Direct hardware control** (GPIO, SPI, I2C)

### 3.2 Development Workflow

```
┌──────────────────────────────────────────────────────────────┐
│           Hosted Application Development Workflow            │
│                                                              │
│  Step 1: Cross-Compile on Host PC                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Host PC (Linux)                                       │ │
│  │  • Use PlutoSDR buildroot SDK                          │ │
│  │  │  OR use cross-compiler toolchain                    │ │
│  │  • Compile with arm-linux-gnueabihf-gcc                │ │
│  │  • Link against libiio, etc.                           │ │
│  │                                                        │ │
│  │  $ arm-linux-gnueabihf-gcc -o myapp myapp.c -liio     │ │
│  └────────────────────────────────────────────────────────┘ │
│               │                                              │
│               ▼                                              │
│  Step 2: Transfer Binary to PlutoSDR                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • SSH/SCP to PlutoSDR                                 │ │
│  │    $ scp myapp root@192.168.2.1:/root/                │ │
│  │  • Mount mass storage and copy                         │ │
│  │  • Build into custom firmware image                    │ │
│  └────────────────────────────────────────────────────────┘ │
│               │                                              │
│               ▼                                              │
│  Step 3: Execute on PlutoSDR                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  PlutoSDR Shell (via SSH)                              │ │
│  │  $ chmod +x myapp                                      │ │
│  │  $ ./myapp                                             │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 3.3 Setting Up Cross-Compilation

#### Option 1: Use PlutoSDR Buildroot SDK

```bash
# On your development PC (Ubuntu/Debian):

# 1. Clone plutosdr-fw repository
git clone --recursive https://github.com/analogdevicesinc/plutosdr-fw.git
cd plutosdr-fw

# 2. Build the SDK
export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2021.2/settings64.sh  # Optional
make TOOLCHAIN

# 3. SDK will be in: buildroot/output/host/
export PATH=$PATH:$(pwd)/buildroot/output/host/bin
export CROSS_COMPILE=arm-linux-gnueabihf-

# 4. Verify
arm-linux-gnueabihf-gcc --version
```

#### Option 2: Use Pre-built Toolchain

```bash
# Download Linaro ARM GCC toolchain
wget https://releases.linaro.org/components/toolchain/binaries/7.3-2018.05/arm-linux-gnueabihf/gcc-linaro-7.3.1-2018.05-x86_64_arm-linux-gnueabihf.tar.xz

tar xf gcc-linaro-7.3.1-2018.05-x86_64_arm-linux-gnueabihf.tar.xz

export PATH=$PATH:$(pwd)/gcc-linaro-7.3.1-2018.05-x86_64_arm-linux-gnueabihf/bin
export CROSS_COMPILE=arm-linux-gnueabihf-
```

### 3.4 Example: Simple Hosted Application (C)

**simple_rx.c** - Capture I/Q samples and process locally:

```c
/*
 * PlutoSDR Hosted Application Example
 * Captures I/Q samples and computes average power
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
#include <iio.h>

#define SAMPLE_COUNT 16384

int main(int argc, char **argv)
{
    struct iio_context *ctx;
    struct iio_device *phy, *dev;
    struct iio_channel *rx0_i, *rx0_q;
    struct iio_buffer *rxbuf;
    int16_t *p_dat;
    double sum_power = 0.0;
    size_t sample_count = 0;

    /* Create local IIO context */
    ctx = iio_create_local_context();
    if (!ctx) {
        fprintf(stderr, "Failed to create IIO context\n");
        return -1;
    }

    printf("IIO context has %u devices\n", iio_context_get_devices_count(ctx));

    /* Get AD9361 PHY and configure */
    phy = iio_context_find_device(ctx, "ad9361-phy");
    if (!phy) {
        fprintf(stderr, "Failed to find ad9361-phy\n");
        goto error;
    }

    /* Set RX LO frequency to 915 MHz */
    iio_channel_attr_write_longlong(
        iio_device_find_channel(phy, "altvoltage0", true),
        "frequency", 915000000);

    /* Set sample rate to 2.084 MSPS */
    iio_channel_attr_write_longlong(
        iio_device_find_channel(phy, "voltage0", false),
        "sampling_frequency", 2084000);

    /* Set RX gain to manual mode, 60 dB */
    iio_channel_attr_write(
        iio_device_find_channel(phy, "voltage0", false),
        "gain_control_mode", "manual");
    iio_channel_attr_write_longlong(
        iio_device_find_channel(phy, "voltage0", false),
        "hardwaregain", 60);

    /* Get RX device */
    dev = iio_context_find_device(ctx, "cf-ad9361-lpc");
    if (!dev) {
        fprintf(stderr, "Failed to find RX device\n");
        goto error;
    }

    /* Enable I/Q channels */
    rx0_i = iio_device_find_channel(dev, "voltage0", false);
    rx0_q = iio_device_find_channel(dev, "voltage1", false);
    iio_channel_enable(rx0_i);
    iio_channel_enable(rx0_q);

    /* Create buffer */
    rxbuf = iio_device_create_buffer(dev, SAMPLE_COUNT, false);
    if (!rxbuf) {
        fprintf(stderr, "Failed to create buffer\n");
        goto error;
    }

    printf("Capturing %d samples...\n", SAMPLE_COUNT);

    /* Refill buffer (blocking read) */
    ssize_t nbytes_rx = iio_buffer_refill(rxbuf);
    if (nbytes_rx < 0) {
        fprintf(stderr, "Error refilling buffer: %ld\n", nbytes_rx);
        goto error_buf;
    }

    printf("Received %ld bytes\n", nbytes_rx);

    /* Process samples */
    p_dat = (int16_t *)iio_buffer_start(rxbuf);

    for (size_t i = 0; i < SAMPLE_COUNT; i++) {
        int16_t i_sample = p_dat[2*i];
        int16_t q_sample = p_dat[2*i + 1];

        /* Calculate power: I^2 + Q^2 */
        double power = (double)i_sample * i_sample +
                       (double)q_sample * q_sample;
        sum_power += power;

        /* Print first 10 samples */
        if (i < 10) {
            printf("Sample %zu: I=%d, Q=%d, Power=%.2f\n",
                   i, i_sample, q_sample, power);
        }
    }

    /* Calculate average power in dB */
    double avg_power = sum_power / SAMPLE_COUNT;
    double avg_power_db = 10.0 * log10(avg_power);

    printf("\nAverage Power: %.2f (%.2f dB)\n", avg_power, avg_power_db);

error_buf:
    iio_buffer_destroy(rxbuf);
error:
    iio_context_destroy(ctx);
    return 0;
}
```

**Compilation:**

```bash
# Cross-compile for PlutoSDR
arm-linux-gnueabihf-gcc -o simple_rx simple_rx.c -liio -lm

# Transfer to PlutoSDR
scp simple_rx root@192.168.2.1:/root/

# Run on PlutoSDR (SSH into it)
ssh root@192.168.2.1
chmod +x simple_rx
./simple_rx
```

### 3.5 Example: Hosted Python Application

PlutoSDR includes Python, so you can run Python scripts directly:

**simple_rx.py**:

```python
#!/usr/bin/env python3
"""
PlutoSDR Hosted Python Example
Captures I/Q samples and computes FFT
"""

import iio
import numpy as np
import time

# Create local context
ctx = iio.Context('local:')
print(f"IIO context has {len(ctx.devices)} devices")

# Get devices
phy = ctx.find_device("ad9361-phy")
dev = ctx.find_device("cf-ad9361-lpc")

# Configure PHY
rx_lo = phy.find_channel("altvoltage0", True)
rx = phy.find_channel("voltage0", False)

# Set frequency to 915 MHz
rx_lo.attrs["frequency"].value = "915000000"

# Set sample rate to 2.084 MSPS
rx.attrs["sampling_frequency"].value = "2084000"

# Set gain
rx.attrs["gain_control_mode"].value = "manual"
rx.attrs["hardwaregain"].value = "60"

# Enable I/Q channels
chn_i = dev.find_channel("voltage0", False)
chn_q = dev.find_channel("voltage1", False)
chn_i.enabled = True
chn_q.enabled = True

# Create buffer
BUFFER_SIZE = 16384
buf = iio.Buffer(dev, BUFFER_SIZE, False)

print(f"Capturing {BUFFER_SIZE} samples...")

# Read samples
buf.refill()
data = buf.read()

# Convert to numpy array
samples = np.frombuffer(data, dtype=np.int16)
i_samples = samples[0::2]
q_samples = samples[1::2]

# Create complex samples
iq = i_samples + 1j * q_samples

# Compute power
power = np.abs(iq)**2
avg_power_db = 10 * np.log10(np.mean(power))

print(f"Average power: {avg_power_db:.2f} dB")

# Compute FFT (basic spectrum)
fft = np.fft.fft(iq)
fft_shifted = np.fft.fftshift(fft)
power_spectrum = 10 * np.log10(np.abs(fft_shifted)**2)

print(f"Peak in spectrum: {np.max(power_spectrum):.2f} dB")
```

**Deploy and run:**

```bash
# Copy to PlutoSDR
scp simple_rx.py root@192.168.2.1:/root/

# SSH and run
ssh root@192.168.2.1
python3 simple_rx.py
```

### 3.6 Adding Custom Apps to Buildroot

To permanently integrate your app into the PlutoSDR firmware:

**Step 1: Create buildroot package**

```bash
cd plutosdr-fw/buildroot
mkdir -p package/myapp
```

**package/myapp/Config.in:**

```
config BR2_PACKAGE_MYAPP
    bool "myapp"
    help
      My custom PlutoSDR application
```

**package/myapp/myapp.mk:**

```makefile
MYAPP_VERSION = 1.0
MYAPP_SITE = $(BR2_EXTERNAL_MYAPP_PATH)/package/myapp/src
MYAPP_SITE_METHOD = local
MYAPP_DEPENDENCIES = libiio

define MYAPP_BUILD_CMDS
    $(TARGET_CC) $(TARGET_CFLAGS) -o $(@D)/myapp \
        $(@D)/myapp.c -liio $(TARGET_LDFLAGS)
endef

define MYAPP_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/myapp $(TARGET_DIR)/usr/bin/myapp
endef

$(eval $(generic-package))
```

**Step 2: Add to buildroot config:**

```bash
# Edit buildroot/configs/zynq_pluto_defconfig
# Add: BR2_PACKAGE_MYAPP=y

# Or run menuconfig:
make -C buildroot menuconfig
# Navigate to Target packages → Custom packages → myapp
# Enable [*] myapp
```

**Step 3: Rebuild firmware:**

```bash
make
# Your app will be in the new pluto.frm firmware image
```

---

## 4. External Applications

### 4.1 libiio (Direct Access)

**Installation:**

```bash
# Ubuntu/Debian
sudo apt install libiio-dev libiio-utils python3-libiio

# Or build from source
git clone https://github.com/analogdevicesinc/libiio.git
cd libiio
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```

**Example C Application (Host PC):**

```c
/* external_rx.c - Run on host PC, connect to PlutoSDR over network */

#include <stdio.h>
#include <iio.h>

#define PLUTO_URI "ip:192.168.2.1"

int main() {
    struct iio_context *ctx = iio_create_context_from_uri(PLUTO_URI);
    if (!ctx) {
        fprintf(stderr, "Failed to connect to %s\n", PLUTO_URI);
        return -1;
    }

    printf("Connected to PlutoSDR\n");
    printf("IIO context has %u devices:\n", iio_context_get_devices_count(ctx));

    for (unsigned int i = 0; i < iio_context_get_devices_count(ctx); i++) {
        struct iio_device *dev = iio_context_get_device(ctx, i);
        printf("  %s (channels: %u)\n",
               iio_device_get_name(dev),
               iio_device_get_channels_count(dev));
    }

    iio_context_destroy(ctx);
    return 0;
}
```

**Compile and run:**

```bash
gcc -o external_rx external_rx.c -liio
./external_rx
```

### 4.2 pyadi-iio (Python Interface)

**Installation:**

```bash
pip install pyadi-iio
```

**Example:**

```python
import adi
import numpy as np
import matplotlib.pyplot as plt

# Connect to PlutoSDR
sdr = adi.Pluto("ip:192.168.2.1")

# Configure
sdr.sample_rate = 2084000  # 2.084 MSPS
sdr.rx_lo = 915000000      # 915 MHz
sdr.rx_rf_bandwidth = 2000000
sdr.rx_buffer_size = 16384
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 60

# Receive samples
samples = sdr.rx()

# Plot spectrum
fft = np.fft.fftshift(np.fft.fft(samples))
freqs = np.fft.fftshift(np.fft.fftfreq(len(samples), 1/sdr.sample_rate))
plt.plot(freqs/1e6, 20*np.log10(np.abs(fft)))
plt.xlabel("Frequency (MHz)")
plt.ylabel("Power (dB)")
plt.title("PlutoSDR Spectrum")
plt.show()
```

### 4.3 GNU Radio

**Installation:**

```bash
sudo apt install gnuradio gr-iio
```

**Example Flowgraph (Python):**

```python
#!/usr/bin/env python3

from gnuradio import gr, blocks, analog, iio

class pluto_rx(gr.top_block):
    def __init__(self):
        gr.top_block.__init__(self, "PlutoSDR RX")

        # PlutoSDR Source
        self.pluto_source = iio.pluto_source(
            uri='ip:192.168.2.1',
            frequency=915000000,
            samplerate=2084000,
            bandwidth=2000000,
            buffer_size=16384,
            quadrature=True,
            rfdc=True,
            bbdc=True,
            gain_mode='manual',
            manual_gain=60,
            filter='')

        # File sink (save I/Q to file)
        self.file_sink = blocks.file_sink(
            gr.sizeof_gr_complex,
            'pluto_rx_samples.bin',
            False)

        # Connect blocks
        self.connect(self.pluto_source, self.file_sink)

if __name__ == '__main__':
    tb = pluto_rx()
    tb.start()
    input("Press Enter to stop...")
    tb.stop()
    tb.wait()
```

### 4.4 SoapySDR

**Installation:**

```bash
# Install SoapySDR and PlutoSDR support
sudo apt install soapysdr-tools soapysdr-module-lms7

# Build SoapyPlutoSDR from source
git clone https://github.com/pothosware/SoapyPlutoSDR.git
cd SoapyPlutoSDR
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
sudo ldconfig
```

**Test:**

```bash
# List devices
SoapySDRUtil --find="driver=plutosdr"

# Test RX
SoapySDRUtil --args="driver=plutosdr" --rate=2e6 --freq=915e6 --rx-file=/tmp/rx.bin
```

**Example C++ Application:**

```cpp
#include <SoapySDR/Device.hpp>
#include <SoapySDR/Formats.hpp>
#include <iostream>
#include <complex>

int main() {
    // Find PlutoSDR
    SoapySDR::Kwargs args;
    args["driver"] = "plutosdr";

    SoapySDR::Device *sdr = SoapySDR::Device::make(args);
    if (!sdr) {
        std::cerr << "Failed to open PlutoSDR" << std::endl;
        return -1;
    }

    // Configure
    sdr->setSampleRate(SOAPY_SDR_RX, 0, 2.084e6);
    sdr->setFrequency(SOAPY_SDR_RX, 0, 915e6);
    sdr->setGain(SOAPY_SDR_RX, 0, 60);

    // Setup stream
    SoapySDR::Stream *rxStream = sdr->setupStream(SOAPY_SDR_RX, SOAPY_SDR_CF32);
    sdr->activateStream(rxStream);

    // Receive samples
    std::complex<float> buffer[16384];
    void *buffs[] = {buffer};
    int flags = 0;
    long long timeNs = 0;

    int ret = sdr->readStream(rxStream, buffs, 16384, flags, timeNs, 1e6);

    std::cout << "Received " << ret << " samples" << std::endl;

    // Cleanup
    sdr->deactivateStream(rxStream);
    sdr->closeStream(rxStream);
    SoapySDR::Device::unmake(sdr);

    return 0;
}
```

### 4.5 MATLAB

**Using Communications Toolbox Support Package:**

```matlab
% Install: Communications Toolbox Support Package for Xilinx Zynq-Based Radio

% Create PlutoSDR object
pluto = sdrrx('Pluto');

% Configure
pluto.RadioID = 'usb:0';
pluto.CenterFrequency = 915e6;
pluto.BasebandSampleRate = 2.084e6;
pluto.SamplesPerFrame = 16384;
pluto.GainSource = 'Manual';
pluto.Gain = 60;

% Receive samples
data = pluto();

% Plot spectrum
nfft = length(data);
f = (-nfft/2:nfft/2-1) * (pluto.BasebandSampleRate/nfft);
spectrum = 20*log10(abs(fftshift(fft(data))));

plot(f/1e6, spectrum);
xlabel('Frequency (MHz)');
ylabel('Power (dB)');
title('PlutoSDR Spectrum');
grid on;

% Cleanup
release(pluto);
```

---

## 5. Interface Options Summary

| Interface | Language | Pros | Cons | Use Case |
|-----------|----------|------|------|----------|
| **libiio (local)** | C/C++ | Low latency, direct | Requires cross-compile | Hosted apps |
| **libiio (network)** | C/C++/Python | Easy development | Network latency | External apps |
| **pyadi-iio** | Python | High-level API, simple | Python overhead | Prototyping, scripts |
| **GNU Radio** | Python/C++ | Flowgraph design, blocks | Learning curve | DSP chains |
| **SoapySDR** | C++/Python | SDR abstraction | Extra layer | Portability |
| **MATLAB** | MATLAB | Built-in functions | Commercial license | Research, analysis |

---

## 6. Custom FPGA Gateware

### 6.1 When to Modify FPGA

- **Hardware acceleration** of DSP algorithms
- **Real-time processing** beyond CPU capability
- **Custom interfaces** (non-standard protocols)
- **Latency-critical** applications

### 6.2 FPGA Modification Workflow

```bash
# 1. Clone HDL repository
git clone https://github.com/analogdevicesinc/hdl.git
cd hdl

# 2. Navigate to PlutoSDR project
cd projects/pluto

# 3. Open in Vivado
source /opt/Xilinx/Vivado/2021.2/settings64.sh
make

# This opens Vivado with the PlutoSDR block design
# You can now add custom IP cores

# 4. After modifications, rebuild
make

# 5. Export XSA
# File → Export → Export Hardware
# Copy system_top.xsa to plutosdr-fw/build/

# 6. Rebuild firmware
cd /path/to/plutosdr-fw
make
```

### 6.3 Example: Adding Custom FIR Filter IP

**Step 1: Create AXI-Stream FIR Filter IP in Vivado HLS:**

```c
#include "ap_int.h"
#include "hls_stream.h"

// Simple moving average filter
void fir_filter(
    hls::stream<ap_uint<32>> &input,
    hls::stream<ap_uint<32>> &output
) {
    #pragma HLS INTERFACE axis port=input
    #pragma HLS INTERFACE axis port=output
    #pragma HLS INTERFACE s_axilite port=return

    static int16_t buffer[8] = {0};

    while (1) {
        ap_uint<32> data_in = input.read();
        int16_t i_in = data_in.range(15, 0);
        int16_t q_in = data_in.range(31, 16);

        // Simple moving average
        int32_t i_sum = 0, q_sum = 0;
        for (int i = 0; i < 7; i++) {
            buffer[i] = buffer[i+1];
        }
        buffer[7] = i_in;

        for (int i = 0; i < 8; i++) {
            i_sum += buffer[i];
        }

        int16_t i_out = i_sum >> 3;  // Divide by 8
        int16_t q_out = q_in;  // Pass Q through

        ap_uint<32> data_out;
        data_out.range(15, 0) = i_out;
        data_out.range(31, 16) = q_out;

        output.write(data_out);
    }
}
```

**Step 2: Add to Block Design:**

1. Package IP in Vivado HLS
2. Add to IP repository in Vivado
3. Insert between AXI_AD9361 and DMA in block design
4. Regenerate bitstream

---

## 7. Example Projects

### 7.1 FM Radio Receiver (GNU Radio)

```python
#!/usr/bin/env python3

from gnuradio import gr, blocks, analog, filter, audio, iio

class fm_radio(gr.top_block):
    def __init__(self):
        gr.top_block.__init__(self, "FM Radio Receiver")

        # PlutoSDR source (88-108 MHz FM band)
        self.pluto = iio.pluto_source(
            uri='ip:192.168.2.1',
            frequency=98000000,  # 98.0 MHz
            samplerate=2084000,
            bandwidth=2000000,
            buffer_size=32768,
            gain_mode='slow_attack',
            manual_gain=50)

        # Low-pass filter for FM demod
        self.lpf = filter.fir_filter_ccf(
            1,
            filter.firdes.low_pass(1, 2084000, 100000, 10000))

        # FM demodulator
        self.fm_demod = analog.wfm_rcv(
            quad_rate=2084000,
            audio_decimation=4)

        # Rational resampler to audio rate
        self.resampler = filter.rational_resampler_fff(
            interpolation=48,
            decimation=521)  # 2084000/4 → 48000

        # Audio sink
        self.audio_sink = audio.sink(48000, '', True)

        # Connect
        self.connect(self.pluto, self.lpf)
        self.connect(self.lpf, self.fm_demod)
        self.connect(self.fm_demod, self.resampler)
        self.connect(self.resampler, self.audio_sink)

if __name__ == '__main__':
    tb = fm_radio()
    tb.start()
    input("Listening to FM radio. Press Enter to stop...")
    tb.stop()
    tb.wait()
```

### 7.2 Spectrum Analyzer (pyadi-iio)

```python
#!/usr/bin/env python3

import adi
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.animation import FuncAnimation

# Connect
sdr = adi.Pluto("ip:192.168.2.1")
sdr.sample_rate = 2084000
sdr.rx_rf_bandwidth = 2000000
sdr.rx_lo = 915000000
sdr.rx_buffer_size = 8192
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 50

# Setup plot
fig, ax = plt.subplots()
freqs = np.fft.fftshift(np.fft.fftfreq(sdr.rx_buffer_size, 1/sdr.sample_rate)) / 1e6
line, = ax.plot(freqs, np.zeros(len(freqs)))
ax.set_xlabel("Frequency (MHz)")
ax.set_ylabel("Power (dB)")
ax.set_title("PlutoSDR Spectrum Analyzer")
ax.set_ylim([-100, 0])
ax.grid(True)

def update(frame):
    samples = sdr.rx()
    fft = np.fft.fftshift(np.fft.fft(samples))
    psd = 20 * np.log10(np.abs(fft) / len(fft))
    line.set_ydata(psd)
    return line,

ani = FuncAnimation(fig, update, interval=100, blit=True)
plt.show()
```

### 7.3 Custom Modulator/Demodulator (C)

*See PLUTOSDR_TRAINING_LABS.md for complete modem examples*

---

## Summary

| Development Goal | Best Approach | Interface |
|------------------|---------------|-----------|
| Quick prototype | Python on host PC | pyadi-iio |
| Flowgraph DSP | GNU Radio | gr-iio |
| Portable SDR app | SoapySDR | SoapyPlutoSDR |
| Standalone device | C/C++ on PlutoSDR | libiio (local) |
| Hardware acceleration | FPGA customization | Vivado HDL |
| MATLAB research | MATLAB | Support Package |

**Next:** See `PLUTOSDR_TRAINING_LABS.md` for hands-on labs covering all SDR concepts!
