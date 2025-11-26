# LAB 3.1: Digital Modulation - ASK, FSK, and PSK

## Overview

This lab explores the three fundamental **digital modulation** schemes: **ASK (Amplitude Shift Keying)**, **FSK (Frequency Shift Keying)**, and **PSK (Phase Shift Keying)**. You'll learn how digital data is encoded onto RF carriers using these methods, and compare their performance characteristics.

## Learning Objectives

After completing this lab, you will understand:
- Binary modulation schemes: BASK, BFSK, BPSK
- Modulation process: bits → symbols → RF waveform
- Demodulation and symbol detection
- Signal constellations and decision regions
- Bandwidth requirements for each scheme
- Bit Error Rate (BER) performance vs SNR
- Spectral efficiency trade-offs
- PlutoSDR implementation of all three schemes

## Prerequisites

- LAB 0: PlutoSDR Setup
- LAB 1.1: Basic IQ Sampling
- LAB 1.3: I/Q Modulation
- LAB 2.1-2.3: Sampling and quantization
- Understanding of complex exponentials
- Basic Python or C programming

## Theory

### 1. Digital Modulation Fundamentals

**Goal**: Transmit digital bits over an RF channel.

```
Data → Modulator → RF Channel → Demodulator → Data
Bits     ↓                           ↓         Bits
       Symbol                      Symbol
       mapping                    detection

Three parameters can be modulated:
  1. Amplitude → ASK (Amplitude Shift Keying)
  2. Frequency → FSK (Frequency Shift Keying)
  3. Phase     → PSK (Phase Shift Keying)
```

**Carrier Signal**:
```
s(t) = A(t) · cos(2πfc·t + φ(t))

Where:
  A(t) = Amplitude (varies in ASK)
  fc   = Carrier frequency
  φ(t) = Phase (varies in PSK)

For FSK: fc itself varies
```

**Complex Baseband Representation**:
```
s(t) = Re{s_c(t) · e^(j2πfc·t)}

where s_c(t) = I(t) + jQ(t) is the complex baseband signal

Modulation schemes differ in how they encode bits into I(t) and Q(t)
```

### 2. ASK (Amplitude Shift Keying)

**Binary ASK (BASK) / On-Off Keying (OOK)**:

```
Bit 0 → Amplitude = 0      (carrier off)
Bit 1 → Amplitude = A      (carrier on)

s(t) = A · b(t) · cos(2πfc·t)

where b(t) ∈ {0, 1} is the bit sequence

Example: T = symbol period = 1 µs
  fc = 1 MHz

  Bit: 1    0    1    1    0
       ___       ___  ___
      |   |     |   ||   |
  ____     _____     ____
  0   1    2    3   4    5  (µs)

Complex baseband:
  Bit 0 → s_c = 0
  Bit 1 → s_c = A
```

**M-ary ASK**: M amplitude levels → log₂(M) bits/symbol

```
4-ASK (M=4): 2 bits/symbol
  00 → A₀ = 0.25
  01 → A₁ = 0.50
  10 → A₂ = 0.75
  11 → A₃ = 1.00

Constellation (I-axis only):
   0.25  0.50  0.75  1.00
    ●     ●     ●     ●   (I)
```

**ASK Advantages**:
- Simple to implement
- Good for optical communication (laser on/off)

**ASK Disadvantages**:
- Poor noise performance
- Susceptible to fading
- DC component (for OOK)
- Requires linear amplifiers

**Bandwidth**:
```
BW_ASK ≈ 2·R_s  (null-to-null)

where R_s = symbol rate = 1/T

Example: 1 Mbps BASK
  R_s = 1 Msps
  BW ≈ 2 MHz
```

### 3. FSK (Frequency Shift Keying)

**Binary FSK (BFSK)**:

```
Bit 0 → Frequency = fc - Δf  (mark frequency)
Bit 1 → Frequency = fc + Δf  (space frequency)

s(t) = A · cos(2π(fc + Δf·m(t))·t)

where m(t) ∈ {-1, +1} represents bits

Example: fc = 1 MHz, Δf = 100 kHz, T = 10 µs
  Bit 0: 900 kHz tone for 10 µs
  Bit 1: 1.1 MHz tone for 10 µs

  Bit: 0         1         0
       ~~~~~~~   ~~~~~     ~~~~~~~
       (slow)    (fast)    (slow)
```

**Complex Baseband** (at fc):
```
Bit 0 → s_c(t) = A · e^(-j2πΔf·t)  (rotate CCW)
Bit 1 → s_c(t) = A · e^(+j2πΔf·t)  (rotate CW)

This is frequency modulation relative to fc
```

**Modulation Index**:
```
h = 2·Δf·T  (modulation index)

h < 1: Narrow-band FSK (frequencies overlap)
h = 0.5: Minimum Shift Keying (MSK)
h ≥ 1: Wide-band FSK (orthogonal tones)

For orthogonality (optimal demodulation):
  Δf = n/(2T), where n = 1, 2, 3, ...

Example: T = 10 µs, n = 1
  Δf = 1/(2×10µs) = 50 kHz
  h = 2×50k×10µ = 1.0 (orthogonal)
```

