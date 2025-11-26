# PlutoSDR Training Curriculum

**Comprehensive training materials for Software-Defined Radio with ADALM-PlutoSDR**

---

## 📚 Repository Structure

```
docs/
│
├── README.md (this file)
│
├── labs/                          # Hands-on laboratory exercises
│   ├── module1_foundations/       # Module 1: SDR Foundations (✅ Complete)
│   ├── module2_sampling/          # Module 2: Sampling Theory (✅ Complete)
│   ├── module3_modulation/        # Module 3: Digital Modulation (✅ Complete)
│   ├── module4_dsp_filtering/     # Module 4: DSP and Filtering (⏳ Planned)
│   ├── module5_fec/               # Module 5: Forward Error Correction (⏳ Planned)
│   └── module6_synchronization/   # Module 6: Synchronization (⏳ Planned)
│
├── projects/                      # Advanced integration projects
│   ├── PROJECT2_IOT_SATELLITE_ENHANCED.md  (✅ Complete)
│   ├── PROJECT3_SECURE_VIDEO_ENHANCED.md   (✅ Complete)
│   └── PROJECT4_FREQUENCY_HOPPING.md       (✅ Complete)
│
├── analysis/                      # Coverage analysis and status tracking
│   ├── SDR_TERM_COVERAGE_ANALYSIS.md
│   ├── CURRICULUM_STATUS.md
│   ├── SESSION_SUMMARY.md
│   └── SESSION_COMPLETION_SUMMARY.md
│
└── guides/                        # Reference guides and documentation
    ├── PLUTOSDR_CUSTOM_APPS.md
    ├── PLUTOSDR_TRAINING_PART1.md
    ├── PLUTOSDR_TRAINING_PART2.md
    └── THREE_METHOD_APPROACH.md
```

---

## 🎓 Learning Path

### Module 1: SDR Foundations (✅ Complete)

**Location**: `labs/module1_foundations/`
**Prerequisites**: Basic programming (Python or C), understanding of complex numbers

| Lab | Title | Duration | Methods |
|-----|-------|----------|---------|
| LAB 0 | Hello PlutoSDR | 1-2 hours | Python + PlutoSDR + C |
| LAB 1.1 | SDR Flexibility | 2-3 hours | Python + PlutoSDR + C |
| LAB 1.2 | RF Gain Staging | 2-3 hours | Python + PlutoSDR |
| LAB 1.3 | I/Q Samples | 3-4 hours | Python + PlutoSDR |

**Key Skills Learned**:
- PlutoSDR setup and configuration
- I/Q sampling fundamentals
- RF gain control and AGC
- Basic signal generation

---

### Module 2: Sampling Theory (✅ Complete)

**Location**: `labs/module2_sampling/`
**Prerequisites**: Module 1, basic calculus

| Lab | Title | Duration | Coverage |
|-----|-------|----------|----------|
| LAB 2.1 | Nyquist Sampling, Aliasing, Bandwidth | 3-4 hours | 1,048 lines |
| LAB 2.2 | Decimation, Interpolation, Sample Rate Conversion | 4-5 hours | 1,369 lines |
| LAB 2.3 | Quantization and ADC Resolution | 3-4 hours | 1,101 lines |

**Key Skills Learned**:
- Nyquist-Shannon theorem application
- Aliasing detection and prevention
- Multi-stage decimation/interpolation
- Quantization noise analysis
- ENOB calculations

**SDR Terms Covered**: Nyquist rate, aliasing, decimation, interpolation, polyphase filters, quantization, SNR formula (6.02N+1.76 dB), ENOB, dynamic range, dithering

---

### Module 3: Digital Modulation (✅ Complete)

**Location**: `labs/module3_modulation/`
**Prerequisites**: Modules 1-2, complex numbers, basic probability

| Lab | Title | Duration | Coverage |
|-----|-------|----------|----------|
| LAB 3.1 | Digital Modulation - ASK, FSK, PSK | 4-5 hours | 1,206 lines |
| LAB 3.2 | QPSK and 8-PSK Modulation | 3-4 hours | 1,147 lines |
| LAB 3.3 | QAM Modulation (16/64/256-QAM) | 4-5 hours | 937 lines |
| LAB 3.4 | BER Testing and Eye Diagrams | 3-4 hours | 769 lines |
| LAB 3.5 | Pulse Shaping and Matched Filtering | 4-5 hours | 709 lines |

