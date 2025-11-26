# SDR Term Coverage Analysis

## Overview

This document analyzes coverage of **150+ SDR terms** across 14 categories in the PlutoSDR training curriculum. Each term is mapped to existing labs/projects and gaps are identified for new lab development.

**Curriculum Status:**
- ✅ **Covered:** Term has dedicated lab or substantial project coverage
- 🟡 **Partial:** Term mentioned but needs deeper practical example
- ❌ **Missing:** Term not covered, needs new lab

---

## Category 1: RF Fundamentals (20 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 1 | **Frequency** | ✅ | LAB 0, 1.1, All Projects | Frequency tuning in all labs |
| 2 | **Wavelength** | 🟡 | LAB 1.1 (theory) | Need practical calculation lab |
| 3 | **Amplitude** | ✅ | LAB 1.2 (RF Gain) | Signal amplitude control |
| 4 | **Phase** | ✅ | LAB 1.3 (I/Q) | Phase representation in complex plane |
| 5 | **Bandwidth** | 🟡 | All projects (implicit) | Need dedicated BW measurement lab |
| 6 | **Center Frequency** | ✅ | LAB 0, All projects | RF tuning parameter |
| 7 | **Carrier Wave** | ✅ | LAB 0 (tone), PROJECT 3/4 | Modulation carrier |
| 8 | **Modulation** | ✅ | PROJECT 2 (BPSK), 3 (QAM), 4 (FSK) | Multiple schemes |
| 9 | **Demodulation** | ✅ | All projects | RX side processing |
| 10 | **Signal-to-Noise Ratio (SNR)** | 🟡 | LAB 1.2 (AGC) | Need SNR measurement lab |
| 11 | **Noise Figure (NF)** | ❌ | Not covered | Need noise analysis lab |
| 12 | **Attenuation** | ✅ | LAB 1.2, All projects | Gain staging |
| 13 | **Amplification** | ✅ | LAB 1.2 | LNA, baseband gain |
| 14 | **Antenna** | ✅ | All projects | Hardware setup |
| 15 | **Propagation** | 🟡 | PROJECT 2/3 (link budget) | Need propagation models lab |
| 16 | **Free-Space Path Loss (FSPL)** | ✅ | PROJECT 2/3 | Link budget calculations |
| 17 | **Reflection** | ❌ | Not covered | Need multipath lab |
| 18 | **Diffraction** | ❌ | Not covered | Need propagation lab |
| 19 | **Interference** | 🟡 | PROJECT 4 (jamming) | Need interference mitigation lab |
| 20 | **Harmonics** | ❌ | Not covered | Need spectrum purity lab |

**Coverage: 10/20 Complete, 5/20 Partial, 5/20 Missing**

**Required New Labs:**
- LAB 1.4: **Wavelength and Propagation Calculations**
- LAB 1.5: **Bandwidth and SNR Measurement**
- LAB 1.6: **Noise Figure and Sensitivity Analysis**
- LAB 2.4: **Multipath and Fading Effects**
- LAB 2.5: **Harmonic Analysis and Spectrum Purity**

---

