# LAB 2.3: Quantization and ADC Resolution

## Overview

This lab explores **quantization** - the process of mapping continuous amplitude values to discrete levels - and how **ADC (Analog-to-Digital Converter) resolution** affects signal quality in SDR systems. You'll learn about quantization noise, dynamic range, and the trade-offs between ADC bit depth and system performance.

## Learning Objectives

After completing this lab, you will understand:
- Quantization: mapping continuous amplitudes to discrete levels
- Quantization noise and its relationship to SNR
- ADC resolution: bits vs. dynamic range vs. SFDR
- Quantization SNR formula: SNR_q = 6.02N + 1.76 dB
- Dynamic range and spurious-free dynamic range (SFDR)
- Dithering techniques to improve effective resolution
- PlutoSDR's AD9361 12-bit ADC performance
- Effects of clipping and overload
- Effective Number of Bits (ENOB)

## Prerequisites

- LAB 0: PlutoSDR Setup
- LAB 1.1: Basic IQ Sampling
- LAB 2.1: Nyquist Sampling and Aliasing
- LAB 2.2: Decimation and Interpolation
- Understanding of decibels and SNR
- Basic Python or C programming

## Theory

### 1. Quantization Fundamentals

**Definition**: Converting continuous amplitude to discrete levels.

```
Continuous signal: x(t) ∈ ℝ (infinite precision)
Sampled signal:    x[n] ∈ ℝ (still infinite precision)
Quantized signal:  x_q[n] ∈ {L₀, L₁, ..., L_{2^N-1}} (finite levels)

For N-bit ADC: 2^N quantization levels
```

**Quantization Process**:
```
Full-scale range: [-Vref, +Vref]
Step size (LSB):  Δ = 2·Vref / 2^N

Example: 12-bit ADC, Vref = 1.0V
  Δ = 2.0 / 4096 = 488 µV
  Levels: 0, 488µV, 976µV, ..., 1.999V

Input:    0.5000 V
Quantized: 0.5000 V (level 2048)
Error:     0 µV

Input:    0.5002 V
Quantized: 0.5003 V (level 2049)
Error:     100 µV (quantization noise!)
```

**Quantization Error**:
```
e_q[n] = x_q[n] - x[n]

Maximum error: |e_q| ≤ Δ/2  (±1/2 LSB)

For uniform quantizer with random input:
  Error is uniformly distributed: e_q ~ U(-Δ/2, +Δ/2)
  Mean: E[e_q] = 0
  Variance: σ²_q = Δ²/12
```

### 2. Quantization Signal-to-Noise Ratio

**Quantization Noise Power**:
```
For N-bit ADC with full-scale range [-1, +1]:
  Δ = 2 / 2^N
  Noise power: σ²_q = Δ²/12 = 1/(3·4^N)
```

**Signal Power** (for full-scale sinusoid):
```
x(t) = A·sin(2πft), where A = 1 (full scale)
Signal power: P_s = A²/2 = 0.5
```

**SNR Formula**:
```
SNR_q = 10·log₁₀(P_s / P_q)
      = 10·log₁₀((A²/2) / (Δ²/12))
      = 10·log₁₀(6·2^(2N))
      = 6.02N + 1.76 dB

This is the THEORETICAL maximum SNR!
```

**Practical Examples**:
```
 8-bit ADC:  SNR = 6.02×8  + 1.76 = 50.0 dB
12-bit ADC:  SNR = 6.02×12 + 1.76 = 74.0 dB
16-bit ADC:  SNR = 6.02×16 + 1.76 = 98.1 dB
24-bit ADC:  SNR = 6.02×24 + 1.76 = 146.2 dB (audio DACs)

Rule of thumb: Each bit adds ~6 dB SNR
```

### 3. Dynamic Range

**Dynamic Range (DR)**: Ratio of largest to smallest measurable signal.

```
DR = 20·log₁₀(V_max / V_min)

For ideal N-bit ADC:
  V_max = Vref (full scale)
  V_min = Δ = Vref / 2^N

  DR = 20·log₁₀(2^N) = 6.02N dB

Example: 12-bit ADC
  DR = 6.02 × 12 = 72.2 dB

This means signals spanning 72 dB (4000:1 voltage ratio)
can be represented without clipping or disappearing in noise.
```

**Spurious-Free Dynamic Range (SFDR)**:
```
SFDR = difference between signal and largest spurious tone

Ideal ADC: SFDR ≈ 6.02N + 1.76 dB (limited by quantization)
Real ADC:  SFDR limited by:
  - Nonlinearity (harmonics)
  - Intermodulation products
  - Clock jitter
  - Thermal noise

AD9361 (PlutoSDR):
  Theoretical: 74 dB (12-bit)
  Actual SFDR: ~65 dB typical
  Reason: Non-idealities, thermal noise, linearity
```

### 4. Effective Number of Bits (ENOB)

**ENOB**: Actual resolution accounting for all noise sources.

```
Measured SNR from real ADC → ENOB

ENOB = (SNR_measured - 1.76) / 6.02

Example: AD9361 RX path
  Specified SNR: ~70 dB (typical, with gain optimization)
  ENOB = (70 - 1.76) / 6.02 = 11.3 bits

Even though ADC is 12-bit, effective resolution ~11.3 bits
due to thermal noise, quantization, linearity errors.
```

**SINAD vs. SNR vs. ENOB**:
```
SINAD = Signal to (Noise + Distortion) ratio
      = Includes quantization noise + harmonics + all spurs

SNR = Signal to Noise ratio
    = Quantization noise only (ideally)

THD = Total Harmonic Distortion
    = Power in harmonics relative to fundamental

ENOB uses SINAD:
  ENOB = (SINAD - 1.76) / 6.02
```

### 5. Overload and Clipping

**Clipping**: When signal exceeds ADC full-scale range.

```
Input signal: x(t) = 1.5·sin(2πft)  (50% overload)
ADC range:    [-1.0, +1.0]

Clipped signal:
       +1.0 ┌─────────┐
           /           \
    ──────              ──────
           \           /
       -1.0 └─────────┘

Effects:
  1. Severe distortion
  2. Harmonics generated
  3. Spectral regrowth
  4. Information loss (irreversible!)
```

**Clipping in Frequency Domain**:
```
Clean sine wave:
  Single spectral line at f₀

Clipped sine wave:
  Fundamental at f₀ (reduced amplitude)
  Odd harmonics: 3f₀, 5f₀, 7f₀, ...
  Power spreads across spectrum

Example: 3 dB overload (√2 amplitude)
  Fundamental: -3 dB (half power in signal)
  3rd harmonic: -15 dBc
  5th harmonic: -25 dBc
  Total harmonic distortion: ~10%
```

**Avoiding Overload**:
```
Set gain so peak signal = 70-80% of full scale

For complex modulation (OFDM, QAM):
  Peak-to-Average Power Ratio (PAPR) = 10-12 dB
  Set RMS level to -12 dB from full scale

PlutoSDR:
  RX gain: 0 to 73 dB (manual or AGC)
  Monitor spectrum to ensure no clipping
  Look for flat-top time domain signal
```

### 6. Dithering

**Dithering**: Adding small random noise to improve effective resolution.

