# SDR Terms Coverage by Lab

**Comprehensive mapping of Software-Defined Radio concepts taught in each laboratory exercise**

**Last Updated**: December 3, 2025
**Total Labs**: 13 labs across 3 modules
**Total SDR Terms Covered**: 130+ terms

---

## Module 1: SDR Foundations (4 Labs)

### LAB 0: Hello PlutoSDR
**Location**: `labs/module1_foundations/LAB_0_HELLO_PLUTOSDR.md`
**Lines**: 2,065 lines
**Duration**: 1-2 hours

**SDR Terms Covered**:
1. **Software-Defined Radio (SDR)** - Fundamental concept
2. **ADALM-PLUTO (PlutoSDR)** - Hardware platform
3. **AD9361 RF Transceiver** - Integrated RF front-end
4. **Zynq-7000 SoC** - Processing system (ARM + FPGA)
5. **I/Q Samples (In-phase/Quadrature)** - Complex baseband representation
6. **Complex Baseband** - Signal representation
7. **Center Frequency (LO - Local Oscillator)** - Carrier frequency
8. **Sample Rate** - Sampling frequency (65 ksps - 61.44 Msps)
9. **RF Bandwidth** - Analog filter bandwidth (200 kHz - 56 MHz)
10. **TX/RX (Transmit/Receive)** - Radio operation modes
11. **libiio** - Industrial I/O library for hardware access
12. **PyADI-IIO** - Python interface to libiio
13. **Cyclic Buffer** - Continuous transmission mode
14. **USB Gadget** - USB device mode (RNDIS/Ethernet)

**Key Concepts**:
- SDR flexibility vs traditional radio
- I/Q sampling fundamentals
- PlutoSDR architecture
- Basic TX/RX operations
- Python/C interface to hardware

---

### LAB 1.1: SDR Flexibility
**Location**: `labs/module1_foundations/LAB_1_1_SDR_FLEXIBILITY.md`
**Lines**: 2,498 lines
**Duration**: 2-3 hours

**SDR Terms Covered**:
1. **Reconfigurable Radio** - Software-defined capabilities
2. **Multi-Protocol Support** - One hardware, multiple protocols
3. **Frequency Agility** - Wide tuning range (70 MHz - 6 GHz)
4. **Bandwidth Flexibility** - Variable bandwidth support
5. **Modulation Independence** - Support for any modulation scheme
6. **Digital Signal Processing (DSP)** - Software-based signal processing
7. **Waveform Generation** - Software-defined waveforms
8. **Protocol Stack** - Layered communication architecture
9. **RF Front-End** - Analog radio components
10. **Digital Back-End** - Baseband processing
11. **Frequency Translation** - Upconversion/downconversion
12. **Zero-IF Architecture** - Direct conversion receiver
13. **Quadrature Mixing** - I/Q generation
14. **LO Leakage** - DC offset in zero-IF receivers

**Key Concepts**:
- Advantages of SDR over traditional radios
- Frequency agility demonstrations
- Multi-protocol implementations
- Software vs hardware radios

---

### LAB 1.2: RF Gain Staging
**Location**: `labs/module1_foundations/LAB_1_2_RF_GAIN_STAGING.md`
**Lines**: 3,175 lines
**Duration**: 2-3 hours

**SDR Terms Covered**:
1. **RF Gain** - Signal amplification
2. **Manual Gain Control (MGC)** - Fixed gain setting
3. **Automatic Gain Control (AGC)** - Adaptive gain adjustment
4. **Slow Attack AGC** - Gradual gain changes
5. **Fast Attack AGC** - Rapid gain changes
6. **Hybrid AGC** - Combined manual/automatic
7. **RX Hardware Gain** - Receive path gain (0-73 dB)
8. **TX Hardware Gain** - Transmit path attenuation (-89 to 0 dB)
9. **Gain Steps** - Discrete gain increments (1 dB steps)
10. **Saturation** - Signal clipping at max level
11. **Noise Floor** - Minimum detectable signal level
12. **Dynamic Range** - Ratio of max to min signal levels
13. **Signal-to-Noise Ratio (SNR)** - Quality metric
14. **dBFS (dB Full Scale)** - Digital signal level measurement
15. **Overload** - Exceeding ADC input range
16. **Link Budget** - System gain/loss analysis

