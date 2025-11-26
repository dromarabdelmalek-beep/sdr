# LAB 2.2: Decimation, Interpolation, and Sample Rate Conversion

## Overview

This lab explores **decimation** (downsampling), **interpolation** (upsampling), and **sample rate conversion** - fundamental operations in digital signal processing for SDR systems. You'll learn how to change sample rates while preserving signal integrity using proper filtering techniques.

## Learning Objectives

After completing this lab, you will understand:
- Decimation: reducing sample rate by integer factor M
- Interpolation: increasing sample rate by integer factor L
- Rational resampling: L/M ratio for arbitrary rate conversion
- Anti-aliasing filters (before decimation)
- Anti-imaging filters (after interpolation)
- Polyphase filter implementations for efficiency
- PlutoSDR's multi-stage decimation/interpolation chain
- Spectral effects of improper sample rate conversion

## Prerequisites

- LAB 0: PlutoSDR Setup
- LAB 1.1: Basic IQ Sampling
- LAB 2.1: Nyquist Sampling and Aliasing
- Understanding of Nyquist theorem
- Basic Python or C programming

## Theory

### 1. Decimation (Downsampling)

**Definition**: Reducing the sample rate by keeping only every Mth sample.

```
Decimation by M:
  Fs_out = Fs_in / M

Example: M=4, Fs_in=4 MSPS
  Input:  [x₀, x₁, x₂, x₃, x₄, x₅, x₆, x₇, ...]
  Output: [x₀,          x₄,          x₈,     ...]
  Fs_out = 1 MSPS
```

**Critical Requirement**: Must apply **anti-aliasing filter** before decimation!

```
Without filtering:
  If input contains frequencies > Fs_out/2, they will alias

With proper anti-aliasing:
  Fs_in = 4 MSPS, M = 4
  1. Apply LPF with cutoff ≤ 500 kHz (new Nyquist)
  2. Keep every 4th sample
  3. Result: Fs_out = 1 MSPS, no aliasing
```

**Frequency Domain View**:
```
Before decimation (Fs=4 MSPS):
  -2 MHz  -1 MHz    0    1 MHz   2 MHz
     |-------|-------|-------|-------|
           [Signal BW < 500 kHz]

After decimation (Fs=1 MSPS):
  -500 kHz    0    500 kHz
     |--------|--------|
     [Signal preserved]
```

### 2. Interpolation (Upsampling)

**Definition**: Increasing the sample rate by inserting L-1 zeros between samples.

```
Interpolation by L:
  Fs_out = L × Fs_in

Example: L=4, Fs_in=1 MSPS
  Input:  [x₀, x₁, x₂, x₃, ...]
  Output: [x₀, 0, 0, 0, x₁, 0, 0, 0, x₂, 0, 0, 0, x₃, ...]
  Fs_out = 4 MSPS
```

**Critical Requirement**: Must apply **anti-imaging filter** after zero-insertion!

```
After zero-insertion (no filtering):
  Original spectrum repeats at multiples of Fs_in
  "Images" appear at ±1 MHz, ±2 MHz, ±3 MHz

After anti-imaging LPF:
  Filter removes images
  Only original baseband signal remains
  Smooth interpolation between original samples
```

**Frequency Domain View**:
```
After zero-insertion (Fs=4 MSPS):
  -2 MHz  -1 MHz    0    1 MHz   2 MHz
     |-------|-------|-------|-------|
       Image  Signal  Signal  Image

After anti-imaging filter:
  -2 MHz  -1 MHz    0    1 MHz   2 MHz
     |-------|-------|-------|-------|
                [Signal only]
```

### 3. Rational Resampling (L/M)

**General Sample Rate Conversion**: Fs_out = (L/M) × Fs_in

```
Process:
  1. Interpolate by L (upsample)
  2. Filter at min(Fs_in/2, Fs_out/2)
  3. Decimate by M (downsample)

Example: Convert 3 MSPS → 2 MSPS
  Ratio: 2/3
  L = 2, M = 3

  3 MSPS → [×2] → 6 MSPS → [LPF] → [÷3] → 2 MSPS
```

**Common SDR Conversions**:
```
ADC/DAC ↔ FPGA ↔ USB ↔ Host
 61.44     30.72   10.24   1-10
  MSPS      MSPS    MSPS    MSPS

PlutoSDR AD9361:
  ADC: 61.44 MSPS (fixed)
  Decimation chain: ÷2, ÷2, ÷1/2/4
  Possible rates: 61.44, 30.72, 15.36, 10.24, ... MSPS
```

### 4. Filter Requirements

**Anti-Aliasing Filter (Decimation)**:
```
Cutoff frequency: fc ≤ Fs_out / 2
Stopband attenuation: > 60 dB typical
Passband ripple: < 0.1 dB
Transition width: narrow as practical

Example: 4 MSPS → 1 MSPS (M=4)
  Passband: 0 - 400 kHz
  Transition: 400 - 500 kHz
  Stopband: 500 kHz+ (>60 dB attenuation)
```

**Anti-Imaging Filter (Interpolation)**:
```
Cutoff frequency: fc ≤ Fs_in / 2
Same attenuation requirements
Removes spectral images

Example: 1 MSPS → 4 MSPS (L=4)
  Passband: 0 - 400 kHz
  Transition: 400 - 500 kHz
  Stopband: 500 kHz+ (removes images)
```

### 5. Polyphase Implementation

**Efficient Decimation**: Instead of filtering at Fs_in then discarding samples...

```
Naive approach (wasteful):
  1. Filter all samples at Fs_in
  2. Keep every Mth sample
  Result: Compute M× more than needed!

Polyphase approach (efficient):
  1. Split filter into M phases
  2. Process only samples we keep
  Result: Same output, M× faster!
```