```
Problem with low-level signals:
  Small signals (< 1 LSB) get quantized to same level
  Results in "stair-step" or "granular" distortion

Solution: Add dither noise (~1 LSB RMS)
  Randomizes quantization error
  Trades noise floor for linearity
  Can reveal sub-LSB signals statistically
```

**Dither Types**:
```
1. Rectangular Dither:
   - Random noise uniform over [-Δ/2, +Δ/2]
   - Simple, decorrelates quantization error

2. Triangular Dither:
   - TPDF (Triangular Probability Density Function)
   - Sum of two uniform random variables
   - Completely whitens quantization noise
   - Optimal for audio and some SDR applications

3. Gaussian Dither:
   - Normal distribution, σ ~ Δ/3
   - Used in high-quality ADCs
```

**Example**: 8-bit ADC with dither
```
Without dither:
  SNR = 50 dB
  Low-level signals distorted

With triangular dither (σ = Δ):
  Noise floor rises by ~3 dB
  SNR = 47 dB (slightly worse)
  But: Linearity improved
  Small signals (-60 dB) now measurable
  Better for weak signal detection
```

### 7. PlutoSDR AD9361 ADC Specifications

**AD9361 RX Path**:
```
Antenna → LNA → Mixer → Baseband Filter → ADC → DDC → Output
                 ↓
            LO (70 MHz - 6 GHz)

ADC Specifications:
  Resolution:   12 bits
  Sample rate:  25-640 MSPS (AD9361 RX ADC max)
  Architecture: Pipeline ADC
  DNL:          < ±0.5 LSB typical
  INL:          < ±1.0 LSB typical
  SFDR:         65 dB typical @ 2.4 GHz, -10 dBFS input
  SNR:          70 dB typical (with optimal gain settings)
  ENOB:         ~11.3 bits
```

**Automatic Gain Control (AGC)**:
```
AD9361 has sophisticated AGC:
  Fast Attack AGC: locks in <1 µs
  Slow Attack AGC: optimizes over longer time

Goal: Keep signal in optimal ADC range
  - Too low: poor SNR (lost in thermal noise)
  - Too high: clipping and distortion
  - Optimal: -10 to -6 dBFS (60-70% of full scale)

Manual gain control also available (0-73 dB RX gain)
```

**Thermal Noise Floor**:
```
Even with perfect ADC, thermal noise limits sensitivity:

Noise power: N = kTB
  k = Boltzmann constant = 1.38×10⁻²³ J/K
  T = Temperature = 290 K (room temp)
  B = Bandwidth

Example: 1 MHz bandwidth
  N = 1.38×10⁻²³ × 290 × 1×10⁶
    = -114 dBm (absolute limit!)

AD9361 noise figure: ~7 dB typical
  Actual noise floor: -114 + 7 = -107 dBm / Hz
```

---

## Part 1: Simulation (Pure Python)

### Implementation 1: Quantization Demonstration

```python
import numpy as np
import matplotlib.pyplot as plt

class QuantizationDemo:
    """Demonstrate quantization effects on signals"""

    def __init__(self, n_bits=12, vref=1.0):
        """
        Args:
            n_bits: ADC resolution (bits)
            vref: Full-scale voltage (±vref)
        """
        self.n_bits = n_bits
        self.vref = vref
        self.n_levels = 2**n_bits
        self.delta = 2 * vref / self.n_levels  # Quantization step (LSB)

        print(f"Quantizer Configuration:")
        print(f"  Resolution:     {n_bits} bits")
        print(f"  Levels:         {self.n_levels}")
        print(f"  Full scale:     ±{vref} V")
        print(f"  LSB size:       {self.delta*1e6:.2f} µV")
        print(f"  Theoretical SNR: {6.02*n_bits + 1.76:.1f} dB")

    def quantize(self, signal):
        """
        Quantize signal to N bits

        Args:
            signal: Input signal (float)

        Returns:
            Quantized signal and quantization error
        """
        # Clip to full-scale range
        signal_clipped = np.clip(signal, -self.vref, self.vref)

        # Quantize: round to nearest level
        levels = np.round(signal_clipped / self.delta)

        # Convert back to voltage
        signal_quantized = levels * self.delta

        # Ensure within range (handles edge case at exactly +vref)
        signal_quantized = np.clip(signal_quantized, -self.vref,
                                   self.vref - self.delta)

        # Quantization error
        error = signal_quantized - signal_clipped

        return signal_quantized, error

    def measure_snr(self, signal, signal_quantized):
        """
        Measure SNR of quantized signal

        Args:
            signal: Original signal
            signal_quantized: Quantized signal

        Returns:
            SNR in dB
        """
        # Signal power (original)
        signal_power = np.mean(signal**2)

        # Noise power (quantization error)
        noise = signal_quantized - signal
        noise_power = np.mean(noise**2)

        # SNR
        snr_linear = signal_power / noise_power
        snr_db = 10 * np.log10(snr_linear)

        return snr_db

    def test_sinusoid(self, freq=1e3, fs=100e3, duration=0.01, amplitude=1.0):
        """
        Test quantization on sinusoid

        Args:
            freq: Signal frequency (Hz)
            fs: Sample rate (Hz)
            duration: Duration (seconds)
            amplitude: Signal amplitude (fraction of full-scale)

        Returns:
            Time, original signal, quantized signal, error
        """
        # Generate sinusoid
        t = np.arange(0, duration, 1/fs)
        signal = amplitude * self.vref * np.sin(2 * np.pi * freq * t)

        # Quantize
        signal_q, error = self.quantize(signal)

        # Measure SNR
        snr = self.measure_snr(signal, signal_q)

        print(f"\nSinusoid Test:")
        print(f"  Frequency:      {freq/1e3:.1f} kHz")
        print(f"  Amplitude:      {amplitude*100:.0f}% of full-scale")
        print(f"  Measured SNR:   {snr:.1f} dB")
        print(f"  Theoretical:    {6.02*self.n_bits + 1.76:.1f} dB")
        print(f"  Error:          {snr - (6.02*self.n_bits + 1.76):.1f} dB")

        return t, signal, signal_q, error

    def plot_quantization(self, t, signal, signal_q, error):
        """Plot quantization results"""

        fig, axes = plt.subplots(3, 1, figsize=(12, 10))

        # Original and quantized signals
        axes[0].plot(t*1e3, signal, 'b-', linewidth=1.5, label='Original', alpha=0.7)
        axes[0].plot(t*1e3, signal_q, 'r-', linewidth=1, label='Quantized', alpha=0.8)
        axes[0].set_xlabel('Time (ms)')
        axes[0].set_ylabel('Amplitude (V)')
        axes[0].set_title(f'{self.n_bits}-bit Quantization')
        axes[0].legend()
        axes[0].grid(True, alpha=0.3)

        # Zoom in to see quantization steps
        t_zoom = 0.002  # 2 ms
        idx = int(t_zoom * len(t) / (t[-1] - t[0]))
        axes[1].plot(t[:idx]*1e3, signal[:idx], 'b-', linewidth=2, label='Original')
        axes[1].plot(t[:idx]*1e3, signal_q[:idx], 'r-', linewidth=1.5,
                    label='Quantized', drawstyle='steps-post')
        axes[1].set_xlabel('Time (ms)')
        axes[1].set_ylabel('Amplitude (V)')
        axes[1].set_title('Quantization Steps (Zoomed)')
        axes[1].legend()
        axes[1].grid(True, alpha=0.3)

        # Quantization error
        axes[2].plot(t*1e3, error*1e6, 'g-', linewidth=0.5)
        axes[2].set_xlabel('Time (ms)')
        axes[2].set_ylabel('Error (µV)')
        axes[2].set_title(f'Quantization Error (max ±{self.delta/2*1e6:.1f} µV)')
        axes[2].grid(True, alpha=0.3)
        axes[2].axhline(self.delta/2*1e6, color='r', linestyle='--',
                       label=f'±Δ/2 = ±{self.delta/2*1e6:.1f} µV')
        axes[2].axhline(-self.delta/2*1e6, color='r', linestyle='--')
        axes[2].legend()

        plt.tight_layout()
        plt.savefig(f'quantization_{self.n_bits}bit.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved quantization_{self.n_bits}bit.png")
        plt.show()


# Test different bit depths
if __name__ == "__main__":
    print("="*70)
    print("QUANTIZATION DEMONSTRATION")
    print("="*70)

    for n_bits in [8, 12, 16]:
        print(f"\n{'='*70}")
        print(f"{n_bits}-BIT ADC")
        print(f"{'='*70}")

        demo = QuantizationDemo(n_bits=n_bits, vref=1.0)
        t, signal, signal_q, error = demo.test_sinusoid(
            freq=1e3, fs=100e3, duration=0.01, amplitude=1.0
        )
        demo.plot_quantization(t, signal, signal_q, error)

        # Analyze frequency domain
        fft_original = np.fft.fft(signal)
        fft_quantized = np.fft.fft(signal_q)
        freqs = np.fft.fftfreq(len(signal), t[1]-t[0])

        # Plot spectrum
        fig, ax = plt.subplots(1, 1, figsize=(12, 6))
        ax.plot(freqs[:len(freqs)//2]/1e3,
               20*np.log10(np.abs(fft_quantized[:len(freqs)//2])/len(signal)),
               'b-', linewidth=1, label=f'{n_bits}-bit quantized')
        ax.set_xlabel('Frequency (kHz)')
        ax.set_ylabel('Magnitude (dB)')
        ax.set_title(f'Spectrum of {n_bits}-bit Quantized Sinusoid')
        ax.grid(True, alpha=0.3)
        ax.legend()
        ax.set_ylim([-120, 0])
        plt.tight_layout()
        plt.savefig(f'spectrum_{n_bits}bit.png', dpi=150, bbox_inches='tight')
        print(f"✓ Saved spectrum_{n_bits}bit.png")
        plt.show()
```