**Key Concepts**:
- RX gain control strategies
- TX power management
- AGC attack time selection
- Preventing saturation and clipping
- Optimizing SNR

---

### LAB 1.3: I/Q Samples
**Location**: `labs/module1_foundations/LAB_1_3_IQ_SAMPLES.md`
**Lines**: 3,622 lines
**Duration**: 3-4 hours

**SDR Terms Covered**:
1. **In-Phase (I) Component** - Real part of complex signal
2. **Quadrature (Q) Component** - Imaginary part of complex signal
3. **Complex Signal** - I+jQ representation
4. **Phasor** - Rotating complex number
5. **Phase** - Angular position of phasor
6. **Magnitude** - Amplitude of complex signal
7. **Constellation Diagram** - I/Q plot of symbols
8. **Quadrature Modulation** - Using both I and Q
9. **Hilbert Transform** - Generating analytic signal
10. **Analytic Signal** - Complex representation of real signal
11. **Negative Frequencies** - Mathematical concept in DSP
12. **Single-Sideband (SSB)** - Using I/Q to eliminate one sideband
13. **Image Rejection** - Suppressing unwanted sideband
14. **I/Q Imbalance** - Gain/phase mismatch between I and Q
15. **DC Offset** - Zero-frequency component
16. **Carrier Feedthrough** - LO leakage in transmitter

**Key Concepts**:
- Why I/Q sampling is used
- Complex baseband representation
- Constellation diagram interpretation
- I/Q imbalance and correction
- Generating arbitrary waveforms with I/Q

**Module 1 Total**: 14,360 lines, ~40 SDR terms

---

## Module 2: Sampling Theory (3 Labs)

### LAB 2.1: Nyquist Sampling, Aliasing, and Bandwidth
**Location**: `labs/module2_sampling/LAB_2_1_NYQUIST_ALIASING.md`
**Lines**: 3,334 lines
**Duration**: 3-4 hours

**SDR Terms Covered**:
1. **Nyquist-Shannon Sampling Theorem** - fs > 2 × fmax
2. **Nyquist Rate** - Minimum sampling rate (2 × fmax)
3. **Nyquist Frequency** - fs/2
4. **Aliasing** - Frequency folding due to undersampling
5. **Sampling Frequency (fs)** - Rate of ADC conversion
6. **Analog Bandwidth** - Signal frequency extent
7. **Anti-Aliasing Filter** - Lowpass filter before ADC
8. **Oversampling** - Sampling above Nyquist rate
9. **Undersampling** - Intentional aliasing (bandpass sampling)
10. **Bandpass Sampling** - Sampling centered signals
11. **Folding Frequency** - fs/2 where aliasing occurs
12. **Spectral Inversion** - Frequency reversal in undersampling
13. **Image Frequency** - Alias of desired signal
14. **RF Bandwidth** - Passband width
15. **Baseband Bandwidth** - Lowpass signal bandwidth

**Key Concepts**:
- Nyquist theorem derivation and proof
- Aliasing visualization
- Anti-aliasing filter design
- Bandpass sampling techniques
- PlutoSDR sample rate limits

---

### LAB 2.2: Decimation, Interpolation, and Sample Rate Conversion
**Location**: `labs/module2_sampling/LAB_2_2_DECIMATION_INTERPOLATION.md`
**Lines**: 4,246 lines
**Duration**: 4-5 hours

**SDR Terms Covered**:
1. **Decimation** - Downsampling by integer factor M
2. **Interpolation** - Upsampling by integer factor L
3. **Sample Rate Conversion (SRC)** - Changing sample rate
4. **Rational Rate Conversion** - L/M sample rate change
5. **Polyphase Filter** - Efficient multirate filter
6. **FIR Filter (Finite Impulse Response)** - Non-recursive filter
7. **Anti-Imaging Filter** - Filter after interpolation
8. **Anti-Aliasing Filter** - Filter before decimation
9. **Passband** - Frequency range of interest
10. **Stopband** - Frequency range to reject
11. **Transition Band** - Between passband and stopband
12. **Filter Order** - Number of taps in FIR filter
13. **CIC Filter (Cascaded Integrator-Comb)** - Efficient decimator/interpolator
14. **Multi-Stage Decimation** - Cascaded decimators
15. **Multi-Stage Interpolation** - Cascaded interpolators
16. **Computational Efficiency** - MIPS reduction with polyphase
17. **Group Delay** - Filter delay characteristics

