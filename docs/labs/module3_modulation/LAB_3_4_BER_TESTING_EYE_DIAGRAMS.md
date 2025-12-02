# LAB 3.4: BER Testing and Eye Diagrams

## Overview

This lab explores **Bit Error Rate (BER) testing** and **eye diagrams** - essential tools for evaluating digital communication system performance. You'll learn how to measure BER vs. SNR, interpret eye diagrams, and understand link quality metrics.

## Learning Objectives

After completing this lab, you will understand:
- Bit Error Rate (BER) measurement techniques
- BER vs. Eb/N0 performance curves
- Symbol Error Rate (SER) vs. BER
- Eye diagrams: what they show and how to interpret them
- Eye opening, eye height, timing jitter
- Q-factor and link quality metrics
- Monte Carlo BER simulation
- PlutoSDR BER testing methodology
- Waterfall diagrams for visualizing signal quality

## Prerequisites

- LAB 0: PlutoSDR Setup
- LAB 2.1-2.3: Sampling and quantization
- LAB 3.1: ASK, FSK, PSK fundamentals
- LAB 3.2: QPSK and 8-PSK
- LAB 3.3: QAM modulation
- Understanding of SNR and noise
- Basic Python or C programming

## Theory

### 1. Bit Error Rate (BER) Fundamentals

**Definition**: Probability of receiving incorrect bit.

```
BER = Number of bit errors / Total bits transmitted

Example:
  Transmit: 1,000,000 bits
  Errors:   10 bits
  BER = 10 / 1,000,000 = 10⁻⁵

BER is dimensionless (ratio), often expressed as power of 10
```

**Typical BER Requirements**:
```
Application              Required BER    Notes
------------------------------------------------------------
Voice (analog quality)   10⁻³            Noticeable errors, acceptable
Data (uncoded)           10⁻⁵            1 error per 100,000 bits
Data (with FEC)          10⁻⁶ to 10⁻⁹    Very reliable
Optical fiber            10⁻¹²           Extremely reliable
Military/space           10⁻⁷ to 10⁻¹⁰   Mission critical

Most wireless standards target 10⁻⁵ to 10⁻⁶ before FEC
After FEC: 10⁻⁸ to 10⁻¹⁰
```

**Statistical Confidence**:
```
To measure BER = 10⁻⁵ with 95% confidence:
  Need at least 100 errors
  Transmit: 100 / 10⁻⁵ = 10,000,000 bits

At 1 Mbps: 10 seconds
At 10 Mbps: 1 second
At 100 kbps: 100 seconds (1.7 minutes)

Lower BER → longer test time!

For BER = 10⁻⁹:
  Need 10⁹ to 10¹⁰ bits
  At 1 Mbps: 1000-10000 seconds (hours!)
```

### 2. Eb/N0 and SNR

**Eb/N0**: Energy per bit to noise power spectral density ratio

```
Eb/N0 = (S/N) × (BW / Rb)

where:
  S = Signal power
  N = Noise power
  BW = Bandwidth
  Rb = Bit rate

In dB:
  Eb/N0 (dB) = SNR (dB) + 10·log₁₀(BW/Rb)
```

**Example Calculation**:
```
QPSK at 1 Mbps, 2 MHz bandwidth, SNR = 10 dB

Eb/N0 = 10 + 10·log₁₀(2 MHz / 1 Mbps)
      = 10 + 10·log₁₀(2)
      = 10 + 3
      = 13 dB

For QPSK BER = 10⁻⁵: need Eb/N0 ≈ 9.6 dB
  13 dB > 9.6 dB → Link margin: 3.4 dB ✓
```

### 3. BER Performance Curves

**Theoretical BER for Common Modulations** (AWGN channel):

```
BPSK:
  BER = Q(√(2·Eb/N0))

QPSK:
  BER ≈ Q(√(2·Eb/N0))  (same as BPSK per bit!)

M-PSK (M > 4):
  BER ≈ (2/log₂M) · Q(√(2·Eb/N0·log₂M) · sin(π/M))

16-QAM:
  BER ≈ (3/8) · Q(√(4·Eb/N0/10))

64-QAM:
  BER ≈ (7/24) · Q(√(6·Eb/N0/42))

where Q(x) = (1/√(2π)) ∫[x,∞] e^(-t²/2) dt (Gaussian Q-function)
```

**Q-Function Values**:
```
Q(x)       x (approx)
-----------------------
10⁻¹       1.28
10⁻²       2.33
10⁻³       3.09
10⁻⁴       3.72
10⁻⁵       4.27
10⁻⁶       4.75
10⁻⁹       6.00
10⁻¹²      7.03
```

**BER vs. Eb/N0 Table** (selected values):

```
Modulation    BER=10⁻³   BER=10⁻⁵   BER=10⁻⁶   BER=10⁻⁹
---------------------------------------------------------------
BPSK          6.8 dB     9.6 dB     10.5 dB    14.0 dB
QPSK          6.8 dB     9.6 dB     10.5 dB    14.0 dB
8-PSK         10.7 dB    14.0 dB    15.0 dB    18.5 dB
16-QAM        10.5 dB    14.5 dB    15.5 dB    19.5 dB
64-QAM        14.8 dB    18.8 dB    20.0 dB    24.0 dB
```

### 4. Eye Diagrams

**Definition**: Overlaid waveforms showing signal transitions.

```
Eye diagram = superposition of many symbol periods

Creation:
  1. Sample received signal
  2. Divide into segments (one symbol period each)
  3. Overlay all segments on same plot
  4. Result: "eye" pattern

Used for:
  - Visualizing signal quality
  - Measuring timing jitter
  - Assessing ISI (inter-symbol interference)
  - Determining optimal sampling time
```

**Ideal Eye Diagram** (BPSK example):
```
Amplitude
    |
 +1 |  ●●●●●●●●●●●●●    ◄── Upper rail (bit = 0)
    | ●            ●
    |●              ●
    |●              ●   ◄── Eye opening (good!)
  0 |●   ○○○○○○○   ●   ◄── Optimal sampling point
    |●              ●
    |●              ●
 -1 | ●            ●    ◄── Lower rail (bit = 1)
    |  ●●●●●●●●●●●●●
    +--|------------|---> Time
       0          Ts

Eye height: 2.0 (full scale)
Eye opening: Wide (low ISI)
Timing jitter: Minimal
```