## Category 2: Digital Modulation (18 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 21 | **Amplitude Shift Keying (ASK)** | ❌ | Not covered | Need basic modulation lab |
| 22 | **Frequency Shift Keying (FSK)** | ✅ | PROJECT 4 (FHSS) | Binary FSK implementation |
| 23 | **Phase Shift Keying (PSK)** | 🟡 | PROJECT 2 (BPSK theory) | Need practical PSK lab |
| 24 | **Binary PSK (BPSK)** | ✅ | PROJECT 2 (DSSS) | DSSS with BPSK |
| 25 | **Quadrature PSK (QPSK)** | ❌ | Not covered | Need QPSK lab |
| 26 | **8-PSK** | ❌ | Not covered | Need higher-order PSK lab |
| 27 | **Quadrature Amplitude Modulation (QAM)** | ✅ | PROJECT 3 (64-QAM OFDM) | High-order QAM |
| 28 | **16-QAM** | 🟡 | PROJECT 2/3 (mentioned) | Need standalone lab |
| 29 | **64-QAM** | ✅ | PROJECT 3 (OFDM) | Video link modulation |
| 30 | **256-QAM** | 🟡 | PROJECT 3 (optimization) | Mentioned as extension |
| 31 | **Constellation Diagram** | ✅ | PROJECT 3 (plots) | QAM constellation |
| 32 | **Symbol Rate** | ✅ | All projects | Data rate parameter |
| 33 | **Bit Error Rate (BER)** | 🟡 | PROJECT 4 (FSK test) | Need comprehensive BER lab |
| 34 | **Eye Diagram** | ❌ | Not covered | Need timing analysis lab |
| 35 | **Intersymbol Interference (ISI)** | ❌ | Not covered | Need pulse shaping lab |
| 36 | **Nyquist Filter** | ❌ | Not covered | Need filtering lab |
| 37 | **Raised Cosine Filter** | ❌ | Not covered | Need pulse shaping lab |
| 38 | **Root Raised Cosine (RRC)** | ❌ | Not covered | Need matched filtering lab |

**Coverage: 5/18 Complete, 5/18 Partial, 8/18 Missing**

**Required New Labs:**
- LAB 3.1: **ASK, FSK, PSK Comparison**
- LAB 3.2: **QPSK and 8-PSK Implementation**
- LAB 3.3: **QAM Modulation (16/64/256-QAM)**
- LAB 3.4: **BER Testing and Eye Diagrams**
- LAB 3.5: **Pulse Shaping and Matched Filtering**

---

## Category 3: I/Q Sampling and Complex Signals (12 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 39 | **In-Phase (I)** | ✅ | LAB 1.3 | Complex baseband I component |
| 40 | **Quadrature (Q)** | ✅ | LAB 1.3 | Complex baseband Q component |
| 41 | **Complex Baseband** | ✅ | LAB 1.3 | I+jQ representation |
| 42 | **I/Q Modulator** | ✅ | LAB 1.3 | Quadrature upconversion |
| 43 | **I/Q Demodulator** | ✅ | LAB 1.3 | Quadrature downconversion |
| 44 | **I/Q Imbalance** | ❌ | Not covered | Need calibration lab |
| 45 | **DC Offset** | ❌ | Not covered | Need impairments lab |
| 46 | **Quadrature Error** | ❌ | Not covered | Need calibration lab |
| 47 | **Image Rejection** | ❌ | Not covered | Need mixer analysis lab |
| 48 | **Complex Mixing** | ✅ | LAB 1.3 | Frequency translation |
| 49 | **Hilbert Transform** | ❌ | Not covered | Need SSB lab |
| 50 | **Analytic Signal** | 🟡 | LAB 1.3 (implicit) | Need signal processing lab |

**Coverage: 6/12 Complete, 1/12 Partial, 5/12 Missing**

**Required New Labs:**
- LAB 2.6: **I/Q Imbalance Correction and Calibration**
- LAB 2.7: **DC Offset and Image Rejection**
- LAB 2.8: **Hilbert Transform and SSB Generation**

---

## Category 4: Sampling and Conversion (10 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 51 | **Sampling Rate** | ✅ | LAB 1.3, All projects | Fs parameter |
| 52 | **Nyquist Rate** | 🟡 | LAB 1.3 (theory) | Need aliasing lab |
| 53 | **Aliasing** | 🟡 | LAB 1.3 (mentioned) | Need practical demo |
| 54 | **Anti-Aliasing Filter** | ❌ | Not covered | Need filter design lab |
| 55 | **Oversampling** | 🟡 | All projects (implicit) | Need decimation lab |
| 56 | **Undersampling** | ❌ | Not covered | Need advanced sampling lab |
| 57 | **Decimation** | ❌ | Not covered | Need multirate DSP lab |
| 58 | **Interpolation** | ❌ | Not covered | Need multirate DSP lab |
| 59 | **Sample Rate Conversion** | ❌ | Not covered | Need resampling lab |
| 60 | **Quantization** | ❌ | Not covered | Need ADC/DAC lab |