**Key Concepts**:
- Decimation/interpolation theory
- Polyphase filter implementation
- Multi-stage rate conversion
- CIC filter design
- Computational efficiency optimization
- NEON SIMD optimization for ARM

---

### LAB 2.3: Quantization and ADC Resolution
**Location**: `labs/module2_sampling/LAB_2_3_QUANTIZATION_ADC_RESOLUTION.md`
**Lines**: 3,830 lines
**Duration**: 3-4 hours

**SDR Terms Covered**:
1. **Quantization** - Amplitude discretization
2. **ADC (Analog-to-Digital Converter)** - Sampling and quantization
3. **DAC (Digital-to-Analog Converter)** - Reconstruction
4. **Resolution** - Number of bits (N)
5. **Quantization Step (Δ or LSB)** - Voltage per bit
6. **Full Scale Range (FSR)** - ADC input range
7. **Quantization Noise** - Error due to quantization
8. **Quantization Error** - Difference between analog and digital
9. **SNR Formula** - SNR = 6.02N + 1.76 dB
10. **SQNR (Signal-to-Quantization-Noise Ratio)** - Ideal SNR
11. **ENOB (Effective Number of Bits)** - Actual resolution
12. **SINAD (Signal-to-Noise-And-Distortion)** - Total quality metric
13. **THD (Total Harmonic Distortion)** - Non-linearity metric
14. **SFDR (Spurious-Free Dynamic Range)** - Largest spurious signal
15. **Dithering** - Adding noise to improve quantization
16. **Oversampling** - Sampling above Nyquist to improve SNR
17. **Noise Shaping** - Pushing quantization noise out of band
18. **Dynamic Range** - Range from noise floor to saturation
19. **AD9361 ADC** - 12-bit ADC in PlutoSDR
20. **Clipping** - Saturation at full scale

**Key Concepts**:
- Quantization theory and noise model
- SNR = 6.02N + 1.76 dB derivation
- ENOB calculation and measurement
- Dithering techniques
- SFDR measurement
- PlutoSDR 12-bit ADC characteristics

**Module 2 Total**: 11,410 lines, ~35 SDR terms

---

## Module 3: Digital Modulation (5 Labs)

### LAB 3.1: Digital Modulation - ASK, FSK, PSK
**Location**: `labs/module3_modulation/LAB_3_1_ASK_FSK_PSK_MODULATION.md`
**Lines**: 3,783 lines
**Duration**: 4-5 hours

**SDR Terms Covered**:
1. **Digital Modulation** - Discrete symbol transmission
2. **Symbol** - Unit of digital information
3. **Bit** - Binary digit (0 or 1)
4. **Symbol Rate (Rs)** - Symbols per second (baud)
5. **Bit Rate (Rb)** - Bits per second (bps)
6. **ASK (Amplitude Shift Keying)** - Amplitude modulation
7. **OOK (On-Off Keying)** - Binary ASK
8. **FSK (Frequency Shift Keying)** - Frequency modulation
9. **BFSK (Binary FSK)** - Two frequencies
10. **MFSK (M-ary FSK)** - Multiple frequencies
11. **PSK (Phase Shift Keying)** - Phase modulation
12. **BPSK (Binary PSK)** - Two phases (0°, 180°)
13. **Constellation Diagram** - Symbol visualization
14. **Symbol Mapping** - Bits to symbols
15. **Modulation Index** - FSK frequency deviation
16. **Phase Shift** - Angular change in carrier
17. **Carrier Frequency** - Center frequency
18. **Baseband Signal** - Information signal
19. **Passband Signal** - Modulated RF signal
20. **Demodulation** - Recovering baseband from passband

**Key Concepts**:
- Binary modulation schemes
- Constellation diagrams
- Modulator/demodulator implementation
- BER performance comparison
- PlutoSDR transmission

---

### LAB 3.2: QPSK and 8-PSK Modulation
**Location**: `labs/module3_modulation/LAB_3_2_QPSK_8PSK_MODULATION.md`
**Lines**: 3,885 lines
**Duration**: 3-4 hours