**FSK Advantages**:
- Constant envelope (efficient power amplifiers)
- Good noise performance
- No DC component
- Works well with non-linear amplifiers

**FSK Disadvantages**:
- Wider bandwidth than PSK
- More complex demodulation
- Frequency stability required

**Bandwidth**:
```
Carson's Rule:
  BW_FSK ≈ 2·(Δf + R_s)

Example: 100 kbps BFSK, Δf = 50 kHz
  BW ≈ 2·(50k + 100k) = 300 kHz

Compare to BASK: 200 kHz (FSK is 1.5× wider)
```

### 4. PSK (Phase Shift Keying)

**Binary PSK (BPSK)**:

```
Bit 0 → Phase = 0°    (or reference phase)
Bit 1 → Phase = 180°  (phase reversal)

s(t) = A · cos(2πfc·t + φ(t))

where φ(t) ∈ {0, π}

Equivalently:
  Bit 0 → s(t) = +A · cos(2πfc·t)
  Bit 1 → s(t) = -A · cos(2πfc·t)

Complex baseband:
  Bit 0 → s_c = +A  (I=+A, Q=0)
  Bit 1 → s_c = -A  (I=-A, Q=0)
```

**BPSK Constellation**:
```
   I-axis

  ●――――――――――●
 -A          +A
 (bit 1)   (bit 0)

Decision boundary: I = 0
  Received I > 0 → bit 0
  Received I < 0 → bit 1

Q-axis not used (can be thought of as BASK on I-channel)
```

**Quadrature PSK (QPSK)**: 4 phases, 2 bits/symbol
```
00 → 45°   (I=+A/√2, Q=+A/√2)
01 → 135°  (I=-A/√2, Q=+A/√2)
11 → 225°  (I=-A/√2, Q=-A/√2)
10 → 315°  (I=+A/√2, Q=-A/√2)

Constellation:
        Q
        |
    01  |  00
   ●    |   ●
  ------+------  I
   ●    |   ●
    11  |  10
        |

Gray coding: Adjacent symbols differ by 1 bit
```

**PSK Advantages**:
- Constant envelope (like FSK)
- Best noise performance (for same bandwidth)
- Bandwidth efficient
- Higher-order PSK: 8-PSK, 16-PSK, etc.

**PSK Disadvantages**:
- Requires carrier phase synchronization
- More complex demodulation than FSK
- Sensitive to phase noise

**Bandwidth**:
```
BW_BPSK ≈ 2·R_s  (same as BASK)

BW_QPSK ≈ R_s  (half of BPSK for same bit rate!)

Example: 1 Mbps
  BPSK: R_s = 1 Msps, BW = 2 MHz
  QPSK: R_s = 0.5 Msps, BW = 1 MHz

QPSK is 2× more bandwidth efficient!
```

### 5. Demodulation Methods

**Coherent Demodulation** (optimal for PSK):
```
Requires carrier phase reference

BPSK demodulator:
  1. Multiply by local oscillator: cos(2πfc·t + φ_0)
  2. Lowpass filter
  3. Sample at symbol rate
  4. Threshold decision: > 0 → bit 0, < 0 → bit 1

Must estimate and track φ_0 (carrier phase)
```

**Non-coherent Demodulation** (for FSK):
```
No carrier phase needed

FSK demodulator (frequency discriminator):
  1. Measure instantaneous frequency
  2. Compare to threshold
  3. f > fc → bit 1
     f < fc → bit 0

Or: Dual bandpass filters + envelope detectors
  - Filter 1 at fc - Δf
  - Filter 2 at fc + Δf
  - Compare energies → decide bit
```

**Envelope Detection** (for ASK):
```
Simplest: Rectifier + lowpass filter

  Input → |Rectifier| → LPF → Threshold → Bits

No carrier phase needed
Poor noise performance (squaring loss ~3 dB)
```

### 6. Bit Error Rate (BER) Performance

**BER vs SNR** (AWGN channel):

```
BPSK (coherent):
  BER = Q(√(2·Eb/N0))

  where Eb/N0 = energy per bit / noise spectral density
        Q(x) = Gaussian Q-function

BFSK (non-coherent):
  BER = 0.5 · e^(-Eb/N0 / 2)

BFSK (coherent):
  BER = Q(√(Eb/N0))

BASK (coherent):
  BER = Q(√(Eb/N0))  (same as BFSK coherent)

BASK (envelope detection):
  BER = Q(√(Eb/N0 / 2))  (~3 dB worse)
```

**Performance Comparison** (BER = 10⁻³):

```
Scheme               Eb/N0 required    Notes
------------------------------------------------------
BPSK (coherent)      6.8 dB            Best
BFSK (coherent)      9.6 dB            2.8 dB worse
BFSK (non-coherent)  12.5 dB           5.7 dB worse
BASK (coherent)      9.6 dB            Same as BFSK
BASK (envelope)      12.5 dB           Poor

For same BER, BPSK needs 3-6 dB less power than FSK/ASK!
```