**Polyphase Filter Structure**:
```
For M=4 decimation:
  h[n] = FIR filter (e.g., 64 taps)

  Split into 4 phases:
    Phase 0: h[0], h[4], h[8],  ..., h[60]
    Phase 1: h[1], h[5], h[9],  ..., h[61]
    Phase 2: h[2], h[6], h[10], ..., h[62]
    Phase 3: h[3], h[7], h[11], ..., h[63]

  For each output sample:
    Rotate through phases
    Compute only 16 MACs (not 64!)
```

**Computational Savings**:
```
Naive FIR decimation:
  MACs per output: N_taps

Polyphase decimation:
  MACs per output: N_taps / M

Example: 64-tap filter, M=4
  Naive: 64 MACs per output
  Polyphase: 16 MACs per output
  Speedup: 4×
```

### 6. PlutoSDR Decimation Chain

The AD9361 uses a multi-stage approach:

```
ADC → HB3 → HB2 → HB1 → RFIR → Output
61.44   30.72   15.36   7.68    Fs
MSPS    MSPS    MSPS    MSPS
        (÷2)    (÷2)    (÷2)    (÷1-4)

HB3, HB2, HB1: Half-Band Filters (÷2 each)
  - Efficient polyphase structure
  - 50% of taps are zero
  - Very low CPU cost

RFIR: Programmable FIR (÷1, ÷2, or ÷4)
  - 128 taps user-programmable
  - Can customize frequency response
```

**Sample Rate Selection**:
```python
sdr.sample_rate = 1000000  # 1 MSPS request

AD9361 automatically selects:
  61.44 MSPS → [÷2] → 30.72 MSPS → [÷2] → 15.36 MSPS
  → [÷2] → 7.68 MSPS → [÷8] → 0.96 MSPS (closest to 1.0)

Actual rate returned: 961,538 SPS
```

---

## Part 1: Simulation (Pure Python)

### Implementation 1: Decimation with Anti-Aliasing