**Degraded Eye Diagram**:
```
Amplitude
    |
 +1 |  ○●●●●●●●●●●○
    | ●            ●     ◄── Noise (fuzzy lines)
    |●              ●
  0 |●●   ●●●●●   ●●    ◄── Eye closure (bad!)
    |●              ●
    | ●            ●
 -1 |  ○●●●●●●●●●●○
    +--|------------|---> Time
       0          Ts

Eye height: Reduced (noise)
Eye opening: Narrow (ISI)
Timing jitter: High (horizontal spread)
```

**Eye Diagram Metrics**:
```
1. Eye Height:
   Vertical opening at optimal sampling instant
   Affected by: Noise, amplitude distortion
   Larger = better

2. Eye Width:
   Horizontal opening (timing margin)
   Affected by: ISI, timing jitter
   Wider = better

3. Eye Opening (%):
   Eye_opening = Eye_height / Signal_amplitude × 100%
   > 80%: Excellent
   60-80%: Good
   40-60%: Marginal
   < 40%: Poor (high BER)

4. Crossing Point:
   Where transitions cross zero
   Should be at mid-point (Ts/2)
   Offset indicates timing errors

5. Jitter:
   Horizontal variation in crossing points
   RMS jitter < 10% Ts: Good
   > 20% Ts: Poor
```

### 5. Q-Factor

**Definition**: Signal-to-noise ratio at decision point.

```
Q = (μ₁ - μ₀) / (σ₁ + σ₀)

where:
  μ₁ = Mean of "1" level
  μ₀ = Mean of "0" level
  σ₁ = Standard deviation of "1" level
  σ₀ = Standard deviation of "0" level

Relationship to BER (Gaussian noise):
  BER ≈ Q(Q-factor)

Example:
  μ₁ = +1.0 V, σ₁ = 0.1 V
  μ₀ = -1.0 V, σ₀ = 0.1 V

  Q = (1.0 - (-1.0)) / (0.1 + 0.1) = 2.0 / 0.2 = 10

  BER ≈ Q(10) ≈ 7.6×10⁻²⁴ (extremely low!)
```

**Q-Factor in dB**:
```
Q_dB = 20·log₁₀(Q-factor)

Typical values:
  Q_dB = 10 dB → BER ≈ 10⁻³
  Q_dB = 15 dB → BER ≈ 10⁻⁵
  Q_dB = 18 dB → BER ≈ 10⁻⁷
```

### 6. Monte Carlo BER Simulation

**Method**: Simulate many random realizations to estimate BER.

```
Algorithm:
  1. Choose Eb/N0 value
  2. Generate random bits
  3. Modulate bits → symbols
  4. Add AWGN noise (based on Eb/N0)
  5. Demodulate → detected bits
  6. Count errors
  7. BER = errors / total_bits
  8. Repeat for different Eb/N0 values

Confidence:
  Need ≥ 100 errors for statistical validity
  If BER too low, increase number of bits
```

---

## Part 1: Simulation (Pure Python)

### Implementation 1: BER Testing Framework

```python
import numpy as np
import matplotlib.pyplot as plt
from scipy.special import erfc

class BERTester:
    """BER testing framework for digital modulations"""

    def __init__(self, modem, name="Modem"):
        """
        Args:
            modem: Modulation modem object (must have modulate/demodulate methods)
            name: Name of modulation scheme
        """
        self.modem = modem
        self.name = name

    def add_awgn(self, signal, snr_db, signal_power=1.0):
        """
        Add AWGN to signal

        Args:
            signal: Input signal
            snr_db: SNR in dB
            signal_power: Signal power (default 1.0 for normalized)

        Returns:
            Noisy signal
        """
        snr_linear = 10**(snr_db / 10)
        noise_power = signal_power / snr_linear

        if np.iscomplexobj(signal):
            noise = np.sqrt(noise_power/2) * (
                np.random.randn(len(signal)) + 1j*np.random.randn(len(signal))
            )
        else:
            noise = np.sqrt(noise_power) * np.random.randn(len(signal))

        return signal + noise

    def measure_ber(self, n_bits, eb_n0_db, bits_per_symbol=1):
        """
        Measure BER at given Eb/N0

        Args:
            n_bits: Number of bits to test
            eb_n0_db: Eb/N0 in dB
            bits_per_symbol: Bits per symbol (for Eb/N0 to SNR conversion)

        Returns:
            Measured BER
        """
        # Generate random bits
        bits_tx = np.random.randint(0, 2, size=n_bits)

        # Modulate (baseband, fc=0 for simulation)
        self.modem.fc = 0
        signal_tx, _ = self.modem.modulate(bits_tx)

        # Convert Eb/N0 to SNR
        # SNR = Eb/N0 + 10*log10(bits_per_symbol)
        snr_db = eb_n0_db + 10*np.log10(bits_per_symbol)

        # Add noise
        signal_rx = self.add_awgn(signal_tx, snr_db)

        # Demodulate
        bits_rx = self.modem.demodulate(signal_rx, n_bits)

        # Count errors
        errors = np.sum(bits_tx != bits_rx)
        ber = errors / n_bits

        return ber, errors

    def sweep_eb_n0(self, eb_n0_range, n_bits=100000, bits_per_symbol=1):
        """
        Sweep Eb/N0 and measure BER curve

        Args:
            eb_n0_range: Array of Eb/N0 values (dB)
            n_bits: Bits per test point
            bits_per_symbol: Bits per symbol

        Returns:
            eb_n0_vals, ber_vals, error_counts
        """
        print(f"\nBER Testing: {self.name}")
        print(f"  Bits per point: {n_bits:,}")
        print(f"  Bits per symbol: {bits_per_symbol}")
        print("-" * 60)

        ber_vals = []
        error_counts = []

        for eb_n0 in eb_n0_range:
            ber, errors = self.measure_ber(n_bits, eb_n0, bits_per_symbol)
            ber_vals.append(ber)
            error_counts.append(errors)

            print(f"  Eb/N0 = {eb_n0:5.1f} dB → BER = {ber:.2e} ({errors} errors)")

        return np.array(eb_n0_range), np.array(ber_vals), np.array(error_counts)

    def plot_ber_curve(self, eb_n0, ber_measured, ber_theoretical=None):
        """Plot BER vs Eb/N0"""
        fig, ax = plt.subplots(1, 1, figsize=(10, 7))

        # Measured BER
        ax.semilogy(eb_n0, ber_measured, 'bo-', linewidth=2, markersize=8,
                   label=f'{self.name} (Measured)')

        # Theoretical BER
        if ber_theoretical is not None:
            ax.semilogy(eb_n0, ber_theoretical, 'r--', linewidth=2,
                       label=f'{self.name} (Theoretical)')

        ax.set_xlabel('Eb/N0 (dB)', fontsize=12)
        ax.set_ylabel('Bit Error Rate (BER)', fontsize=12)
        ax.set_title(f'BER Performance: {self.name}', fontsize=14, fontweight='bold')
        ax.grid(True, alpha=0.3, which='both')
        ax.legend(fontsize=11)
        ax.set_ylim([1e-6, 1])

        # Add reference lines
        for ber_ref in [1e-3, 1e-5, 1e-6]:
            ax.axhline(ber_ref, color='gray', linestyle=':', alpha=0.3)

        plt.tight_layout()
        filename = f'ber_curve_{self.name.lower().replace(" ", "_")}.png'
        plt.savefig(filename, dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved {filename}")
        plt.show()


# Theoretical BER functions
def theoretical_ber_bpsk(eb_n0_db):
    """Theoretical BER for BPSK"""
    eb_n0_linear = 10**(eb_n0_db / 10)
    return 0.5 * erfc(np.sqrt(eb_n0_linear))

def theoretical_ber_qpsk(eb_n0_db):
    """Theoretical BER for QPSK (same as BPSK per bit)"""
    return theoretical_ber_bpsk(eb_n0_db)


# Test BER measurement
if __name__ == "__main__":
    from LAB_3_1_ASK_FSK_PSK_MODULATION import PSKModem as BPSKModem
    from LAB_3_2_QPSK_8PSK_MODULATION import QPSKModem

    print("="*70)
    print("BER TESTING DEMONSTRATION")
    print("="*70)

    # Test BPSK
    bpsk_modem = BPSKModem(carrier_freq=0, symbol_rate=100e3, samples_per_symbol=20)
    bpsk_tester = BERTester(bpsk_modem, name="BPSK")

    eb_n0_range = np.arange(0, 11, 2)
    eb_n0, ber_bpsk, errors = bpsk_tester.sweep_eb_n0(
        eb_n0_range, n_bits=100000, bits_per_symbol=1
    )

    # Theoretical BER
    ber_theory_bpsk = theoretical_ber_bpsk(eb_n0)

    # Plot
    bpsk_tester.plot_ber_curve(eb_n0, ber_bpsk, ber_theory_bpsk)

    # Test QPSK
    print("\n" + "="*70)
    qpsk_modem = QPSKModem(carrier_freq=0, symbol_rate=100e3, samples_per_symbol=20)
    qpsk_tester = BERTester(qpsk_modem, name="QPSK")

    eb_n0, ber_qpsk, errors = qpsk_tester.sweep_eb_n0(
        eb_n0_range, n_bits=100000, bits_per_symbol=2
    )

    ber_theory_qpsk = theoretical_ber_qpsk(eb_n0)
    qpsk_tester.plot_ber_curve(eb_n0, ber_qpsk, ber_theory_qpsk)
```

