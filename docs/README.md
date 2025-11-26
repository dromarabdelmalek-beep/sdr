# PlutoSDR Complete Professional Training Documentation

## Welcome to the PlutoSDR Training Course

This comprehensive training course covers all aspects of Software-Defined Radio (SDR) using the **Analog Devices ADALM-PlutoSDR** platform. The course includes detailed theory, PlutoSDR-specific implementation details, and hands-on laboratory exercises with complete code examples.

---

## 📚 Documentation Structure

### 1. **PLUTOSDR_ARCHITECTURE.md**
Complete firmware and gateware architecture guide

**Contents:**
- System Overview and Block Diagrams
- Hardware Architecture (Zynq SoC + AD9363)
- Firmware Components and Build System
- FPGA Gateware Architecture
- Software Stack (Linux IIO Framework)
- Boot Process and Memory Maps
- Complete RX/TX Signal Processing Chain

**Who should read this:**
- System architects
- Firmware developers
- Anyone wanting deep understanding of PlutoSDR internals

---

### 2. **PLUTOSDR_CUSTOM_APPS.md**
Development guide for custom applications

**Contents:**
- **Hosted Applications** (running on PlutoSDR Linux)
  - Cross-compilation setup
  - C/C++/Python examples
  - Adding apps to buildroot
  - GPIO/SPI integration

- **External Applications** (running on host PC)
  - libiio (C/Python)
  - pyadi-iio (Python high-level API)
  - GNU Radio flowgraphs
  - SoapySDR integration
  - MATLAB interface

- **Custom FPGA Gateware**
  - When to modify FPGA
  - Vivado workflow
  - Adding custom IP cores

- **Example Projects**
  - FM radio receiver
  - Spectrum analyzer
  - Custom modems

**Who should read this:**
- Application developers
- GNU Radio users
- Embedded systems developers
- FPGA engineers

---

### 3. **PLUTOSDR_TRAINING_PART1.md**
Core SDR concepts with hands-on labs (Beginner to Intermediate)

**Contents:**

#### Section 1: Core SDR Concepts
- **LAB 1.1:** SDR Flexibility - Frequency Hopping Demo
- **LAB 1.2:** RF Front-End Gain Staging
- **LAB 1.3:** I/Q Samples - Complex Baseband
- **LAB 1.4:** Nyquist Sampling and Aliasing
- **LAB 1.5:** Dynamic Range and SNR Measurement
- **LAB 1.6:** AGC Performance Comparison

**Concepts Covered:**
- SDR vs traditional radio
- RF front-end components (LNA, mixer, filters)
- I/Q sampling (complex baseband)
- Sample rate and bandwidth
- Dynamic range, SNR, noise figure
- Automatic Gain Control (AGC)

#### Section 2: Digital Signal Processing
- **LAB 2.1:** Decimation and Interpolation
- **LAB 2.2:** FIR vs IIR Filter Comparison
- **LAB 2.3:** FFT Analysis and Windowing

**Concepts Covered:**
- Multi-rate signal processing
- Filter design (FIR/IIR)
- FFT for spectrum analysis
- Window functions (Hann, Hamming, Blackman)
- Frequency resolution vs time resolution

**Who should read this:**
- SDR beginners
- Students learning DSP
- Anyone new to PlutoSDR

---

### 4. **PLUTOSDR_TRAINING_PART2.md**
Advanced modulation, coding, and applications (Intermediate to Advanced)

**Contents:**

#### Section 3: Modulation & Waveforms
- **LAB 3.1:** BPSK Transmitter and Receiver
- **LAB 3.2:** QPSK with Gray Coding
- **LAB 3.3:** 16-QAM Implementation
- **LAB 3.4:** OFDM System (WiFi/LTE-like)

**Concepts Covered:**
- Phase shift keying (BPSK, QPSK, 8-PSK)
- Quadrature amplitude modulation (16/64/256-QAM)
- OFDM (multicarrier modulation)
- Constellation diagrams and EVM
- Pulse shaping (RRC filters)
- Spread spectrum (DSSS, FHSS)

#### Section 4: Coding, FEC, and Framing
- **LAB 4.1:** Frame Detection with Preamble
- **LAB 4.2:** CRC Error Detection
- **LAB 4.3:** Convolutional Coding
- **LAB 4.4:** LDPC Codes