```python
import numpy as np
import matplotlib.pyplot as plt
from scipy import signal

class DecimationDemo:
    """Demonstrate proper and improper decimation"""

    def __init__(self, fs_in=4e6, decimation_factor=4):
        """
        Args:
            fs_in: Input sample rate (Hz)
            decimation_factor: M (integer downsampling factor)
        """
        self.fs_in = fs_in
        self.M = decimation_factor
        self.fs_out = fs_in / decimation_factor

        print(f"Decimation Configuration:")
        print(f"  Input rate:  {fs_in/1e6:.2f} MSPS")
        print(f"  Factor:      M = {decimation_factor}")
        print(f"  Output rate: {self.fs_out/1e6:.2f} MSPS")
        print(f"  New Nyquist: {self.fs_out/2/1e6:.2f} MHz")

    def design_antialiasing_filter(self, transition_width=0.1):
        """
        Design anti-aliasing lowpass filter

        Args:
            transition_width: Fraction of Nyquist for transition band

        Returns:
            b, a: Filter coefficients
        """
        # Cutoff at 90% of new Nyquist frequency
        nyquist_out = self.fs_out / 2
        cutoff = 0.9 * nyquist_out

        # Normalized frequency (fraction of fs_in/2)
        wn = cutoff / (self.fs_in / 2)

        # Design Butterworth filter (8th order for steep rolloff)
        b, a = signal.butter(8, wn, btype='low')

        print(f"\nAnti-aliasing Filter:")
        print(f"  Type:    8th order Butterworth")
        print(f"  Cutoff:  {cutoff/1e3:.1f} kHz")
        print(f"  -3dB at: {cutoff/1e3:.1f} kHz")
        print(f"  -60dB at: ~{nyquist_out/1e3:.1f} kHz")

        return b, a

    def decimate_improper(self, samples):
        """
        Improper decimation: just drop samples (WRONG!)

        Args:
            samples: Input signal

        Returns:
            Decimated samples (with aliasing!)
        """
        return samples[::self.M]

    def decimate_proper(self, samples):
        """
        Proper decimation: filter then downsample

        Args:
            samples: Input signal

        Returns:
            Decimated samples (no aliasing)
        """
        # Design and apply anti-aliasing filter
        b, a = self.design_antialiasing_filter()
        filtered = signal.filtfilt(b, a, samples)

        # Downsample
        return filtered[::self.M]

    def test_signal(self, freq_low=200e3, freq_high=800e3, duration=0.001):
        """
        Create test signal with two tones

        Args:
            freq_low: Low frequency tone (within new Nyquist)
            freq_high: High frequency tone (above new Nyquist)
            duration: Signal duration (seconds)

        Returns:
            t, signal: Time vector and test signal
        """
        t = np.arange(0, duration, 1/self.fs_in)

        # Two tones
        signal_low = np.sin(2 * np.pi * freq_low * t)
        signal_high = 0.5 * np.sin(2 * np.pi * freq_high * t)

        test_signal = signal_low + signal_high

        print(f"\nTest Signal:")
        print(f"  Tone 1: {freq_low/1e3:.0f} kHz (below new Nyquist)")
        print(f"  Tone 2: {freq_high/1e3:.0f} kHz (ABOVE new Nyquist)")
        print(f"  Expected alias: {abs(freq_high - self.fs_out)/1e3:.0f} kHz")

        return t, test_signal

    def analyze_spectrum(self, samples, fs, label="Signal"):
        """
        Compute and return spectrum

        Args:
            samples: Input signal
            fs: Sample rate
            label: Plot label

        Returns:
            freqs, spectrum: Frequency vector and magnitude spectrum
        """
        n = len(samples)
        fft_result = np.fft.fftshift(np.fft.fft(samples))
        freqs = np.fft.fftshift(np.fft.fftfreq(n, 1/fs))
        spectrum = 20 * np.log10(np.abs(fft_result) / n + 1e-12)

        return freqs, spectrum

    def run_comparison(self):
        """Run complete decimation comparison"""

        # Generate test signal
        t_in, signal_in = self.test_signal()

        # Improper decimation (aliasing will occur!)
        signal_improper = self.decimate_improper(signal_in)

        # Proper decimation (no aliasing)
        signal_proper = self.decimate_proper(signal_in)

        # Analyze spectra
        freqs_in, spectrum_in = self.analyze_spectrum(signal_in, self.fs_in, "Input")
        freqs_out, spectrum_improper = self.analyze_spectrum(signal_improper, self.fs_out, "Improper")
        freqs_out2, spectrum_proper = self.analyze_spectrum(signal_proper, self.fs_out, "Proper")

        # Plot results
        fig, axes = plt.subplots(3, 1, figsize=(12, 10))

        # Original spectrum
        axes[0].plot(freqs_in/1e6, spectrum_in, 'b-', linewidth=1.5)
        axes[0].axvline(self.fs_out/2/1e6, color='r', linestyle='--',
                       label=f'New Nyquist: {self.fs_out/2/1e6:.2f} MHz')
        axes[0].axvline(-self.fs_out/2/1e6, color='r', linestyle='--')
        axes[0].set_xlabel('Frequency (MHz)')
        axes[0].set_ylabel('Magnitude (dB)')
        axes[0].set_title(f'Original Signal at {self.fs_in/1e6:.1f} MSPS')
        axes[0].grid(True, alpha=0.3)
        axes[0].legend()
        axes[0].set_ylim([-80, 0])

        # Improper decimation (with aliasing)
        axes[1].plot(freqs_out/1e6, spectrum_improper, 'r-', linewidth=1.5)
        axes[1].set_xlabel('Frequency (MHz)')
        axes[1].set_ylabel('Magnitude (dB)')
        axes[1].set_title(f'IMPROPER Decimation (no filter) - ALIASING OCCURS!')
        axes[1].grid(True, alpha=0.3)
        axes[1].set_ylim([-80, 0])
        axes[1].annotate('Aliased tone!', xy=(0.2, -10), fontsize=12, color='red')

        # Proper decimation (no aliasing)
        axes[2].plot(freqs_out2/1e6, spectrum_proper, 'g-', linewidth=1.5)
        axes[2].set_xlabel('Frequency (MHz)')
        axes[2].set_ylabel('Magnitude (dB)')
        axes[2].set_title(f'PROPER Decimation (with anti-aliasing filter) - Clean!')
        axes[2].grid(True, alpha=0.3)
        axes[2].set_ylim([-80, 0])
        axes[2].annotate('High freq removed', xy=(0.3, -60), fontsize=12, color='green')

        plt.tight_layout()
        plt.savefig('decimation_comparison.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved decimation_comparison.png")
        plt.show()

        return signal_in, signal_improper, signal_proper


# Test the decimation
if __name__ == "__main__":
    print("="*70)
    print("DECIMATION DEMONSTRATION")
    print("="*70)

    demo = DecimationDemo(fs_in=4e6, decimation_factor=4)
    signal_in, signal_improper, signal_proper = demo.run_comparison()

    print("\n" + "="*70)
    print("KEY OBSERVATIONS:")
    print("="*70)
    print("1. Improper decimation causes ALIASING")
    print("   - High frequency (800 kHz) folds into baseband")
    print("   - Appears at 200 kHz (alias)")
    print("   - Cannot distinguish from real 200 kHz tone!")
    print()
    print("2. Proper decimation prevents aliasing")
    print("   - Anti-aliasing filter removes high frequencies")
    print("   - Only 200 kHz tone remains")
    print("   - Clean spectrum, no artifacts")
    print()
    print("RULE: Always filter BEFORE decimation!")
    print("="*70)
```

### Implementation 2: Interpolation with Anti-Imaging

