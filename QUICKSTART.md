# PlutoSDR Educational Curriculum - Quick Start Guide

**Get started with PlutoSDR and Software-Defined Radio in 30 minutes!**

This quick start guide will help you:
1. Set up your PlutoSDR hardware
2. Install required software
3. Run your first SDR program
4. Navigate the comprehensive curriculum

---

## Prerequisites

- **Hardware**: ADALM-PLUTO (PlutoSDR) - [Purchase here](https://www.analog.com/en/design-center/evaluation-hardware-and-software/evaluation-boards-kits/adalm-pluto.html) (~$150 USD)
- **Computer**: Linux (Ubuntu 20.04+), Windows 10/11, or macOS
- **Skills**: Basic programming knowledge (Python or C)
- **Time**: 30 minutes for setup

---

## Step 1: Connect Your PlutoSDR (5 minutes)

### Physical Connection

1. **Connect PlutoSDR** via USB to your computer
2. **Wait for drivers** to install automatically
3. **Verify connection**:

**Linux**:
```bash
lsusb | grep Analog
# Should show: "Analog Devices, Inc. PlutoSDR (ADALM-PLUTO)"

ip addr show usb0
# Should show: 192.168.2.x network interface
```

**Windows**:
- Open Device Manager → Network Adapters
- Look for "Remote NDIS Compatible Device"

**macOS**:
- System Preferences → Network
- Look for "RNDIS/Ethernet Gadget"

### Test Network Connection

```bash
ping 192.168.2.1
# PlutoSDR should respond
```

### Access PlutoSDR Web Interface

Open browser: **http://192.168.2.1**

Default credentials:
- Username: `root`
- Password: `analog`

You should see the PlutoSDR dashboard showing:
- Firmware version
- TX/RX sample rates
- Center frequency
- Bandwidth

---

## Step 2: Install Software (10 minutes)

### Option A: Quick Setup (Linux - Recommended)

```bash
# Update package lists
sudo apt update

# Install Python and dependencies
sudo apt install -y python3 python3-pip libiio-utils libiio-dev

# Install PyADI-IIO (Analog Devices Python library)
pip3 install pyadi-iio numpy scipy matplotlib

# Install GNU Radio (optional, for Method 1)
sudo apt install -y gnuradio

# Install ARM cross-compiler (for Method 3)
sudo apt install -y gcc-arm-linux-gnueabihf

# Verify installation
python3 -c "import adi; print('✓ PyADI-IIO installed')"
iio_info -n 192.168.2.1 | grep "PlutoSDR"
```

### Option B: Manual Setup (All Platforms)

#### Windows

1. **Install Python 3.9+**: https://www.python.org/downloads/
2. **Install libiio**: https://github.com/analogdevicesinc/libiio/releases
3. **Install PyADI-IIO**:
   ```cmd
   pip install pyadi-iio numpy scipy matplotlib
   ```

#### macOS

```bash
# Install Homebrew if not installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install dependencies
brew install python libiio

# Install PyADI-IIO
pip3 install pyadi-iio numpy scipy matplotlib
```

### USB Permissions (Linux Only)

```bash
# Install udev rules for PlutoSDR
sudo cp scripts/53-adi-plutosdr-usb.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger

# Add yourself to plugdev group
sudo usermod -a -G plugdev $USER

# Log out and log back in for changes to take effect
```

---

## Step 3: Run Your First SDR Program (10 minutes)

### Test 1: Receive and Plot FM Radio

Create a file `test_fm_radio.py`:

```python
import adi
import numpy as np
import matplotlib.pyplot as plt

# Connect to PlutoSDR
sdr = adi.Pluto("ip:192.168.2.1")

# Configure for FM radio reception (88-108 MHz)
sdr.sample_rate = int(2.4e6)        # 2.4 MSPS
sdr.rx_rf_bandwidth = int(2e6)      # 2 MHz
sdr.rx_lo = int(100.1e6)            # 100.1 MHz (tune to your local FM station)
sdr.gain_control_mode = "slow_attack"

# Receive samples
sdr.rx_buffer_size = 1024 * 16
samples = sdr.rx()

# Plot spectrum
plt.figure(figsize=(10, 6))
plt.subplot(2, 1, 1)
plt.plot(np.real(samples[:1000]))
plt.title("FM Radio Signal - Time Domain (I component)")
plt.xlabel("Sample")
plt.ylabel("Amplitude")
plt.grid(True)

plt.subplot(2, 1, 2)
fft = np.fft.fftshift(np.fft.fft(samples))
freqs = np.fft.fftshift(np.fft.fftfreq(len(samples), 1/sdr.sample_rate))
plt.plot(freqs/1e6, 20*np.log10(np.abs(fft)))
plt.title("FM Radio Signal - Frequency Domain")
plt.xlabel("Frequency (MHz)")
plt.ylabel("Power (dB)")
plt.grid(True)

plt.tight_layout()
plt.savefig('fm_radio_test.png')
print("✓ FM radio reception successful!")
print(f"  Center frequency: {sdr.rx_lo/1e6:.1f} MHz")
print(f"  Sample rate: {sdr.sample_rate/1e6:.1f} MSPS")
print(f"  Samples captured: {len(samples)}")
print("  Plot saved to: fm_radio_test.png")
```

Run the script:
```bash
python3 test_fm_radio.py
```

**Expected output**:
- Console message confirming successful reception
- Plot saved as `fm_radio_test.png` showing time and frequency domain

### Test 2: Transmit a Simple Tone

Create a file `test_tx_tone.py`:

```python
import adi
import numpy as np

# Connect to PlutoSDR
sdr = adi.Pluto("ip:192.168.2.1")

# Configure TX
sdr.sample_rate = int(2.4e6)
sdr.tx_rf_bandwidth = int(2e6)
sdr.tx_lo = int(915e6)              # 915 MHz (ISM band)
sdr.tx_hardwaregain = -30           # -30 dB (low power for testing)

# Generate 100 kHz tone
fs = int(sdr.sample_rate)
N = 1024 * 10
t = np.arange(N) / fs
tone_freq = 100e3                   # 100 kHz offset
tone = 0.5 * np.exp(2j * np.pi * tone_freq * t)

# Transmit continuously
sdr.tx_cyclic_buffer = True
sdr.tx(tone)

print("✓ Transmitting 100 kHz tone at 915 MHz")
print(f"  TX frequency: {sdr.tx_lo/1e6:.1f} MHz")
print(f"  TX power: {sdr.tx_hardwaregain} dB")
print(f"  Tone offset: {tone_freq/1e3:.0f} kHz")
print("  Press Ctrl+C to stop...")

try:
    import time
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    sdr.tx_destroy_buffer()
    print("\n✓ Transmission stopped")
```

**⚠️ Warning**: This transmits RF! Use only in shielded environment or with proper antenna/license.

Run the script:
```bash
python3 test_tx_tone.py
```

Use another SDR or spectrum analyzer to verify the 100 kHz tone at 915 MHz.

---

## Step 4: Navigate the Curriculum (5 minutes)

### Curriculum Structure

The educational content is organized as follows:

```
docs/
├── labs/                    # 13 hands-on labs (41,385 lines)
│   ├── module1_foundations/ # Start here! (4 labs)
│   ├── module2_sampling/    # Nyquist, aliasing (3 labs)
│   └── module3_modulation/  # Digital modulation (5 labs)
│
├── projects/                # 4 expert-level projects (8,383 lines)
│   ├── PROJECT2_IOT_SATELLITE_ENHANCED.md
│   ├── PROJECT3_SECURE_VIDEO_ENHANCED.md
│   ├── PROJECT4_FREQUENCY_HOPPING.md
│   └── PROJECT5_TACTICAL_COMMS_MIL_STD.md
│
└── guides/                  # Reference guides (5,810 lines)
    ├── PLUTOSDR_BUILD_SCRIPTS.md      # Firmware compilation
    ├── PLUTOSDR_ARCHITECTURE.md       # Hardware/software architecture
    ├── PLUTOSDR_CUSTOM_APPS.md        # Writing C applications
    └── THREE_METHOD_APPROACH.md       # GNU Radio + Python + C
```

### Recommended Learning Path

**Beginner** (2-3 weeks):
1. **LAB 0**: Hello PlutoSDR → [docs/labs/module1_foundations/LAB_0_HELLO_PLUTOSDR.md](docs/labs/module1_foundations/LAB_0_HELLO_PLUTOSDR.md)
2. **LAB 1.1**: SDR Flexibility
3. **LAB 1.2**: RF Gain Staging
4. **LAB 1.3**: I/Q Samples

**Intermediate** (3-4 weeks):
5. **LAB 2.1**: Nyquist Sampling and Aliasing
6. **LAB 2.2**: Decimation and Interpolation
7. **LAB 2.3**: Quantization and ADC Resolution

**Advanced** (4-6 weeks):
8. **LAB 3.1**: ASK, FSK, PSK Modulation
9. **LAB 3.2**: QPSK and 8-PSK
10. **LAB 3.3**: QAM Modulation (16/64/256-QAM)
11. **LAB 3.4**: BER Testing and Eye Diagrams
12. **LAB 3.5**: Pulse Shaping and Matched Filtering

**Expert** (6-10 weeks):
13. **PROJECT 2**: IoT-Satellite DSSS Link
14. **PROJECT 3**: Secure Video Streaming
15. **PROJECT 4**: Frequency-Hopping Datalink
16. **PROJECT 5**: Tactical Voice/Data Radio (MIL-STD)

### Three Implementation Methods

Each lab uses **three different approaches**:

**Method 1: GNU Radio** (Visual Programming)
- Drag-and-drop flowgraph design
- Rapid prototyping
- Great for learning signal flow

**Method 2: Python** (Scripting)
- PyADI-IIO library
- NumPy/SciPy for DSP
- Flexible and interactive

**Method 3: C** (Embedded/Production)
- Cross-compiled for ARM Cortex-A9
- Runs directly on PlutoSDR
- NEON SIMD optimized (6-15× speedup)
- Production-ready performance

---

## Quick Reference: Common Tasks

### Check PlutoSDR Status

```python
import adi
sdr = adi.Pluto("ip:192.168.2.1")
print(f"Sample rate: {sdr.sample_rate/1e6:.2f} MSPS")
print(f"RX LO: {sdr.rx_lo/1e6:.2f} MHz")
print(f"TX LO: {sdr.tx_lo/1e6:.2f} MHz")
print(f"RX Gain Mode: {sdr.gain_control_mode}")
```

### Capture IQ Samples to File

```python
import adi
import numpy as np

sdr = adi.Pluto("ip:192.168.2.1")
sdr.sample_rate = int(2.4e6)
sdr.rx_lo = int(915e6)
sdr.rx_buffer_size = 1024 * 100

samples = sdr.rx()
np.save('iq_capture.npy', samples)
print(f"Saved {len(samples)} samples to iq_capture.npy")
```

### SSH into PlutoSDR

```bash
ssh root@192.168.2.1
# Password: analog

# Once logged in:
uname -a        # Check kernel version
free -m         # Check memory usage
df -h           # Check storage
iio_info        # List IIO devices
```

### Update PlutoSDR Firmware

1. Download latest firmware: https://github.com/analogdevicesinc/plutosdr-fw/releases
2. Copy `pluto.frm` to PlutoSDR mass storage device
3. Eject the device safely
4. PlutoSDR will reboot and update automatically

**OR** via DFU mode:
```bash
# Enter DFU mode (hold button while plugging in)
dfu-util -a firmware.dfu -D pluto.dfu
dfu-util -a pluto.frm -D pluto.frm
```

### Cross-Compile C Code for PlutoSDR

```bash
# Compile on host PC
arm-linux-gnueabihf-gcc -o myapp myapp.c \
    -lm -O3 -march=armv7-a -mfpu=neon -mfloat-abi=hard

# Deploy to PlutoSDR
scp myapp root@192.168.2.1:/root/

# Run on PlutoSDR
ssh root@192.168.2.1 './myapp'
```

---

## Troubleshooting

### PlutoSDR Not Detected

**Linux**:
```bash
# Check USB connection
lsusb | grep Analog

# Check network interface
ip addr show usb0

# Manually configure IP
sudo ip addr add 192.168.2.10/24 dev usb0
sudo ip link set usb0 up
```

**Windows**:
- Install libiio drivers from: https://github.com/analogdevicesinc/libiio/releases
- Reboot after installation

### "No such device" Error

```python
# Try these alternatives:
sdr = adi.Pluto("ip:192.168.2.1")      # Network (preferred)
sdr = adi.Pluto("usb:1.2.5")           # USB (auto-detect)
sdr = adi.Pluto()                       # First available device
```

### Import Error: "No module named 'adi'"

```bash
# Install PyADI-IIO
pip3 install pyadi-iio

# If that fails, try:
pip3 install --user pyadi-iio

# Verify installation
python3 -c "import adi; print(adi.__version__)"
```

### Low Performance / Dropped Samples

```python
# Increase buffer size
sdr.rx_buffer_size = 1024 * 64  # Larger buffer

# Reduce sample rate
sdr.sample_rate = int(1e6)      # 1 MSPS instead of 2.4

# Use manual gain control
sdr.gain_control_mode = "manual"
sdr.rx_hardwaregain = 50        # 50 dB gain
```

### PlutoSDR Web Interface Not Loading

```bash
# Check if web server is running on PlutoSDR
ssh root@192.168.2.1
ps | grep httpd

# Restart web server if needed
killall lighttpd
/usr/sbin/lighttpd -f /etc/lighttpd.conf
```

---

## Next Steps

### 1. Start with LAB 0

Open and work through:
**[docs/labs/module1_foundations/LAB_0_HELLO_PLUTOSDR.md](docs/labs/module1_foundations/LAB_0_HELLO_PLUTOSDR.md)**

This lab covers:
- Detailed PlutoSDR setup
- Understanding I/Q data
- Your first transmit/receive experiments
- All three implementation methods

### 2. Explore the Documentation

**Main documentation index**: [docs/README.md](docs/README.md)
- Complete curriculum overview
- 13 labs, 4 projects, 6 guides
- 55,600+ lines of tutorials and code

### 3. Join the Community

- **PlutoSDR Wiki**: https://wiki.analog.com/university/tools/pluto
- **Analog Devices Forum**: https://ez.analog.com/
- **GitHub Issues**: Report problems or suggest improvements

### 4. Dive Deeper

**For firmware developers**:
- [PlutoSDR Build Scripts Guide](docs/guides/PLUTOSDR_BUILD_SCRIPTS.md)
- Complete FPGA, FSBL, U-Boot, Linux kernel compilation

**For application developers**:
- [PlutoSDR Custom Applications](docs/guides/PLUTOSDR_CUSTOM_APPS.md)
- Writing optimized C code for ARM Cortex-A9

**For comprehensive learning**:
- [Three Method Approach](docs/guides/THREE_METHOD_APPROACH.md)
- When to use GNU Radio vs Python vs C

---

## Quick Wins: 5-Minute Experiments

### Experiment 1: Spectrum Analyzer

```python
import adi
import numpy as np

sdr = adi.Pluto("ip:192.168.2.1")
sdr.sample_rate = int(2.4e6)
sdr.rx_lo = int(2.4e9)  # WiFi band
sdr.gain_control_mode = "slow_attack"

samples = sdr.rx()
fft = np.fft.fftshift(np.abs(np.fft.fft(samples)))
peak_idx = np.argmax(fft)
peak_freq = sdr.rx_lo + (peak_idx - len(fft)//2) * sdr.sample_rate / len(fft)
print(f"Strongest signal at: {peak_freq/1e6:.2f} MHz")
```

### Experiment 2: Signal Strength Monitor

```python
import adi
import numpy as np
import time

sdr = adi.Pluto("ip:192.168.2.1")
sdr.sample_rate = int(2.4e6)
sdr.rx_lo = int(915e6)  # Your frequency of interest

while True:
    samples = sdr.rx()
    power_dbfs = 10*np.log10(np.mean(np.abs(samples)**2))
    print(f"Signal power: {power_dbfs:.1f} dBFS")
    time.sleep(0.5)
```

### Experiment 3: Frequency Scanner

```python
import adi
import numpy as np

sdr = adi.Pluto("ip:192.168.2.1")
sdr.sample_rate = int(2.4e6)
sdr.gain_control_mode = "slow_attack"

# Scan FM band (88-108 MHz)
for freq in range(88, 108, 1):
    sdr.rx_lo = int(freq * 1e6)
    samples = sdr.rx()
    power = 10*np.log10(np.mean(np.abs(samples)**2))
    print(f"{freq:3d} MHz: {'█' * int(power + 40)} {power:.1f} dBFS")
```

---

## Cheat Sheet: PlutoSDR Parameters

### Frequency Ranges

| Parameter | Min | Max | Notes |
|-----------|-----|-----|-------|
| RX LO | 70 MHz | 6 GHz | Can be extended to 325 MHz - 3.8 GHz (default) |
| TX LO | 47 MHz | 6 GHz | Can be extended |
| Sample Rate | 65.105 ksps | 61.44 Msps | Configurable |
| RF Bandwidth | 200 kHz | 56 MHz | Analog filter |

### Gain Settings

| Mode | Description | Range |
|------|-------------|-------|
| Manual | Fixed gain | 0-74 dB (RX) |
| Slow Attack | AGC slow response | Auto |
| Fast Attack | AGC fast response | Auto |
| Hybrid | AGC + manual threshold | Auto + threshold |

### Buffer Sizes

| Application | Recommended Buffer Size |
|-------------|------------------------|
| Spectrum Analysis | 1024 - 4096 samples |
| Data Streaming | 16384 - 65536 samples |
| Burst Capture | 131072+ samples |

### TX Power

| Setting | Typical Output Power |
|---------|---------------------|
| 0 dB | ~5 dBm |
| -10 dB | ~-5 dBm |
| -20 dB | ~-15 dBm |
| -30 dB | ~-25 dBm |

---

## Resources

### Official Documentation
- **PlutoSDR Wiki**: https://wiki.analog.com/university/tools/pluto
- **PyADI-IIO Documentation**: https://analogdevicesinc.github.io/pyadi-iio/
- **AD9361 Datasheet**: https://www.analog.com/en/products/ad9361.html

### Additional Learning
- **GNU Radio Tutorials**: https://wiki.gnuradio.org/index.php/Tutorials
- **DSP Guide**: http://www.dspguide.com/
- **Software Defined Radio Academy**: https://www.sdr-academy.com/

### Community
- **Analog Devices EngineerZone**: https://ez.analog.com/
- **PlutoSDR subreddit**: https://www.reddit.com/r/RTLSDR/
- **GitHub Repository**: https://github.com/analogdevicesinc/plutosdr-fw

---

## Summary

You've completed the quick start! You should now have:
- ✅ PlutoSDR connected and verified
- ✅ Software installed (Python, PyADI-IIO)
- ✅ First SDR programs running
- ✅ Understanding of curriculum structure

**Next**: Start **[LAB 0: Hello PlutoSDR](docs/labs/module1_foundations/LAB_0_HELLO_PLUTOSDR.md)** for a comprehensive introduction.

**Questions?** Check the troubleshooting section or visit the [PlutoSDR Wiki](https://wiki.analog.com/university/tools/pluto).

---

**Version**: 1.0
**Last Updated**: December 3, 2025
**Maintainer**: PlutoSDR Educational Documentation Team

Happy SDR learning! 📡🎓
