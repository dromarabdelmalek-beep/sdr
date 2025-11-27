# LAB 2.1: Nyquist Sampling, Aliasing, and Bandwidth Measurement

## Learning Objectives

After completing this lab, you will be able to:
1. **Understand** the Nyquist sampling theorem and its implications
2. **Demonstrate** aliasing effects with undersampled signals
3. **Design** anti-aliasing filters to prevent spectral folding
4. **Measure** signal bandwidth accurately using FFT analysis
5. **Implement** proper sampling strategies for real-world signals

**Covered SDR Terms:** Nyquist Rate, Aliasing, Bandwidth, Sampling Rate, Anti-Aliasing Filter

**Prerequisites:** LAB 1.3 (I/Q Samples and Complex Baseband)

---

## Part 1: Nyquist-Shannon Sampling Theorem

### 1.1 Theory

**Nyquist-Shannon Sampling Theorem:**
> A continuous-time signal can be perfectly reconstructed from its samples if:
> 1. The signal is band-limited to B Hz (no frequency content above B Hz)
> 2. The sampling rate Fs ≥ 2B (at least twice the highest frequency)

**Key Terms:**
- **Nyquist Rate:** Fs_min = 2 × B (minimum sampling rate)
- **Nyquist Frequency:** Fn = Fs / 2 (maximum frequency representable)
- **Folding Frequency:** Same as Nyquist frequency (where aliasing occurs)

**Examples:**
```
Audio CD: 44.1 kHz sampling rate
  - Maximum frequency: 20 kHz (human hearing limit)
  - Nyquist rate: 2 × 20 kHz = 40 kHz
  - Actual rate: 44.1 kHz (provides 2.05× oversampling margin)

PlutoSDR: 61.44 Msps maximum
  - Nyquist frequency: 30.72 MHz
  - Can represent signals up to ±30.72 MHz baseband

Telephone: 8 kHz sampling rate
  - Nyquist frequency: 4 kHz
  - Voice bandwidth: 300 Hz - 3.4 kHz (fits within Nyquist limit)
```

### 1.2 What Happens When Fs < 2B? (Aliasing)

**Aliasing:**
When a signal contains frequencies above the Nyquist frequency (Fs/2), those frequencies "fold back" into the baseband and appear as false lower frequencies.

**Mathematical Description:**
```
If a signal contains frequency f > Fs/2:
  The alias appears at: f_alias = |f - n×Fs|  (where n chosen so f_alias < Fs/2)

Example:
  Fs = 10 kHz (Fn = 5 kHz)
  Signal at 7 kHz (above Fn)
  Alias appears at: |7 kHz - 10 kHz| = 3 kHz

  Signal at 13 kHz:
  Alias appears at: |13 kHz - 10 kHz| = 3 kHz (same alias!)
```

**Visualization:**
```
Frequency Spectrum:

Without Aliasing (Fs > 2B):
   Power
     │    ┌───┐
     │    │   │
     │────┴───┴────────────
     0   B   Fn=Fs/2     Frequency

With Aliasing (Fs < 2B):
   Power
     │    ┌───┐
     │    │   │╲ ╱  ← Folded components
     │────┴───┴─┼────────
     0   B  Fn│Fs/2     Frequency
            Alias!
```

---

## Part 2: Demonstrating Aliasing (Method 1 - Simulation)

### 2.1 Aliasing Demonstration Code