```python
class InterpolationDemo:
    """Demonstrate proper and improper interpolation"""

    def __init__(self, fs_in=1e6, interpolation_factor=4):
        """
        Args:
            fs_in: Input sample rate (Hz)
            interpolation_factor: L (integer upsampling factor)
        """
        self.fs_in = fs_in
        self.L = interpolation_factor
        self.fs_out = fs_in * interpolation_factor

        print(f"Interpolation Configuration:")
        print(f"  Input rate:  {fs_in/1e6:.2f} MSPS")
        print(f"  Factor:      L = {interpolation_factor}")
        print(f"  Output rate: {self.fs_out/1e6:.2f} MSPS")

    def design_antiimaging_filter(self):
        """
        Design anti-imaging lowpass filter

        Returns:
            b, a: Filter coefficients
        """
        # Cutoff at 90% of original Nyquist
        nyquist_in = self.fs_in / 2
        cutoff = 0.9 * nyquist_in

        # Normalized frequency (fraction of fs_out/2)
        wn = cutoff / (self.fs_out / 2)

        # Design filter
        b, a = signal.butter(8, wn, btype='low')

        print(f"\nAnti-imaging Filter:")
        print(f"  Type:    8th order Butterworth")
        print(f"  Cutoff:  {cutoff/1e3:.1f} kHz")
        print(f"  Purpose: Remove spectral images")

        return b, a

    def interpolate_improper(self, samples):
        """
        Improper interpolation: just insert zeros (WRONG!)

        Args:
            samples: Input signal

        Returns:
            Upsampled samples (with images!)
        """
        # Insert L-1 zeros between each sample
        n_out = len(samples) * self.L
        upsampled = np.zeros(n_out, dtype=samples.dtype)
        upsampled[::self.L] = samples

        return upsampled

    def interpolate_proper(self, samples):
        """
        Proper interpolation: insert zeros then filter

        Args:
            samples: Input signal

        Returns:
            Upsampled samples (no images)
        """
        # Insert zeros
        upsampled = self.interpolate_improper(samples)

        # Apply anti-imaging filter (and scale by L)
        b, a = self.design_antiimaging_filter()
        filtered = signal.filtfilt(b, a, upsampled)

        # Scale by L to preserve energy
        return filtered * self.L

    def test_signal(self, freq=200e3, duration=0.001):
        """
        Create test signal

        Args:
            freq: Signal frequency
            duration: Duration (seconds)

        Returns:
            t, signal: Time vector and test signal
        """
        t = np.arange(0, duration, 1/self.fs_in)
        test_signal = np.sin(2 * np.pi * freq * t)

        print(f"\nTest Signal:")
        print(f"  Frequency: {freq/1e3:.0f} kHz")
        print(f"  Samples:   {len(test_signal)}")

        return t, test_signal

    def run_comparison(self):
        """Run complete interpolation comparison"""

        # Generate test signal
        t_in, signal_in = self.test_signal()

        # Improper interpolation (images will appear!)
        signal_improper = self.interpolate_improper(signal_in)

        # Proper interpolation (no images)
        signal_proper = self.interpolate_proper(signal_in)

        # Analyze spectra
        freqs_in, spectrum_in = self.analyze_spectrum(signal_in, self.fs_in)
        freqs_out, spectrum_improper = self.analyze_spectrum(signal_improper, self.fs_out)
        freqs_out2, spectrum_proper = self.analyze_spectrum(signal_proper, self.fs_out)

        # Plot results
        fig, axes = plt.subplots(3, 1, figsize=(12, 10))

        # Original spectrum
        axes[0].plot(freqs_in/1e6, spectrum_in, 'b-', linewidth=1.5)
        axes[0].set_xlabel('Frequency (MHz)')
        axes[0].set_ylabel('Magnitude (dB)')
        axes[0].set_title(f'Original Signal at {self.fs_in/1e6:.1f} MSPS')
        axes[0].grid(True, alpha=0.3)
        axes[0].set_ylim([-80, 0])

        # Improper interpolation (with images)
        axes[1].plot(freqs_out/1e6, spectrum_improper, 'r-', linewidth=1.5)
        axes[1].axvline(self.fs_in/2/1e6, color='orange', linestyle='--',
                       label=f'Original Nyquist: {self.fs_in/2/1e6:.2f} MHz')
        axes[1].axvline(-self.fs_in/2/1e6, color='orange', linestyle='--')
        axes[1].set_xlabel('Frequency (MHz)')
        axes[1].set_ylabel('Magnitude (dB)')
        axes[1].set_title(f'IMPROPER Interpolation (no filter) - IMAGES APPEAR!')
        axes[1].grid(True, alpha=0.3)
        axes[1].legend()
        axes[1].set_ylim([-80, 0])
        axes[1].annotate('Spectral images!', xy=(0.8, -10), fontsize=12, color='red')

        # Proper interpolation (no images)
        axes[2].plot(freqs_out2/1e6, spectrum_proper, 'g-', linewidth=1.5)
        axes[2].axvline(self.fs_in/2/1e6, color='orange', linestyle='--',
                       label=f'Original Nyquist: {self.fs_in/2/1e6:.2f} MHz')
        axes[2].axvline(-self.fs_in/2/1e6, color='orange', linestyle='--')
        axes[2].set_xlabel('Frequency (MHz)')
        axes[2].set_ylabel('Magnitude (dB)')
        axes[2].set_title(f'PROPER Interpolation (with anti-imaging filter) - Clean!')
        axes[2].grid(True, alpha=0.3)
        axes[2].legend()
        axes[2].set_ylim([-80, 0])
        axes[2].annotate('Images removed', xy=(0.8, -60), fontsize=12, color='green')

        plt.tight_layout()
        plt.savefig('interpolation_comparison.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved interpolation_comparison.png")
        plt.show()

        return signal_in, signal_improper, signal_proper

    def analyze_spectrum(self, samples, fs):
        """Compute spectrum"""
        n = len(samples)
        fft_result = np.fft.fftshift(np.fft.fft(samples))
        freqs = np.fft.fftshift(np.fft.fftfreq(n, 1/fs))
        spectrum = 20 * np.log10(np.abs(fft_result) / n + 1e-12)
        return freqs, spectrum


# Test the interpolation
if __name__ == "__main__":
    print("\n" + "="*70)
    print("INTERPOLATION DEMONSTRATION")
    print("="*70)

    demo = InterpolationDemo(fs_in=1e6, interpolation_factor=4)
    signal_in, signal_improper, signal_proper = demo.run_comparison()

    print("\n" + "="*70)
    print("KEY OBSERVATIONS:")
    print("="*70)
    print("1. Improper interpolation creates SPECTRAL IMAGES")
    print("   - Original spectrum repeats at multiples of fs_in")
    print("   - Images appear at ±1 MHz, ±2 MHz, etc.")
    print("   - Wastes bandwidth, creates interference")
    print()
    print("2. Proper interpolation removes images")
    print("   - Anti-imaging filter removes replicas")
    print("   - Only original baseband signal remains")
    print("   - Smooth interpolation between samples")
    print()
    print("RULE: Always filter AFTER zero-insertion!")
    print("="*70)
```

### Implementation 3: Rational Resampling (L/M)