**Coverage: 1/10 Complete, 3/10 Partial, 6/10 Missing**

**Required New Labs:**
- LAB 2.1: **Nyquist Sampling and Aliasing Demonstration**
- LAB 2.2: **Decimation and Interpolation**
- LAB 2.3: **Quantization and ADC Resolution**

---

## Category 5: Filtering and Signal Processing (15 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 61 | **Low-Pass Filter (LPF)** | 🟡 | PROJECT 3 (OFDM, implicit) | Need filter design lab |
| 62 | **High-Pass Filter (HPF)** | ❌ | Not covered | Need filter lab |
| 63 | **Band-Pass Filter (BPF)** | 🟡 | All projects (RF path) | Need filter design lab |
| 64 | **Band-Stop Filter (Notch)** | ❌ | Not covered | Need interference rejection lab |
| 65 | **Finite Impulse Response (FIR)** | ❌ | Not covered | Need digital filter lab |
| 66 | **Infinite Impulse Response (IIR)** | ❌ | Not covered | Need digital filter lab |
| 67 | **Filter Order** | ❌ | Not covered | Need filter design lab |
| 68 | **Cutoff Frequency** | 🟡 | All projects (bandwidth) | Need filter lab |
| 69 | **Passband** | 🟡 | All projects | Need filter response lab |
| 70 | **Stopband** | 🟡 | All projects | Need filter response lab |
| 71 | **Rolloff** | ❌ | Not covered | Need filter characteristics lab |
| 72 | **Group Delay** | ❌ | Not covered | Need phase response lab |
| 73 | **Windowing** | ❌ | Not covered | Need spectral analysis lab |
| 74 | **Convolution** | ❌ | Not covered | Need DSP fundamentals lab |
| 75 | **Correlation** | ✅ | PROJECT 2 (DSSS) | Gold code correlation |

**Coverage: 1/15 Complete, 5/15 Partial, 9/15 Missing**

**Required New Labs:**
- LAB 2.9: **FIR and IIR Filter Design**
- LAB 2.10: **Filter Response Analysis (Bode plots)**
- LAB 2.11: **Windowing and Spectral Leakage**

---

## Category 6: Frequency Domain Analysis (12 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 76 | **Fast Fourier Transform (FFT)** | ✅ | LAB 0 (spectrum), PROJECT 3 (OFDM) | Core DSP tool |
| 77 | **Discrete Fourier Transform (DFT)** | 🟡 | LAB 0 (implicit) | Need theory lab |
| 78 | **Inverse FFT (IFFT)** | ✅ | PROJECT 3 (OFDM TX) | Time-domain synthesis |
| 79 | **Power Spectral Density (PSD)** | 🟡 | LAB 0 (plot) | Need Welch method lab |
| 80 | **Spectrogram** | ❌ | Not covered | Need time-frequency lab |
| 81 | **Waterfall Plot** | ❌ | Not covered | Need visualization lab |
| 82 | **Frequency Resolution** | ❌ | Not covered | Need FFT parameters lab |
| 83 | **Window Function** | ❌ | Not covered | Need spectral analysis lab |
| 84 | **Zero-Padding** | ❌ | Not covered | Need FFT techniques lab |
| 85 | **Frequency Bin** | 🟡 | LAB 0 (implicit) | Need FFT theory lab |
| 86 | **Leakage** | ❌ | Not covered | Need windowing lab |
| 87 | **Scalloping Loss** | ❌ | Not covered | Need FFT artifacts lab |