### Implementation 2: Clipping Demonstration

```python
class ClippingDemo:
    """Demonstrate effects of ADC overload/clipping"""

    def __init__(self, n_bits=12, vref=1.0):
        """Initialize ADC model"""
        self.quantizer = QuantizationDemo(n_bits, vref)
        self.vref = vref

    def test_clipping(self, overload_db=0):
        """
        Test clipping at various overload levels

        Args:
            overload_db: Overload amount in dB (0 = full scale, +3 = 2× amplitude)

        Returns:
            Time, signal, clipped signal
        """
        # Generate signal with specified amplitude
        freq = 1e3
        fs = 100e3
        duration = 0.01

        amplitude_linear = 10**(overload_db/20)

        t = np.arange(0, duration, 1/fs)
        signal = amplitude_linear * self.vref * np.sin(2 * np.pi * freq * t)

        # Quantize (includes clipping)
        signal_clipped, _ = self.quantizer.quantize(signal)

        print(f"\nClipping Test:")
        print(f"  Overload:       {overload_db:+.1f} dB")
        print(f"  Peak input:     {np.max(np.abs(signal)):.3f} V")
        print(f"  ADC full scale: {self.vref:.3f} V")

        if np.max(np.abs(signal)) > self.vref:
            clip_pct = np.sum(np.abs(signal) > self.vref) / len(signal) * 100
            print(f"  Clipping:       {clip_pct:.1f}% of samples")
        else:
            print(f"  Clipping:       None")

        # Measure distortion
        fft_signal = np.fft.fft(signal_clipped)
        freqs = np.fft.fftfreq(len(signal), 1/fs)

        # Find fundamental and harmonics
        fundamental_idx = np.argmax(np.abs(fft_signal[:len(fft_signal)//2]))
        fundamental_power = np.abs(fft_signal[fundamental_idx])**2

        # Find 3rd harmonic
        harmonic3_idx = fundamental_idx * 3
        if harmonic3_idx < len(fft_signal)//2:
            harmonic3_power = np.abs(fft_signal[harmonic3_idx])**2
            thd3 = 10 * np.log10(harmonic3_power / fundamental_power)
            print(f"  3rd harmonic:   {thd3:.1f} dBc")

        return t, signal, signal_clipped

    def plot_clipping(self, results_list):
        """
        Plot clipping results for multiple overload levels

        Args:
            results_list: List of (overload_db, t, signal, clipped) tuples
        """
        n_tests = len(results_list)
        fig, axes = plt.subplots(n_tests, 2, figsize=(14, 4*n_tests))

        if n_tests == 1:
            axes = axes.reshape(1, -1)

        for i, (overload_db, t, signal, clipped) in enumerate(results_list):
            # Time domain
            axes[i, 0].plot(t*1e3, signal, 'b-', linewidth=1, alpha=0.6, label='Input')
            axes[i, 0].plot(t*1e3, clipped, 'r-', linewidth=1.5, label='ADC Output')
            axes[i, 0].axhline(self.vref, color='k', linestyle='--', alpha=0.3)
            axes[i, 0].axhline(-self.vref, color='k', linestyle='--', alpha=0.3)
            axes[i, 0].set_xlabel('Time (ms)')
            axes[i, 0].set_ylabel('Amplitude (V)')
            axes[i, 0].set_title(f'Overload: {overload_db:+.0f} dB (Time Domain)')
            axes[i, 0].legend()
            axes[i, 0].grid(True, alpha=0.3)
            axes[i, 0].set_xlim([0, 5])

            # Frequency domain
            fft_result = np.fft.fft(clipped)
            freqs = np.fft.fftfreq(len(clipped), t[1]-t[0])
            spectrum = 20 * np.log10(np.abs(fft_result)/len(clipped) + 1e-12)

            axes[i, 1].plot(freqs[:len(freqs)//2]/1e3, spectrum[:len(freqs)//2],
                           'b-', linewidth=1)
            axes[i, 1].set_xlabel('Frequency (kHz)')
            axes[i, 1].set_ylabel('Magnitude (dB)')
            axes[i, 1].set_title(f'Overload: {overload_db:+.0f} dB (Spectrum)')
            axes[i, 1].grid(True, alpha=0.3)
            axes[i, 1].set_ylim([-100, 0])
            axes[i, 1].set_xlim([0, 10])

            # Annotate harmonics
            if overload_db > 0:
                axes[i, 1].annotate('Harmonics\nfrom clipping',
                                   xy=(3, -20), fontsize=10, color='red')

        plt.tight_layout()
        plt.savefig('clipping_effects.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved clipping_effects.png")
        plt.show()


# Test clipping
if __name__ == "__main__":
    print("\n" + "="*70)
    print("CLIPPING DEMONSTRATION")
    print("="*70)

    demo = ClippingDemo(n_bits=12, vref=1.0)

    results = []
    for overload_db in [0, 3, 6, 9]:
        t, signal, clipped = demo.test_clipping(overload_db)
        results.append((overload_db, t, signal, clipped))

    demo.plot_clipping(results)

    print("\n" + "="*70)
    print("KEY OBSERVATIONS:")
    print("="*70)
    print("1. 0 dB overload: Clean sinusoid, minimal harmonics")
    print("2. 3 dB overload: Slight clipping, 3rd harmonic appears")
    print("3. 6 dB overload: Severe clipping, strong harmonics")
    print("4. 9 dB overload: Heavy clipping, almost square wave")
    print()
    print("LESSON: Keep signal level 3-6 dB below full scale!")
    print("="*70)
```