**Concepts Covered:**
- Frame structure design
- Preambles (Barker, Zadoff-Chu)
- Error detection (CRC)
- Forward error correction (FEC)
- Convolutional codes with Viterbi decoding
- LDPC and Turbo codes
- Interleaving and scrambling

**Who should read this:**
- Wireless protocol designers
- Digital communications engineers
- Advanced SDR users

---

## 🎯 Learning Paths

### Path 1: Beginner - First Steps with PlutoSDR
1. Read **PLUTOSDR_ARCHITECTURE.md** (Sections 1-2)
2. Read **PLUTOSDR_CUSTOM_APPS.md** (Section 4 - pyadi-iio)
3. Complete **PLUTOSDR_TRAINING_PART1.md** Labs 1.1-1.6

**Time:** 1-2 weeks
**Outcome:** Understand PlutoSDR basics, capture and analyze RF signals

### Path 2: Intermediate - DSP and Modulation
1. Complete **PLUTOSDR_TRAINING_PART1.md** Section 2 (DSP)
2. Complete **PLUTOSDR_TRAINING_PART2.md** Section 3 (Modulation)
3. Read **PLUTOSDR_CUSTOM_APPS.md** (GNU Radio section)

**Time:** 2-3 weeks
**Outcome:** Implement digital modems, understand wireless standards

### Path 3: Advanced - Custom Applications
1. Read **PLUTOSDR_ARCHITECTURE.md** (complete)
2. Read **PLUTOSDR_CUSTOM_APPS.md** Section 3 (Hosted apps)
3. Complete **PLUTOSDR_TRAINING_PART2.md** Section 4 (Coding/FEC)
4. Study **PLUTOSDR_CUSTOM_APPS.md** Section 6 (FPGA)

**Time:** 4-6 weeks
**Outcome:** Develop standalone SDR applications, FPGA acceleration

### Path 4: Expert - Specialized Applications
1. Complete all previous paths
2. Study advanced topics (Radar, Satellite, Waveform Engineering)
3. Design custom PHY layers
4. Implement FPGA-accelerated DSP

**Time:** 2-3 months
**Outcome:** Design and deploy custom SDR systems

---

## 🔬 Laboratory Setup

### Required Hardware
- **ADALM-PlutoSDR** (Rev B, C, or D)
- **Host PC** (Windows, Linux, or Mac)
- **USB cable** (included with PlutoSDR)
- **SMA antenna** (or 50Ω termination for loopback tests)
- *Optional:* Second PlutoSDR for full-duplex testing

### Required Software
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install python3 python3-pip libiio-utils

# Python libraries
pip3 install pyadi-iio numpy scipy matplotlib

# Optional: GNU Radio
sudo apt install gnuradio gr-iio