**Coverage: 2/12 Complete, 3/12 Partial, 7/12 Missing**

**Required New Labs:**
- LAB 2.12: **FFT Analysis and Parameters**
- LAB 2.13: **Spectrograms and Waterfall Plots**
- LAB 2.14: **Window Functions and Spectral Leakage**

---

## Category 7: Spread Spectrum (15 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 88 | **Spread Spectrum** | ✅ | PROJECT 2, 3, 4 | Three techniques |
| 89 | **Direct Sequence Spread Spectrum (DSSS)** | ✅ | PROJECT 2 | CDMA with Gold codes |
| 90 | **Frequency Hopping Spread Spectrum (FHSS)** | ✅ | PROJECT 4 | 500 hops/sec |
| 91 | **Chirp Spread Spectrum (CSS)** | ❌ | Not covered | Need LoRa-style lab |
| 92 | **Code Division Multiple Access (CDMA)** | ✅ | PROJECT 2 | Multi-user DSSS |
| 93 | **Pseudo-Random Noise (PN) Sequence** | ✅ | PROJECT 2 (Gold), 4 (hopping) | Sequence generation |
| 94 | **Spreading Factor** | ✅ | PROJECT 2 (SF10) | Processing gain |
| 95 | **Despreading** | ✅ | PROJECT 2 | Correlation receiver |
| 96 | **Processing Gain** | ✅ | PROJECT 2 (30 dB), 4 (17 dB) | Anti-jamming gain |
| 97 | **Chip Rate** | ✅ | PROJECT 2 | Spreading clock rate |
| 98 | **Gold Codes** | ✅ | PROJECT 2 | Low cross-correlation |
| 99 | **Walsh Codes** | ❌ | Not covered | Need orthogonal codes lab |
| 100 | **Barker Codes** | ❌ | Not covered | Need pulse compression lab |
| 101 | **Maximal Length Sequence (m-sequence)** | ✅ | PROJECT 2 (LFSR) | PN generator |
| 102 | **Linear Feedback Shift Register (LFSR)** | ✅ | PROJECT 2 | Gold code generation |

**Coverage: 10/15 Complete, 0/15 Partial, 5/15 Missing**

**Required New Labs:**
- LAB 4.1: **Chirp Spread Spectrum (LoRa-style)**
- LAB 4.2: **Walsh and Barker Codes**

---

## Category 8: OFDM and Multicarrier (12 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 103 | **Orthogonal Frequency Division Multiplexing (OFDM)** | ✅ | PROJECT 3 | Complete implementation |
| 104 | **Subcarrier** | ✅ | PROJECT 3 (256 subcarriers) | Parallel channels |
| 105 | **Cyclic Prefix (CP)** | ✅ | PROJECT 3 | ISI mitigation |
| 106 | **Guard Interval** | ✅ | PROJECT 3 | Same as CP |
| 107 | **Pilot Tones** | ✅ | PROJECT 3 | Channel estimation |
| 108 | **Channel Estimation** | ✅ | PROJECT 3 | Pilot-based |
| 109 | **Equalization** | ✅ | PROJECT 3 | Zero-forcing |
| 110 | **Peak-to-Average Power Ratio (PAPR)** | ✅ | PROJECT 3 (8.3 dB) | OFDM characteristic |
| 111 | **Carrier Frequency Offset (CFO)** | ❌ | Not covered | Need synchronization lab |
| 112 | **Timing Offset** | ❌ | Not covered | Need synchronization lab |
| 113 | **Phase Noise** | ❌ | Not covered | Need oscillator impairments lab |
| 114 | **Multipath Channel** | 🟡 | PROJECT 3 (theory) | Need fading lab |

**Coverage: 9/12 Complete, 1/12 Partial, 2/12 Missing**

**Required New Labs:**
- LAB 4.3: **OFDM Synchronization (CFO and Timing)**
- LAB 4.4: **Multipath and Fading Channels**

---