### Implementation 2: Eye Diagram Generator

```python
class EyeDiagramGenerator:
    """Generate eye diagrams for digital signals"""

    def __init__(self, samples_per_symbol):
        """
        Args:
            samples_per_symbol: Samples per symbol period
        """
        self.sps = samples_per_symbol

    def generate_eye(self, signal, n_symbols_per_trace=2):
        """
        Generate eye diagram from signal

        Args:
            signal: Input signal (real)
            n_symbols_per_trace: Symbol periods per trace

        Returns:
            Eye diagram traces
        """
        trace_length = self.sps * n_symbols_per_trace
        n_traces = len(signal) // trace_length

        # Extract traces
        traces = []
        for i in range(n_traces):
            start = i * trace_length
            end = start + trace_length
            if end <= len(signal):
                traces.append(signal[start:end])

        return np.array(traces)

    def plot_eye_diagram(self, signal, title="Eye Diagram"):
        """Plot eye diagram"""
        traces = self.generate_eye(signal)

        fig, ax = plt.subplots(1, 1, figsize=(10, 7))

        # Plot all traces
        t = np.arange(traces.shape[1]) / self.sps
        for trace in traces:
            ax.plot(t, trace, 'b-', alpha=0.1, linewidth=0.5)

        ax.set_xlabel('Time (symbol periods)', fontsize=12)
        ax.set_ylabel('Amplitude', fontsize=12)
        ax.set_title(title, fontsize=14, fontweight='bold')
        ax.grid(True, alpha=0.3)
        ax.axvline(1, color='r', linestyle='--', alpha=0.5, label='Optimal sampling')
        ax.legend()

        plt.tight_layout()
        filename = 'eye_diagram.png'
        plt.savefig(filename, dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved {filename}")
        plt.show()

    def measure_eye_metrics(self, signal):
        """Measure eye diagram quality metrics"""
        traces = self.generate_eye(signal)

        # Sample at symbol center
        sample_idx = self.sps

        # Get samples at decision point
        samples = traces[:, sample_idx]

        # Separate by threshold
        threshold = np.median(samples)
        samples_high = samples[samples > threshold]
        samples_low = samples[samples < threshold]

        # Compute metrics
        if len(samples_high) > 0 and len(samples_low) > 0:
            mu_high = np.mean(samples_high)
            mu_low = np.mean(samples_low)
            sigma_high = np.std(samples_high)
            sigma_low = np.std(samples_low)

            eye_height = mu_high - mu_low
            q_factor = eye_height / (sigma_high + sigma_low)
            q_db = 20 * np.log10(q_factor)

            print(f"\nEye Diagram Metrics:")
            print(f"  Mean high level:  {mu_high:.3f}")
            print(f"  Mean low level:   {mu_low:.3f}")
            print(f"  Eye height:       {eye_height:.3f}")
            print(f"  Q-factor:         {q_factor:.2f}")
            print(f"  Q-factor (dB):    {q_db:.1f} dB")
            print(f"  Estimated BER:    ~10^{int(np.log10(theoretical_ber_bpsk(q_db*0.5)))}")

        return {
            'eye_height': eye_height,
            'q_factor': q_factor,
            'q_db': q_db
        }


# Test eye diagram
if __name__ == "__main__":
    print("\n" + "="*70)
    print("EYE DIAGRAM DEMONSTRATION")
    print("="*70)

    from LAB_3_1_ASK_FSK_PSK_MODULATION import PSKModem as BPSKModem

    modem = BPSKModem(carrier_freq=0, symbol_rate=100e3, samples_per_symbol=20)

    # Generate signal
    bits = np.random.randint(0, 2, size=1000)
    signal_clean, _ = modem.modulate(bits)

    # Add noise
    snr_db = 15
    signal_power = np.mean(signal_clean**2)
    snr_linear = 10**(snr_db/10)
    noise_power = signal_power / snr_linear
    noise = np.sqrt(noise_power) * np.random.randn(len(signal_clean))
    signal_noisy = signal_clean + noise

    # Generate eye diagrams
    eye_gen = EyeDiagramGenerator(samples_per_symbol=20)

    # Clean signal
    eye_gen.plot_eye_diagram(signal_clean, title="Eye Diagram (Clean Signal)")
    eye_gen.measure_eye_metrics(signal_clean)

    # Noisy signal
    eye_gen.plot_eye_diagram(signal_noisy, title=f"Eye Diagram (SNR = {snr_db} dB)")
    eye_gen.measure_eye_metrics(signal_noisy)
```