```python
class RationalResampler:
    """Arbitrary sample rate conversion using L/M ratio"""

    def __init__(self, fs_in, fs_out):
        """
        Args:
            fs_in: Input sample rate
            fs_out: Desired output sample rate
        """
        self.fs_in = fs_in
        self.fs_out = fs_out

        # Find L and M for rational conversion
        from fractions import Fraction
        ratio = Fraction(int(fs_out), int(fs_in)).limit_denominator(1000)
        self.L = ratio.numerator
        self.M = ratio.denominator

        print(f"Rational Resampling:")
        print(f"  Input rate:    {fs_in/1e6:.3f} MSPS")
        print(f"  Output rate:   {fs_out/1e6:.3f} MSPS")
        print(f"  Ratio:         {fs_out/fs_in:.6f}")
        print(f"  L/M:           {self.L}/{self.M}")
        print(f"  Intermediate:  {fs_in*self.L/1e6:.3f} MSPS")

    def resample(self, samples):
        """
        Perform L/M resampling

        Process:
          1. Upsample by L (insert zeros)
          2. Lowpass filter
          3. Downsample by M

        Args:
            samples: Input signal

        Returns:
            Resampled signal
        """
        print(f"\nResampling {len(samples)} samples...")

        # Step 1: Upsample by L
        n_up = len(samples) * self.L
        upsampled = np.zeros(n_up, dtype=samples.dtype)
        upsampled[::self.L] = samples
        print(f"  After ×{self.L}: {len(upsampled)} samples")

        # Step 2: Lowpass filter
        # Cutoff at min(fs_in/2, fs_out/2)
        nyquist_in = self.fs_in / 2
        nyquist_out = self.fs_out / 2
        cutoff = min(nyquist_in, nyquist_out)

        fs_intermediate = self.fs_in * self.L
        wn = cutoff / (fs_intermediate / 2)

        b, a = signal.butter(8, wn, btype='low')
        filtered = signal.filtfilt(b, a, upsampled) * self.L
        print(f"  After filter: {len(filtered)} samples (cutoff: {cutoff/1e3:.0f} kHz)")

        # Step 3: Downsample by M
        downsampled = filtered[::self.M]
        print(f"  After ÷{self.M}: {len(downsampled)} samples")

        return downsampled

    def test_resampling(self, freq=200e3, duration=0.001):
        """Test rational resampling with sine wave"""

        # Generate test signal
        t_in = np.arange(0, duration, 1/self.fs_in)
        signal_in = np.sin(2 * np.pi * freq * t_in)

        print(f"\nTest signal: {freq/1e3:.0f} kHz tone")
        print(f"Input samples: {len(signal_in)}")

        # Resample
        signal_out = self.resample(signal_in)

        print(f"Output samples: {len(signal_out)}")
        print(f"Expected samples: {int(len(signal_in) * self.fs_out / self.fs_in)}")

        # Analyze
        t_out = np.arange(len(signal_out)) / self.fs_out

        # Plot time domain
        fig, axes = plt.subplots(2, 1, figsize=(12, 8))

        axes[0].plot(t_in*1e6, signal_in, 'b-', linewidth=1, label='Input')
        axes[0].plot(t_in[::10]*1e6, signal_in[::10], 'bo', markersize=4)
        axes[0].set_xlabel('Time (µs)')
        axes[0].set_ylabel('Amplitude')
        axes[0].set_title(f'Input Signal at {self.fs_in/1e6:.1f} MSPS')
        axes[0].grid(True, alpha=0.3)
        axes[0].legend()
        axes[0].set_xlim([0, 50])

        axes[1].plot(t_out*1e6, signal_out, 'g-', linewidth=1, label='Output')
        axes[1].plot(t_out[::10]*1e6, signal_out[::10], 'go', markersize=4)
        axes[1].set_xlabel('Time (µs)')
        axes[1].set_ylabel('Amplitude')
        axes[1].set_title(f'Output Signal at {self.fs_out/1e6:.1f} MSPS')
        axes[1].grid(True, alpha=0.3)
        axes[1].legend()
        axes[1].set_xlim([0, 50])

        plt.tight_layout()
        plt.savefig('rational_resampling.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved rational_resampling.png")
        plt.show()

        return signal_in, signal_out


# Test rational resampling
if __name__ == "__main__":
    print("\n" + "="*70)
    print("RATIONAL RESAMPLING DEMONSTRATION")
    print("="*70)

    # Example: Convert 3 MSPS to 2 MSPS
    resampler = RationalResampler(fs_in=3e6, fs_out=2e6)
    signal_in, signal_out = resampler.test_resampling()

    print("\n" + "="*70)
    print("COMMON SDR CONVERSIONS:")
    print("="*70)
    print("61.44 MSPS → 30.72 MSPS: L/M = 1/2")
    print("30.72 MSPS → 10.24 MSPS: L/M = 1/3")
    print(" 3.00 MSPS →  2.00 MSPS: L/M = 2/3")
    print(" 1.92 MSPS →  1.00 MSPS: L/M = 25/48")
    print("="*70)
```

---

## Part 2: PlutoSDR Hardware Implementation

### Method 2A: Decimation Testing