```python
#!/usr/bin/env python3
"""
aliasing_demo.py - Demonstrate aliasing effects in sampling
Method 1: Pure Python simulation
"""

import numpy as np
import matplotlib.pyplot as plt

def demonstrate_aliasing(signal_freq, sampling_rate, duration=0.01):
    """
    Demonstrate aliasing by sampling a sinusoid

    Args:
        signal_freq: Frequency of the signal (Hz)
        sampling_rate: Sampling rate (Hz)
        duration: Signal duration (seconds)

    Returns:
        Time arrays, signals, and alias frequency
    """
    # Generate high-resolution "continuous" time
    t_continuous = np.linspace(0, duration, 10000)
    signal_continuous = np.sin(2 * np.pi * signal_freq * t_continuous)

    # Generate sampled version
    t_sampled = np.arange(0, duration, 1/sampling_rate)
    signal_sampled = np.sin(2 * np.pi * signal_freq * t_sampled)

    # Calculate Nyquist frequency
    nyquist_freq = sampling_rate / 2

    # Calculate alias frequency
    if signal_freq > nyquist_freq:
        # Find the alias in the first Nyquist zone
        n = int(signal_freq / sampling_rate) + 1
        alias_freq = abs(signal_freq - n * sampling_rate)
        if alias_freq > nyquist_freq:
            alias_freq = sampling_rate - alias_freq
    else:
        alias_freq = signal_freq  # No aliasing

    return t_continuous, signal_continuous, t_sampled, signal_sampled, alias_freq, nyquist_freq


def plot_aliasing_demo(signal_freq, sampling_rate):
    """Plot aliasing demonstration"""
    t_cont, sig_cont, t_samp, sig_samp, alias, nyquist = \
        demonstrate_aliasing(signal_freq, sampling_rate)

    plt.figure(figsize=(14, 10))

    # Time domain
    plt.subplot(3, 1, 1)
    plt.plot(t_cont * 1000, sig_cont, 'b-', linewidth=2, alpha=0.7, label='Original Signal')
    plt.stem(t_samp * 1000, sig_samp, 'r', markerfmt='ro', label='Sampled Points', basefmt=' ')
    plt.xlabel('Time (ms)')
    plt.ylabel('Amplitude')
    plt.title(f'Time Domain: {signal_freq/1e3:.1f} kHz Signal Sampled at {sampling_rate/1e3:.1f} kHz')
    plt.legend()
    plt.grid(True, alpha=0.3)
    plt.xlim(0, 2)  # First 2 ms

    # Reconstructed signal from samples (shows alias)
    plt.subplot(3, 1, 2)
    t_recon = np.linspace(0, t_samp[-1], 1000)
    # Sinc interpolation would be ideal, but we'll use simple linear for visualization
    sig_recon = np.interp(t_recon, t_samp, sig_samp)
    plt.plot(t_cont * 1000, sig_cont, 'b-', linewidth=2, alpha=0.5, label='Original')
    plt.plot(t_recon * 1000, sig_recon, 'r-', linewidth=2, label='Reconstructed (aliased)')
    plt.xlabel('Time (ms)')
    plt.ylabel('Amplitude')
    plt.title(f'Reconstructed Signal: Appears as {alias/1e3:.1f} kHz (Alias!)')
    plt.legend()
    plt.grid(True, alpha=0.3)
    plt.xlim(0, 10)

    # Frequency domain
    plt.subplot(3, 1, 3)
    freq_orig = np.fft.fftfreq(len(t_cont), t_cont[1] - t_cont[0])
    fft_orig = np.fft.fft(sig_cont)

    freq_samp = np.fft.fftfreq(len(t_samp), 1/sampling_rate)
    fft_samp = np.fft.fft(sig_samp)

    plt.plot(freq_orig / 1e3, np.abs(fft_orig), 'b-', alpha=0.5, label='Original Spectrum')
    plt.stem(freq_samp / 1e3, np.abs(fft_samp), 'r', markerfmt='ro',
             label='Sampled Spectrum', basefmt=' ')

    # Mark Nyquist frequency
    plt.axvline(nyquist / 1e3, color='g', linestyle='--', linewidth=2,
                label=f'Nyquist Freq = {nyquist/1e3:.1f} kHz')
    plt.axvline(-nyquist / 1e3, color='g', linestyle='--', linewidth=2)

    plt.xlabel('Frequency (kHz)')
    plt.ylabel('Magnitude')
    plt.title('Frequency Domain')
    plt.legend()
    plt.grid(True, alpha=0.3)
    plt.xlim(-sampling_rate/1e3, sampling_rate/1e3)

    plt.tight_layout()
    return plt


# Test Cases
if __name__ == "__main__":
    print("=== Aliasing Demonstration ===\n")

    test_cases = [
        # (signal_freq, sampling_rate, description)
        (5e3, 20e3, "Proper Sampling (Fs > 2×f)"),
        (5e3, 10e3, "Nyquist Limit (Fs = 2×f)"),
        (5e3, 8e3, "Undersampling (Fs < 2×f) - ALIASING!"),
        (7e3, 10e3, "Undersampling - Alias appears at 3 kHz"),
    ]

    for i, (sig_f, samp_f, desc) in enumerate(test_cases, 1):
        print(f"\nTest {i}: {desc}")
        print(f"  Signal frequency: {sig_f/1e3:.1f} kHz")
        print(f"  Sampling rate: {samp_f/1e3:.1f} kHz")
        print(f"  Nyquist frequency: {samp_f/2/1e3:.1f} kHz")

        _, _, _, _, alias, nyquist = demonstrate_aliasing(sig_f, samp_f, duration=0.01)

        if sig_f > nyquist:
            print(f"  ⚠️  ALIASING! Signal appears as {alias/1e3:.1f} kHz")
        else:
            print(f"  ✓ No aliasing (signal at {alias/1e3:.1f} kHz)")

        # Plot first and last test cases
        if i in [1, 4]:
            plot_aliasing_demo(sig_f, samp_f)
            plt.savefig(f'aliasing_demo_case{i}.png', dpi=150)
            print(f"  Saved: aliasing_demo_case{i}.png")

    print("\n=== Interactive Test ===")
    print("Try your own frequencies:")

    # Interactive example
    fig = plot_aliasing_demo(signal_freq=7e3, sampling_rate=10e3)
    plt.savefig('aliasing_interactive.png', dpi=150)
    print("\n✓ Saved: aliasing_interactive.png")

    # Calculate and display alias table
    print("\n=== Alias Frequency Table ===")
    print("Sampling Rate: 10 kHz (Nyquist Freq: 5 kHz)")
    print("-" * 50)
    print(f"{'Signal Freq':<15} {'Alias Freq':<15} {'Status'}")
    print("-" * 50)

    for f in [1, 3, 5, 7, 9, 11, 13, 15]:
        _, _, _, _, alias, _ = demonstrate_aliasing(f * 1e3, 10e3)
        status = "OK" if f <= 5 else "ALIASED"
        print(f"{f:>6} kHz{'':<7} {alias/1e3:>6.1f} kHz{'':<7} {status}")
```