---

## Part 2: PlutoSDR BER Testing

```python
import adi
import numpy as np
import time

class PlutoBERTest:
    """BER testing on PlutoSDR"""

    def __init__(self, uri="ip:192.168.2.1"):
        self.sdr = adi.Pluto(uri)
        self.sdr.sample_rate = int(1e6)
        self.sdr.tx_lo = int(915e6)
        self.sdr.rx_lo = int(915e6)
        self.sdr.tx_cyclic_buffer = True

        print("PlutoSDR BER Tester")
        print(f"Sample rate: {self.sdr.sample_rate/1e6:.1f} MSPS")

    def run_ber_test(self, n_bits=10000, modulation='BPSK', gain=40):
        """
        Run BER test on PlutoSDR

        Args:
            n_bits: Number of bits to test
            modulation: 'BPSK', 'QPSK', or '16QAM'
            gain: RX gain (0-73 dB)

        Returns:
            BER, error count
        """
        from LAB_3_1_ASK_FSK_PSK_MODULATION import PSKModem as BPSKModem
        from LAB_3_2_QPSK_8PSK_MODULATION import QPSKModem
        from LAB_3_3_QAM_MODULATION import QAM16Modem

        # Select modem
        symbol_rate = 10e3
        sps = int(self.sdr.sample_rate / symbol_rate)

        if modulation == 'BPSK':
            modem = BPSKModem(0, symbol_rate, sps)
            bits_per_symbol = 1
        elif modulation == 'QPSK':
            modem = QPSKModem(0, symbol_rate, sps)
            bits_per_symbol = 2
        elif modulation == '16QAM':
            modem = QAM16Modem(0, symbol_rate, sps)
            bits_per_symbol = 4
        else:
            raise ValueError(f"Unknown modulation: {modulation}")

        # Generate bits
        bits_tx = np.random.randint(0, 2, size=n_bits)

        # Modulate
        symbols = modem.bits_to_symbols(bits_tx)
        tx_samples = np.repeat(symbols, sps) * 0.5

        # Set gain
        self.sdr.rx_hardwaregain_chan0 = gain
        self.sdr.tx_hardwaregain_chan0 = -20

        # Transmit
        self.sdr.tx(tx_samples)
        time.sleep(0.1)  # Allow TX to start

        # Receive
        rx_samples = self.sdr.rx()

        # Demodulate (simplified - should add synchronization)
        rx_symbols = rx_samples[sps//2::sps][:len(symbols)]

        # Decode symbols (nearest neighbor)
        symbol_indices = np.zeros(len(rx_symbols), dtype=int)
        for i, sym in enumerate(rx_symbols):
            distances = np.abs(modem.constellation - sym)
            symbol_indices[i] = np.argmin(distances)

        bits_rx = modem.symbols_to_bits(symbol_indices, n_bits)

        # Count errors
        errors = np.sum(bits_tx != bits_rx)
        ber = errors / n_bits

        print(f"\n{modulation} BER Test (Gain = {gain} dB):")
        print(f"  Bits sent:   {n_bits:,}")
        print(f"  Errors:      {errors}")
        print(f"  BER:         {ber:.2e}")

        return ber, errors


# Run test
if __name__ == "__main__":
    tester = PlutoBERTest()

    # Test different gain settings
    for gain in [20, 30, 40, 50]:
        tester.run_ber_test(n_bits=10000, modulation='QPSK', gain=gain)
```

---

## METHOD 3: HOSTED APPLICATION IN C (PART 3/6 - THEORY DEEP DIVE)

This section provides **deep theoretical understanding** of BER testing and eye diagrams, essential for implementing professional-grade measurement tools on PlutoSDR.

**What you'll master**:

✅ **BER Statistics**: Confidence intervals, sample size calculations, error probability distributions

✅ **Eye Diagram Mathematics**: Sampling theory, optimal decision thresholds, eye closure analysis

✅ **Q-Factor Theory**: Relationship to BER, Gaussian noise assumptions, measurement techniques

✅ **ISI and Jitter**: Causes, mathematical models, impact on BER

✅ **Practical Algorithms**: Efficient BER counting, real-time eye diagram construction, histogram-based Q-factor

---

### Section 1: BER Measurement Theory and Statistics

#### **1.1 Probability of Bit Error**

For binary signaling with AWGN (Additive White Gaussian Noise):

```
P_e = Q(√(2·SNR))

where Q(x) is the Q-function (tail probability of Gaussian):

Q(x) = (1/√(2π)) ∫[x to ∞] exp(-t²/2) dt

For large x:
Q(x) ≈ (1/(x√(2π))) · exp(-x²/2)
```

**For specific modulations**:

```
BPSK:   P_e = Q(√(2·Eb/N0))
QPSK:   P_e = Q(√(2·Es/N0))     where Es = 2·Eb
16-QAM: P_e ≈ (3/2)·Q(√(Es/(5·N0)))
64-QAM: P_e ≈ (7/3)·Q(√(Es/(21·N0)))
```

**Key Insight**: BER decreases exponentially with SNR:
- 3 dB SNR increase → ~10× lower BER
- 6 dB SNR increase → ~100× lower BER

#### **1.2 Statistical Confidence and Sample Size**

**Problem**: How many bits to test for accurate BER measurement?