## Category 9: Multiple Access (8 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 115 | **Frequency Division Multiple Access (FDMA)** | 🟡 | PROJECT 4 (channels) | Need multi-user demo |
| 116 | **Time Division Multiple Access (TDMA)** | ❌ | Not covered | Need time-slot lab |
| 117 | **Code Division Multiple Access (CDMA)** | ✅ | PROJECT 2 | DSSS multi-user |
| 118 | **Orthogonal FDMA (OFDMA)** | 🟡 | PROJECT 3 (OFDM) | Need LTE-style lab |
| 119 | **Space Division Multiple Access (SDMA)** | ❌ | Not covered | Need MIMO lab |
| 120 | **Carrier Sense Multiple Access (CSMA)** | ❌ | Not covered | Need MAC protocol lab |
| 121 | **Random Access** | ❌ | Not covered | Need collision handling lab |
| 122 | **Multiple Input Multiple Output (MIMO)** | ❌ | Not covered | Need spatial multiplexing lab |

**Coverage: 1/8 Complete, 2/8 Partial, 5/8 Missing**

**Required New Labs:**
- LAB 5.1: **FDMA and TDMA Multi-User Systems**
- LAB 5.2: **CSMA and MAC Protocols**
- LAB 5.3: **2×2 MIMO with PlutoSDR** (requires 2 TX/RX)

---

## Category 10: Forward Error Correction (10 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 123 | **Forward Error Correction (FEC)** | ✅ | PROJECT 3/4 (Reed-Solomon) | Error correction |
| 124 | **Block Codes** | ✅ | PROJECT 3/4 (RS) | RS(255,223) |
| 125 | **Convolutional Codes** | 🟡 | PROJECT 2/3 (mentioned) | Need Viterbi lab |
| 126 | **Turbo Codes** | ❌ | Not covered | Need advanced FEC lab |
| 127 | **Low-Density Parity Check (LDPC)** | ❌ | Not covered | Need modern FEC lab |
| 128 | **Reed-Solomon Codes** | ✅ | PROJECT 3/4 | RS(255,223) |
| 129 | **Hamming Codes** | ❌ | Not covered | Need simple FEC lab |
| 130 | **Viterbi Decoder** | ❌ | Not covered | Need convolutional coding lab |
| 131 | **Interleaving** | ❌ | Not covered | Need burst error lab |
| 132 | **Coding Gain** | 🟡 | PROJECT 2 (link budget) | Need BER comparison lab |

**Coverage: 3/10 Complete, 2/10 Partial, 5/10 Missing**

**Required New Labs:**
- LAB 4.5: **Hamming and Convolutional Codes**
- LAB 4.6: **Viterbi Decoding**
- LAB 4.7: **Interleaving and Burst Errors**

---

## Category 11: Synchronization (12 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 133 | **Carrier Synchronization** | 🟡 | All projects (implicit) | Need PLL lab |
| 134 | **Symbol Timing Recovery** | 🟡 | PROJECT 4 (preamble) | Need timing loop lab |
| 135 | **Frame Synchronization** | ✅ | PROJECT 4 (sync word) | Packet detection |
| 136 | **Phase-Locked Loop (PLL)** | 🟡 | PROJECT 4 (settling time) | Need PLL design lab |
| 137 | **Costas Loop** | ❌ | Not covered | Need carrier recovery lab |
| 138 | **Gardner Timing Recovery** | ❌ | Not covered | Need symbol timing lab |
| 139 | **Early-Late Gate** | ❌ | Not covered | Need timing recovery lab |
| 140 | **Preamble** | ✅ | PROJECT 4 | Sync sequence |
| 141 | **Training Sequence** | 🟡 | PROJECT 3 (pilots) | Need burst mode lab |
| 142 | **Synchronization Word (Sync Word)** | ✅ | PROJECT 4 | 0xAA5533CC |
| 143 | **Correlation Peak** | ✅ | PROJECT 2 (DSSS), 4 | Sync detection |
| 144 | **Frequency Offset Estimation** | ❌ | Not covered | Need AFC lab |