**SDR Terms Covered**:
1. **QPSK (Quadrature PSK)** - 4-phase PSK (2 bits/symbol)
2. **8-PSK** - 8-phase PSK (3 bits/symbol)
3. **M-ary Modulation** - Multiple symbols (M > 2)
4. **Bits per Symbol (m)** - log2(M)
5. **Gray Coding** - Adjacent symbols differ by 1 bit
6. **Hamming Distance** - Bit differences between symbols
7. **Symbol Error Rate (SER)** - Probability of symbol error
8. **Bit Error Rate (BER)** - Probability of bit error
9. **Eb/N0 (Energy per Bit to Noise Ratio)** - SNR metric
10. **Euclidean Distance** - Separation between symbols
11. **Minimum Distance** - Closest symbol pair
12. **Phase Ambiguity** - Carrier phase uncertainty
13. **Differential Encoding** - Resolving phase ambiguity
14. **DQPSK (Differential QPSK)** - Phase change encoding
15. **Spectral Efficiency** - Bits per second per Hz
16. **Power Efficiency** - Energy per bit required

**Key Concepts**:
- M-ary PSK theory
- Gray coding benefits
- QPSK vs 8-PSK tradeoffs
- Differential encoding
- Spectral efficiency analysis

---

### LAB 3.3: QAM Modulation (16/64/256-QAM)
**Location**: `labs/module3_modulation/LAB_3_3_QAM_MODULATION.md`
**Lines**: 3,851 lines (comprehensive Method 3)
**Duration**: 5-6 hours

**SDR Terms Covered**:
1. **QAM (Quadrature Amplitude Modulation)** - Combined amplitude and phase
2. **16-QAM** - 16 symbols (4 bits/symbol)
3. **64-QAM** - 64 symbols (6 bits/symbol)
4. **256-QAM** - 256 symbols (8 bits/symbol)
5. **Square QAM** - Rectangular constellation (M = 2^2k)
6. **Cross QAM** - Non-square constellation
7. **Constellation Normalization** - Unit average power
8. **Gray Coding** - Minimizing bit errors
9. **I/Q Mapping** - Bits to I/Q coordinates
10. **Decision Boundaries** - Symbol detection regions
11. **PAPR (Peak-to-Average Power Ratio)** - Dynamic range requirement
12. **Clipping** - PAPR reduction technique
13. **Spectral Efficiency** - Up to 8 bits/s/Hz for 256-QAM
14. **Noise Sensitivity** - Higher-order QAM more sensitive
15. **SNR Requirements** - dB needed for target BER
16. **Adaptive Modulation** - Changing QAM order based on conditions

**Key Concepts**:
- QAM constellation design
- Gray coding for QAM
- Normalization and PAPR
- BER vs SNR curves
- Production-ready C implementation with NEON
- Adaptive modulation systems

---

### LAB 3.4: BER Testing and Eye Diagrams
**Location**: `labs/module3_modulation/LAB_3_4_BER_TESTING_EYE_DIAGRAMS.md`
**Lines**: 4,264 lines (comprehensive Method 3)
**Duration**: 5-6 hours

**SDR Terms Covered**:
1. **BER (Bit Error Rate)** - Pe = errors / total_bits
2. **SER (Symbol Error Rate)** - Symbol errors
3. **Monte Carlo Simulation** - Statistical BER estimation
4. **Confidence Interval** - Statistical certainty
5. **Eb/N0** - Energy per bit to noise power density ratio
6. **AWGN (Additive White Gaussian Noise)** - Channel model
7. **Q-factor** - Eye diagram quality metric
8. **Eye Diagram** - Oscilloscope-like display
9. **Eye Opening** - Clear region in eye
10. **Eye Height** - Voltage margin
11. **Eye Width** - Timing margin
12. **ISI (Inter-Symbol Interference)** - Pulse overlap
13. **Jitter** - Timing variation
14. **Random Jitter (RJ)** - Gaussian timing noise
15. **Deterministic Jitter (DJ)** - Systematic timing error
16. **Rise Time** - Signal edge speed
17. **Fall Time** - Signal edge speed
18. **Crossing Point** - Zero-crossing location
19. **Sampling Instant** - Optimal decision time
20. **Decision Threshold** - Voltage detection level
21. **Noise Margin** - Tolerance to noise
22. **Welford's Algorithm** - Online statistics computation