**Binomial Distribution**: Number of errors follows binomial distribution:
```
n_errors ~ Binomial(n_bits, BER)

Mean: μ = n_bits · BER
Standard deviation: σ = √(n_bits · BER · (1 - BER))
```

**Confidence Interval** (95%):
```
BER_measured ± 1.96 · √(BER · (1 - BER) / n_bits)

Example: BER = 10⁻⁴, n_bits = 10⁶
  σ = √(10⁻⁴ · (1 - 10⁻⁴) / 10⁶) ≈ 10⁻⁵
  CI: [10⁻⁴ - 1.96×10⁻⁵, 10⁻⁴ + 1.96×10⁻⁵]
    = [8.04×10⁻⁵, 1.196×10⁻⁴]
```

**Rule of Thumb**: For reliable BER measurement:
```
Required bits ≈ 100 / BER

Target BER    Required Bits    Time @ 1 Mbps
----------------------------------------
10⁻³          100,000          0.1 s
10⁻⁴          1,000,000        1 s
10⁻⁵          10,000,000       10 s
10⁻⁶          100,000,000      100 s
10⁻⁹          1,000,000,000    1000 s (17 min!)
```

**Practical Compromise**: Test until you observe 100-1000 errors:
```
if (bit_errors >= 100 && total_bits >= 10⁶) {
    BER = bit_errors / total_bits;
    // 95% confidence achieved
}
```

#### **1.3 Symbol Error Rate (SER) vs. BER**

**Relationship**: For Gray-coded M-ary modulation:
```
BER ≈ SER / log₂(M)

Example: 16-QAM with Gray coding
  log₂(16) = 4 bits/symbol
  If SER = 4×10⁻⁴
  Then BER ≈ 4×10⁻⁴ / 4 = 10⁻⁴
```

**Why Gray coding helps**:
- Adjacent symbols differ by 1 bit
- Single-symbol error typically causes 1 bit error (not all 4 bits)
- Without Gray coding: SER ≈ BER (worst case)

**C Implementation Strategy**:
```c
// Option 1: Count bit errors (more accurate)
for (size_t i = 0; i < num_bits; i++) {
    if (tx_bits[i] != rx_bits[i]) bit_errors++;
}
BER = (double)bit_errors / num_bits;

// Option 2: Count symbol errors (faster, less accurate for high-order QAM)
for (size_t i = 0; i < num_symbols; i++) {
    if (tx_symbols[i] != rx_symbols[i]) symbol_errors++;
}
SER = (double)symbol_errors / num_symbols;
BER_estimate = SER / bits_per_symbol;
```

---

### Section 2: Eye Diagram Theory

#### **2.1 What is an Eye Diagram?**

An eye diagram is created by:
1. Sampling received signal at multiple points per symbol
2. Overlaying many symbol intervals (typically 100-1000 symbols)
3. Displaying on oscilloscope-like plot

**Mathematical Description**:
```
For symbol period T_s, sample at times:
  t = k·T_s + τ,  where τ ∈ [0, T_s] and k = 0, 1, 2, ...

Eye diagram: Plot of all samples (τ, r(k·T_s + τ))
  where r(t) is received signal
```

#### **2.2 Eye Diagram Components**

**Anatomy of an eye**:
```
         ┌──────────────────┐
         │   Eye Opening    │← Maximum vertical opening
    +1 ──┤                  ├── Decision threshold
         │                  │
     0 ──┼──────────────────┼──
         │                  │
    -1 ──┤                  ├──
         │                  │
         └──────────────────┘
         ◄────────────────►
         Eye Width = T_eye

         ◄──►                 Timing jitter

Optimal sampling time: τ_opt = T_s / 2 (center of eye)
```

**Key Measurements**:

1. **Eye Height** (V_eye):
   ```
   V_eye = |μ₁ - μ₀| - 3·(σ₁ + σ₀)

   where:
     μ₁ = mean of "1" samples
     μ₀ = mean of "0" samples
     σ₁ = std dev of "1" samples
     σ₀ = std dev of "0" samples

   Factor of 3σ ensures 99.7% of samples within bounds
   ```

2. **Eye Width** (T_eye):
   ```
   T_eye = T_s - 2·t_jitter

   where t_jitter is timing uncertainty (jitter)

   Typical: T_eye > 0.6·T_s (40% margin)
   ```

3. **Eye Opening** (Area):
   ```
   Eye_opening = V_eye × T_eye

   Larger opening → Better signal quality
   Smaller opening → Approaching error threshold
   ```

#### **2.3 Factors that Close the Eye**

**1. Inter-Symbol Interference (ISI)**:
```
Caused by:
- Limited bandwidth (incomplete pulse shaping)
- Multipath propagation
- Imperfect filtering

Mathematical model:
  r(t) = Σ[k] a_k·h(t - k·T_s) + n(t)

  where h(t) is channel impulse response

If h(t) has long tail → symbols overlap → ISI
```

**2. Timing Jitter**:
```
Types:
- Random jitter (Gaussian, from thermal noise)
- Deterministic jitter (from clock instability)

Effect: Horizontal eye closure

Jitter RMS = √(E[(t_actual - t_ideal)²])
```

**3. Additive Noise (AWGN)**:
```
Effect: Vertical eye closure

Noise blurs each trace → thicker lines → reduced V_eye
```

**4. Carrier Frequency Offset (CFO)**:
```
In PlutoSDR: TX and RX LOs may differ by Δf

Effect: Rotating constellation → eye rotates → closure

Maximum tolerable Δf:
  Δf_max < 1 / (10·T_symbol)

  For 100 ksps: Δf_max < 10 kHz
```

#### **2.4 Optimal Sampling Time**

**Decision Rule**: Sample at time τ_opt that maximizes eye opening

**Algorithm**:
```
1. Generate eye diagram samples for τ ∈ [0, T_s]
2. For each τ, compute eye height V_eye(τ)
3. Find: τ_opt = argmax V_eye(τ)
4. Use τ_opt for symbol decisions

Mathematically:
  τ_opt = argmax |μ₁(τ) - μ₀(τ)| / (σ₁(τ) + σ₀(τ))
```