### Implementation 3: Dithering Demonstration

```python
class DitheringDemo:
    """Demonstrate dithering to improve low-level linearity"""

    def __init__(self, n_bits=8, vref=1.0):
        """Initialize with low resolution ADC"""
        self.quantizer = QuantizationDemo(n_bits, vref)
        self.n_bits = n_bits
        self.vref = vref
        self.delta = self.quantizer.delta

        print(f"Dithering Demo: {n_bits}-bit ADC")
        print(f"  LSB size: {self.delta*1e6:.1f} µV")

    def add_dither(self, signal, dither_type='triangular'):
        """
        Add dither to signal

        Args:
            signal: Input signal
            dither_type: 'none', 'rectangular', or 'triangular'

        Returns:
            Dithered signal
        """
        if dither_type == 'none':
            return signal

        elif dither_type == 'rectangular':
            # Uniform dither over [-Δ/2, +Δ/2]
            dither = np.random.uniform(-self.delta/2, self.delta/2, size=len(signal))

        elif dither_type == 'triangular':
            # TPDF: sum of two uniform random variables
            dither1 = np.random.uniform(-self.delta/2, self.delta/2, size=len(signal))
            dither2 = np.random.uniform(-self.delta/2, self.delta/2, size=len(signal))
            dither = dither1 + dither2

        else:
            raise ValueError(f"Unknown dither type: {dither_type}")

        return signal + dither

    def test_dithering(self, signal_level_db=-40):
        """
        Test dithering on low-level signal

        Args:
            signal_level_db: Signal level relative to full-scale (dB)

        Returns:
            Results dict
        """
        # Generate low-level sinusoid
        freq = 1e3
        fs = 100e3
        duration = 0.1

        amplitude = 10**(signal_level_db/20) * self.vref

        t = np.arange(0, duration, 1/fs)
        signal = amplitude * np.sin(2 * np.pi * freq * t)

        print(f"\nLow-level signal test:")
        print(f"  Level:      {signal_level_db} dBFS")
        print(f"  Amplitude:  {amplitude*1e6:.2f} µV")
        print(f"  LSB:        {self.delta*1e6:.2f} µV")
        print(f"  Signal/LSB: {amplitude/self.delta:.2f}")

        results = {}

        for dither_type in ['none', 'rectangular', 'triangular']:
            # Add dither
            signal_dithered = self.add_dither(signal, dither_type)

            # Quantize
            signal_q, _ = self.quantizer.quantize(signal_dithered)

            # Analyze spectrum
            fft_q = np.fft.fft(signal_q)
            freqs = np.fft.fftfreq(len(signal_q), 1/fs)
            spectrum = 20 * np.log10(np.abs(fft_q)/len(signal_q) + 1e-12)

            # Find fundamental
            fund_idx = np.argmax(np.abs(fft_q[:len(fft_q)//2]))
            fund_power = spectrum[fund_idx]

            # Measure noise floor (excluding DC and fundamental)
            noise_bins = np.concatenate([spectrum[10:fund_idx-5],
                                         spectrum[fund_idx+5:len(spectrum)//2]])
            noise_floor = np.median(noise_bins)

            snr = fund_power - noise_floor

            print(f"\n{dither_type.capitalize()} dither:")
            print(f"  Signal:      {fund_power:.1f} dB")
            print(f"  Noise floor: {noise_floor:.1f} dB")
            print(f"  SNR:         {snr:.1f} dB")

            results[dither_type] = {
                'freqs': freqs,
                'spectrum': spectrum,
                'signal_q': signal_q,
                'snr': snr
            }

        return t, signal, results

    def plot_dithering(self, t, signal, results):
        """Plot dithering results"""

        fig, axes = plt.subplots(3, 2, figsize=(14, 12))

        dither_types = ['none', 'rectangular', 'triangular']

        for i, dither_type in enumerate(dither_types):
            res = results[dither_type]

            # Time domain (zoomed)
            n_zoom = 500
            axes[i, 0].plot(t[:n_zoom]*1e3, signal[:n_zoom]*1e6, 'b-',
                           linewidth=1, alpha=0.6, label='Original')
            axes[i, 0].plot(t[:n_zoom]*1e3, res['signal_q'][:n_zoom]*1e6, 'r-',
                           linewidth=1, label='Quantized')
            axes[i, 0].set_xlabel('Time (ms)')
            axes[i, 0].set_ylabel('Amplitude (µV)')
            axes[i, 0].set_title(f'{dither_type.capitalize()} - Time Domain')
            axes[i, 0].legend()
            axes[i, 0].grid(True, alpha=0.3)

            # Frequency domain
            axes[i, 1].plot(res['freqs'][:len(res['freqs'])//2]/1e3,
                           res['spectrum'][:len(res['spectrum'])//2],
                           'b-', linewidth=1)
            axes[i, 1].set_xlabel('Frequency (kHz)')
            axes[i, 1].set_ylabel('Magnitude (dB)')
            axes[i, 1].set_title(f'{dither_type.capitalize()} - Spectrum (SNR: {res["snr"]:.1f} dB)')
            axes[i, 1].grid(True, alpha=0.3)
            axes[i, 1].set_ylim([-120, -20])
            axes[i, 1].set_xlim([0, 10])

        plt.tight_layout()
        plt.savefig('dithering_comparison.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved dithering_comparison.png")
        plt.show()


# Test dithering
if __name__ == "__main__":
    print("\n" + "="*70)
    print("DITHERING DEMONSTRATION")
    print("="*70)

    demo = DitheringDemo(n_bits=8, vref=1.0)
    t, signal, results = demo.test_dithering(signal_level_db=-40)
    demo.plot_dithering(t, signal, results)

    print("\n" + "="*70)
    print("DITHERING BENEFITS:")
    print("="*70)
    print("1. Without dither: Low-level signals show granular distortion")
    print("2. With dither: Noise floor rises slightly, but linearity improves")
    print("3. Triangular (TPDF) dither: Best - completely whitens quantization noise")
    print("4. Trade-off: +3 dB noise floor for better low-level linearity")
    print("5. Used in: High-quality audio DACs, some SDR receivers")
    print("="*70)
```

---

## Part 2: PlutoSDR Hardware Testing

### Method 2: PlutoSDR ADC Analysis