### 2.2 Running the Demonstration

```bash
# Run aliasing demonstration
python3 aliasing_demo.py

# Expected output shows:
# - Proper sampling: signal preserved
# - Nyquist limit: barely sufficient
# - Undersampling: false frequencies appear!
```

**Key Observations:**
1. **Below Nyquist rate:** Signal appears as lower frequency (alias)
2. **Multiple frequencies can alias to the same value**
3. **Cannot distinguish original from alias after sampling**

---

## Part 3: Aliasing with PlutoSDR (Method 2 - External App)

### 3.1 Real-World Aliasing Test

```python
#!/usr/bin/env python3
"""
plutosdr_aliasing_test.py - Demonstrate aliasing with PlutoSDR
Method 2: External application using PlutoSDR hardware
"""

import adi
import numpy as np
import matplotlib.pyplot as plt
import time

class AliasingTester:
    def __init__(self, pluto_uri="ip:192.168.2.1"):
        """Initialize PlutoSDR for aliasing tests"""
        print("Initializing PlutoSDR Aliasing Tester...")

        # Connect to PlutoSDR
        self.sdr = adi.Pluto(pluto_uri)

        # We'll use different sample rates to demonstrate aliasing
        self.test_sample_rates = [1e6, 2e6, 4e6, 8e6]  # 1, 2, 4, 8 Msps

        print("✓ PlutoSDR connected")

    def generate_tone(self, frequency, sample_rate, duration=0.001):
        """
        Generate a complex tone

        Args:
            frequency: Tone frequency relative to baseband (Hz)
            sample_rate: Sample rate (Hz)
            duration: Duration in seconds

        Returns:
            Complex samples
        """
        t = np.arange(0, duration, 1/sample_rate)
        tone = np.exp(2j * np.pi * frequency * t)
        # Normalize
        tone = tone / np.max(np.abs(tone)) * 0.8
        return tone

    def test_aliasing_effect(self, baseband_freq=1.5e6, sample_rate=2e6):
        """
        Test aliasing by transmitting and receiving at same frequency

        Args:
            baseband_freq: Frequency offset from center (Hz)
            sample_rate: Sample rate (Hz)
        """
        print(f"\n=== Test: {baseband_freq/1e6:.2f} MHz offset @ {sample_rate/1e6:.0f} Msps ===")

        # Configure TX
        self.sdr.sample_rate = int(sample_rate)
        self.sdr.tx_rf_bandwidth = int(sample_rate)
        self.sdr.tx_lo = int(915e6)  # Center frequency
        self.sdr.tx_hardwaregain_chan0 = -30  # Low power

        # Configure RX
        self.sdr.rx_lo = int(915e6)
        self.sdr.rx_rf_bandwidth = int(sample_rate)
        self.sdr.gain_control_mode_chan0 = "manual"
        self.sdr.rx_hardwaregain_chan0 = 40
        self.sdr.rx_buffer_size = 8192

        # Generate tone
        tx_samples = self.generate_tone(baseband_freq, sample_rate, duration=0.01)

        # Transmit
        self.sdr.tx(tx_samples)
        time.sleep(0.1)  # Let TX settle

        # Receive
        rx_samples = self.sdr.rx()

        # Analyze spectrum
        fft_rx = np.fft.fftshift(np.fft.fft(rx_samples))
        freqs = np.fft.fftshift(np.fft.fftfreq(len(rx_samples), 1/sample_rate))

        # Find peak
        peak_idx = np.argmax(np.abs(fft_rx))
        peak_freq = freqs[peak_idx]

        # Nyquist frequency
        nyquist = sample_rate / 2

        print(f"  Transmitted: {baseband_freq/1e6:.2f} MHz offset")
        print(f"  Nyquist frequency: ±{nyquist/1e6:.2f} MHz")
        print(f"  Received peak at: {peak_freq/1e6:.2f} MHz offset")

        # Check for aliasing
        if abs(baseband_freq) > nyquist:
            # Calculate expected alias
            n = int(abs(baseband_freq) / sample_rate) + 1
            expected_alias = abs(baseband_freq - n * sample_rate)
            if expected_alias > nyquist:
                expected_alias = sample_rate - expected_alias
            if baseband_freq < 0:
                expected_alias = -expected_alias

            print(f"  Expected alias: {expected_alias/1e6:.2f} MHz")
            print(f"  ⚠️  ALIASING DETECTED!")

            # Check if measurement matches theory
            error = abs(peak_freq - expected_alias)
            if error < 50e3:  # 50 kHz tolerance
                print(f"  ✓ Alias matches theory (error: {error/1e3:.1f} kHz)")
            else:
                print(f"  ✗ Unexpected alias (error: {error/1e3:.1f} kHz)")
        else:
            print(f"  ✓ No aliasing (within Nyquist limit)")

        return freqs, fft_rx, peak_freq

    def comprehensive_test(self):
        """
        Run comprehensive aliasing tests across different sample rates
        """
        print("\n" + "="*60)
        print("COMPREHENSIVE ALIASING TEST")
        print("="*60)

        fig, axes = plt.subplots(2, 2, figsize=(14, 10))
        fig.suptitle('Aliasing Effects at Different Sample Rates', fontsize=16)

        test_configs = [
            (1e6, 2e6, axes[0, 0], "2 Msps: 1 MHz offset (OK)"),
            (1.5e6, 2e6, axes[0, 1], "2 Msps: 1.5 MHz offset (ALIASED!)"),
            (3e6, 4e6, axes[1, 0], "4 Msps: 3 MHz offset (ALIASED!)"),
            (2e6, 8e6, axes[1, 1], "8 Msps: 2 MHz offset (OK)"),
        ]

        for baseband_f, samp_rate, ax, title in test_configs:
            freqs, fft_rx, peak_f = self.test_aliasing_effect(baseband_f, samp_rate)

            # Plot
            ax.plot(freqs / 1e6, 20 * np.log10(np.abs(fft_rx) + 1e-12))
            ax.axvline(samp_rate / 2 / 1e6, color='r', linestyle='--',
                      label=f'Nyquist: ±{samp_rate/2/1e6:.1f} MHz')
            ax.axvline(-samp_rate / 2 / 1e6, color='r', linestyle='--')
            ax.axvline(peak_f / 1e6, color='g', linestyle=':',
                      label=f'Peak: {peak_f/1e6:.2f} MHz')
            ax.set_xlabel('Frequency (MHz)')
            ax.set_ylabel('Magnitude (dB)')
            ax.set_title(title)
            ax.legend()
            ax.grid(True, alpha=0.3)
            ax.set_xlim(-samp_rate/1e6, samp_rate/1e6)

        plt.tight_layout()
        plt.savefig('plutosdr_aliasing_test.png', dpi=150)
        print("\n✓ Saved: plutosdr_aliasing_test.png")

        return fig


# Run tests
if __name__ == "__main__":
    tester = AliasingTester()

    # Run comprehensive test
    tester.comprehensive_test()

    print("\n" + "="*60)
    print("ALIASING SUMMARY")
    print("="*60)
    print("Key Takeaways:")
    print("1. Frequencies above Fs/2 alias back into baseband")
    print("2. Alias frequency: f_alias = |f - n×Fs|")
    print("3. Cannot distinguish original from alias after sampling")
    print("4. Solution: Use anti-aliasing filter before ADC")
    print("="*60)
```