**C Implementation Approach**:
```c
// Collect samples at multiple phases
#define PHASES_PER_SYMBOL 8

for (int phase = 0; phase < PHASES_PER_SYMBOL; phase++) {
    double tau = (double)phase / PHASES_PER_SYMBOL;

    // Sample at this phase across all symbols
    for (int k = 0; k < num_symbols; k++) {
        int sample_idx = k * sps + (int)(tau * sps);
        eye_samples[phase][k] = rx_signal[sample_idx];
    }

    // Compute eye height at this phase
    compute_eye_metrics(eye_samples[phase], num_symbols);
}

// Find phase with maximum eye opening
int best_phase = find_max_eye_opening();
```

---

### Section 3: Q-Factor and Signal Quality

#### **3.1 Q-Factor Definition**

**Q-factor**: Signal-to-noise ratio at decision point

```
Q = |μ₁ - μ₀| / (σ₁ + σ₀)

where:
  μ₁, μ₀ = mean levels for "1" and "0"
  σ₁, σ₀ = standard deviations

Interpretation:
  Q = 6:  99.73% correct decisions (BER ≈ 10⁻³)
  Q = 7:  BER ≈ 10⁻⁶
  Q = 8:  BER ≈ 10⁻⁹
```

**Relationship to BER** (for Gaussian noise):
```
BER = Q(Q_factor)

Approximation:
  BER ≈ (1/(Q_factor·√(2π))) · exp(-Q_factor² / 2)

Example:
  Q = 6  → BER = Q(6) ≈ 9.87×10⁻¹⁰
  Q = 7  → BER = Q(7) ≈ 1.28×10⁻¹²
```

**In dB**:
```
Q_dB = 20·log₁₀(Q)

Q = 6  → Q_dB = 15.56 dB
Q = 7  → Q_dB = 16.90 dB

Relationship to SNR:
  For BPSK: Q² = 2·SNR
  For QPSK: Q² = SNR (per dimension)
```

#### **3.2 Measuring Q-Factor from Eye Diagram**

**Histogram Method** (most practical for PlutoSDR):

**Step 1**: Sample signal at optimal decision time τ_opt
```c
for (int k = 0; k < num_symbols; k++) {
    int sample_idx = k * samples_per_symbol + optimal_phase;
    double sample = rx_signal[sample_idx];
    samples[k] = sample;
}
```

**Step 2**: Separate "1" and "0" samples based on known TX bits
```c
for (int k = 0; k < num_symbols; k++) {
    if (tx_bits[k] == 1) {
        ones_samples[n_ones++] = samples[k];
    } else {
        zeros_samples[n_zeros++] = samples[k];
    }
}
```

**Step 3**: Compute means and standard deviations
```c
double mu_1 = mean(ones_samples, n_ones);
double mu_0 = mean(zeros_samples, n_zeros);
double sigma_1 = stddev(ones_samples, n_ones);
double sigma_0 = stddev(zeros_samples, n_zeros);
```

**Step 4**: Calculate Q-factor
```c
double Q_factor = fabs(mu_1 - mu_0) / (sigma_1 + sigma_0);
double Q_dB = 20 * log10(Q_factor);
```

**Step 5**: Estimate BER from Q
```c
double estimated_BER = 0.5 * erfc(Q_factor / sqrt(2));
// erfc = complementary error function (available in <math.h>)
```

#### **3.3 Alternative Q-Factor Methods**

**Peak-to-Peak Method** (less accurate, but simpler):
```c
double peak_1 = max(ones_samples, n_ones);
double peak_0 = min(zeros_samples, n_zeros);
double Q_approx = fabs(peak_1 - peak_0) / (sigma_1 + sigma_0);
```

**Percentile Method** (robust to outliers):
```c
// Use 0.1% and 99.9% percentiles instead of min/max
double p999_1 = percentile(ones_samples, n_ones, 0.999);
double p001_0 = percentile(zeros_samples, n_zeros, 0.001);
double Q_robust = (p999_1 - p001_0) / (sigma_1 + sigma_0);
```

---

### Section 4: ISI (Inter-Symbol Interference) Analysis

#### **4.1 ISI Mathematical Model**

**Baseband Signal Model**:
```
r(t) = Σ[k=-∞ to ∞] a_k · p(t - k·T_s) + n(t)

where:
  a_k = transmitted symbols
  p(t) = combined TX filter, channel, RX filter response
  n(t) = AWGN

At sampling time t = m·T_s:
  r(m·T_s) = a_m · p(0) + Σ[k≠m] a_k · p((m-k)·T_s) + n(m·T_s)
             ^^^^^^^^^^^^   ^^^^^^^^^^^^^^^^^^^^^^^   ^^^^^^^^^^
             desired        ISI                       noise
```

**ISI Power**:
```
P_ISI = Σ[k≠0] p²(k·T_s)

For zero-ISI (Nyquist criterion):
  p(k·T_s) = 0  for all k ≠ 0

Achieved by raised-cosine or RRC pulse shaping
```

**Impact on BER**:
```
Effective SNR with ISI:
  SNR_eff = SNR / (1 + ISI_factor)

  where ISI_factor = P_ISI / p²(0)

Example:
  SNR = 20 dB, ISI_factor = 0.2
  SNR_eff = 20 / 1.2 = 16.67 dB
  → BER degrades by 3.3 dB!
```

#### **4.2 Eye Diagram ISI Indicators**

**Closed Eye** (severe ISI):
```
Eye height → 0
Multiple crossing points
Thick, fuzzy traces
```

**Moderate ISI**:
```
Eye still open but distorted
Asymmetric eye shape
Unequal rise/fall times
```

**Minimal ISI** (ideal):
```
Wide eye opening
Clean crossing at 50% level
Symmetric shape
```

**C Implementation - ISI Measurement**:
```c
// Measure eye closure due to ISI
double measure_isi_closure(complex double *rx_signal, size_t len, int sps) {
    // Sample at non-optimal times (edges of symbol period)
    size_t num_symbols = len / sps;

    double edge_samples[num_symbols];
    double center_samples[num_symbols];

    for (size_t k = 0; k < num_symbols; k++) {
        center_samples[k] = cabs(rx_signal[k * sps + sps/2]);
        edge_samples[k] = cabs(rx_signal[k * sps]);  // Symbol boundary
    }

    double sigma_center = stddev(center_samples, num_symbols);
    double sigma_edge = stddev(edge_samples, num_symbols);

    // High sigma_edge relative to sigma_center indicates ISI
    double isi_ratio = sigma_edge / sigma_center;

    return isi_ratio;  // > 1.5 indicates significant ISI
}
```

---

### Section 5: Jitter Analysis