```python
import adi
import numpy as np
import matplotlib.pyplot as plt

class PlutoADCTest:
    """Test PlutoSDR AD9361 ADC performance"""

    def __init__(self, uri="ip:192.168.2.1"):
        """Initialize PlutoSDR"""
        self.sdr = adi.Pluto(uri)
        self.sdr.sample_rate = int(2e6)
        self.sdr.tx_lo = int(915e6)
        self.sdr.rx_lo = int(915e6)
        self.sdr.tx_cyclic_buffer = True

        print("PlutoSDR ADC Performance Test")
        print(f"Sample rate: {self.sdr.sample_rate/1e6:.1f} MSPS")
        print(f"AD9361: 12-bit ADC, theoretical SNR = 74 dB")

    def test_snr_vs_gain(self, tone_freq=100e3):
        """
        Measure SNR vs RX gain

        Finds optimal gain setting for maximum SNR

        Args:
            tone_freq: Test tone frequency (baseband)

        Returns:
            gains, snrs
        """
        print(f"\nSNR vs Gain Test:")
        print(f"  Tone: {tone_freq/1e3:.0f} kHz")

        # Generate test tone
        fs = self.sdr.sample_rate
        n_samples = 2**16
        t = np.arange(n_samples) / fs
        tx_signal = 0.5 * np.exp(2j * np.pi * tone_freq * t)

        # Set TX gain
        self.sdr.tx_hardwaregain_chan0 = -20

        # Transmit
        self.sdr.tx(tx_signal)

        # Test multiple RX gain settings
        gains = np.arange(0, 74, 5)
        snrs = []
        signal_levels = []

        for gain in gains:
            self.sdr.rx_hardwaregain_chan0 = int(gain)

            # Receive
            rx_samples = self.sdr.rx()

            # Compute spectrum
            fft_rx = np.fft.fftshift(np.fft.fft(rx_samples))
            freqs = np.fft.fftshift(np.fft.fftfreq(len(rx_samples), 1/fs))
            spectrum = 20 * np.log10(np.abs(fft_rx) / len(rx_samples) + 1e-12)

            # Find signal peak
            center_idx = len(spectrum) // 2
            search_start = center_idx + int(tone_freq / fs * len(spectrum)) - 10
            search_end = search_start + 20
            peak_idx = search_start + np.argmax(spectrum[search_start:search_end])

            signal_power = spectrum[peak_idx]

            # Measure noise floor (exclude signal bin)
            noise_bins = np.concatenate([spectrum[center_idx+50:peak_idx-20],
                                         spectrum[peak_idx+20:center_idx+len(spectrum)//4]])
            noise_floor = np.median(noise_bins)

            snr = signal_power - noise_floor

            print(f"  Gain: {gain:2d} dB → Signal: {signal_power:6.1f} dB, Noise: {noise_floor:6.1f} dB, SNR: {snr:5.1f} dB")

            snrs.append(snr)
            signal_levels.append(signal_power)

        # Find optimal gain
        optimal_idx = np.argmax(snrs)
        optimal_gain = gains[optimal_idx]
        optimal_snr = snrs[optimal_idx]

        print(f"\nOptimal RX Gain: {optimal_gain} dB")
        print(f"Maximum SNR:     {optimal_snr:.1f} dB")
        print(f"AD9361 ENOB:     {(optimal_snr - 1.76) / 6.02:.1f} bits")

        # Plot results
        fig, axes = plt.subplots(2, 1, figsize=(12, 8))

        axes[0].plot(gains, snrs, 'bo-', linewidth=2, markersize=6)
        axes[0].axvline(optimal_gain, color='r', linestyle='--',
                       label=f'Optimal: {optimal_gain} dB')
        axes[0].set_xlabel('RX Gain (dB)')
        axes[0].set_ylabel('SNR (dB)')
        axes[0].set_title('SNR vs RX Gain (AD9361 12-bit ADC)')
        axes[0].grid(True, alpha=0.3)
        axes[0].legend()

        axes[1].plot(gains, signal_levels, 'go-', linewidth=2, markersize=6, label='Signal')
        axes[1].set_xlabel('RX Gain (dB)')
        axes[1].set_ylabel('Signal Level (dB)')
        axes[1].set_title('Signal Level vs RX Gain')
        axes[1].grid(True, alpha=0.3)
        axes[1].legend()
        axes[1].axhline(-6, color='r', linestyle='--', alpha=0.5, label='Target: -6 dBFS')

        plt.tight_layout()
        plt.savefig('pluto_snr_vs_gain.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved pluto_snr_vs_gain.png")
        plt.show()

        return gains, snrs

    def test_sfdr(self, tone_freq=100e3, rx_gain=40):
        """
        Measure Spurious-Free Dynamic Range (SFDR)

        Args:
            tone_freq: Test tone frequency
            rx_gain: RX gain setting

        Returns:
            SFDR in dB
        """
        print(f"\nSFDR Test:")
        print(f"  RX Gain: {rx_gain} dB")

        # Configure
        self.sdr.rx_hardwaregain_chan0 = rx_gain
        self.sdr.tx_hardwaregain_chan0 = -20

        # Generate tone
        fs = self.sdr.sample_rate
        n_samples = 2**16
        t = np.arange(n_samples) / fs
        tx_signal = 0.5 * np.exp(2j * np.pi * tone_freq * t)

        self.sdr.tx(tx_signal)

        # Receive
        rx_samples = self.sdr.rx()

        # Spectrum
        window = np.blackman(len(rx_samples))
        fft_rx = np.fft.fftshift(np.fft.fft(rx_samples * window))
        freqs = np.fft.fftshift(np.fft.fftfreq(len(rx_samples), 1/fs))
        spectrum = 20 * np.log10(np.abs(fft_rx) / len(rx_samples) + 1e-12)

        # Find fundamental
        center_idx = len(spectrum) // 2
        search_start = center_idx + int(tone_freq / fs * len(spectrum)) - 10
        search_end = search_start + 20
        peak_idx = search_start + np.argmax(spectrum[search_start:search_end])
        fundamental_power = spectrum[peak_idx]

        # Find largest spur (excluding DC and fundamental)
        spectrum_copy = spectrum.copy()
        spectrum_copy[center_idx-10:center_idx+10] = -200  # Mask DC
        spectrum_copy[peak_idx-10:peak_idx+10] = -200      # Mask fundamental

        # Only search positive frequencies
        spur_idx = center_idx + np.argmax(spectrum_copy[center_idx:])
        spur_power = spectrum[spur_idx]

        sfdr = fundamental_power - spur_power

        print(f"  Fundamental: {fundamental_power:.1f} dB @ {freqs[peak_idx]/1e3:.1f} kHz")
        print(f"  Largest spur: {spur_power:.1f} dB @ {freqs[spur_idx]/1e3:.1f} kHz")
        print(f"  SFDR:        {sfdr:.1f} dB")

        # Plot
        fig, ax = plt.subplots(1, 1, figsize=(12, 6))
        ax.plot(freqs/1e6, spectrum, 'b-', linewidth=1)
        ax.axvline(freqs[peak_idx]/1e6, color='g', linestyle='--', label='Fundamental')
        ax.axvline(freqs[spur_idx]/1e6, color='r', linestyle='--', label='Largest Spur')
        ax.set_xlabel('Frequency (MHz)')
        ax.set_ylabel('Magnitude (dB)')
        ax.set_title(f'SFDR Test: {sfdr:.1f} dB')
        ax.grid(True, alpha=0.3)
        ax.legend()
        ax.set_ylim([-120, 0])

        plt.tight_layout()
        plt.savefig('pluto_sfdr.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved pluto_sfdr.png")
        plt.show()

        return sfdr


# Run tests
if __name__ == "__main__":
    tester = PlutoADCTest()

    # Test 1: SNR vs Gain
    gains, snrs = tester.test_snr_vs_gain()

    # Test 2: SFDR
    sfdr = tester.test_sfdr(rx_gain=40)

    print("\n" + "="*70)
    print("AD9361 ADC PERFORMANCE SUMMARY")
    print("="*70)
    print(f"Theoretical SNR (12-bit):  74.0 dB")
    print(f"Measured SNR (optimal):    {np.max(snrs):.1f} dB")
    print(f"ENOB:                      {(np.max(snrs) - 1.76) / 6.02:.1f} bits")
    print(f"SFDR:                      {sfdr:.1f} dB")
    print()
    print("Degradation from ideal due to:")
    print("  - Thermal noise (~7 dB NF)")
    print("  - ADC nonlinearity (INL/DNL)")
    print("  - Clock jitter")
    print("  - Quantization noise")
    print("="*70)
```