```python
import adi
import numpy as np
import matplotlib.pyplot as plt
from scipy import signal

class PlutoDecimationTest:
    """Test PlutoSDR's decimation chain"""

    def __init__(self, uri="ip:192.168.2.1"):
        """Initialize PlutoSDR"""
        self.sdr = adi.Pluto(uri)
        self.sdr.tx_lo = int(915e6)
        self.sdr.rx_lo = int(915e6)
        self.sdr.tx_cyclic_buffer = True
        self.sdr.tx_hardwaregain_chan0 = -30
        self.sdr.rx_hardwaregain_chan0 = 0

        print("PlutoSDR Decimation Test")
        print(f"Center frequency: 915 MHz")

    def test_sample_rates(self, rates=[0.5e6, 1e6, 2e6, 4e6, 10e6]):
        """
        Test multiple sample rates

        Args:
            rates: List of sample rates to test
        """
        results = []

        for requested_rate in rates:
            # Set sample rate
            self.sdr.sample_rate = int(requested_rate)
            actual_rate = self.sdr.sample_rate

            print(f"\n{'='*60}")
            print(f"Requested: {requested_rate/1e6:.2f} MSPS")
            print(f"Actual:    {actual_rate/1e6:.6f} MSPS")
            print(f"Error:     {abs(actual_rate-requested_rate)/requested_rate*100:.2f}%")

            # Calculate decimation from ADC
            adc_rate = 61.44e6
            decimation = adc_rate / actual_rate
            print(f"Decimation from ADC: {decimation:.2f}×")

            # Generate and transmit test tone at 100 kHz offset
            tone_freq = 100e3
            n_samples = 2**14
            t = np.arange(n_samples) / actual_rate
            tx_samples = 0.5 * np.exp(2j * np.pi * tone_freq * t)

            # Transmit
            self.sdr.tx(tx_samples)

            # Receive
            rx_samples = self.sdr.rx()

            # Analyze spectrum
            fft_rx = np.fft.fftshift(np.fft.fft(rx_samples))
            freqs = np.fft.fftshift(np.fft.fftfreq(len(rx_samples), 1/actual_rate))
            spectrum = 20 * np.log10(np.abs(fft_rx) / len(rx_samples) + 1e-12)

            # Find peak
            peak_idx = np.argmax(np.abs(fft_rx))
            peak_freq = freqs[peak_idx]

            print(f"Tone frequency: {tone_freq/1e3:.1f} kHz")
            print(f"Measured peak:  {peak_freq/1e3:.1f} kHz")
            print(f"SNR:            {np.max(spectrum) - np.median(spectrum):.1f} dB")

            results.append({
                'requested': requested_rate,
                'actual': actual_rate,
                'freqs': freqs,
                'spectrum': spectrum
            })

        # Plot all spectra
        fig, axes = plt.subplots(len(results), 1, figsize=(12, 3*len(results)))
        if len(results) == 1:
            axes = [axes]

        for i, result in enumerate(results):
            axes[i].plot(result['freqs']/1e6, result['spectrum'], 'b-', linewidth=1)
            axes[i].set_xlabel('Frequency (MHz)')
            axes[i].set_ylabel('Magnitude (dB)')
            axes[i].set_title(f"Sample Rate: {result['actual']/1e6:.3f} MSPS")
            axes[i].grid(True, alpha=0.3)
            axes[i].set_ylim([-100, 0])

            # Mark Nyquist boundaries
            nyquist = result['actual'] / 2
            axes[i].axvline(nyquist/1e6, color='r', linestyle='--', alpha=0.5, label='Nyquist')
            axes[i].axvline(-nyquist/1e6, color='r', linestyle='--', alpha=0.5)
            axes[i].legend()

        plt.tight_layout()
        plt.savefig('pluto_decimation_test.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved pluto_decimation_test.png")
        plt.show()

        return results


# Run the test
if __name__ == "__main__":
    tester = PlutoDecimationTest()
    results = tester.test_sample_rates([0.5e6, 1e6, 2e6, 4e6, 10e6])

    print("\n" + "="*70)
    print("OBSERVATIONS:")
    print("="*70)
    print("1. PlutoSDR supports sample rates from ~521 kSPS to 61.44 MSPS")
    print("2. Actual rates may differ slightly from requested rates")
    print("3. AD9361 uses multi-stage decimation (HB3, HB2, HB1, RFIR)")
    print("4. Each stage includes anti-aliasing filters")
    print("5. Lower sample rates show cleaner spectra (more filtering)")
    print("="*70)
```

### Method 2B: Interpolation Testing