# Optional: SoapySDR
sudo apt install soapysdr-tools soapysdr-module-remote
```

### PlutoSDR Configuration
1. Connect PlutoSDR via USB
2. Access web interface: http://192.168.2.1
3. Default credentials: `root` / `analog`
4. Verify IP address: `192.168.2.1` (PlutoSDR), `192.168.2.10` (host)

### Test Installation
```python
import adi
sdr = adi.Pluto("ip:192.168.2.1")
print(f"Sample rate: {sdr.sample_rate}")
print("✓ PlutoSDR connected successfully!")
```

---

## 📖 Concept Reference - Complete Glossary

### Core SDR Concepts
| Term | Definition | PlutoSDR Implementation |
|------|------------|------------------------|
| **SDR** | Software-Defined Radio | Zynq SoC + AD9363 transceiver |
| **RF Front-End** | Analog RX/TX chain | AD9363 (LNA, mixers, filters) |
| **I/Q Samples** | Complex baseband representation | 16-bit signed integer pairs |
| **Sample Rate** | Samples per second | 520 kSPS - 61.44 MSPS |
| **Bandwidth** | Usable RF spectrum | 200 kHz - 56 MHz |
| **Dynamic Range** | Noise floor to saturation | ~60-65 dB (12-bit ADC) |
| **AGC** | Automatic gain control | Fast/slow/hybrid modes |

### DSP Concepts
| Term | Definition | PlutoSDR Implementation |
|------|------------|------------------------|
| **Decimation** | Reduce sample rate | AD9363 FIR filters (programmable) |
| **FIR Filter** | Finite impulse response | AD9361 TX/RX FIR (128 taps) |
| **FFT** | Fast Fourier Transform | Software (NumPy/FFTW) or FPGA |
| **Windowing** | Spectral leakage reduction | Hann, Hamming, Blackman windows |

### Modulation Schemes
| Modulation | Bits/Symbol | Complexity | Use Cases |
|------------|-------------|------------|-----------|
| **BPSK** | 1 | Low | Satellite, long-range |
| **QPSK** | 2 | Medium | Satellite, LTE |
| **16-QAM** | 4 | High | WiFi, LTE |
| **64-QAM** | 6 | Very High | WiFi 802.11ac, LTE |
| **OFDM** | Variable | High | WiFi, LTE, 5G, DAB |

### Hardware Components
| Component | Specification | Function |
|-----------|---------------|----------|
| **Zynq-7000** | XC7Z010 SoC | Dual ARM Cortex-A9 + FPGA |
| **AD9363** | RF transceiver | 325 MHz - 3.8 GHz, 12-bit ADC/DAC |
| **DDR3** | 512 MB | DMA buffers, Linux memory |
| **USB 2.0** | 480 Mbps | Data + control interface |

---

## 🛠️ Common Tasks - Quick Reference

### Capture I/Q Samples
```python
import adi
sdr = adi.Pluto("ip:192.168.2.1")
sdr.rx_lo = 915e6  # 915 MHz center frequency
sdr.sample_rate = 2.084e6  # 2.084 MSPS
sdr.rx_buffer_size = 16384
sdr.rx_hardwaregain_chan0 = 60  # RX gain in dB

samples = sdr.rx()  # Capture samples
# samples is complex numpy array (I + jQ)
```

### Transmit Signal
```python
import numpy as np
# Generate test tone
fs = 2.084e6
t = np.arange(0, 0.001, 1/fs)
tone = np.exp(2j * np.pi * 100e3 * t)  # 100 kHz tone
tone_scaled = (tone * 0.8 * 2**14).astype(np.int16)

sdr.tx_lo = 915e6
sdr.tx_hardwaregain_chan0 = -10  # TX attenuation (dB)
sdr.tx_cyclic_buffer = True
sdr.tx(tone_scaled)
```

### Configure RF Parameters
```python
# Frequency (70 MHz - 6 GHz)
sdr.rx_lo = 433e6  # RX local oscillator
sdr.tx_lo = 433e6  # TX local oscillator

# Sample rate (520 kSPS - 61.44 MSPS)
sdr.sample_rate = 4e6  # 4 MSPS

# RF bandwidth (200 kHz - 56 MHz)
sdr.rx_rf_bandwidth = 4e6  # RX analog filter

# Gain (RX: 0-73 dB, TX: -89.75 to 0 dB)
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 50
sdr.tx_hardwaregain_chan0 = -20
```

### Spectrum Analysis
```python
import matplotlib.pyplot as plt

samples = sdr.rx()
fft = np.fft.fftshift(np.fft.fft(samples))
freqs = np.fft.fftshift(np.fft.fftfreq(len(samples), 1/sdr.sample_rate))
psd = 20 * np.log10(np.abs(fft) / len(fft))

plt.plot(freqs/1e6, psd)
plt.xlabel('Frequency (MHz)')
plt.ylabel('Power (dB)')
plt.grid(True)
plt.show()
```

---

## 🚀 Example Projects

### 1. FM Radio Receiver
**Difficulty:** Beginner
**Files:** See PLUTOSDR_CUSTOM_APPS.md Section 7.1
**Concepts:** Wide-band FM, audio demodulation, filtering

### 2. Spectrum Analyzer
**Difficulty:** Beginner
**Files:** See PLUTOSDR_CUSTOM_APPS.md Section 7.2
**Concepts:** FFT, real-time plotting, frequency scanning

### 3. BPSK/QPSK Modem
**Difficulty:** Intermediate
**Files:** See PLUTOSDR_TRAINING_PART2.md Labs 3.1-3.2
**Concepts:** Digital modulation, matched filtering, synchronization

### 4. OFDM Transceiver
**Difficulty:** Advanced
**Files:** See PLUTOSDR_TRAINING_PART2.md Lab 3.4
**Concepts:** Multicarrier, IFFT/FFT, pilot symbols, cyclic prefix

### 5. LoRa-like Chirp Modulation
**Difficulty:** Advanced
**Concepts:** Chirp spread spectrum, frequency hopping, low SNR detection

### 6. Passive Radar
**Difficulty:** Expert
**Concepts:** Cross-correlation, Doppler processing, range-Doppler maps

---

## 📊 SDR Concept Map

```
┌─────────────────────────────────────────────────────────────────┐
│                         SDR Fundamentals                        │
└─────────────────┬───────────────────────────────────────────────┘
                  │
      ┌───────────┴───────────┬──────────────┬──────────────────┐
      │                       │              │                  │