---

## Method 3: Hosted Application (Compiled C on PlutoSDR ARM)

### Overview

In this method, you'll develop a **production-grade C application** that runs directly on PlutoSDR's ARM processor to analyze quantization effects, measure ADC performance, and demonstrate ENOB, SFDR, and dithering techniques.

**What you'll build**:
- Quantization noise analyzer (measure SNR vs bit depth)
- ENOB calculator (effective bits from measured SNR)
- SFDR tester (spurious-free dynamic range measurement)
- Clipping detector (identify and quantify signal saturation)
- Dithering demonstrator (triangular dither for linearity improvement)
- Dynamic range analyzer (measure usable signal range)

**Prerequisites**:
- Completed LAB 2.1 Method 3 (Nyquist/Aliasing)
- Completed LAB 2.2 Method 3 (Decimation/Interpolation)
- ARM cross-compiler installed
- libiio library (ARM version) available

---

## Part 5: Quantization Theory (Deep Dive)

Before implementing the quantization analyzer, let's establish comprehensive theoretical foundations.

### 1. Quantization Process

#### **1.1: Analog-to-Digital Conversion**

**Simple Analogy**: Measuring temperature with a thermometer that only shows whole degrees. 98.6°F → 99°F (quantization error = 0.4°F).

**Mathematical Definition**:
```
Ideal ADC Model:
  Input:  x(t) ∈ [-Vref, +Vref] (continuous voltage)
  Output: x_q[n] ∈ {0, 1, 2, ..., 2^N-1} (discrete digital code)

Quantization function:
  Q(x) = Δ · ⌊x/Δ + 0.5⌋  (round to nearest level)

Where:
  Δ = Full-scale range / Number of levels = 2Vref / 2^N
  ⌊·⌋ = floor function
```

**Numerical Example** (12-bit ADC, Vref = 1.25V):
```
Full-scale range: ±1.25V → 2.5V total
Number of levels: 2^12 = 4096
Step size: Δ = 2.5 / 4096 = 610.35 µV

Input voltage: 0.500000 V
Digital code: ⌊0.500000 / 0.00061035 + 0.5⌋ = 819
Quantized output: 819 × 0.00061035 = 0.499877 V
Quantization error: 0.500000 - 0.499877 = 123 µV

Input voltage: 0.500500 V
Digital code: ⌊0.500500 / 0.00061035 + 0.5⌋ = 820
Quantized output: 820 × 0.00061035 = 0.500488 V
Quantization error: 0.500500 - 0.500488 = 12 µV
```

#### **1.2: Quantization Error Statistics**

**Error Distribution** (uniform quantizer, random input):
```
Quantization error: e_q[n] = x_q[n] - x[n]

Assuming x[n] uniformly distributed within each quantization interval:
  e_q ~ U(-Δ/2, +Δ/2)  (uniform distribution)

Mean (DC bias):
  E[e_q] = 0  (unbiased for rounding quantizer)

Variance (noise power):
  σ²_q = E[e_q²] = ∫_{-Δ/2}^{+Δ/2} e² · (1/Δ) de
       = [e³/(3Δ)]_{-Δ/2}^{+Δ/2}
       = Δ²/12

RMS error:
  σ_q = Δ / √12 ≈ 0.289 · Δ
```

**Example** (12-bit ADC):
```
Δ = 610.35 µV
RMS quantization noise: 610.35 / √12 = 176.2 µV
```

### 2. Signal-to-Quantization-Noise Ratio (SQNR)

#### **2.1: Theoretical SQNR Derivation**

**Signal Power** (full-scale sinusoid):
```
x(t) = A · sin(2πf₀t), where A = Vref (full scale)

Instantaneous power: p(t) = x²(t) = A² sin²(2πf₀t)

Average power: P_s = (1/T) ∫₀^T A² sin²(2πf₀t) dt
             = A² · (1/T) · T/2
             = A²/2

For A = 1.0 V:
  P_s = 0.5 W (normalized)
```

**Quantization Noise Power**:
```
From variance calculation:
  P_q = σ²_q = Δ²/12

For N-bit ADC with full-scale ±A:
  Δ = 2A / 2^N

  P_q = (2A/2^N)² / 12
      = 4A² / (12 · 2^(2N))
      = A² / (3 · 2^(2N))
```

**SQNR Formula**:
```
SQNR = P_s / P_q
     = (A²/2) / (A² / (3 · 2^(2N)))
     = (3 · 2^(2N)) / 2
     = (3/2) · 4^N

In decibels:
  SQNR_dB = 10 log₁₀((3/2) · 4^N)
          = 10 log₁₀(3/2) + 10 log₁₀(4^N)
          = 1.76 + 10N · log₁₀(4)
          = 1.76 + 10N · 0.602
          = 6.02N + 1.76 dB

This is the FAMOUS quantization SNR formula!
```

**Verification**:
```
8-bit ADC:   SQNR = 6.02 × 8  + 1.76 = 49.92 dB ≈ 50 dB
10-bit ADC:  SQNR = 6.02 × 10 + 1.76 = 61.96 dB ≈ 62 dB
12-bit ADC:  SQNR = 6.02 × 12 + 1.76 = 73.96 dB ≈ 74 dB
16-bit ADC:  SQNR = 6.02 × 16 + 1.76 = 98.08 dB ≈ 98 dB
24-bit ADC:  SQNR = 6.02 × 24 + 1.76 = 146.24 dB ≈ 146 dB

Rule of thumb: Each bit adds 6 dB of SNR
```

#### **2.2: SQNR vs Input Signal Level**

**Problem**: Formula assumes FULL-SCALE sinusoid. What if input is smaller?

**General SQNR**:
```
For sinusoid with amplitude A_input < A_fullscale:

Signal power: P_s = A_input² / 2

Quantization noise power unchanged: P_q = Δ²/12

SQNR = (A_input² / 2) / (Δ² / 12)
     = 6 A_input² / Δ²

In dB:
  SQNR_dB = 6.02N + 1.76 + 20 log₁₀(A_input / A_fullscale)

Backoff penalty: 20 log₁₀(A_input / A_fullscale)
```