**Key Concepts**:
- BER measurement techniques
- Eye diagram generation and interpretation
- Q-factor calculation
- ISI and jitter analysis
- Statistical confidence
- Fast BER counting with XOR + popcount
- NEON SIMD optimization (15× speedup)

---

### LAB 3.5: Pulse Shaping and Matched Filtering
**Location**: `labs/module3_modulation/LAB_3_5_PULSE_SHAPING_MATCHED_FILTERING.md`
**Lines**: 2,832 lines (comprehensive Method 3)
**Duration**: 5-6 hours

**SDR Terms Covered**:
1. **Pulse Shaping** - Limiting signal bandwidth
2. **Matched Filter** - Optimal receiver filter
3. **ISI (Inter-Symbol Interference)** - Symbol overlap
4. **Nyquist ISI Criterion** - Zero ISI condition
5. **Raised Cosine (RC) Filter** - Nyquist pulse shape
6. **Root-Raised Cosine (RRC) Filter** - Matched filter pair
7. **Roll-off Factor (β)** - Bandwidth parameter (0 to 1)
8. **Excess Bandwidth** - Additional BW beyond Nyquist
9. **Occupied Bandwidth** - BW = (1+β)/(2T)
10. **Symbol Period (T)** - Time per symbol
11. **Samples per Symbol (sps)** - Oversampling factor
12. **Eye Diagram** - Pulse shape quality visualization
13. **Timing Recovery** - Symbol clock extraction
14. **Gardner Algorithm** - Timing error detector (TED)
15. **Mueller-Müller Algorithm** - Alternative TED
16. **Early-Late Gate** - Simple TED
17. **Timing Error Detector (TED)** - Phase detector for timing
18. **Loop Filter** - PLL filter for timing recovery
19. **NCO (Numerically Controlled Oscillator)** - Digital frequency source
20. **PI Controller** - Proportional-Integral loop filter
21. **Lock Range** - PLL capture range
22. **Tracking Range** - PLL hold range
23. **Acquisition** - Initial lock process
24. **Tracking** - Maintaining lock

**Key Concepts**:
- Nyquist ISI criterion derivation
- RC and RRC filter design
- Matched filter theory
- Gardner timing recovery implementation
- PLL loop dynamics
- RRC singularity handling (t=0, t=±T/4β)
- NEON SIMD optimization (7.7× speedup)
- FCC occupied bandwidth compliance

**Module 3 Total**: 18,615 lines, ~60 SDR terms

---

## Advanced Projects (4 Projects)

### PROJECT 2: IoT-Satellite DSSS Link
**Location**: `projects/PROJECT2_IOT_SATELLITE_ENHANCED.md`
**Lines**: 2,074 lines
**Duration**: 10-15 hours

**SDR Terms Covered**:
1. **DSSS (Direct Sequence Spread Spectrum)** - Spreading technique
2. **Spreading Code** - Pseudo-random sequence
3. **Spreading Gain** - Processing gain = BW_spread / BW_data
4. **Gold Codes** - Optimal spreading sequences
5. **PN Sequence (Pseudo-Noise)** - Spreading sequence
6. **Chip Rate** - Spreading code rate
7. **Processing Gain** - SNR improvement from spreading
8. **Despread** - Correlating with spreading code
9. **Correlation** - Detecting spread signal
10. **Autocorrelation** - Code correlation with itself
11. **Cross-Correlation** - Code correlation with other codes
12. **CDMA (Code Division Multiple Access)** - Multi-user DSSS
13. **OFDM (Orthogonal Frequency Division Multiplexing)** - Multi-carrier
14. **Subcarrier** - Individual OFDM tone
15. **FFT/IFFT** - OFDM implementation
16. **Cyclic Prefix (CP)** - Guard interval for OFDM
17. **Pilot Tones** - Reference subcarriers
18. **Channel Estimation** - Using pilot tones
19. **STM32WL33** - IoT radio transceiver
20. **Link Budget** - Power budget analysis

**Key Concepts**:
- DSSS spreading and despreading
- Gold code generation
- OFDM fundamentals
- Multi-protocol integration (IoT + SDR)
- Link budget calculations

---