┌─────▼─────┐          ┌──────▼─────┐  ┌─────▼──────┐   ┌──────▼────────┐
│ Hardware  │          │    DSP     │  │ Modulation │   │    Coding     │
└─────┬─────┘          └──────┬─────┘  └─────┬──────┘   └──────┬────────┘
      │                       │              │                  │
┌─────▼──────────┐   ┌────────▼────────┐  ┌─▼────────────┐  ┌──▼─────────┐
│ • RF Front-End │   │ • Filters       │  │ • BPSK/QPSK  │  │ • CRC      │
│ • ADC/DAC      │   │ • FFT/IFFT      │  │ • QAM        │  │ • FEC      │
│ • LNA/PA       │   │ • Decimation    │  │ • OFDM       │  │ • Viterbi  │
│ • Mixer        │   │ • Interpolation │  │ • FSK        │  │ • LDPC     │
│ • PLL          │   │ • Windowing     │  │ • Spread     │  │ • Turbo    │
└────────────────┘   └─────────────────┘  └──────────────┘  └────────────┘
```

---

## 🤝 Contributing and Support

### Community Resources
- **ADI PlutoSDR Wiki:** https://wiki.analog.com/university/tools/pluto
- **GitHub Issues:** https://github.com/analogdevicesinc/plutosdr-fw/issues
- **ADI EngineerZone:** https://ez.analog.com/
- **GNU Radio Mailing List:** discuss-gnuradio@gnu.org

### Reporting Issues
For issues with this training material:
1. Check if concept is unclear or code has errors
2. Open an issue with specific lab number and error description
3. Include PlutoSDR firmware version and host OS

### Extending This Course
To add new labs or sections:
1. Follow existing lab format (Theory → Code → Expected Results)
2. Test all code with PlutoSDR
3. Include plots/visualizations
4. Document prerequisites

---

## 📝 License and Disclaimer

This training material is provided for educational purposes.

**Hardware:**
- ADALM-PlutoSDR is a product of Analog Devices, Inc.
- Ensure compliance with local regulations for RF transmission

**Software:**
- Code examples use open-source libraries (libiio, NumPy, SciPy)
- Modify and distribute freely for educational use

**Safety:**
- Always use appropriate RF attenuation
- Do not transmit without proper licensing
- Follow FCC/local regulations

---

## 🎓 Certification Path (Self-Study)

### Level 1: SDR Fundamentals (Complete Training Part 1)
- Understand I/Q sampling
- Configure PlutoSDR parameters
- Capture and analyze RF signals
- Implement basic DSP (filtering, FFT)

### Level 2: Digital Communications (Complete Training Part 2)
- Implement BPSK/QPSK/QAM modems
- Design frame structures
- Implement error detection/correction
- Build OFDM transceiver

### Level 3: Advanced Applications
- Develop custom protocols
- FPGA acceleration
- Real-time processing
- System integration

---

## 📞 Quick Start Checklist

- [ ] Connect PlutoSDR to PC via USB
- [ ] Install Python and pyadi-iio
- [ ] Test connection: `import adi; sdr = adi.Pluto("ip:192.168.2.1")`
- [ ] Read PLUTOSDR_ARCHITECTURE.md (Section 1)
- [ ] Complete LAB 1.1 (Frequency Hopping Demo)
- [ ] Capture your first I/Q samples
- [ ] Plot your first spectrum
- [ ] Transmit your first signal

**Welcome to the exciting world of Software-Defined Radio!**

---

*Last Updated: 2025-11*
*PlutoSDR Firmware Version: v0.38+*
*Compatible with: PlutoSDR Rev B, C, D*