**Example**:
```
12-bit ADC, signal at -6 dBFS (half amplitude):

A_input / A_fullscale = 10^(-6/20) = 0.501

SQNR = 74 + 20 log₁₀(0.501)
     = 74 - 6
     = 68 dB

Lost 6 dB SNR by not using full dynamic range!
```

**Optimal operating point**: Keep signal at -3 to -6 dBFS (87-71% of full scale) for headroom against clipping.

### 3. Effective Number of Bits (ENOB)

#### **3.1: ENOB Definition**

**Problem**: Real ADCs have noise sources beyond quantization.

**ENOB**: Number of ideal bits that would produce the measured SNR.

**Formula**:
```
Measured SNR from real ADC → ENOB

ENOB = (SNR_measured - 1.76) / 6.02

Example: AD9361 measured SNR = 70 dB
  ENOB = (70 - 1.76) / 6.02
       = 68.24 / 6.02
       = 11.33 bits

Interpretation: 12-bit ADC performs like ideal 11.3-bit ADC
Lost: 12 - 11.3 = 0.7 bits due to non-idealities
```

#### **3.2: Noise Sources in Real ADCs**

**Total Noise Budget**:
```
SNR_total = -10 log₁₀(10^(-SNR_quant/10) + 10^(-SNR_thermal/10)
                      + 10^(-SNR_jitter/10) + 10^(-SNR_distortion/10))

Where:
  SNR_quant:      Quantization noise (6.02N + 1.76 dB)
  SNR_thermal:    Thermal noise from front-end
  SNR_jitter:     Clock jitter (phase noise)
  SNR_distortion: Nonlinearity (INL, DNL, harmonics)
```

**AD9361 Example** (12-bit ADC):
```
Component contributions to noise:

1. Quantization noise:    74.0 dB (theoretical)
2. Thermal noise (7 dB NF): ~50 dB (dominates!)
3. Clock jitter (100 fs):   ~80 dB
4. ADC nonlinearity:        ~75 dB (INL < 1 LSB)

Total SNR (power sum):
  1/SNR_total = 1/10^7.4 + 1/10^5.0 + 1/10^8.0 + 1/10^7.5
              ≈ 1/10^5.0  (thermal noise dominates)

  SNR_total ≈ 70 dB

ENOB = (70 - 1.76) / 6.02 = 11.3 bits
```

**Key Insight**: Thermal noise from RF front-end limits practical ENOB, not quantization noise!

### 4. Spurious-Free Dynamic Range (SFDR)

#### **4.1: SFDR Definition**

**Definition**: Ratio of fundamental signal power to largest spurious (unwanted) component.

**Mathematical Expression**:
```
SFDR_dB = 20 log₁₀(A_fundamental / A_spur)

Where A_spur is the largest spurious tone from:
  - Harmonics (2f₀, 3f₀, 4f₀, ...)
  - Intermodulation products (f₁ ± f₂)
  - Clock feedthrough
  - Power supply noise
```

**Example**:
```
Fundamental at 500 kHz: -10 dBFS
2nd harmonic at 1 MHz:  -75 dBFS (largest spur)

SFDR = -10 - (-75) = 65 dB
```

#### **4.2: SFDR vs SNR vs ENOB**

**Comparison**:
```
SNR:  Measures total noise floor (integrated across all frequencies)
SFDR: Measures worst-case spurious tone (single frequency)
ENOB: Effective resolution accounting for all noise

For ideal ADC:
  SFDR ≈ SNR ≈ 6.02N + 1.76 dB (quantization-limited)

For real ADC:
  SFDR < SNR (spurs stick out above noise floor)

AD9361 typical:
  SNR:  70 dB
  SFDR: 65 dB (2nd/3rd harmonics from nonlinearity)
  ENOB: 11.3 bits
```

#### **4.3: Sources of Spurious Tones**

**1. Harmonic Distortion** (ADC nonlinearity):
```
2nd harmonic: HD2 = 20 log₁₀(A_2f / A_f)
3rd harmonic: HD3 = 20 log₁₀(A_3f / A_f)

For N-bit ADC with good linearity:
  HD2, HD3 > -80 dBc typical

AD9361:
  HD2: -75 dBc typical
  HD3: -80 dBc typical
```

**2. Intermodulation Distortion** (two-tone test):
```
Input: f₁ = 500 kHz, f₂ = 600 kHz

2nd-order products: f₁ ± f₂ = 1.1 MHz, 100 kHz
3rd-order products: 2f₁ - f₂ = 400 kHz (in-band!)
                    2f₂ - f₁ = 700 kHz (in-band!)

IMD3 (3rd-order intercept point):
  For ideal 12-bit ADC: IIP3 ~ +20 dBm
  For AD9361:           IIP3 ~ +10 dBm
```

### 5. Dynamic Range

#### **5.1: Dynamic Range Definition**

**Dynamic Range**: Ratio of largest to smallest measurable signal.

**Mathematical Expression**:
```
DR_dB = 20 log₁₀(V_max / V_min)

Where:
  V_max: Full-scale voltage (clipping threshold)
  V_min: Noise floor (smallest detectable signal)

For N-bit ADC:
  V_max = Vref
  V_min = Δ = Vref / 2^N  (1 LSB)

  DR = 20 log₁₀(2^N) = 6.02N dB

Note: DR ≈ SNR (for full-scale signal)
```

**Example**:
```
12-bit ADC, Vref = 1.25V:

V_max = 1.25 V
V_min = 1.25 / 4096 = 305 µV (1 LSB)

DR = 20 log₁₀(1.25 / 0.000305)
   = 20 log₁₀(4096)
   = 72.2 dB

This means signals spanning 72 dB (voltage ratio of 4096:1)
can be digitized without clipping or being buried in noise.
```

#### **5.2: Usable Dynamic Range**

**Problem**: Practical dynamic range less than theoretical.

**Factors reducing DR**:
```
1. Clipping headroom:    -3 to -6 dB (prevent occasional peaks)
2. Noise floor margin:    +3 to +6 dB (detection threshold)
3. Spurious tones:       SFDR limit

Usable DR ≈ SFDR - 6 dB (typical)

AD9361 example:
  Theoretical DR: 72 dB
  SFDR:           65 dB
  Usable DR:      ~59 dB (with margins)
```

### 6. Dithering

#### **6.1: Dithering Concept**

**Problem**: Low-level signals exhibit quantization distortion (non-random error pattern).

**Dithering**: Add small random noise BEFORE quantization to randomize error.

**Simple Analogy**: Old film cameras added grain to smooth color transitions. Dithering does the same for ADCs.

**Mathematical Process**:
```
Without dithering:
  y[n] = Q(x[n])

With dithering:
  y[n] = Q(x[n] + d[n]) - d[n]

Where d[n] is dither signal (usually uniform or triangular noise)
```

#### **6.2: Types of Dither**

**1. Rectangular (Uniform) Dither**:
```
d[n] ~ U(-Δ/2, +Δ/2)  (uniform distribution)

Effect:
  - Linearizes quantizer
  - Increases noise floor by 3 dB
  - Simple to implement

Noise power added: Δ²/12 (same as quantization noise)
Total noise: 2 × (Δ²/12) → SNR penalty: 3 dB
```