### PROJECT 3: Secure Video Streaming
**Location**: `projects/PROJECT3_SECURE_VIDEO_ENHANCED.md`
**Lines**: 2,914 lines
**Duration**: 12-18 hours

**SDR Terms Covered**:
1. **AES-256-GCM** - Encryption algorithm
2. **Encryption** - Securing data
3. **Authentication** - Verifying integrity
4. **AEAD (Authenticated Encryption with Associated Data)** - GCM mode
5. **Reed-Solomon FEC** - Forward error correction
6. **RS(255, 223)** - RS code parameters
7. **Interleaving** - Spreading burst errors
8. **FEC (Forward Error Correction)** - Error correction
9. **Coding Gain** - SNR improvement from FEC
10. **64-QAM** - High-order modulation
11. **OFDM** - Multi-carrier modulation
12. **H.264** - Video compression
13. **Bitrate** - Data rate in bps
14. **Latency** - End-to-end delay
15. **Packet Loss** - Missing data
16. **Jitter** - Delay variation
17. **UDP** - User Datagram Protocol
18. **RTP** - Real-time Transport Protocol
19. **Raspberry Pi** - Embedded computer
20. **Pi Camera** - Video source

**Key Concepts**:
- End-to-end encryption
- Reed-Solomon coding
- OFDM video streaming
- Real-time video processing
- Integration of multiple technologies

---

### PROJECT 4: Frequency-Hopping Datalink
**Location**: `projects/PROJECT4_FREQUENCY_HOPPING.md`
**Lines**: 1,640 lines
**Duration**: 8-12 hours

**SDR Terms Covered**:
1. **FHSS (Frequency Hopping Spread Spectrum)** - Hopping technique
2. **Hop Rate** - Hops per second
3. **Hop Set** - Available frequencies
4. **Hop Sequence** - Order of frequency hops
5. **Dwell Time** - Time per frequency
6. **Synchronization** - Hop timing alignment
7. **Fast Hopping** - Multiple hops per symbol
8. **Slow Hopping** - Multiple symbols per hop
9. **Anti-Jamming** - Resistance to interference
10. **LPI (Low Probability of Intercept)** - Hard to detect
11. **LPD (Low Probability of Detection)** - Stealth operation
12. **Jamming** - Intentional interference
13. **Frequency Agility** - Rapid frequency changes
14. **Hop Collision** - Multiple users on same frequency
15. **Bluetooth** - FHSS system example
16. **IEEE 802.15.1** - Bluetooth standard
17. **ISM Band** - Industrial, Scientific, Medical frequencies
18. **Settling Time** - PLL lock time
19. **Frequency Synthesizer** - Generating hop frequencies
20. **PN Generator** - Pseudo-random hop sequence

**Key Concepts**:
- FHSS theory and implementation
- 500 hops/second on PlutoSDR
- Anti-jamming techniques
- Dual-SDR synchronization
- FSK modulation with hopping

---

### PROJECT 5: Tactical Voice/Data Radio (MIL-STD)
**Location**: `projects/PROJECT5_TACTICAL_COMMS_MIL_STD.md`
**Lines**: 1,755 lines
**Duration**: 15-20 hours

**SDR Terms Covered**:
1. **MIL-STD-188-181D** - Data modem standard
2. **MIL-STD-188-141E** - HF radio standard
3. **STANAG 4285** - NATO PSK modem standard
4. **FIPS 197** - AES encryption standard
5. **MELP (Mixed Excitation Linear Prediction)** - Voice codec
6. **Vocoder** - Voice encoder/decoder
7. **Voice Codec** - Compression algorithm
8. **2400 bps** - MELP bitrate
9. **9600 bps** - Data rate
10. **Tactical Radio** - Military communications
11. **COMSEC (Communications Security)** - Encryption
12. **TRANSEC (Transmission Security)** - LPI/LPD
13. **AES-256-GCM** - Military-grade encryption
14. **Key Management** - Cryptographic key handling
15. **FHSS** - 50 hops/second
16. **DSSS** - 16-chip spreading
17. **Processing Gain** - 12 dB (16-chip)
18. **Anti-Jam Margin** - Jamming resistance
19. **Link Quality** - BER, RSSI, SNR monitoring
20. **RSSI (Received Signal Strength Indicator)** - Power level
21. **Interoperability** - Working with other radios
22. **AN/PRC-152** - Example tactical radio
23. **SINCGARS** - Single Channel Ground and Airborne Radio System
24. **Squad-Level Communications** - Small unit operations