**Why is BPSK better?**
```
BPSK symbols are maximally separated:
  Distance between +A and -A: d = 2A

BFSK (orthogonal):
  Distance: d = √2 · A  (less than BPSK)

Larger separation → better noise immunity → lower BER
```

### 7. Spectral Efficiency

**Comparison**:

```
Scheme    Bits/symbol    Bandwidth       Efficiency (bits/s/Hz)
---------------------------------------------------------------
BASK      1              2·Rs            0.5
BFSK      1              2·(Δf + Rs)     ~0.3-0.4 (depends on h)
BPSK      1              2·Rs            0.5
QPSK      2              Rs              1.0
16-QAM    4              Rs              2.0

Higher-order modulation → Better spectral efficiency
But: Lower noise immunity!
```

**Trade-off**:
```
                  Spectral Efficiency
                         ↑
    16-QAM ●
           |
     8-PSK ●
           |
      QPSK ●
           |
      BPSK ●――――――――――→ Noise Immunity
           |
      BFSK ●
      BASK ●

Choose modulation based on:
  - Available bandwidth (limited → higher order)
  - SNR (low SNR → BPSK/BFSK)
  - Complexity (simple → FSK)
  - Power efficiency (critical → BPSK)
```

---

## Part 1: Simulation (Pure Python)

### Implementation 1: ASK Modulator and Demodulator

```python
import numpy as np
import matplotlib.pyplot as plt

class ASKModem:
    """Binary ASK (On-Off Keying) modem"""

    def __init__(self, carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20):
        """
        Args:
            carrier_freq: Carrier frequency (Hz)
            symbol_rate: Symbol rate (symbols/s)
            samples_per_symbol: Samples per symbol period
        """
        self.fc = carrier_freq
        self.Rs = symbol_rate
        self.Ts = 1 / symbol_rate
        self.sps = samples_per_symbol
        self.fs = symbol_rate * samples_per_symbol

        print(f"ASK Modem Configuration:")
        print(f"  Carrier freq:     {carrier_freq/1e6:.2f} MHz")
        print(f"  Symbol rate:      {symbol_rate/1e3:.0f} ksps")
        print(f"  Symbol period:    {self.Ts*1e6:.1f} µs")
        print(f"  Bandwidth (null): {2*symbol_rate/1e6:.2f} MHz")

    def modulate(self, bits):
        """
        Modulate bits using ASK

        Args:
            bits: Binary data (0s and 1s)

        Returns:
            Modulated signal (real samples)
        """
        n_symbols = len(bits)
        n_samples = n_symbols * self.sps

        # Upsample bits (pulse shaping - rectangular for now)
        symbols = np.repeat(bits, self.sps)

        # Generate carrier
        t = np.arange(n_samples) / self.fs
        carrier = np.cos(2 * np.pi * self.fc * t)

        # ASK modulation: multiply symbols by carrier
        signal = symbols * carrier

        return signal, t

    def demodulate(self, signal, method='coherent'):
        """
        Demodulate ASK signal

        Args:
            signal: Received ASK signal
            method: 'coherent' or 'envelope'

        Returns:
            Detected bits
        """
        n_samples = len(signal)
        n_symbols = n_samples // self.sps

        if method == 'coherent':
            # Coherent demodulation: multiply by carrier and integrate
            t = np.arange(n_samples) / self.fs
            carrier = np.cos(2 * np.pi * self.fc * t)

            # Multiply by local oscillator
            demod = signal * carrier

            # Lowpass filter (simple moving average)
            # In practice, use proper filter
            from scipy import signal as sig
            b, a = sig.butter(4, self.Rs / (self.fs/2), btype='low')
            demod_filt = sig.filtfilt(b, a, demod)

            # Sample at symbol rate
            samples = demod_filt[self.sps//2::self.sps]

            # Threshold decision
            threshold = np.mean(samples) / 2
            bits = (samples > threshold).astype(int)

        elif method == 'envelope':
            # Envelope detection: |signal| → LPF → sample → threshold
            envelope = np.abs(signal)

            # Lowpass filter
            from scipy import signal as sig
            b, a = sig.butter(4, self.Rs / (self.fs/2), btype='low')
            envelope_filt = sig.filtfilt(b, a, envelope)

            # Sample at symbol rate
            samples = envelope_filt[self.sps//2::self.sps]

            # Threshold
            threshold = np.mean(samples) / 2
            bits = (samples > threshold).astype(int)

        else:
            raise ValueError(f"Unknown method: {method}")

        return bits[:n_symbols]

    def plot_constellation(self):
        """Plot ASK constellation"""
        fig, ax = plt.subplots(1, 1, figsize=(6, 6))

        # BASK: two points on I-axis
        ax.plot([0], [0], 'ro', markersize=15, label='Bit 0')
        ax.plot([1], [0], 'bo', markersize=15, label='Bit 1')

        ax.set_xlabel('In-phase (I)')
        ax.set_ylabel('Quadrature (Q)')
        ax.set_title('BASK Constellation')
        ax.grid(True, alpha=0.3)
        ax.legend()
        ax.set_xlim([-0.5, 1.5])
        ax.set_ylim([-0.5, 0.5])
        ax.axhline(0, color='k', linewidth=0.5)
        ax.axvline(0, color='k', linewidth=0.5)

        plt.tight_layout()
        plt.savefig('ask_constellation.png', dpi=150, bbox_inches='tight')
        print("✓ Saved ask_constellation.png")
        plt.show()


# Test ASK modem
if __name__ == "__main__":
    print("="*70)
    print("ASK MODULATION DEMONSTRATION")
    print("="*70)

    modem = ASKModem(carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20)

    # Generate random bits
    np.random.seed(42)
    bits = np.random.randint(0, 2, size=20)
    print(f"\nTransmit bits: {bits[:10]}...")

    # Modulate
    signal, t = modem.modulate(bits)

    # Demodulate (both methods)
    bits_coherent = modem.demodulate(signal, method='coherent')
    bits_envelope = modem.demodulate(signal, method='envelope')

    # Check errors
    errors_coherent = np.sum(bits != bits_coherent)
    errors_envelope = np.sum(bits != bits_envelope)

    print(f"\nCoherent demod: {errors_coherent} errors")
    print(f"Envelope demod: {errors_envelope} errors")

    # Plot
    fig, axes = plt.subplots(3, 1, figsize=(12, 9))

    # Transmitted bits
    t_bits = np.arange(len(bits)) * modem.Ts
    axes[0].step(t_bits*1e6, bits, 'b-', linewidth=2, where='post')
    axes[0].set_xlabel('Time (µs)')
    axes[0].set_ylabel('Bit value')
    axes[0].set_title('Digital Data')
    axes[0].grid(True, alpha=0.3)
    axes[0].set_ylim([-0.2, 1.2])

    # Modulated signal (zoom to first few symbols)
    n_show = 5 * modem.sps
    axes[1].plot(t[:n_show]*1e6, signal[:n_show], 'r-', linewidth=1)
    axes[1].set_xlabel('Time (µs)')
    axes[1].set_ylabel('Amplitude')
    axes[1].set_title('ASK Modulated Signal')
    axes[1].grid(True, alpha=0.3)

    # Spectrum
    fft_signal = np.fft.fftshift(np.fft.fft(signal))
    freqs = np.fft.fftshift(np.fft.fftfreq(len(signal), 1/modem.fs))
    spectrum = 20 * np.log10(np.abs(fft_signal) / len(signal) + 1e-12)

    axes[2].plot(freqs/1e6, spectrum, 'b-', linewidth=1)
    axes[2].set_xlabel('Frequency (MHz)')
    axes[2].set_ylabel('Magnitude (dB)')
    axes[2].set_title('ASK Spectrum')
    axes[2].grid(True, alpha=0.3)
    axes[2].set_xlim([0, 3])
    axes[2].axvline(modem.fc/1e6, color='r', linestyle='--', label='Carrier')
    axes[2].legend()

    plt.tight_layout()
    plt.savefig('ask_modulation.png', dpi=150, bbox_inches='tight')
    print("✓ Saved ask_modulation.png")
    plt.show()

    # Plot constellation
    modem.plot_constellation()
```