```python
class PlutoInterpolationTest:
    """Test PlutoSDR's interpolation chain"""

    def __init__(self, uri="ip:192.168.2.1"):
        """Initialize PlutoSDR"""
        self.sdr = adi.Pluto(uri)
        self.sdr.tx_lo = int(915e6)
        self.sdr.rx_lo = int(915e6)
        self.sdr.tx_cyclic_buffer = True
        self.sdr.tx_hardwaregain_chan0 = -20
        self.sdr.rx_hardwaregain_chan0 = 0

        print("PlutoSDR Interpolation Test")

    def test_interpolation_quality(self, fs_low=1e6, fs_high=10e6):
        """
        Compare low and high sample rate transmission

        Tests whether PlutoSDR's interpolation preserves signal quality

        Args:
            fs_low: Low sample rate
            fs_high: High sample rate
        """
        print(f"\nTesting interpolation quality:")
        print(f"  Low rate:  {fs_low/1e6:.1f} MSPS")
        print(f"  High rate: {fs_high/1e6:.1f} MSPS")
        print(f"  Interpolation factor: {fs_high/fs_low:.0f}×")

        results = []

        for fs_tx in [fs_low, fs_high]:
            self.sdr.sample_rate = int(fs_tx)

            # Generate multi-tone signal (three tones)
            n_samples = 2**14
            t = np.arange(n_samples) / fs_tx

            signal_tx = (
                0.3 * np.exp(2j * np.pi * 100e3 * t) +
                0.3 * np.exp(2j * np.pi * 200e3 * t) +
                0.3 * np.exp(2j * np.pi * 300e3 * t)
            )

            # Transmit
            self.sdr.tx(signal_tx)

            # Receive at high rate
            self.sdr.sample_rate = int(fs_high)
            rx_samples = self.sdr.rx()

            # Analyze
            fft_rx = np.fft.fftshift(np.fft.fft(rx_samples))
            freqs = np.fft.fftshift(np.fft.fftfreq(len(rx_samples), 1/fs_high))
            spectrum = 20 * np.log10(np.abs(fft_rx) / len(rx_samples) + 1e-12)

            results.append({
                'tx_rate': fs_tx,
                'freqs': freqs,
                'spectrum': spectrum
            })

            print(f"\nTX at {fs_tx/1e6:.1f} MSPS:")
            print(f"  Noise floor: {np.median(spectrum):.1f} dB")
            print(f"  Peak power:  {np.max(spectrum):.1f} dB")
            print(f"  SNR:         {np.max(spectrum) - np.median(spectrum):.1f} dB")

        # Plot comparison
        fig, axes = plt.subplots(2, 1, figsize=(12, 8))

        for i, result in enumerate(results):
            axes[i].plot(result['freqs']/1e6, result['spectrum'], 'b-', linewidth=1)
            axes[i].set_xlabel('Frequency (MHz)')
            axes[i].set_ylabel('Magnitude (dB)')
            axes[i].set_title(f"TX at {result['tx_rate']/1e6:.1f} MSPS, RX at {fs_high/1e6:.1f} MSPS")
            axes[i].grid(True, alpha=0.3)
            axes[i].set_ylim([-100, 0])
            axes[i].set_xlim([-5, 5])

            # Mark tone frequencies
            for freq in [0.1, 0.2, 0.3]:
                axes[i].axvline(freq, color='r', linestyle='--', alpha=0.3)
                axes[i].axvline(-freq, color='r', linestyle='--', alpha=0.3)

        plt.tight_layout()
        plt.savefig('pluto_interpolation_test.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved pluto_interpolation_test.png")
        plt.show()

        print("\n" + "="*70)
        print("CONCLUSION:")
        print("="*70)
        print("If both plots look similar:")
        print("  ✓ PlutoSDR's interpolation is working correctly")
        print("  ✓ Anti-imaging filters remove spectral replicas")
        print("  ✓ Signal quality preserved across sample rates")
        print()
        print("If low-rate TX shows artifacts:")
        print("  ✗ Possible interpolation issues")
        print("  ✗ Check for spectral images")
        print("="*70)

        return results


# Run the test
if __name__ == "__main__":
    tester = PlutoInterpolationTest()
    results = tester.test_interpolation_quality(fs_low=1e6, fs_high=10e6)
```

---

## Part 3: Advanced Topics

### Polyphase Filter Implementation

```python
class PolyphaseDecimator:
    """
    Efficient polyphase decimation implementation

    Uses polyphase decomposition to reduce computational cost
    """

    def __init__(self, decimation_factor, n_taps=64):
        """
        Args:
            decimation_factor: M (downsampling factor)
            n_taps: Total filter taps (should be multiple of M)
        """
        self.M = decimation_factor
        self.n_taps = n_taps

        # Design prototype lowpass filter
        cutoff = 0.9 / decimation_factor  # Normalized cutoff
        self.h = signal.firwin(n_taps, cutoff, window='hamming')

        # Decompose into polyphase branches
        self.polyphase_filters = self._decompose_polyphase()

        print(f"Polyphase Decimator:")
        print(f"  Decimation: M = {decimation_factor}")
        print(f"  Total taps: {n_taps}")
        print(f"  Branches:   {len(self.polyphase_filters)}")
        print(f"  Taps/branch: {len(self.polyphase_filters[0])}")
        print(f"  Computational savings: {decimation_factor}×")

    def _decompose_polyphase(self):
        """
        Decompose FIR filter into M polyphase branches

        Branch i: h[i], h[i+M], h[i+2M], ...

        Returns:
            List of M filter branches
        """
        branches = []
        for i in range(self.M):
            branch = self.h[i::self.M]
            branches.append(branch)
        return branches

    def decimate_naive(self, x):
        """
        Naive decimation: filter all samples, then downsample

        Computational cost: N_taps MACs per output sample
        """
        # Filter
        y_filtered = np.convolve(x, self.h, mode='same')

        # Downsample
        y_decimated = y_filtered[::self.M]

        return y_decimated

    def decimate_polyphase(self, x):
        """
        Polyphase decimation: only compute samples we keep

        Computational cost: N_taps/M MACs per output sample
        """
        n_out = len(x) // self.M
        y = np.zeros(n_out)

        for n in range(n_out):
            # Compute one output sample using all M branches
            for i in range(self.M):
                # Branch i processes x[n*M + i], x[n*M + i + M], ...
                branch = self.polyphase_filters[i]
                for k, tap in enumerate(branch):
                    idx = n * self.M + i - k * self.M
                    if 0 <= idx < len(x):
                        y[n] += tap * x[idx]

        return y

    def compare_methods(self, signal_length=10000):
        """Compare naive vs polyphase performance"""

        # Generate test signal
        x = np.random.randn(signal_length) + 1j * np.random.randn(signal_length)

        import time

        # Time naive method
        t_start = time.time()
        y_naive = self.decimate_naive(x)
        t_naive = time.time() - t_start

        # Time polyphase method
        t_start = time.time()
        y_polyphase = self.decimate_polyphase(x)
        t_polyphase = time.time() - t_start

        # Check results match
        error = np.max(np.abs(y_naive[:len(y_polyphase)] - y_polyphase))

        print(f"\nPerformance Comparison:")
        print(f"  Input samples:  {signal_length}")
        print(f"  Output samples: {len(y_polyphase)}")
        print(f"  Naive time:     {t_naive*1000:.2f} ms")
        print(f"  Polyphase time: {t_polyphase*1000:.2f} ms")
        print(f"  Speedup:        {t_naive/t_polyphase:.1f}×")
        print(f"  Max error:      {error:.2e}")

        return y_naive, y_polyphase


# Test polyphase implementation
if __name__ == "__main__":
    print("="*70)
    print("POLYPHASE DECIMATION")
    print("="*70)

    decimator = PolyphaseDecimator(decimation_factor=4, n_taps=64)
    y_naive, y_polyphase = decimator.compare_methods(signal_length=100000)

    print("\n" + "="*70)
    print("KEY POINTS:")
    print("="*70)
    print("1. Polyphase structure exploits downsampling")
    print("2. Only computes samples that are kept")
    print("3. Achieves M× speedup for M-fold decimation")
    print("4. Used in all practical SDR decimators")
    print("5. AD9361 uses polyphase half-band filters")
    print("="*70)
```