### 3.2 Running PlutoSDR Aliasing Test

```bash
# Run PlutoSDR aliasing demonstration
python3 plutosdr_aliasing_test.py

# Expected output:
# - Shows aliasing at different sample rates
# - Generates spectrum plots
# - Compares theoretical vs measured alias frequencies
```

---

## Part 4: Method 3 - Hosted Application (Nyquist/Aliasing Tests on PlutoSDR ARM)

### Overview

This guide provides **complete step-by-step instructions** for implementing Nyquist sampling and aliasing demonstration directly on PlutoSDR's ARM processor. This C program tests sampling theory by transmitting tones and measuring aliasing effects - all executing on the embedded Linux system within PlutoSDR.

**Benefits over Method 2 (External Application)**:
- **Real-time sampling control**: Direct ADC/DAC access
- **Standalone operation**: No PC required for testing
- **Embedded DSP**: Production-ready sampling analysis on ARM
- **Hardware-level aliasing**: Test actual AD9361 sampling behavior

---

## THEORY: Deep Dive into Nyquist Sampling and Aliasing

Before implementing this lab, let's deeply understand **why** the Nyquist theorem matters and **how** aliasing affects real radio systems.

### What is the Nyquist Theorem? (Simple Explanation)

**Simple Analogy**: Imagine you're watching a helicopter's rotor blades:

- **Fast camera (high sample rate)**: You see the blades rotating smoothly forward ✓
- **Slow camera (low sample rate)**: The blades appear to rotate **backward** or stand still! ✗

This "backward rotation" illusion is **aliasing** in the time domain - it's the same phenomenon that causes false frequencies in radio.

**In Sampling Theory**:
- **Signal**: Contains frequencies up to B Hz
- **Nyquist Rate**: Minimum sampling rate = 2B samples/second
- **Nyquist Frequency**: Maximum representable frequency = Fs/2

**Simple Rule**:
```
Sample at LEAST 2× the highest frequency in your signal
Otherwise: FALSE frequencies appear (aliasing)
```

### Why Does Aliasing Happen? (Mathematical Insight)

**Sampling = Multiplication**:
When you sample a continuous signal, you're multiplying it by an impulse train:

```
Sampled signal = Continuous signal × Impulses at rate Fs

Time domain multiplication → Frequency domain CONVOLUTION
```

**Result in frequency domain**:
- Original spectrum gets **copied** (replicated) at every multiple of Fs
- These copies **overlap** if Fs < 2B
- Overlapping regions create **aliases** (false frequencies)

**Mathematical formula**:
```
Alias frequency: f_alias = |f_signal - n×Fs|

where n is chosen so that: 0 ≤ f_alias ≤ Fs/2
```