### Implementation 2: FSK Modulator and Demodulator

```python
class FSKModem:
    """Binary FSK modem"""

    def __init__(self, carrier_freq=1e6, freq_deviation=50e3,
                 symbol_rate=100e3, samples_per_symbol=20):
        """
        Args:
            carrier_freq: Center frequency (Hz)
            freq_deviation: Frequency deviation Δf (Hz)
            symbol_rate: Symbol rate (symbols/s)
            samples_per_symbol: Samples per symbol
        """
        self.fc = carrier_freq
        self.delta_f = freq_deviation
        self.Rs = symbol_rate
        self.Ts = 1 / symbol_rate
        self.sps = samples_per_symbol
        self.fs = symbol_rate * samples_per_symbol

        # Modulation index
        self.h = 2 * freq_deviation * self.Ts

        print(f"FSK Modem Configuration:")
        print(f"  Center freq:      {carrier_freq/1e6:.2f} MHz")
        print(f"  Deviation:        {freq_deviation/1e3:.0f} kHz")
        print(f"  Modulation index: {self.h:.2f}")
        print(f"  Mark freq:        {(carrier_freq - freq_deviation)/1e6:.3f} MHz (bit 0)")
        print(f"  Space freq:       {(carrier_freq + freq_deviation)/1e6:.3f} MHz (bit 1)")
        print(f"  Symbol rate:      {symbol_rate/1e3:.0f} ksps")
        print(f"  Bandwidth:        {2*(freq_deviation + symbol_rate)/1e6:.2f} MHz (Carson's rule)")

    def modulate(self, bits):
        """
        Modulate bits using FSK

        Args:
            bits: Binary data

        Returns:
            Modulated signal
        """
        n_symbols = len(bits)
        n_samples = n_symbols * self.sps

        # Map bits to frequencies
        # Bit 0 → -Δf, Bit 1 → +Δf
        freq_deviations = 2*bits - 1  # {0,1} → {-1,+1}
        freq_deviations = np.repeat(freq_deviations, self.sps) * self.delta_f

        # Generate FSK signal using continuous phase
        t = np.arange(n_samples) / self.fs

        # Instantaneous frequency: fc + Δf(t)
        # Phase: integral of 2π·f(t)
        phase = 2 * np.pi * self.fc * t + 2 * np.pi * np.cumsum(freq_deviations) / self.fs

        signal = np.cos(phase)

        return signal, t

    def demodulate(self, signal):
        """
        Demodulate FSK using dual filter method

        Args:
            signal: Received FSK signal

        Returns:
            Detected bits
        """
        from scipy import signal as sig

        # Design bandpass filters for mark and space frequencies
        f0 = self.fc - self.delta_f  # Mark frequency (bit 0)
        f1 = self.fc + self.delta_f  # Space frequency (bit 1)

        bw = self.Rs  # Filter bandwidth ~ symbol rate

        # Bandpass filter 1 (mark)
        b0, a0 = sig.butter(4, [f0 - bw/2, f0 + bw/2], btype='band', fs=self.fs)
        filtered0 = sig.filtfilt(b0, a0, signal)

        # Bandpass filter 2 (space)
        b1, a1 = sig.butter(4, [f1 - bw/2, f1 + bw/2], btype='band', fs=self.fs)
        filtered1 = sig.filtfilt(b1, a1, signal)

        # Envelope detection
        env0 = np.abs(filtered0)
        env1 = np.abs(filtered1)

        # Sample at symbol rate
        env0_sampled = env0[self.sps//2::self.sps]
        env1_sampled = env1[self.sps//2::self.sps]

        # Decide bit based on which envelope is larger
        bits = (env1_sampled > env0_sampled).astype(int)

        return bits

    def plot_constellation(self):
        """
        Plot FSK constellation (as points on unit circle rotating)
        FSK doesn't have a traditional constellation, but we can visualize
        the two frequency tones
        """
        fig, ax = plt.subplots(1, 1, figsize=(6, 6))

        # For visualization: show start and end points of symbol period
        # Bit 0: rotates at -Δf for time Ts
        # Bit 1: rotates at +Δf for time Ts

        theta0 = -2 * np.pi * self.delta_f * self.Ts
        theta1 = +2 * np.pi * self.delta_f * self.Ts

        # Draw rotation arcs
        t = np.linspace(0, self.Ts, 100)
        phase0 = -2 * np.pi * self.delta_f * t
        phase1 = +2 * np.pi * self.delta_f * t

        ax.plot(np.cos(phase0), np.sin(phase0), 'r-', linewidth=2, label='Bit 0 trajectory')
        ax.plot(np.cos(phase1), np.sin(phase1), 'b-', linewidth=2, label='Bit 1 trajectory')

        ax.plot([1], [0], 'ko', markersize=10, label='Start')
        ax.plot([np.cos(theta0)], [np.sin(theta0)], 'ro', markersize=10)
        ax.plot([np.cos(theta1)], [np.sin(theta1)], 'bo', markersize=10)

        # Unit circle
        circle = plt.Circle((0, 0), 1, fill=False, color='gray', linestyle='--', alpha=0.5)
        ax.add_patch(circle)

        ax.set_xlabel('In-phase (I)')
        ax.set_ylabel('Quadrature (Q)')
        ax.set_title(f'BFSK Phase Trajectory (h={self.h:.2f})')
        ax.grid(True, alpha=0.3)
        ax.legend()
        ax.set_xlim([-1.5, 1.5])
        ax.set_ylim([-1.5, 1.5])
        ax.set_aspect('equal')

        plt.tight_layout()
        plt.savefig('fsk_constellation.png', dpi=150, bbox_inches='tight')
        print("✓ Saved fsk_constellation.png")
        plt.show()


# Test FSK modem
if __name__ == "__main__":
    print("\n" + "="*70)
    print("FSK MODULATION DEMONSTRATION")
    print("="*70)

    modem = FSKModem(carrier_freq=1e6, freq_deviation=50e3,
                     symbol_rate=100e3, samples_per_symbol=20)

    # Generate random bits
    np.random.seed(42)
    bits = np.random.randint(0, 2, size=20)
    print(f"\nTransmit bits: {bits[:10]}...")

    # Modulate
    signal, t = modem.modulate(bits)

    # Demodulate
    bits_rx = modem.demodulate(signal)

    # Check errors
    errors = np.sum(bits != bits_rx)
    print(f"\nDemodulation errors: {errors}")

    # Plot
    fig, axes = plt.subplots(3, 1, figsize=(12, 9))

    # Modulated signal (zoom)
    n_show = 5 * modem.sps
    axes[0].plot(t[:n_show]*1e6, signal[:n_show], 'b-', linewidth=1)
    axes[0].set_xlabel('Time (µs)')
    axes[0].set_ylabel('Amplitude')
    axes[0].set_title('FSK Modulated Signal (note frequency changes)')
    axes[0].grid(True, alpha=0.3)

    # Instantaneous frequency (visualization)
    from scipy import signal as sig
    analytic = sig.hilbert(signal)
    inst_phase = np.unwrap(np.angle(analytic))
    inst_freq = np.diff(inst_phase) / (2*np.pi) * modem.fs

    axes[1].plot(t[1:n_show]*1e6, inst_freq[:n_show-1]/1e6, 'r-', linewidth=1)
    axes[1].axhline((modem.fc - modem.delta_f)/1e6, color='g', linestyle='--', label='Mark (bit 0)')
    axes[1].axhline((modem.fc + modem.delta_f)/1e6, color='b', linestyle='--', label='Space (bit 1)')
    axes[1].set_xlabel('Time (µs)')
    axes[1].set_ylabel('Frequency (MHz)')
    axes[1].set_title('Instantaneous Frequency')
    axes[1].grid(True, alpha=0.3)
    axes[1].legend()

    # Spectrum
    fft_signal = np.fft.fftshift(np.fft.fft(signal))
    freqs = np.fft.fftshift(np.fft.fftfreq(len(signal), 1/modem.fs))
    spectrum = 20 * np.log10(np.abs(fft_signal) / len(signal) + 1e-12)

    axes[2].plot(freqs/1e6, spectrum, 'b-', linewidth=1)
    axes[2].set_xlabel('Frequency (MHz)')
    axes[2].set_ylabel('Magnitude (dB)')
    axes[2].set_title('FSK Spectrum (two tones visible)')
    axes[2].grid(True, alpha=0.3)
    axes[2].set_xlim([0.5, 1.5])
    axes[2].axvline((modem.fc - modem.delta_f)/1e6, color='g', linestyle='--', alpha=0.5)
    axes[2].axvline((modem.fc + modem.delta_f)/1e6, color='b', linestyle='--', alpha=0.5)

    plt.tight_layout()
    plt.savefig('fsk_modulation.png', dpi=150, bbox_inches='tight')
    print("✓ Saved fsk_modulation.png")
    plt.show()

    # Plot constellation
    modem.plot_constellation()
```