---

## Testing Procedures

### Test 1: Verify Anti-Aliasing (Simulation)

```bash
# Run decimation demo
python3 decimation_demo.py

# Expected results:
# - Improper decimation shows aliased 800 kHz tone at 200 kHz
# - Proper decimation shows only original 200 kHz tone
# - Spectrum plots saved to decimation_comparison.png
```

### Test 2: Verify Anti-Imaging (Simulation)

```bash
# Run interpolation demo
python3 interpolation_demo.py

# Expected results:
# - Improper interpolation shows spectral images
# - Proper interpolation shows clean spectrum
# - Plots saved to interpolation_comparison.png
```

### Test 3: PlutoSDR Sample Rate Conversion

```bash
# Test PlutoSDR's decimation chain
python3 pluto_decimation_test.py

# Verify:
# 1. All requested sample rates are supported
# 2. Actual rates match requested (within tolerance)
# 3. Spectrum shows clean tone at all rates
# 4. No aliasing artifacts visible
```

### Test 4: Polyphase Performance

```bash
# Run polyphase comparison
python3 polyphase_demo.py

# Verify:
# 1. Polyphase output matches naive implementation
# 2. Speedup approximately equals decimation factor
# 3. Max error < 1e-10 (numerical precision)
```

---

## Troubleshooting

### Problem 1: Aliasing in Decimated Signal

**Symptoms**:
- Unexpected tones appear in spectrum
- High frequencies "fold" into baseband

**Causes**:
- No anti-aliasing filter applied
- Filter cutoff too high
- Filter stopband attenuation insufficient

**Solutions**:
```python
# Increase filter order
b, a = signal.butter(12, wn, btype='low')  # Higher order

# Or use sharper filter design
h = signal.firwin(128, cutoff, window='kaiser', beta=8.0)
```

### Problem 2: Spectral Images in Interpolated Signal

**Symptoms**:
- Multiple copies of signal appear in spectrum
- Signal repeats at multiples of original Fs

**Causes**:
- No anti-imaging filter applied
- Filter cutoff too high

**Solutions**:
```python
# Apply proper anti-imaging filter
upsampled = np.zeros(len(samples) * L)
upsampled[::L] = samples

# Filter with cutoff at original Nyquist
b, a = signal.butter(10, cutoff/(Fs_out/2), btype='low')
interpolated = signal.filtfilt(b, a, upsampled) * L
```

### Problem 3: PlutoSDR Sample Rate Mismatch

**Symptoms**:
- Requested rate != actual rate
- Frequency measurements incorrect

**Causes**:
- AD9361 has limited rate selection
- Requested rate not achievable with integer decimation

**Solutions**:
```python
# Always read back actual rate
sdr.sample_rate = int(1e6)
actual_rate = sdr.sample_rate
print(f"Actual rate: {actual_rate}")

# Use actual rate in calculations
freqs = np.fft.fftfreq(len(samples), 1/actual_rate)
```

### Problem 4: Poor Polyphase Performance

**Symptoms**:
- Polyphase not faster than naive
- High computational cost

**Causes**:
- Implementation not optimized
- Python overhead dominates
- Not using NumPy vectorization

**Solutions**:
```python
# Use SciPy's built-in resampler (optimized C code)
from scipy.signal import resample_poly
y = resample_poly(x, up=1, down=M)

# Or use GNU Radio's polyphase implementation
```

---

## Summary

In this lab, you learned:

✅ **Decimation (M)**: Reduce sample rate by factor M
   - Requires anti-aliasing filter BEFORE downsampling
   - Prevents high frequencies from aliasing into baseband

✅ **Interpolation (L)**: Increase sample rate by factor L
   - Insert L-1 zeros between samples
   - Requires anti-imaging filter AFTER zero-insertion
   - Removes spectral replicas

✅ **Rational Resampling (L/M)**: Arbitrary rate conversion
   - Upsample by L, filter, downsample by M
   - Used for non-integer rate ratios

✅ **Polyphase Filters**: Efficient implementation
   - Exploit downsampling to reduce computation
   - M× speedup for M-fold decimation
   - Used in all practical SDR systems

✅ **PlutoSDR Multi-Stage Chain**:
   - AD9361 uses HB3 → HB2 → HB1 → RFIR
   - Supports 521 kSPS to 61.44 MSPS
   - Automatic anti-aliasing at each stage

---

## References

1. **Multirate Signal Processing**:
   - "Multirate Digital Signal Processing" by Crochiere & Rabiner
   - Understanding decimation and interpolation theory

2. **Polyphase Filters**:
   - "Multirate Systems and Filter Banks" by Vaidyanathan
   - Efficient implementations

3. **AD9361 Datasheet**:
   - Analog Devices AD9361 Reference Manual
   - Details of RX/TX filter chains

4. **GNU Radio**:
   - Rational resampler block documentation
   - Polyphase filter bank implementations

---

## Next Steps

Continue to:
- **LAB 2.3**: Quantization and ADC Resolution
- **LAB 2.4**: I/Q Imbalance Correction
- **LAB 3.1**: ASK, FSK, PSK Modulation Schemes