**Coverage: 4/12 Complete, 4/12 Partial, 4/12 Missing**

**Required New Labs:**
- LAB 4.8: **PLL and Carrier Recovery**
- LAB 4.9: **Symbol Timing Recovery**
- LAB 4.10: **Frequency Offset Estimation and Correction**

---

## Category 12: RF Hardware (10 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 145 | **Low Noise Amplifier (LNA)** | ✅ | LAB 1.2 | RX front-end |
| 146 | **Power Amplifier (PA)** | ✅ | All projects (TX gain) | TX output stage |
| 147 | **Mixer** | ✅ | LAB 1.3 | Frequency conversion |
| 148 | **Local Oscillator (LO)** | ✅ | LAB 1.3, PROJECT 4 | Synthesizer |
| 149 | **Voltage-Controlled Oscillator (VCO)** | 🟡 | PROJECT 4 (PLL) | Need VCO tuning lab |
| 150 | **Synthesizer** | 🟡 | PROJECT 4 (hopping) | Need PLL synth lab |
| 151 | **Automatic Gain Control (AGC)** | ✅ | LAB 1.2 | Dynamic range |
| 152 | **Duplexer** | ❌ | Not covered | Need full-duplex lab |
| 153 | **Circulator** | ❌ | Not covered | Need antenna isolation lab |
| 154 | **Balun** | ❌ | Not covered | Need impedance matching lab |

**Coverage: 6/10 Complete, 2/10 Partial, 2/10 Missing**

**Required New Labs:**
- LAB 1.7: **VCO and Synthesizer Tuning**
- LAB 1.8: **Duplexer and Full-Duplex Operation**

---

## Category 13: Link Budget and Propagation (8 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 155 | **Link Budget** | ✅ | PROJECT 2/3 | Complete calculations |
| 156 | **Friis Equation** | ✅ | PROJECT 2 | Path loss calculation |
| 157 | **Effective Isotropic Radiated Power (EIRP)** | ✅ | PROJECT 2 | TX power + gain |
| 158 | **G/T (Gain-to-Temperature Ratio)** | ❌ | Not covered | Need link analysis lab |
| 159 | **Fade Margin** | 🟡 | PROJECT 2 (link margin) | Need fading lab |
| 160 | **Rain Attenuation** | ❌ | Not covered | Need weather effects lab |
| 161 | **Fresnel Zone** | ❌ | Not covered | Need propagation lab |
| 162 | **Line-of-Sight (LOS)** | ✅ | All projects | Range testing |

**Coverage: 4/8 Complete, 1/8 Partial, 3/8 Missing**

**Required New Labs:**
- LAB 5.4: **Advanced Link Budget (G/T, fade margin)**
- LAB 5.5: **Propagation Effects (rain, Fresnel zones)**

---

## Category 14: Advanced Topics (8 terms)

| # | Term | Status | Current Coverage | Notes |
|---|------|--------|------------------|-------|
| 163 | **Software-Defined Radio (SDR)** | ✅ | Entire curriculum | PlutoSDR platform |
| 164 | **Cognitive Radio** | ❌ | Not covered | Need spectrum sensing lab |
| 165 | **Dynamic Spectrum Access** | 🟡 | PROJECT 4 (hopping) | Need spectrum sensing |
| 166 | **Blind Signal Detection** | ❌ | Not covered | Need classification lab |
| 167 | **Beamforming** | ❌ | Not covered | Need phased array lab |
| 168 | **Direction Finding** | ❌ | Not covered | Need DOA lab |
| 169 | **Jamming** | ✅ | PROJECT 4 (anti-jamming) | Jammer demo |
| 170 | **Electronic Warfare** | 🟡 | PROJECT 4 (LPI/LPD) | Need EW lab |