### Implementation 3: PSK Modulator and Demodulator

```python
class PSKModem:
    """Binary PSK (BPSK) modem"""

    def __init__(self, carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20):
        """
        Args:
            carrier_freq: Carrier frequency (Hz)
            symbol_rate: Symbol rate (symbols/s)
            samples_per_symbol: Samples per symbol
        """
        self.fc = carrier_freq
        self.Rs = symbol_rate
        self.Ts = 1 / symbol_rate
        self.sps = samples_per_symbol
        self.fs = symbol_rate * samples_per_symbol

        print(f"BPSK Modem Configuration:")
        print(f"  Carrier freq:     {carrier_freq/1e6:.2f} MHz")
        print(f"  Symbol rate:      {symbol_rate/1e3:.0f} ksps")
        print(f"  Bandwidth (null): {2*symbol_rate/1e6:.2f} MHz")

    def modulate(self, bits):
        """
        Modulate bits using BPSK

        Args:
            bits: Binary data

        Returns:
            Modulated signal
        """
        n_symbols = len(bits)
        n_samples = n_symbols * self.sps

        # Map bits to phase: 0 → 0°, 1 → 180°
        # Equivalently: 0 → +1, 1 → -1
        symbols = 2*bits - 1  # {0,1} → {+1,-1}
        symbols = np.repeat(symbols, self.sps)

        # Generate carrier
        t = np.arange(n_samples) / self.fs
        carrier = np.cos(2 * np.pi * self.fc * t)

        # BPSK: multiply symbols by carrier (phase reversal)
        signal = symbols * carrier

        return signal, t

    def demodulate(self, signal):
        """
        Coherent demodulation of BPSK

        Args:
            signal: Received BPSK signal

        Returns:
            Detected bits
        """
        from scipy import signal as sig

        n_samples = len(signal)
        t = np.arange(n_samples) / self.fs

        # Multiply by local oscillator (coherent)
        lo = np.cos(2 * np.pi * self.fc * t)
        demod = signal * lo

        # Lowpass filter
        b, a = sig.butter(4, self.Rs / (self.fs/2), btype='low')
        demod_filt = sig.filtfilt(b, a, demod)

        # Sample at symbol rate
        samples = demod_filt[self.sps//2::self.sps]

        # Threshold decision (0 is decision boundary)
        bits = (samples < 0).astype(int)  # Negative → bit 1, Positive → bit 0

        return bits

    def plot_constellation(self):
        """Plot BPSK constellation"""
        fig, ax = plt.subplots(1, 1, figsize=(6, 6))

        # BPSK: two points on I-axis at ±1
        ax.plot([-1], [0], 'ro', markersize=15, label='Bit 1 (180°)')
        ax.plot([+1], [0], 'bo', markersize=15, label='Bit 0 (0°)')

        # Decision boundary
        ax.axvline(0, color='k', linestyle='--', linewidth=2, alpha=0.5, label='Decision boundary')

        ax.set_xlabel('In-phase (I)')
        ax.set_ylabel('Quadrature (Q)')
        ax.set_title('BPSK Constellation')
        ax.grid(True, alpha=0.3)
        ax.legend()
        ax.set_xlim([-1.5, 1.5])
        ax.set_ylim([-0.5, 0.5])
        ax.axhline(0, color='k', linewidth=0.5)
        ax.axvline(0, color='k', linewidth=0.5)
        ax.set_aspect('equal')

        plt.tight_layout()
        plt.savefig('bpsk_constellation.png', dpi=150, bbox_inches='tight')
        print("✓ Saved bpsk_constellation.png")
        plt.show()


# Test BPSK modem
if __name__ == "__main__":
    print("\n" + "="*70)
    print("BPSK MODULATION DEMONSTRATION")
    print("="*70)

    modem = PSKModem(carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20)

    # Generate random bits
    np.random.seed(42)
    bits = np.random.randint(0, 2, size=20)
    print(f"\nTransmit bits: {bits[:10]}...")

    # Modulate
    signal, t = modem.modulate(bits)

    # Demodulate
    bits_rx = modem.demodulate(signal)

    # Check errors
    errors = np.sum(bits != bits_rx)
    print(f"\nDemodulation errors: {errors}")

    # Plot
    fig, axes = plt.subplots(3, 1, figsize=(12, 9))

    # Bits
    t_bits = np.arange(len(bits)) * modem.Ts
    axes[0].step(t_bits*1e6, bits, 'b-', linewidth=2, where='post')
    axes[0].set_xlabel('Time (µs)')
    axes[0].set_ylabel('Bit')
    axes[0].set_title('Digital Data')
    axes[0].grid(True, alpha=0.3)
    axes[0].set_ylim([-0.2, 1.2])

    # Modulated signal (note phase reversals)
    n_show = 5 * modem.sps
    axes[1].plot(t[:n_show]*1e6, signal[:n_show], 'r-', linewidth=1)
    axes[1].set_xlabel('Time (µs)')
    axes[1].set_ylabel('Amplitude')
    axes[1].set_title('BPSK Modulated Signal (note 180° phase shifts)')
    axes[1].grid(True, alpha=0.3)

    # Spectrum
    fft_signal = np.fft.fftshift(np.fft.fft(signal))
    freqs = np.fft.fftshift(np.fft.fftfreq(len(signal), 1/modem.fs))
    spectrum = 20 * np.log10(np.abs(fft_signal) / len(signal) + 1e-12)

    axes[2].plot(freqs/1e6, spectrum, 'b-', linewidth=1)
    axes[2].set_xlabel('Frequency (MHz)')
    axes[2].set_ylabel('Magnitude (dB)')
    axes[2].set_title('BPSK Spectrum (centered at carrier)')
    axes[2].grid(True, alpha=0.3)
    axes[2].set_xlim([0, 3])
    axes[2].axvline(modem.fc/1e6, color='r', linestyle='--', label='Carrier')
    axes[2].legend()

    plt.tight_layout()
    plt.savefig('bpsk_modulation.png', dpi=150, bbox_inches='tight')
    print("✓ Saved bpsk_modulation.png")
    plt.show()

    # Plot constellation
    modem.plot_constellation()
```

