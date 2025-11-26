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