**Example Calculation**:
```
Fs = 10 kHz (Nyquist frequency = 5 kHz)
Signal at 7 kHz (above Nyquist)

Calculate alias:
  n = 1: |7 - 1×10| = 3 kHz ← This is the alias!

Verification:
  Both 7 kHz and 3 kHz produce the SAME samples when Fs = 10 kHz
  After sampling, you CANNOT tell them apart!
```

### Real-World Impact: Why Aliasing Breaks Radios

**Scenario 1: GPS Receiver** (without anti-aliasing filter)
```
GPS L1 signal: 1575.42 MHz
Receiver's ADC: 10 MHz sampling rate
Nyquist frequency: 5 MHz

After down-conversion to baseband: 2.046 MHz (within Nyquist)
But strong jammer at 12 MHz gets aliased to:
  |12 - 10| = 2 MHz ← Alias appears ON TOP of GPS signal!

Result: GPS signal lost, receiver fails to get position ✗
Solution: Anti-aliasing filter removes everything above 5 MHz ✓
```

**Scenario 2: Tactical Radio** (improper sampling)
```
Desired signal: 915.000 MHz
Enemy jammer: 915.008 MHz (8 MHz offset)

If Fs = 10 MHz (Nyquist = 5 MHz):
  Desired: 0 Hz baseband ✓
  Jammer: 8 MHz → |8 - 10| = 2 MHz alias

Even though jammer is OUTSIDE your intended bandwidth,
it aliases back and jams your communication!

Solution: Set Fs > 16 MHz OR use anti-aliasing filter
```

**Scenario 3: Audio Digitization** (why CDs use 44.1 kHz)
```
Human hearing: 20 Hz - 20 kHz
Nyquist rate: 2 × 20 kHz = 40 kHz minimum

CD standard: 44.1 kHz (10% margin)
  - Allows imperfect anti-aliasing filter
  - Filter transition band: 20-22 kHz
  - Everything above 22 kHz must be blocked

If used 40 kHz exactly:
  - Filter would need brick-wall response (impossible)
  - Slight filter roll-off would cause aliasing
  - Audio would have artifacts
```

### Bandwidth vs. Nyquist Rate (Critical Distinction)

**Common Confusion**:
- "My signal is at 915 MHz, so I need 1.83 GHz sampling rate"
- **WRONG!** You only need to sample the **bandwidth**, not the carrier!

**Correct Understanding**:
```
RF Signal: 915.000 - 915.020 MHz (20 kHz bandwidth)

After down-conversion to baseband:
  Signal occupies: -10 kHz to +10 kHz
  Bandwidth B = 20 kHz
  Nyquist rate = 2 × 10 kHz = 20 kHz

Practical sampling rate: 40-100 kHz (2-5× oversampling)
NOT 1.83 GHz!
```

**Why this works**:
1. **Mixer** down-converts 915 MHz → baseband (0 Hz center)
2. **Low-pass filter** removes everything above B/2
3. **ADC** samples at Fs ≥ 2B
4. All information about original 915 MHz signal is preserved!

**PlutoSDR Example**:
```
AD9361 specifications:
  - RF frequency: 325 MHz - 3.8 GHz
  - Sampling rate: 2.084 - 61.44 MHz

How can 61 MHz sampling capture 3.8 GHz signal?

Answer: Quadrature down-conversion!
  1. LO tunes to 3.8 GHz
  2. Mixer creates baseband I/Q
  3. ADCs sample at 61 MHz → Captures ±30 MHz bandwidth
  4. Total instantaneous bandwidth: 60 MHz

NOT sampling 3.8 GHz directly - sampling the 60 MHz baseband!
```