**Coverage: 2/8 Complete, 2/8 Partial, 4/8 Missing**

**Required New Labs:**
- LAB 6.1: **Cognitive Radio and Spectrum Sensing**
- LAB 6.2: **Signal Classification and Blind Detection**
- LAB 6.3: **Direction Finding with Phase Arrays**

---

## Summary Statistics

### Overall Coverage

| Category | Total Terms | ✅ Complete | 🟡 Partial | ❌ Missing | % Coverage |
|----------|-------------|-------------|-----------|-----------|------------|
| 1. RF Fundamentals | 20 | 10 | 5 | 5 | 75% |
| 2. Digital Modulation | 18 | 5 | 5 | 8 | 56% |
| 3. I/Q Sampling | 12 | 6 | 1 | 5 | 58% |
| 4. Sampling/Conversion | 10 | 1 | 3 | 6 | 40% |
| 5. Filtering/DSP | 15 | 1 | 5 | 9 | 40% |
| 6. Frequency Domain | 12 | 2 | 3 | 7 | 42% |
| 7. Spread Spectrum | 15 | 10 | 0 | 5 | 67% |
| 8. OFDM | 12 | 9 | 1 | 2 | 83% |
| 9. Multiple Access | 8 | 1 | 2 | 5 | 38% |
| 10. FEC | 10 | 3 | 2 | 5 | 50% |
| 11. Synchronization | 12 | 4 | 4 | 4 | 67% |
| 12. RF Hardware | 10 | 6 | 2 | 2 | 80% |
| 13. Link Budget | 8 | 4 | 1 | 3 | 63% |
| 14. Advanced Topics | 8 | 2 | 2 | 4 | 50% |
| **TOTAL** | **170** | **64** | **36** | **70** | **59%** |

**Current Status:**
- **64 terms (38%)** have complete practical coverage
- **36 terms (21%)** have partial coverage (mentioned but need deeper labs)
- **70 terms (41%)** are missing and need new labs

**Strong Areas (>75% coverage):**
- ✅ OFDM and Multicarrier (83%)
- ✅ RF Hardware (80%)
- ✅ RF Fundamentals (75%)

**Weak Areas (<50% coverage):**
- ❌ Multiple Access (38%)
- ❌ Sampling and Conversion (40%)
- ❌ Filtering and Signal Processing (40%)
- ❌ Frequency Domain Analysis (42%)

---

## Recommended Lab Additions

### Priority 1: Critical Gaps (Fundamental Concepts)

**LAB 2.1: Nyquist Sampling, Aliasing, and Bandwidth**
- Demonstrate aliasing with undersampled signals
- Anti-aliasing filter design
- Nyquist zones and undersampling
- **Covers:** Nyquist rate, aliasing, bandwidth measurement (Terms 52-53, 5)

**LAB 2.2: Decimation, Interpolation, and Sample Rate Conversion**
- Multirate signal processing
- CIC filters for decimation
- Polyphase filters for interpolation
- **Covers:** Decimation, interpolation, sample rate conversion (Terms 57-59)

**LAB 2.3: Quantization, ADC Resolution, and Dynamic Range**
- Quantization noise analysis
- ENOB (Effective Number of Bits)
- SFDR (Spurious-Free Dynamic Range)
- **Covers:** Quantization, SNR measurement, dynamic range (Terms 60, 10)

**LAB 3.1: ASK, FSK, PSK Comparison**
- Implement all three basic modulations
- BER vs SNR curves
- Spectral efficiency comparison
- **Covers:** ASK, FSK, PSK basics (Terms 21-23)

**LAB 3.2: QPSK and 8-PSK Implementation**
- Gray coding for QPSK
- Phase ambiguity and differential encoding
- Higher-order PSK (8-PSK, 16-PSK)
- **Covers:** QPSK, 8-PSK (Terms 25-26)

Continue with additional labs...