#### **5.1 Types of Jitter**

**Random Jitter (RJ)**:
```
Distribution: Gaussian

  t_jitter ~ N(0, σ_RJ²)

Causes:
  - Thermal noise
  - Shot noise in clock circuits
  - Phase noise in PLLs

Unbounded: Can theoretically be arbitrarily large
```

**Deterministic Jitter (DJ)**:
```
Distribution: Bounded (peaks at specific values)

Types:
  1. Periodic jitter: From interfering signals
  2. Data-dependent jitter: From ISI
  3. Bounded uncorrelated jitter: From EMI

Bounded: |DJ| ≤ DJ_max
```

**Total Jitter (TJ)**:
```
TJ = RJ + DJ

At BER = 10⁻¹²:
  TJ = DJ + 14·σ_RJ  (14σ covers 10⁻¹² probability)
```

#### **5.2 Jitter Measurement from Eye Diagram**

**Method 1: Zero-Crossing Analysis**
```c
// Find all zero crossings in eye diagram
for (int k = 0; k < num_symbols - 1; k++) {
    if (rx_signal[k] * rx_signal[k+1] < 0) {
        // Zero crossing detected between k and k+1
        double t_cross = interpolate_zero_crossing(rx_signal, k);
        zero_crossings[n_crossings++] = t_cross;
    }
}

// Compute jitter as std dev of crossing times
double t_mean = mean(zero_crossings, n_crossings);
double jitter_rms = 0;
for (int i = 0; i < n_crossings; i++) {
    jitter_rms += (zero_crossings[i] - t_mean) * (zero_crossings[i] - t_mean);
}
jitter_rms = sqrt(jitter_rms / n_crossings);

// Express as fraction of symbol period
double jitter_UI = jitter_rms / T_symbol;  // UI = Unit Interval
```

**Method 2: Eye Width Measurement**
```c
// Measure horizontal eye opening at threshold level
double threshold = (mu_1 + mu_0) / 2;

// Find leftmost and rightmost times where signal crosses threshold
double t_left = find_crossing_time(rx_signal, threshold, RISING_EDGE, LEFT);
double t_right = find_crossing_time(rx_signal, threshold, FALLING_EDGE, RIGHT);

double eye_width = t_right - t_left;
double jitter = (T_symbol - eye_width) / 2;
```

#### **5.3 Jitter Impact on BER**

**BER Degradation Formula**:
```
BER_with_jitter = BER_no_jitter · (1 + (σ_jitter / T_eye)²)

Example:
  BER without jitter: 10⁻⁶
  Jitter: σ_jitter = 0.1·T_symbol
  Eye width: T_eye = 0.8·T_symbol

  BER_with_jitter = 10⁻⁶ · (1 + (0.1/0.8)²)
                  = 10⁻⁶ · 1.0156
                  ≈ 1.016×10⁻⁶  (1.6% increase)
```

**Rule of Thumb**:
```
For BER < 10⁻⁹:
  Total jitter < 0.2·T_symbol (20% of symbol period)

For BER < 10⁻⁶:
  Total jitter < 0.3·T_symbol (30% of symbol period)
```

---

### Section 6: PlutoSDR-Specific Considerations

#### **6.1 AD9361 Sampling Limitations**

**ADC/DAC Resolution**: 12 bits
```
Quantization levels: 2¹² = 4096

Quantization SNR: 6.02·N + 1.76 dB
                = 6.02·12 + 1.76 = 74 dB

This is theoretical maximum SNR - actual SNR lower due to:
  - Thermal noise
  - Phase noise
  - Spurious signals

Practical SNR: 60-65 dB for clean signals
```

**Impact on Q-Factor**:
```
Maximum measurable Q_factor ≈ 10·log₁₀(SNR) / 2
                            ≈ 10·log₁₀(60 dB) / 2
                            ≈ 9 (corresponds to BER ≈ 10⁻¹⁹)

In practice, PlutoSDR can measure Q up to ~8-9
  → Minimum measurable BER ≈ 10⁻¹⁵
```

#### **6.2 Clock Accuracy**

**AD9361 Clock Sources**:
```
Internal oscillator: ±25 ppm accuracy
External clock: Accuracy depends on source

Frequency error at 2.4 GHz:
  Δf = 2.4 GHz · 25 ppm = 60 kHz

For 100 ksps symbol rate:
  CFO/symbol_rate = 60 kHz / 100 kHz = 0.6

This is significant! Need carrier frequency correction.
```

**Jitter from Clock**:
```
AD9361 phase noise: -145 dBc/Hz @ 1 MHz offset

Integrated jitter (10 kHz to 40 MHz):
  σ_jitter ≈ 0.5 ps (picoseconds)

At 100 ksps (T_symbol = 10 μs):
  σ_jitter / T_symbol = 0.5 ps / 10 μs = 5×10⁻⁸

This is negligible for typical SDR applications
```

#### **6.3 I/Q Imbalance**

**Gain Imbalance**:
```
I_actual = α·I_ideal
Q_actual = β·Q_ideal

Typical: |α - β| < 0.5 dB

Effect on constellation:
  - Elliptical instead of circular
  - Increased BER
```

**Phase Imbalance**:
```
I_actual = I_ideal
Q_actual = Q_ideal·cos(Δφ) + I_ideal·sin(Δφ)

Typical: |Δφ| < 2°

Effect:
  - Rotated constellation
  - Crosstalk between I and Q
```

**Correction in C**:
```c
// Estimate and correct I/Q imbalance
typedef struct {
    double alpha;  // I-channel gain
    double beta;   // Q-channel gain
    double phi;    // Phase imbalance (radians)
} IQImbalance;

void correct_iq_imbalance(complex double *samples, size_t len, IQImbalance *imb) {
    for (size_t i = 0; i < len; i++) {
        double I = creal(samples[i]);
        double Q = cimag(samples[i]);

        // Correct gain imbalance
        I /= imb->alpha;
        Q /= imb->beta;

        // Correct phase imbalance
        double I_corr = I;
        double Q_corr = Q * cos(imb->phi) - I * sin(imb->phi);

        samples[i] = I_corr + I * Q_corr;
    }
}
```

#### **6.4 Memory and Processing Constraints**

**ARM Cortex-A9 @ 667 MHz**:
```
Available memory: ~512 MB total
  - ~300 MB free for applications

For BER testing:
  - Need to store TX and RX bits
  - Need buffers for eye diagram samples

Memory budget:
  10⁷ bits = 1.25 MB (manageable)
  10⁸ bits = 12.5 MB (still OK)
  10⁹ bits = 125 MB (approaching limit)
```