### Implementation 4: Modulation Comparison

```python
def compare_modulations():
    """Compare ASK, FSK, PSK side-by-side"""

    print("\n" + "="*70)
    print("MODULATION COMPARISON")
    print("="*70)

    # Common parameters
    fc = 1e6
    Rs = 100e3
    sps = 20

    # Create modems
    ask_modem = ASKModem(fc, Rs, sps)
    fsk_modem = FSKModem(fc, 50e3, Rs, sps)
    bpsk_modem = PSKModem(fc, Rs, sps)

    # Generate test data
    np.random.seed(42)
    bits = np.random.randint(0, 2, size=10)

    # Modulate with all three schemes
    ask_sig, t = ask_modem.modulate(bits)
    fsk_sig, _ = fsk_modem.modulate(bits)
    bpsk_sig, _ = bpsk_modem.modulate(bits)

    # Plot comparison
    fig, axes = plt.subplots(4, 2, figsize=(14, 12))

    n_show = 5 * sps

    # Time domain
    axes[0, 0].plot(t[:n_show]*1e6, ask_sig[:n_show], 'b-', linewidth=1)
    axes[0, 0].set_title('ASK - Time Domain')
    axes[0, 0].set_ylabel('Amplitude')
    axes[0, 0].grid(True, alpha=0.3)

    axes[1, 0].plot(t[:n_show]*1e6, fsk_sig[:n_show], 'g-', linewidth=1)
    axes[1, 0].set_title('FSK - Time Domain')
    axes[1, 0].set_ylabel('Amplitude')
    axes[1, 0].grid(True, alpha=0.3)

    axes[2, 0].plot(t[:n_show]*1e6, bpsk_sig[:n_show], 'r-', linewidth=1)
    axes[2, 0].set_title('BPSK - Time Domain')
    axes[2, 0].set_ylabel('Amplitude')
    axes[2, 0].set_xlabel('Time (µs)')
    axes[2, 0].grid(True, alpha=0.3)

    # Frequency domain
    def plot_spectrum(ax, signal, fs, fc, title):
        fft_sig = np.fft.fftshift(np.fft.fft(signal))
        freqs = np.fft.fftshift(np.fft.fftfreq(len(signal), 1/fs))
        spectrum = 20 * np.log10(np.abs(fft_sig) / len(signal) + 1e-12)
        ax.plot(freqs/1e6, spectrum, linewidth=1)
        ax.set_title(title)
        ax.set_ylabel('Magnitude (dB)')
        ax.set_xlim([0.5, 1.5])
        ax.set_ylim([-80, 0])
        ax.grid(True, alpha=0.3)
        ax.axvline(fc/1e6, color='r', linestyle='--', alpha=0.5)

    fs = ask_modem.fs
    plot_spectrum(axes[0, 1], ask_sig, fs, fc, 'ASK - Spectrum')
    plot_spectrum(axes[1, 1], fsk_sig, fs, fc, 'FSK - Spectrum (two tones)')
    plot_spectrum(axes[2, 1], bpsk_sig, fs, fc, 'BPSK - Spectrum')
    axes[2, 1].set_xlabel('Frequency (MHz)')

    # Summary table
    axes[3, 0].axis('off')
    axes[3, 1].axis('off')

    table_data = [
        ['Parameter', 'ASK', 'FSK', 'BPSK'],
        ['Bandwidth', '2·Rs', '2·(Δf+Rs)', '2·Rs'],
        ['Power Eff.', 'Poor', 'Good', 'Best'],
        ['Noise Imm.', 'Poor', 'Good', 'Best'],
        ['Complexity', 'Low', 'Medium', 'Medium'],
        ['Envelope', 'Varies', 'Constant', 'Constant'],
        ['Coherent?', 'Optional', 'Optional', 'Required']
    ]

    table = axes[3, 0].table(cellText=table_data, cellLoc='left',
                             bbox=[0, 0, 2, 1])
    table.auto_set_font_size(False)
    table.set_fontsize(10)
    table.scale(1, 2)

    # Color header row
    for i in range(4):
        table[(0, i)].set_facecolor('#40466e')
        table[(0, i)].set_text_props(weight='bold', color='white')

    plt.tight_layout()
    plt.savefig('modulation_comparison.png', dpi=150, bbox_inches='tight')
    print("\n✓ Saved modulation_comparison.png")
    plt.show()


if __name__ == "__main__":
    compare_modulations()
```