### Complex (I/Q) Sampling and Aliasing

**Real Sampling** (one ADC):
```
Nyquist frequency: Fs/2
Representable range: 0 to Fs/2 (positive frequencies only)

Problem: Cannot distinguish +f from -f
  e.g., +10 kHz and -10 kHz look identical
```

**Complex (I/Q) Sampling** (two ADCs):
```
Nyquist frequency: Still Fs/2
Representable range: -Fs/2 to +Fs/2 (BOTH positive and negative!)

Advantage: Can distinguish +f from -f
  e.g., +10 kHz (I leading Q) vs -10 kHz (Q leading I)
```

**Aliasing in I/Q**:
```
For complex baseband at rate Fs:
  Signal at +f: No aliasing if |f| < Fs/2
  Signal at -f: No aliasing if |f| < Fs/2

Total usable bandwidth: Fs (full sampling rate!)

Example: Fs = 10 MHz
  Real sampling: 0 to 5 MHz (5 MHz bandwidth)
  I/Q sampling: -5 to +5 MHz (10 MHz bandwidth) ← 2× better!
```

**Why PlutoSDR uses I/Q**:
- 2× bandwidth efficiency
- Can distinguish upper/lower sidebands
- Required for frequency-domain multiplexing

### Anti-Aliasing Filters (The Solution)

**What is an anti-aliasing filter?**
A **low-pass filter** placed **before the ADC** that removes all frequencies above Fs/2.

**Analog vs Digital Implementation**:

1. **Analog anti-aliasing filter** (before ADC):
   ```
   Signal → [Analog LPF] → ADC → Digital processing
             Removes f > Fs/2

   Advantage: Prevents aliasing completely
   Disadvantage: Fixed cutoff frequency, analog components age
   ```

2. **Oversampling + Digital filter** (modern approach):
   ```
   Signal → [Wide analog LPF] → [Fast ADC] → [Digital decimation filter] → Output
            Removes f > 5×Fs             Removes f > Fs/2

   Example: AD9361 approach
     - Analog filter: Very gradual roll-off
     - ADC: Samples at 640 MHz max
     - Digital filter: Sharp brick-wall response
     - Decimation: Down to user-selected rate (2-61 MHz)

   Advantages:
     - Flexible sample rate (software-configurable)
     - Better filter performance (digital FIR)
     - Consistent performance (no analog drift)
   ```

**AD9361 Filter Chain**:
```
┌──────────────────────────────────────────────────────────────┐
│                     AD9361 RX Path                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  RF Input                                                    │
│     │                                                        │
│     v                                                        │
│  [LNA + Mixer] → I/Q baseband                                │
│     │                                                        │
│     v                                                        │
│  [Analog LPF] ← Programmable cutoff (200 kHz - 20 MHz)       │
│     │                                                        │
│     v                                                        │
│  [ADC @ 640 MHz] ← Fixed-rate oversampling                   │
│     │                                                        │
│     v                                                        │
│  [Digital FIR filter] ← Programmable decimation              │
│     │                                                        │
│     v                                                        │
│  Output @ Fs (2.084 - 61.44 MHz)                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘

Anti-aliasing occurs in BOTH stages:
  1. Analog LPF: Removes very high frequencies
  2. Digital FIR: Removes frequencies > Fs/2 after decimation
```

**Filter Performance Metrics**:
```
Ideal anti-aliasing filter:
  - Passband: |H(f)| = 1 for f < Fs/2
  - Stopband: |H(f)| = 0 for f > Fs/2
  - Transition: Instantaneous (brick wall)

Real anti-aliasing filter:
  - Passband ripple: ±0.1 dB typical
  - Stopband attenuation: -80 dB typical
  - Transition band: 10-20% of Fs

Example: Fs = 10 MHz
  - Passband: 0 - 4.5 MHz (±0.1 dB)
  - Transition: 4.5 - 5.5 MHz (gradual roll-off)
  - Stopband: > 5.5 MHz (< -80 dB)
```