**2. Triangular (TPDF - Triangular Probability Density Function) Dither**:
```
d[n] = d₁[n] + d₂[n], where d₁, d₂ ~ U(-Δ/2, +Δ/2)

Triangular PDF: higher probability near zero, lower at extremes

Effect:
  - Linearizes quantizer
  - White quantization noise (frequency-independent)
  - Increases noise floor by 4.77 dB
  - OPTIMAL for audio (inaudible noise shaping)

Noise power added: Δ²/6
Total noise: Δ²/12 + Δ²/6 = Δ²/4 → SNR penalty: ~4.77 dB
```

**Trade-off**:
```
Without dither:
  - Higher SNR for large signals
  - Quantization distortion on small signals (harmonics, IMD)
  - Non-random error pattern

With triangular dither:
  - Lower SNR (~5 dB worse)
  - No quantization distortion (fully linearized)
  - Random white noise (psychoacoustically better)

Recommendation: Use dithering for audio (human ear prefers white noise over distortion)
                 Skip dithering for communications (maximize SNR)
```

#### **6.3: Dithering Implementation**

**Hardware (at ADC input)**:
```
Add analog noise before ADC:
  - Noise amplitude: ~1 LSB RMS
  - Gaussian or triangular distribution
  - Wideband (white noise)

Advantage: True analog dithering
Disadvantage: Adds physical noise source
```

**Software (digital dithering before requantization)**:
```c
// Example: 16-bit signal → 12-bit output with dithering

int16_t input = ...; // 16-bit value

// Generate triangular dither (sum of two uniform random)
int dither = (rand() & 0xF) + (rand() & 0xF) - 15; // ±15 range

// Add dither, quantize (shift right 4 bits), remove dither bias
int16_t output = (input + dither) >> 4;

// Result: 12-bit quantized value with reduced distortion
```

### 7. Clipping and Overload

#### **7.1: Hard Clipping**

**Definition**: Signal exceeds ADC full-scale range → saturates at max/min code.

**Mathematical Model**:
```
Hard clipper:
  y[n] = clip(x[n]) = {  +Vmax,  if x[n] > +Vmax
                       {  x[n],   if -Vmax ≤ x[n] ≤ +Vmax
                       {  -Vmax,  if x[n] < -Vmax

Clipping is NONLINEAR → generates harmonics!
```

**Harmonic Content** (clipped sinusoid):
```
Fourier series of clipped sine wave shows:
  - Fundamental (f₀): reduced amplitude
  - Odd harmonics (3f₀, 5f₀, 7f₀, ...): significant energy

Example: 10% clipping (90% of samples unclipped)
  Harmonic levels:
    1st (fundamental): -0.5 dB (slightly reduced)
    3rd harmonic:       -30 dBc
    5th harmonic:       -40 dBc

  Severe clipping (50% of samples clipped):
    3rd harmonic:       -10 dBc (very audible!)
```

**Clipping Detection**:
```
Peak-to-average ratio (PAR):
  PAR_dB = 20 log₁₀(|x_peak| / x_RMS)

For sinusoid: PAR = 20 log₁₀(√2) = 3.01 dB

If signal occasionally clips:
  - Probability of clipping > 0.1% → audible distortion
  - Keep peak 3-6 dB below full scale (headroom)
```

#### **7.2: Clipping Recovery**

**Problem**: Clipping is IRREVERSIBLE. Cannot recover original signal.

**Mitigation strategies**:
```
1. AGC (Automatic Gain Control):
   - Monitor signal level
   - Reduce gain if peaks approach clipping
   - Attack time: 10-100 ms (fast enough to prevent clipping)
   - Release time: 1-10 seconds (slow to avoid pumping)

2. Peak limiting:
   - Soft clipper before ADC (analog limiter)
   - Reduces harmonic distortion compared to hard clipping
   - Example: tanh() soft clipper

3. Headroom planning:
   - Design for PAR + 6 dB headroom
   - Example: OFDM signal with 12 dB PAR → keep RMS at -18 dBFS
```

### 8. AD9361 ADC Characteristics

#### **8.1: AD9361 Specifications**

**Key Parameters**:
```
ADC Resolution:        12 bits
Sample Rate:          25 MSPS (max per channel)
Full-Scale Input:     2.5 Vpp differential (±1.25V)
LSB Size:             2.5V / 4096 = 610 µV

Theoretical Performance:
  SNR:    6.02 × 12 + 1.76 = 74.0 dB
  SFDR:   74 dB (quantization-limited)
  DR:     72.2 dB
  ENOB:   12.0 bits

Actual Performance (typical @ 2.4 GHz):
  SNR:    70 dB (thermal noise-limited)
  SFDR:   65 dBc (nonlinearity-limited)
  ENOB:   11.3 bits
  NF:     ~7 dB (receive path noise figure)
```

**Degradation Factors**:
```
1. Thermal Noise (~7 dB NF):
   - LNA, mixer, baseband amplifier noise
   - Dominates at low/medium gain settings
   - Contribution: ~4 dB SNR loss

2. Clock Jitter (~100 fs RMS):
   - Phase noise on 640 MHz ADC clock
   - SNR_jitter = -20 log₁₀(2π f_in t_jitter)
   - At f_in = 10 MHz: SNR_jitter ≈ 80 dB (negligible)

3. ADC Nonlinearity (INL < 1 LSB):
   - Differential nonlinearity (DNL)
   - Integral nonlinearity (INL)
   - Generates harmonics → SFDR ≈ 65 dB
```

#### **8.2: Optimal Operating Range**

**RX Gain Settings**:
```
Manual Gain Mode (recommended for testing):

Low gain (0-30 dB):
  - Large input signals (> -30 dBm)
  - Risk of clipping
  - SNR: ~60-65 dB

Medium gain (30-50 dB):
  - Optimal for most signals (-60 to -30 dBm)
  - Best SNR: ~70 dB
  - Good linearity

High gain (50-73 dB):
  - Weak signals (< -60 dBm)
  - Increased noise floor
  - SNR: 55-65 dB (noise figure degrades)

Recommendation: Use 40-50 dB for lab tests
```

---

## Part 6: Complete C Source Code

(To be continued in next section...)

---

## Summary

In this lab, you learned:

✅ **Quantization Formula**: SNR_q = 6.02N + 1.76 dB
   - Each bit adds ~6 dB of SNR
   - 12-bit ADC: 74 dB theoretical SNR

✅ **Dynamic Range**: 6.02N dB for N-bit ADC
   - Range from full-scale to noise floor
   - 12-bit: 72 dB dynamic range

✅ **ENOB (Effective Number of Bits)**:
   - ENOB = (SNR_measured - 1.76) / 6.02
   - AD9361: ~11.3 bits effective (70 dB SNR)

✅ **Clipping Effects**:
   - Keep signal 3-6 dB below full scale
   - Clipping generates harmonics
   - Irreversible distortion

✅ **Dithering**:
   - Adds noise to improve linearity
   - Triangular dither whitens quantization noise
   - Trade noise floor for low-level performance

✅ **PlutoSDR AD9361**:
   - 12-bit ADC, 70 dB typical SNR
   - SFDR ~65 dB
   - Optimal RX gain: 30-50 dB typical

---

## Next Steps

Continue to:
- **LAB 2.4**: I/Q Imbalance and DC Offset Correction
- **LAB 3.1**: Digital Modulation (ASK, FSK, PSK)
- **LAB 4.1**: FIR and IIR Filter Design