---

## Part 2: PlutoSDR Hardware Implementation

**Note**: Full PlutoSDR implementation with BER testing continues in LAB 3.4 (BER Testing). Here's a basic transmission example:

```python
import adi
import numpy as np

class PlutoModemTest:
    """Test modulation schemes on PlutoSDR"""

    def __init__(self, uri="ip:192.168.2.1"):
        self.sdr = adi.Pluto(uri)
        self.sdr.sample_rate = int(1e6)
        self.sdr.tx_lo = int(915e6)
        self.sdr.rx_lo = int(915e6)
        self.sdr.tx_cyclic_buffer = True
        self.sdr.tx_hardwaregain_chan0 = -30
        self.sdr.rx_hardwaregain_chan0 = 40

        print("PlutoSDR Modulation Test")
        print(f"Center freq: 915 MHz")
        print(f"Sample rate: 1 MSPS")

    def test_bpsk(self, symbol_rate=10e3):
        """Test BPSK transmission"""

        # Generate BPSK signal
        bits = np.random.randint(0, 2, size=1000)
        sps = int(self.sdr.sample_rate / symbol_rate)

        # BPSK: bits → ±1
        symbols = 2*bits - 1
        symbols_up = np.repeat(symbols, sps)

        # Complex baseband (BPSK on I-channel)
        tx_samples = symbols_up.astype(complex)

        # Transmit
        self.sdr.tx(tx_samples * 0.5)  # Scale to prevent clipping

        # Receive
        rx_samples = self.sdr.rx()

        print(f"\nBPSK Test:")
        print(f"  TX bits: {len(bits)}")
        print(f"  TX samples: {len(tx_samples)}")
        print(f"  RX samples: {len(rx_samples)}")

        return tx_samples, rx_samples


# Run test
tester = PlutoModemTest()
tx, rx = tester.test_bpsk()
```

---

## Summary

In this lab, you learned:

✅ **ASK (Amplitude Shift Keying)**:
   - On-Off Keying (OOK): simplest form
   - Poor noise performance
   - Used in simple applications (IR remotes, etc.)

✅ **FSK (Frequency Shift Keying)**:
   - Two frequencies for 0 and 1
   - Constant envelope, good for non-linear amplifiers
   - Carson's rule: BW ≈ 2·(Δf + Rs)

✅ **PSK (Phase Shift Keying)**:
   - BPSK: 0° and 180° phases
   - Best noise performance
   - Requires carrier synchronization

✅ **Performance Comparison**:
   - BPSK: Best power efficiency (~3-6 dB better than FSK/ASK)
   - FSK: Best for non-coherent detection
   - ASK: Simplest, but poorest performance

---

## Next Steps

Continue to:
- **LAB 3.2**: QPSK and 8-PSK Modulation
- **LAB 3.3**: QAM Modulation (16/64/256-QAM)
- **LAB 3.4**: BER Testing and Eye Diagrams
- **LAB 3.5**: Pulse Shaping and Matched Filtering