### Bandwidth Measurement Techniques

**What is bandwidth?**
The range of frequencies occupied by a signal.

**3 dB Bandwidth** (most common):
- Frequencies where power is at least half the peak power
- Math: Points where |H(f)| ≥ -3 dB from peak
- Used for: Channel bandwidth, filter response

**Occupied Bandwidth** (regulatory):
- Contains 99% of signal power
- Used for: FCC compliance, spectrum allocation
- Example: Wi-Fi 20 MHz channels actually use ~18 MHz occupied bandwidth

**Null-to-Null Bandwidth**:
- Distance between spectral nulls (zeros)
- Used for: Modulated signals (BPSK, QPSK)
- Example: BPSK at 1 Mbps → 2 MHz null-to-null bandwidth

**How to measure bandwidth in code**:

1. **FFT Method** (frequency domain):
   ```c
   // Capture I/Q samples
   capture_iq_samples(sdr, &i, &q, N);

   // Compute FFT
   fft(i, q, N, fft_mag);

   // Find peak
   peak_idx = find_max(fft_mag, N);
   peak_power = fft_mag[peak_idx];

   // Find -3 dB points
   threshold = peak_power / sqrt(2);  // -3 dB = 0.707× power
   lower_idx = find_first_below(fft_mag, 0, peak_idx, threshold);
   upper_idx = find_first_below(fft_mag, peak_idx, N, threshold);

   // Calculate bandwidth
   bandwidth = (upper_idx - lower_idx) * (Fs / N);
   ```

2. **Power Method** (time domain):
   ```c
   // Measure signal power in bins
   total_power = sum(fft_mag);
   target_power = 0.99 * total_power;  // 99% occupied bandwidth

   // Integrate from center outward until reaching target
   integrated = 0;
   bins = 0;
   while (integrated < target_power) {
       integrated += power_in_bin[bins];
       bins++;
   }

   occupied_bandwidth = bins * (Fs / N);
   ```

3. **Correlation Method** (for known signals):
   ```c
   // Correlate with reference signal at different frequencies
   for (f = -Fs/2; f < Fs/2; f += df) {
       correlation[f] = correlate(received, reference_at_freq(f));
   }

   // Find correlation > threshold
   bandwidth = count_bins_above_threshold(correlation) * df;
   ```

### What This Lab Demonstrates

**4 Key Tests on PlutoSDR ARM**:

1. **Proper Sampling Test**:
   - Transmit tone at 500 kHz
   - Sample at 2 MHz (Nyquist = 1 MHz)
   - Verify tone appears at 500 kHz ✓ No aliasing

2. **Aliasing Test**:
   - Transmit tone at 1.5 MHz
   - Sample at 2 MHz (Nyquist = 1 MHz)
   - Verify tone aliases to -500 kHz ← Aliasing demonstrated!

3. **Bandwidth Measurement**:
   - Transmit QPSK signal
   - Measure occupied bandwidth using FFT
   - Compare to theoretical bandwidth

4. **Nyquist Zone Test**:
   - Sweep tone frequency from -Fs to +Fs
   - Map where frequencies appear after sampling
   - Demonstrate Nyquist zones (1st, 2nd, 3rd...)

**Why run on PlutoSDR ARM?**:
- **Real ADC/DAC behavior**: See actual hardware quantization
- **Loopback testing**: TX → RX on same device
- **Embedded deployment**: Standalone sampling analyzer
- **No PC dependency**: Battery-powered field testing

**Real-world application**:
```
Spectrum analyzer calibration:
  1. Transmit known tone frequencies
  2. Measure where they appear
  3. Verify no aliasing within operating bandwidth
  4. Test anti-aliasing filter performance
  5. ✓ Certify analyzer for field use
```

This hosted application gives you **direct access** to the AD9361's sampling system for validating Nyquist theorem and characterizing aliasing behavior.

---