**Key Skills Learned**:
- Complete modulation suite implementation
- Gray coding for M-ary modulations
- BER testing and performance analysis
- Eye diagram interpretation
- Pulse shaping for ISI control

**SDR Terms Covered**: ASK/OOK, FSK, BPSK, QPSK, 8-PSK, 16-QAM, 64-QAM, 256-QAM, constellation diagrams, BER, SER, Eb/N0, eye diagrams, Q-factor, raised cosine, root-raised cosine, matched filtering, ISI

---

### Module 4: DSP and Filtering (⏳ Planned)

**Location**: `labs/module4_dsp_filtering/`
**Prerequisites**: Module 2, z-transforms

| Lab | Title | Status |
|-----|-------|--------|
| LAB 4.1 | FIR and IIR Filter Design | ⏳ Pending |
| LAB 4.2 | Window Functions and Spectral Leakage | ⏳ Pending |
| LAB 4.3 | FFT Analysis and Spectrograms | ⏳ Pending |
| LAB 4.4 | Digital Filter Applications | ⏳ Pending |

**Planned Learning**: FIR/IIR design, Butterworth, Chebyshev, window functions, FFT, spectrograms

---

### Module 5: Forward Error Correction (⏳ Planned)

**Location**: `labs/module5_fec/`
**Prerequisites**: Module 3

| Lab | Title | Status |
|-----|-------|--------|
| LAB 4.5 | Hamming and Block Codes | ⏳ Pending |
| LAB 4.6 | Convolutional Codes and Viterbi | ⏳ Pending |
| LAB 4.7 | Reed-Solomon Codes | ✅ Covered in PROJECT 3 |

**Planned Learning**: Error detection/correction, coding gain, Viterbi decoding

---

### Module 6: Synchronization (⏳ Planned)

**Location**: `labs/module6_synchronization/`
**Prerequisites**: Modules 3-4

| Lab | Title | Status |
|-----|-------|--------|
| LAB 4.8 | Carrier Phase Recovery (Costas Loop) | ⏳ Pending |
| LAB 4.9 | Symbol Timing Recovery | ⏳ Pending |
| LAB 4.10 | PLL Design and Tracking | ⏳ Pending |

**Planned Learning**: Carrier recovery, timing recovery, PLL design

---

## 🚀 Advanced Projects

**Location**: `projects/`

### PROJECT 2: IoT-Satellite DSSS Link (✅ Complete)
- **Lines**: 2,074
- **Technologies**: STM32WL33 + PlutoSDR, DSSS with Gold codes, OFDM
- **Difficulty**: Expert
- **Duration**: 10-15 hours

### PROJECT 3: Secure Video Streaming (✅ Complete)
- **Lines**: 2,914
- **Technologies**: Raspberry Pi + Camera, AES-256-GCM, 64-QAM OFDM, Reed-Solomon FEC
- **Difficulty**: Expert
- **Duration**: 12-18 hours

### PROJECT 4: Frequency-Hopping Datalink (✅ Complete)
- **Lines**: 1,641
- **Technologies**: Dual PlutoSDRs, FHSS (500 hops/sec), FSK, anti-jamming
- **Difficulty**: Expert
- **Duration**: 8-12 hours

---

## 📊 Curriculum Statistics

### Current Status
- **Total Labs Created**: 13 labs
- **Total Projects**: 3 projects
- **Total Documentation**: ~18,000 lines
- **SDR Terms Covered**: 120+ / 170 (71%)
- **Categories at 100%**: Sampling Theory, Digital Modulation
- **Average Lab Length**: 1,035 lines
- **Code-to-Theory Ratio**: 60% code, 40% theory

### Coverage by Category
1. ✅ **Sampling/Conversion** - 100%
2. ✅ **Digital Modulation** - 100%
3. 🟡 **DSP/Filtering** - 40% (planned expansion)
4. 🟡 **FEC** - 25% (Reed-Solomon complete)
5. 🟡 **Synchronization** - 30% (basic concepts covered)

---

## 🛠️ Three-Method Approach

Each lab implements up to **three methods**:

### Method 1: Pure Python Simulation
- No hardware required
- Mathematical/algorithmic focus
- Rapid prototyping and learning
- ~400-600 lines per lab