**Key Concepts**:
- Military communications standards
- Secure voice and data
- Multi-layer security (COMSEC + TRANSEC)
- Anti-jamming waveforms
- Real-world tactical applications

**Projects Total**: 8,383 lines, ~65 SDR terms (some overlap with labs)

---

## Summary Statistics

### Coverage by Module

| Module | Labs | Lines | SDR Terms | Terms/Lab |
|--------|------|-------|-----------|-----------|
| **Module 1** | 4 | 11,360 | ~40 | 10 |
| **Module 2** | 3 | 11,410 | ~35 | 12 |
| **Module 3** | 5 | 18,615 | ~60 | 12 |
| **Projects** | 4 | 8,383 | ~65* | 16 |
| **TOTAL** | **16** | **49,768** | **130+** | **~9** |

*Some project terms overlap with lab terms

### Coverage by Category

| Category | Terms | Covered In | Coverage |
|----------|-------|------------|----------|
| **Fundamentals** | 20 | Module 1 (all labs) | 100% |
| **Sampling Theory** | 35 | Module 2 (all labs) | 100% |
| **Digital Modulation** | 60 | Module 3 (all labs) | 100% |
| **Spread Spectrum** | 20 | Projects 2, 4, 5 | 100% |
| **FEC** | 15 | Project 3 | 53% |
| **Security** | 10 | Projects 3, 5 | 100% |
| **Military Standards** | 10 | Project 5 | 100% |
| **Hardware** | 15 | Module 1, all projects | 100% |

### Term Density

**Average terms per lab**: ~9 terms
**Highest density**: LAB 3.4 (BER/Eye Diagrams) - 22 terms
**Most comprehensive**: LAB 3.5 (Pulse Shaping) - 24 terms

### Implementation Coverage

All terms covered with:
- ✅ **Theory**: Mathematical foundations
- ✅ **Python**: Simulation and visualization
- ✅ **C**: Production implementation (Method 3)
- ✅ **PlutoSDR**: Hardware validation

---

## Terms Not Yet Covered (Planned for Future Modules)

### Module 4: DSP and Filtering (Planned)
- FIR filter design (window method, Parks-McClellan)
- IIR filter design (Butterworth, Chebyshev, Elliptic)
- Window functions (Hamming, Hann, Blackman, Kaiser)
- FFT algorithms (Cooley-Tukey, radix-2, radix-4)
- Spectrogram and waterfall displays
- Adaptive filtering (LMS, RLS, Kalman)

### Module 5: Forward Error Correction (Planned)
- Hamming codes
- Cyclic codes
- BCH codes
- Convolutional codes
- Viterbi algorithm
- Turbo codes
- LDPC (Low-Density Parity Check)

### Module 6: Synchronization (Planned)
- Costas loop (QPSK carrier recovery)
- PLL (Phase-Locked Loop) design
- Mueller-Müller timing recovery
- Frame synchronization
- Preamble detection
- Sync word correlation

**Estimated additional terms**: ~50 terms
**Target total coverage**: ~180 terms (beyond original 170 goal)

---

## How to Use This Document

### For Students
- Use this as a reference to understand what each lab teaches
- Check prerequisites before starting a lab
- Track your learning progress through the curriculum

### For Educators
- Map course objectives to specific labs
- Create quizzes/exams based on terms covered
- Assess student understanding of SDR concepts

### For Curriculum Planning
- Identify gaps in coverage
- Plan future lab development
- Ensure comprehensive SDR education

---

## Cross-References

- **Main Curriculum**: [docs/README.md](../README.md)
- **Curriculum Status**: [CURRICULUM_STATUS.md](CURRICULUM_STATUS.md)
- **Quick Start**: [QUICKSTART.md](../../QUICKSTART.md)
- **Build Scripts**: [guides/PLUTOSDR_BUILD_SCRIPTS.md](../guides/PLUTOSDR_BUILD_SCRIPTS.md)

---

**Last Updated**: December 3, 2025
**Total Coverage**: 130+ SDR terms across 16 labs/projects
**Completion**: 76% of planned 170 terms
**Status**: Production-ready comprehensive SDR curriculum