**Processing Time**:
```
BER counting: ~10⁶ bits/second (naive loop)
With NEON: ~10⁷ bits/second (10× faster)

Eye diagram: ~10⁴ symbols/second (without optimization)
With NEON: ~10⁵ symbols/second

Practical approach:
  - Process in chunks (streaming)
  - Use circular buffers
  - Offload computation to host PC if needed
```

---

### Section 7: Algorithms for Efficient Implementation

#### **7.1 Fast BER Counting**

**Naive Method** (slow):
```c
for (size_t i = 0; i < n_bits; i++) {
    if (tx_bits[i] != rx_bits[i]) errors++;
}
// O(n) bit-by-bit comparison
```

**Optimized Method** (byte-wise XOR + popcount):
```c
// Pack bits into bytes (8 bits per byte)
size_t n_bytes = n_bits / 8;
uint8_t *tx_bytes = pack_bits_to_bytes(tx_bits, n_bits);
uint8_t *rx_bytes = pack_bits_to_bytes(rx_bits, n_bits);

size_t errors = 0;
for (size_t i = 0; i < n_bytes; i++) {
    uint8_t diff = tx_bytes[i] ^ rx_bytes[i];
    errors += __builtin_popcount(diff);  // Count set bits (errors)
}
// O(n/8) with hardware popcount instruction
```

**NEON SIMD Method** (fastest on ARM):
```c
// Process 16 bytes at a time
for (size_t i = 0; i < n_bytes; i += 16) {
    uint8x16_t tx_vec = vld1q_u8(&tx_bytes[i]);
    uint8x16_t rx_vec = vld1q_u8(&rx_bytes[i]);
    uint8x16_t diff_vec = veorq_u8(tx_vec, rx_vec);
    errors += vcntq_u8(diff_vec);  // NEON popcount
}
// ~100× faster than naive method
```

#### **7.2 Streaming Eye Diagram Construction**

**Memory-Efficient Approach**:
```c
// Don't store all samples - build histogram directly

#define EYE_PHASES 16
#define EYE_LEVELS 256

uint32_t eye_histogram[EYE_PHASES][EYE_LEVELS];

void update_eye_diagram(complex double *rx_signal, size_t len, int sps) {
    for (size_t k = 0; k < len / sps; k++) {
        for (int phase = 0; phase < EYE_PHASES; phase++) {
            int sample_idx = k * sps + (phase * sps) / EYE_PHASES;
            double amplitude = cabs(rx_signal[sample_idx]);

            // Quantize to histogram bin
            int level = (int)((amplitude + 1.0) * 127.5);
            if (level < 0) level = 0;
            if (level > 255) level = 255;

            eye_histogram[phase][level]++;
        }
    }
}
// Memory: 16 × 256 × 4 bytes = 16 KB (vs MB for raw samples)
```

#### **7.3 Online Q-Factor Estimation**

**Welford's Algorithm** (numerically stable, one-pass):
```c
typedef struct {
    size_t n;
    double mean;
    double M2;  // Sum of squared deviations
} OnlineStats;

void update_stats(OnlineStats *stats, double value) {
    stats->n++;
    double delta = value - stats->mean;
    stats->mean += delta / stats->n;
    stats->M2 += delta * (value - stats->mean);
}

double get_variance(OnlineStats *stats) {
    return stats->M2 / stats->n;
}

double get_stddev(OnlineStats *stats) {
    return sqrt(get_variance(stats));
}

// Usage for Q-factor:
OnlineStats ones_stats = {0};
OnlineStats zeros_stats = {0};

for (size_t k = 0; k < num_samples; k++) {
    double sample = rx_signal[k];
    if (tx_bits[k] == 1) {
        update_stats(&ones_stats, sample);
    } else {
        update_stats(&zeros_stats, sample);
    }
}

double mu_1 = ones_stats.mean;
double mu_0 = zeros_stats.mean;
double sigma_1 = get_stddev(&ones_stats);
double sigma_0 = get_stddev(&zeros_stats);

double Q_factor = fabs(mu_1 - mu_0) / (sigma_1 + sigma_0);
```

---

### Summary of Theory

This theory section covered:

✅ **BER Statistics**: Binomial distribution, confidence intervals, required sample sizes for accurate measurement

✅ **Eye Diagrams**: Mathematical construction, key metrics (height, width, opening), factors causing closure

✅ **Q-Factor**: Definition, relationship to BER, histogram-based measurement technique

✅ **ISI Analysis**: Mathematical model, impact on eye diagrams, power measurement

✅ **Jitter Analysis**: Random vs deterministic jitter, measurement methods, BER degradation formulas

✅ **PlutoSDR Constraints**: AD9361 resolution limits, clock accuracy, I/Q imbalance correction

✅ **Efficient Algorithms**: Fast BER counting with NEON, streaming eye diagrams, online Q-factor estimation

**Key Formulas for Implementation**:
```
BER = errors / total_bits
Q_factor = |μ₁ - μ₀| / (σ₁ + σ₀)
Eye_height = |μ₁ - μ₀| - 3·(σ₁ + σ₀)
Confidence_interval = BER ± 1.96·√(BER·(1-BER)/n_bits)
```

**Next**: Part 4 will provide complete C source code implementing:
- Multi-modulation BER testing framework
- Real-time eye diagram generator
- Q-factor measurement with statistical analysis
- ISI and jitter measurement tools
- PlutoSDR hardware integration

---

## Summary

In this lab, you learned:

✅ **BER Measurement**:
   - Definition and calculation
   - Statistical confidence requirements
   - Typical BER targets for applications

✅ **Eb/N0 vs. SNR**:
   - Relationship and conversion
   - BER performance curves
   - Theoretical vs. measured BER

✅ **Eye Diagrams**:
   - Visual representation of signal quality
   - Eye opening, height, width
   - Timing jitter and ISI

✅ **Q-Factor**:
   - Signal quality metric
   - Relationship to BER
   - Practical measurement

✅ **Monte Carlo Simulation**:
   - BER testing methodology
   - AWGN channel modeling
   - Performance verification

---

## Next Steps

Continue to:
- **LAB 3.5**: Pulse Shaping and Matched Filtering
- **LAB 4.1**: FIR and IIR Filter Design