### Method 2: PlutoSDR as External Device
- Python + `pyadi-iio` library
- USB/Ethernet connection to PlutoSDR
- Real RF hardware validation
- ~200-300 lines per lab

### Method 3: Compiled C on PlutoSDR
- Cross-compiled for ARM Cortex-A9
- Runs directly on PlutoSDR Linux
- Maximum performance
- ~300-400 lines per lab

**Current Status**:
- Methods 1 & 2: ✅ Complete for all labs
- Method 3: ✅ Complete for LAB 0, 1.1 | ⏳ Pending for LAB 1.2-3.5

---

## 🔧 Quick Start

### Hardware Requirements
- ADALM-PlutoSDR (Rev B/C/D) - $150 USD
- USB 2.0/3.0 cable
- Host PC (Windows/Linux/Mac)
- Optional: Second PlutoSDR, RF attenuators, antennas

### Software Installation

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install python3 python3-pip libiio-utils

# Python libraries
pip3 install pyadi-iio numpy scipy matplotlib

# Verify installation
python3 -c "import adi; print('✓ pyadi-iio installed')"
```

### Test PlutoSDR Connection

```python
import adi

sdr = adi.Pluto("ip:192.168.2.1")
print(f"Sample rate: {sdr.sample_rate/1e6:.2f} MSPS")
print(f"✓ PlutoSDR connected successfully!")
```

---

## 📖 How to Navigate This Curriculum

### For Self-Study

1. Start with **Module 1** (Foundations)
2. Progress sequentially through modules
3. Complete all labs in each module before moving on
4. Attempt projects after completing relevant modules
5. Use `analysis/` folder to track your progress

### For Instructors

- **Modular design**: Each module is self-contained
- **2-3 weeks per module** recommended
- **Assessment ready**: Clear learning objectives
- **Project-based learning**: Capstone projects integrate multiple modules

### For Researchers

- Jump to relevant modules based on your research needs
- Use projects as templates for custom applications
- Reference `guides/` for architecture details

---

## 📈 Progress Tracking

Use the `analysis/` folder to monitor your progress:

- **CURRICULUM_STATUS.md**: Overall completion tracking
- **SDR_TERM_COVERAGE_ANALYSIS.md**: Detailed term-by-term coverage
- **SESSION_COMPLETION_SUMMARY.md**: Development roadmap and recommendations

---

## 🎯 Learning Objectives by Module

### Module 1: Foundations
✅ Configure PlutoSDR
✅ Understand I/Q representation
✅ Control RF gain and AGC
✅ Generate/receive basic signals

### Module 2: Sampling
✅ Apply Nyquist criterion
✅ Prevent aliasing
✅ Implement sample rate conversion
✅ Analyze quantization effects

### Module 3: Modulation
✅ Implement ASK/FSK/PSK/QAM
✅ Design Gray-coded constellations
✅ Measure BER vs. Eb/N0
✅ Apply pulse shaping
✅ Interpret eye diagrams

### Module 4-6: Advanced (Planned)
⏳ Design FIR/IIR filters
⏳ Implement FEC codes
⏳ Build synchronization loops

---

## 📚 External Resources

### Official Documentation
- [PlutoSDR Wiki](https://wiki.analog.com/university/tools/pluto)
- [AD9361 Datasheet](https://www.analog.com/en/products/ad9361.html)
- [pyadi-iio Docs](https://analogdevicesinc.github.io/pyadi-iio/)

### Textbooks
- "Software Defined Radio for Engineers" - Collins et al.
- "Digital Communications" - Proakis & Salehi
- "Understanding Digital Signal Processing" - Lyons

---

## 🤝 Contributing

This curriculum is continuously evolving:

- **Bug fixes**: Correct errors in code/theory
- **Enhancements**: Improve explanations
- **New labs**: Expand SDR topic coverage
- **Translations**: Make content accessible globally

---

## 📞 Support

- Review lab troubleshooting sections
- Check PlutoSDR wiki and forums
- Consult DSP Stack Exchange for theory

---

## 🎓 Credits

**Developed by**: Claude (Anthropic) with user collaboration
**Platform**: ADALM-PlutoSDR by Analog Devices
**Community**: Open-source SDR community

---

**Last Updated**: November 26, 2025
**Version**: 2.0 - Reorganized Structure
**Status**: 71% complete, ongoing expansion
