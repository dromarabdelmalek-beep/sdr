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

## Part 5: Complete C Source Code for PlutoSDR ARM

### lab2_1_method3_hosted.c

This program implements comprehensive Nyquist sampling and aliasing tests directly on PlutoSDR.

```c
/*
 * LAB 2.1 - Method 3: Nyquist Sampling and Aliasing Hosted Application
 *
 * Demonstrates Nyquist-Shannon sampling theorem and aliasing effects
 * by transmitting test tones and analyzing received spectrum on PlutoSDR ARM.
 *
 * Features:
 *  - Tone generation at configurable frequencies
 *  - Aliasing demonstration (frequencies above Nyquist)
 *  - Bandwidth measurement using FFT
 *  - Nyquist zone mapping
 *  - Sample rate verification
 *
 * Compile: See compile_lab2_1.sh
 * Deploy:  See deploy_lab2_1.sh
 * Run:     ./lab2_1_hosted [test_mode]
 *          Modes: proper_sampling, aliasing, bandwidth, nyquist_zones
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <stdbool.h>
#include <math.h>
#include <time.h>
#include <unistd.h>
#include <iio.h>
#include <complex.h>

// ============================================================================
// CONFIGURATION PARAMETERS
// ============================================================================

// RF configuration
#define CENTER_FREQ        915000000  // 915 MHz
#define RX_GAIN            60         // RX gain in dB
#define TX_GAIN            -20        // TX gain in dB (low power for loopback)
#define BUFFER_SIZE        16384      // Samples per capture

// Test configurations (different sample rates for aliasing tests)
#define TEST_SAMPLE_RATE_1  2000000   // 2 MSPS
#define TEST_SAMPLE_RATE_2  4000000   // 4 MSPS
#define TEST_SAMPLE_RATE_3  8000000   // 8 MSPS

// Tone generation
#define TONE_AMPLITUDE     0.8        // 80% of full scale
#define TONE_DURATION_MS   100        // 100 ms tone duration

// Analysis parameters
#define FFT_SIZE           8192       // FFT size for spectrum analysis
#define BANDWIDTH_THRESHOLD_DB  -3.0  // 3 dB bandwidth threshold

// ============================================================================
// DATA STRUCTURES
// ============================================================================

typedef struct {
    double frequency_hz;
    double magnitude;
    double phase_deg;
    bool detected;
} ToneInfo;

typedef struct {
    double center_freq_hz;
    double bandwidth_3db_hz;
    double bandwidth_occupied_hz;
    double peak_power_db;
    int num_peaks;
} BandwidthMeasurement;

typedef struct {
    int zone_number;
    double input_freq;
    double observed_freq;
    double alias_freq;
    bool aliased;
} NyquistZoneTest;

typedef struct {
    struct iio_context *ctx;
    struct iio_device *phy;
    struct iio_device *tx_dev;
    struct iio_device *rx_dev;
    struct iio_channel *rx_phy_ch;
    struct iio_channel *tx_phy_ch;
    struct iio_channel *rx_i;
    struct iio_channel *rx_q;
    struct iio_channel *tx_i;
    struct iio_channel *tx_q;
    struct iio_buffer *rxbuf;
    struct iio_buffer *txbuf;
    long long sample_rate;
} PlutoSDR;

// ============================================================================
// UTILITY FUNCTIONS
// ============================================================================

/**
 * Get high-resolution timestamp in seconds
 */
static double get_time_seconds(void)
{
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec + ts.tv_nsec / 1e9;
}

/**
 * Set IIO channel attribute (long long value)
 */
static int set_channel_attr_ll(struct iio_channel *chn, const char *attr, long long val)
{
    int ret = iio_channel_attr_write_longlong(chn, attr, val);
    if (ret < 0) {
        fprintf(stderr, "Failed to set %s: %s\n", attr, strerror(-ret));
    }
    return ret;
}

/**
 * Set IIO channel attribute (string value)
 */
static int set_channel_attr_str(struct iio_channel *chn, const char *attr, const char *val)
{
    int ret = iio_channel_attr_write(chn, attr, val);
    if (ret < 0) {
        fprintf(stderr, "Failed to set %s to %s: %s\n", attr, val, strerror(-ret));
    }
    return ret;
}

// ============================================================================
// PLUTOSDR INITIALIZATION
// ============================================================================

/**
 * Initialize PlutoSDR hardware via local IIO
 */
static int init_plutosdr(PlutoSDR *sdr, long long sample_rate)
{
    printf("Initializing PlutoSDR (local IIO context)...\n");

    // Create local IIO context
    sdr->ctx = iio_create_local_context();
    if (!sdr->ctx) {
        fprintf(stderr, "Failed to create IIO context\n");
        return -1;
    }
    printf("  ✓ IIO context created (local)\n");

    // Get AD9361 PHY device
    sdr->phy = iio_context_find_device(sdr->ctx, "ad9361-phy");
    if (!sdr->phy) {
        fprintf(stderr, "Failed to find ad9361-phy device\n");
        return -1;
    }
    printf("  ✓ Found ad9361-phy\n");

    // Get TX device
    sdr->tx_dev = iio_context_find_device(sdr->ctx, "cf-ad9361-dds-core-lpc");
    if (!sdr->tx_dev) {
        fprintf(stderr, "Failed to find TX device\n");
        return -1;
    }
    printf("  ✓ Found TX device\n");

    // Get RX device
    sdr->rx_dev = iio_context_find_device(sdr->ctx, "cf-ad9361-lpc");
    if (!sdr->rx_dev) {
        fprintf(stderr, "Failed to find RX device\n");
        return -1;
    }
    printf("  ✓ Found RX device\n");

    // Get PHY channels
    sdr->rx_phy_ch = iio_device_find_channel(sdr->phy, "voltage0", false);
    sdr->tx_phy_ch = iio_device_find_channel(sdr->phy, "voltage0", true);

    if (!sdr->rx_phy_ch || !sdr->tx_phy_ch) {
        fprintf(stderr, "Failed to find PHY channels\n");
        return -1;
    }

    // Get RX I/Q channels
    sdr->rx_i = iio_device_find_channel(sdr->rx_dev, "voltage0", false);
    sdr->rx_q = iio_device_find_channel(sdr->rx_dev, "voltage1", false);

    if (!sdr->rx_i || !sdr->rx_q) {
        fprintf(stderr, "Failed to find RX I/Q channels\n");
        return -1;
    }

    iio_channel_enable(sdr->rx_i);
    iio_channel_enable(sdr->rx_q);
    printf("  ✓ RX I/Q channels enabled\n");

    // Get TX I/Q channels
    sdr->tx_i = iio_device_find_channel(sdr->tx_dev, "voltage0", true);
    sdr->tx_q = iio_device_find_channel(sdr->tx_dev, "voltage1", true);

    if (sdr->tx_i && sdr->tx_q) {
        iio_channel_enable(sdr->tx_i);
        iio_channel_enable(sdr->tx_q);
        printf("  ✓ TX I/Q channels enabled\n");
    }

    // Configure sample rate
    sdr->sample_rate = sample_rate;
    set_channel_attr_ll(sdr->rx_phy_ch, "sampling_frequency", sample_rate);
    set_channel_attr_ll(sdr->tx_phy_ch, "sampling_frequency", sample_rate);
    printf("  ✓ Sample rate: %.3f MSPS\n", sample_rate / 1e6);

    // Configure RX frequency and gain
    struct iio_channel *rx_lo = iio_device_find_channel(sdr->phy, "altvoltage0", true);
    set_channel_attr_ll(rx_lo, "frequency", CENTER_FREQ);
    set_channel_attr_str(sdr->rx_phy_ch, "gain_control_mode", "manual");
    set_channel_attr_ll(sdr->rx_phy_ch, "hardwaregain", RX_GAIN);
    printf("  ✓ RX: %.3f MHz, %d dB gain\n", CENTER_FREQ / 1e6, RX_GAIN);

    // Configure TX frequency and gain
    struct iio_channel *tx_lo = iio_device_find_channel(sdr->phy, "altvoltage1", true);
    set_channel_attr_ll(tx_lo, "frequency", CENTER_FREQ);
    set_channel_attr_ll(sdr->tx_phy_ch, "hardwaregain", TX_GAIN);
    printf("  ✓ TX: %.3f MHz, %d dB gain\n", CENTER_FREQ / 1e6, TX_GAIN);

    // Create RX buffer
    sdr->rxbuf = iio_device_create_buffer(sdr->rx_dev, BUFFER_SIZE, false);
    if (!sdr->rxbuf) {
        fprintf(stderr, "Failed to create RX buffer\n");
        return -1;
    }
    printf("  ✓ RX buffer created (%d samples)\n", BUFFER_SIZE);

    // Create TX buffer
    sdr->txbuf = iio_device_create_buffer(sdr->tx_dev, BUFFER_SIZE, true);
    if (!sdr->txbuf) {
        fprintf(stderr, "Failed to create TX buffer\n");
        return -1;
    }
    printf("  ✓ TX buffer created (%d samples)\n", BUFFER_SIZE);

    printf("PlutoSDR initialization complete!\n\n");
    return 0;
}

/**
 * Cleanup PlutoSDR resources
 */
static void cleanup_plutosdr(PlutoSDR *sdr)
{
    if (sdr->txbuf) iio_buffer_destroy(sdr->txbuf);
    if (sdr->rxbuf) iio_buffer_destroy(sdr->rxbuf);
    if (sdr->ctx) iio_context_destroy(sdr->ctx);
    printf("PlutoSDR resources released\n");
}

// ============================================================================
// TONE GENERATION
// ============================================================================

/**
 * Generate complex tone at specified frequency
 */
static int generate_tone(PlutoSDR *sdr, double tone_freq_hz)
{
    printf("Generating tone at %.3f kHz offset...\n", tone_freq_hz / 1e3);

    // Get TX buffer pointer
    void *tx_buf_start = iio_buffer_start(sdr->txbuf);
    int16_t *tx_samples = (int16_t *)tx_buf_start;

    // Generate complex tone: I(t) + jQ(t) = A * e^(j*2*pi*f*t)
    for (size_t n = 0; n < BUFFER_SIZE; n++) {
        double t = (double)n / sdr->sample_rate;
        double phase = 2.0 * M_PI * tone_freq_hz * t;

        // I and Q components
        double i_val = TONE_AMPLITUDE * cos(phase);
        double q_val = TONE_AMPLITUDE * sin(phase);

        // Convert to 12-bit signed integer (-2048 to 2047)
        tx_samples[2*n]     = (int16_t)(i_val * 2047.0);
        tx_samples[2*n + 1] = (int16_t)(q_val * 2047.0);
    }

    // Push buffer to TX
    ssize_t nbytes = iio_buffer_push(sdr->txbuf);
    if (nbytes < 0) {
        fprintf(stderr, "Failed to push TX buffer: %s\n", strerror(-nbytes));
        return -1;
    }

    printf("  ✓ Tone transmitted (%zd bytes)\n", nbytes);
    return 0;
}

// ============================================================================
// SIGNAL ANALYSIS
// ============================================================================

/**
 * Capture I/Q samples from RX buffer
 */
static int capture_iq_samples(PlutoSDR *sdr, int16_t **i_samples, int16_t **q_samples)
{
    // Capture samples
    ssize_t nbytes = iio_buffer_refill(sdr->rxbuf);
    if (nbytes < 0) {
        fprintf(stderr, "Failed to refill RX buffer: %s\n", strerror(-nbytes));
        return -1;
    }

    int16_t *rx_data = (int16_t *)iio_buffer_start(sdr->rxbuf);

    // Allocate separate I and Q arrays
    *i_samples = malloc(BUFFER_SIZE * sizeof(int16_t));
    *q_samples = malloc(BUFFER_SIZE * sizeof(int16_t));

    if (!*i_samples || !*q_samples) {
        fprintf(stderr, "Failed to allocate I/Q arrays\n");
        return -1;
    }

    // De-interleave I and Q
    for (size_t i = 0; i < BUFFER_SIZE; i++) {
        (*i_samples)[i] = rx_data[2*i];
        (*q_samples)[i] = rx_data[2*i + 1];
    }

    return 0;
}

/**
 * Simple DFT to find peak frequency
 * (Using DFT instead of full FFT library for simplicity on embedded)
 */
static int find_peak_frequency(int16_t *i_samples, int16_t *q_samples,
                               size_t num_samples, double sample_rate,
                               ToneInfo *tone)
{
    double max_magnitude = 0.0;
    double best_freq = 0.0;
    int num_bins = 512;  // Check 512 frequency bins

    double freq_step = sample_rate / num_bins;

    // Search from -Fs/2 to +Fs/2
    for (int bin = -num_bins/2; bin < num_bins/2; bin++) {
        double test_freq = bin * freq_step;

        // Correlation with test frequency
        double corr_i = 0.0, corr_q = 0.0;

        for (size_t n = 0; n < num_samples; n++) {
            double t = (double)n / sample_rate;
            double phase = 2.0 * M_PI * test_freq * t;

            double ref_i = cos(phase);
            double ref_q = sin(phase);

            double sig_i = i_samples[n] / 2048.0;
            double sig_q = q_samples[n] / 2048.0;

            // Complex correlation
            corr_i += sig_i * ref_i + sig_q * ref_q;
            corr_q += sig_q * ref_i - sig_i * ref_q;
        }

        double magnitude = sqrt(corr_i * corr_i + corr_q * corr_q) / num_samples;

        if (magnitude > max_magnitude) {
            max_magnitude = magnitude;
            best_freq = test_freq;
        }
    }

    tone->frequency_hz = best_freq;
    tone->magnitude = max_magnitude;
    tone->detected = (max_magnitude > 0.1);

    return 0;
}

/**
 * Measure bandwidth using simple power-based method
 */
static int measure_bandwidth(int16_t *i_samples, int16_t *q_samples,
                             size_t num_samples, double sample_rate,
                             BandwidthMeasurement *bw)
{
    // Calculate power spectrum (simplified DFT approach)
    int num_bins = 256;
    double *power_spectrum = calloc(num_bins, sizeof(double));

    if (!power_spectrum) {
        return -1;
    }

    double freq_step = sample_rate / num_bins;

    for (int bin = 0; bin < num_bins; bin++) {
        double test_freq = (bin - num_bins/2) * freq_step;

        double corr_i = 0.0, corr_q = 0.0;

        for (size_t n = 0; n < num_samples; n++) {
            double t = (double)n / sample_rate;
            double phase = 2.0 * M_PI * test_freq * t;

            double ref_i = cos(phase);
            double ref_q = sin(phase);

            double sig_i = i_samples[n] / 2048.0;
            double sig_q = q_samples[n] / 2048.0;

            corr_i += sig_i * ref_i + sig_q * ref_q;
            corr_q += sig_q * ref_i - sig_i * ref_q;
        }

        power_spectrum[bin] = (corr_i * corr_i + corr_q * corr_q) / (num_samples * num_samples);
    }

    // Find peak
    int peak_bin = 0;
    double peak_power = 0.0;

    for (int bin = 0; bin < num_bins; bin++) {
        if (power_spectrum[bin] > peak_power) {
            peak_power = power_spectrum[bin];
            peak_bin = bin;
        }
    }

    bw->center_freq_hz = (peak_bin - num_bins/2) * freq_step;
    bw->peak_power_db = 10.0 * log10(peak_power + 1e-12);

    // Find 3 dB bandwidth
    double threshold = peak_power / 2.0;  // -3 dB = half power

    int lower_bin = peak_bin;
    while (lower_bin > 0 && power_spectrum[lower_bin] > threshold) {
        lower_bin--;
    }

    int upper_bin = peak_bin;
    while (upper_bin < num_bins - 1 && power_spectrum[upper_bin] > threshold) {
        upper_bin++;
    }

    bw->bandwidth_3db_hz = (upper_bin - lower_bin) * freq_step;

    // Calculate 99% occupied bandwidth
    double total_power = 0.0;
    for (int bin = 0; bin < num_bins; bin++) {
        total_power += power_spectrum[bin];
    }

    double target_power = 0.99 * total_power;
    double integrated_power = 0.0;
    int occupied_bins = 0;

    // Integrate from peak outward
    for (int offset = 0; offset < num_bins/2; offset++) {
        if (peak_bin - offset >= 0) {
            integrated_power += power_spectrum[peak_bin - offset];
            occupied_bins++;
        }
        if (peak_bin + offset < num_bins) {
            integrated_power += power_spectrum[peak_bin + offset];
            occupied_bins++;
        }

        if (integrated_power >= target_power) {
            break;
        }
    }

    bw->bandwidth_occupied_hz = occupied_bins * freq_step;

    free(power_spectrum);
    return 0;
}

// ============================================================================
// TEST FUNCTIONS
// ============================================================================

/**
 * Test 1: Proper Sampling (frequency within Nyquist limit)
 */
static int test_proper_sampling(PlutoSDR *sdr)
{
    printf("======================================================================\n");
    printf("TEST 1: PROPER SAMPLING (Within Nyquist Limit)\n");
    printf("======================================================================\n\n");

    double nyquist_freq = sdr->sample_rate / 2.0;
    double tone_freq = nyquist_freq * 0.25;  // 25% of Nyquist (well within limit)

    printf("Sample rate: %.3f MSPS\n", sdr->sample_rate / 1e6);
    printf("Nyquist frequency: %.3f MHz\n", nyquist_freq / 1e6);
    printf("Transmitting tone at: %.3f kHz (%.1f%% of Nyquist)\n\n",
           tone_freq / 1e3, 100.0 * tone_freq / nyquist_freq);

    // Generate and transmit tone
    if (generate_tone(sdr, tone_freq) < 0) {
        return -1;
    }

    usleep(100000);  // 100 ms

    // Capture and analyze
    int16_t *i_samples = NULL, *q_samples = NULL;
    if (capture_iq_samples(sdr, &i_samples, &q_samples) < 0) {
        return -1;
    }

    ToneInfo tone;
    find_peak_frequency(i_samples, q_samples, BUFFER_SIZE, sdr->sample_rate, &tone);

    printf("Analysis Results:\n");
    printf("  Expected frequency: %.3f kHz\n", tone_freq / 1e3);
    printf("  Measured frequency: %.3f kHz\n", tone.frequency_hz / 1e3);
    printf("  Magnitude:          %.4f\n", tone.magnitude);

    double error_hz = fabs(tone.frequency_hz - tone_freq);
    double error_pct = 100.0 * error_hz / tone_freq;

    printf("  Error:              %.3f kHz (%.2f%%)\n\n", error_hz / 1e3, error_pct);

    if (error_pct < 5.0) {
        printf("✓ PASS: Frequency correctly represented (no aliasing)\n");
    } else {
        printf("✗ FAIL: Large frequency error detected\n");
    }

    free(i_samples);
    free(q_samples);

    return 0;
}

/**
 * Test 2: Aliasing (frequency above Nyquist limit)
 */
static int test_aliasing(PlutoSDR *sdr)
{
    printf("\n======================================================================\n");
    printf("TEST 2: ALIASING (Frequency Above Nyquist Limit)\n");
    printf("======================================================================\n\n");

    double nyquist_freq = sdr->sample_rate / 2.0;
    double tone_freq = nyquist_freq * 0.75;  // 75% of Nyquist (ABOVE limit!)

    printf("Sample rate: %.3f MSPS\n", sdr->sample_rate / 1e6);
    printf("Nyquist frequency: ±%.3f MHz\n", nyquist_freq / 1e6);
    printf("Transmitting tone at: %.3f MHz (%.1f%% ABOVE Nyquist!)\n\n",
           tone_freq / 1e6, 100.0 * (tone_freq - nyquist_freq) / nyquist_freq);

    // Calculate expected alias
    double expected_alias = tone_freq - sdr->sample_rate;
    if (expected_alias < -nyquist_freq) {
        expected_alias = sdr->sample_rate + expected_alias;
    }

    printf("Expected alias frequency: %.3f kHz\n\n", expected_alias / 1e3);

    // Generate and transmit tone
    if (generate_tone(sdr, tone_freq) < 0) {
        return -1;
    }

    usleep(100000);  // 100 ms

    // Capture and analyze
    int16_t *i_samples = NULL, *q_samples = NULL;
    if (capture_iq_samples(sdr, &i_samples, &q_samples) < 0) {
        return -1;
    }

    ToneInfo tone;
    find_peak_frequency(i_samples, q_samples, BUFFER_SIZE, sdr->sample_rate, &tone);

    printf("Analysis Results:\n");
    printf("  Transmitted frequency: %.3f MHz\n", tone_freq / 1e6);
    printf("  Expected alias:        %.3f kHz\n", expected_alias / 1e3);
    printf("  Measured frequency:    %.3f kHz\n", tone.frequency_hz / 1e3);
    printf("  Magnitude:             %.4f\n", tone.magnitude);

    double alias_error = fabs(tone.frequency_hz - expected_alias);
    double alias_error_pct = 100.0 * alias_error / fabs(expected_alias);

    printf("  Alias error:           %.3f kHz (%.2f%%)\n\n", alias_error / 1e3, alias_error_pct);

    if (alias_error_pct < 10.0) {
        printf("✓ ALIASING CONFIRMED: Measured alias matches theory!\n");
        printf("  Original %.3f MHz → Aliased to %.3f kHz\n",
               tone_freq / 1e6, tone.frequency_hz / 1e3);
    } else {
        printf("✗ UNEXPECTED: Alias does not match theory\n");
    }

    free(i_samples);
    free(q_samples);

    return 0;
}

/**
 * Test 3: Bandwidth Measurement
 */
static int test_bandwidth_measurement(PlutoSDR *sdr)
{
    printf("\n======================================================================\n");
    printf("TEST 3: BANDWIDTH MEASUREMENT\n");
    printf("======================================================================\n\n");

    double tone_freq = 100000.0;  // 100 kHz offset

    printf("Transmitting tone at %.3f kHz...\n", tone_freq / 1e3);

    // Generate and transmit tone
    if (generate_tone(sdr, tone_freq) < 0) {
        return -1;
    }

    usleep(100000);  // 100 ms

    // Capture and analyze
    int16_t *i_samples = NULL, *q_samples = NULL;
    if (capture_iq_samples(sdr, &i_samples, &q_samples) < 0) {
        return -1;
    }

    BandwidthMeasurement bw;
    measure_bandwidth(i_samples, q_samples, BUFFER_SIZE, sdr->sample_rate, &bw);

    printf("\nBandwidth Measurement Results:\n");
    printf("  Center frequency:   %.3f kHz\n", bw.center_freq_hz / 1e3);
    printf("  Peak power:         %.1f dB\n", bw.peak_power_db);
    printf("  3 dB bandwidth:     %.3f kHz\n", bw.bandwidth_3db_hz / 1e3);
    printf("  Occupied bandwidth: %.3f kHz (99%% power)\n\n", bw.bandwidth_occupied_hz / 1e3);

    // A pure tone should have very narrow bandwidth
    if (bw.bandwidth_3db_hz < sdr->sample_rate / 50.0) {
        printf("✓ PASS: Measured narrow bandwidth (pure tone)\n");
    } else {
        printf("⚠ WARNING: Bandwidth larger than expected for pure tone\n");
    }

    free(i_samples);
    free(q_samples);

    return 0;
}

/**
 * Test 4: Nyquist Zone Mapping
 */
static int test_nyquist_zones(PlutoSDR *sdr)
{
    printf("\n======================================================================\n");
    printf("TEST 4: NYQUIST ZONE MAPPING\n");
    printf("======================================================================\n\n");

    double nyquist_freq = sdr->sample_rate / 2.0;

    printf("Sample rate: %.3f MSPS\n", sdr->sample_rate / 1e6);
    printf("Nyquist frequency: ±%.3f MHz\n", nyquist_freq / 1e6);
    printf("\nTesting frequencies across multiple Nyquist zones...\n\n");

    printf("%-20s %-20s %-20s %s\n", "Zone", "Input Freq", "Measured Freq", "Status");
    printf("%-20s %-20s %-20s %s\n", "----", "----------", "-------------", "------");

    // Test frequencies in different zones
    double test_freqs[] = {
        nyquist_freq * 0.25,   // Zone 1 (within Nyquist)
        nyquist_freq * 0.75,   // Zone 2 (aliased)
        nyquist_freq * 1.25,   // Zone 3 (aliased)
        nyquist_freq * 1.75    // Zone 4 (aliased)
    };

    for (int i = 0; i < 4; i++) {
        double tone_freq = test_freqs[i];

        // Generate and transmit
        generate_tone(sdr, tone_freq);
        usleep(100000);

        // Capture and analyze
        int16_t *i_samples = NULL, *q_samples = NULL;
        capture_iq_samples(sdr, &i_samples, &q_samples);

        ToneInfo tone;
        find_peak_frequency(i_samples, q_samples, BUFFER_SIZE, sdr->sample_rate, &tone);

        // Determine zone and aliasing status
        int zone = (int)(fabs(tone_freq) / nyquist_freq) + 1;
        bool aliased = (fabs(tone_freq) > nyquist_freq);

        printf("Zone %-15d %-20.3f %-20.3f %s\n",
               zone,
               tone_freq / 1e6,
               tone.frequency_hz / 1e6,
               aliased ? "ALIASED" : "OK");

        free(i_samples);
        free(q_samples);
    }

    printf("\n✓ Nyquist zone mapping complete\n");

    return 0;
}

// ============================================================================
// MAIN PROGRAM
// ============================================================================

int main(int argc, char **argv)
{
    printf("\n");
    printf("======================================================================\n");
    printf("LAB 2.1 - Method 3: Nyquist Sampling and Aliasing Tests\n");
    printf("Running on PlutoSDR ARM Cortex-A9\n");
    printf("======================================================================\n\n");

    PlutoSDR sdr = {0};
    int ret = 0;

    // Use default sample rate (can be changed for different tests)
    long long sample_rate = TEST_SAMPLE_RATE_1;  // 2 MSPS

    // Initialize hardware
    if (init_plutosdr(&sdr, sample_rate) < 0) {
        fprintf(stderr, "Failed to initialize PlutoSDR\n");
        return 1;
    }

    // Run all tests
    double total_start = get_time_seconds();

    // Test 1: Proper sampling
    ret = test_proper_sampling(&sdr);
    if (ret < 0) {
        fprintf(stderr, "Test 1 failed\n");
    }
    sleep(1);

    // Test 2: Aliasing
    ret = test_aliasing(&sdr);
    if (ret < 0) {
        fprintf(stderr, "Test 2 failed\n");
    }
    sleep(1);

    // Test 3: Bandwidth measurement
    ret = test_bandwidth_measurement(&sdr);
    if (ret < 0) {
        fprintf(stderr, "Test 3 failed\n");
    }
    sleep(1);

    // Test 4: Nyquist zones
    ret = test_nyquist_zones(&sdr);
    if (ret < 0) {
        fprintf(stderr, "Test 4 failed\n");
    }

    double total_duration = get_time_seconds() - total_start;

    // Summary
    printf("\n");
    printf("======================================================================\n");
    printf("ALL TESTS COMPLETE\n");
    printf("======================================================================\n");
    printf("Total Duration: %.2f seconds\n", total_duration);
    printf("Sample Rate:    %.3f MSPS\n", sdr.sample_rate / 1e6);
    printf("Nyquist Freq:   ±%.3f MHz\n", sdr.sample_rate / 2.0 / 1e6);
    printf("\nKey Takeaways:\n");
    printf("  1. Frequencies within Nyquist limit are correctly represented\n");
    printf("  2. Frequencies above Nyquist limit alias to lower frequencies\n");
    printf("  3. Alias frequency: f_alias = |f - n×Fs|\n");
    printf("  4. Cannot distinguish original from alias after sampling\n");
    printf("  5. Solution: Use anti-aliasing filter before ADC\n");
    printf("\n✓ Nyquist sampling and aliasing demonstration complete!\n\n");

    // Cleanup
    cleanup_plutosdr(&sdr);

    return 0;
}
```

---

## Part 6: Detailed Step-by-Step Compilation Guide

This section explains **exactly how** to compile the Nyquist sampling and aliasing application for PlutoSDR's ARM processor, with every step explained in detail.

### What is Cross-Compilation? (Concept Review)

**Problem**: PlutoSDR has an ARM Cortex-A9 processor, but your PC has an x86/x64 processor. You can't use your regular `gcc` compiler.

**Solution**: Use a **cross-compiler** that runs on x86 but produces ARM binaries.

**Analogy**: It's like writing instructions in English (x86 development environment) but translating them to Japanese (ARM machine code) for someone who only reads Japanese (PlutoSDR).

```
Your PC (x86/x64)                  PlutoSDR (ARM Cortex-A9)
     |                                  |
     v                                  v
┌──────────────┐                 ┌──────────────┐
│ Intel/AMD    │                 │ ARM CPU      │
│ x86-64 CPU   │                 │ (Cortex-A9)  │
└──────────────┘                 └──────────────┘
     |                                  |
     v                                  v
arm-linux-gnueabihf-gcc    →    lab2_1_hosted (ARM binary)
(Cross-compiler on x86)         (Runs on ARM only)
```

### Step-by-Step Compilation Process

#### **STEP 1: Prepare Your Workspace**

Create a dedicated directory for this lab:

```bash
mkdir -p ~/pluto_labs/lab2_1_method3
cd ~/pluto_labs/lab2_1_method3
```

**What this does**:
- Creates folder structure: `~/pluto_labs/lab2_1_method3/`
- `-p` flag creates parent directories if they don't exist
- `cd` changes to the new directory so all files go here

**Verification**:
```bash
pwd
# Should show: /home/your_username/pluto_labs/lab2_1_method3
```

#### **STEP 2: Create the C Source File**

Copy the complete C code from Part 5 above into a file:

```bash
nano lab2_1_method3_hosted.c
```

**What to do**:
1. Opens text editor (`nano`)
2. Paste the entire C program from Part 5 (lines 867-1662)
3. Save: `Ctrl+X`, then `Y`, then `Enter`

**Verification**:
```bash
wc -l lab2_1_method3_hosted.c
# Should show: ~796 lines

ls -lh lab2_1_method3_hosted.c
# Should show file size ~28-32 KB
```

**Quick check for syntax errors**:
```bash
head -n 20 lab2_1_method3_hosted.c
# Should see: /* LAB 2.1 - Method 3: Nyquist Sampling and Aliasing Hosted Application */
```

#### **STEP 3: Create Compilation Script**

Create the automated build script:

```bash
nano compile_lab2_1.sh
```

**Paste this content**:

```bash
#!/bin/bash
#
# Compile LAB 2.1 Method 3 for PlutoSDR ARM architecture
#

set -e  # Exit on any error

# Configuration
SOURCE_FILE="lab2_1_method3_hosted.c"
OUTPUT_FILE="lab2_1_hosted"
ARM_LIBS="/opt/arm-libs"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'  # No Color

echo -e "${GREEN}Compiling LAB 2.1 Method 3 for PlutoSDR${NC}"
echo "=================================================="

# Check if source file exists
if [ ! -f "$SOURCE_FILE" ]; then
    echo -e "${RED}Error: Source file '$SOURCE_FILE' not found${NC}"
    exit 1
fi

# Check if cross-compiler is installed
if ! command -v arm-linux-gnueabihf-gcc &> /dev/null; then
    echo -e "${RED}Error: ARM cross-compiler not found${NC}"
    echo "Please install: sudo apt-get install gcc-arm-linux-gnueabihf"
    exit 1
fi

# Check if libiio is available
if [ ! -f "$ARM_LIBS/lib/libiio.so" ]; then
    echo -e "${RED}Error: libiio for ARM not found${NC}"
    echo "Please build libiio for ARM first (see LAB 0 or LAB 1.1)"
    exit 1
fi

echo "Compiler:    arm-linux-gnueabihf-gcc"
echo "Source:      $SOURCE_FILE"
echo "Output:      $OUTPUT_FILE"
echo "libiio path: $ARM_LIBS"
echo ""

# Compilation flags
CC="arm-linux-gnueabihf-gcc"
CFLAGS="-Wall -Wextra -O2 -std=c99"
INCLUDES="-I${ARM_LIBS}/include"
LDFLAGS="-L${ARM_LIBS}/lib"
LIBS="-liio -lm -lpthread"

echo "Compiling..."
$CC $CFLAGS $INCLUDES -o $OUTPUT_FILE $SOURCE_FILE $LDFLAGS $LIBS

if [ $? -eq 0 ]; then
    echo -e "${GREEN}✓ Compilation successful${NC}"
else
    echo -e "${RED}✗ Compilation failed${NC}"
    exit 1
fi

# Strip debug symbols to reduce size
echo "Stripping debug symbols..."
arm-linux-gnueabihf-strip $OUTPUT_FILE

# Check file
echo ""
echo "Binary information:"
file $OUTPUT_FILE
ls -lh $OUTPUT_FILE

# Verify ARM architecture
if file $OUTPUT_FILE | grep -q "ARM"; then
    echo -e "${GREEN}✓ Binary is ARM architecture${NC}"
else
    echo -e "${RED}✗ Binary is NOT ARM architecture${NC}"
    exit 1
fi

echo ""
echo -e "${GREEN}Build complete!${NC}"
echo "Next step: Deploy to PlutoSDR using deploy_lab2_1.sh"
```

**Save and make executable**:
```bash
# Save: Ctrl+X, then Y, then Enter

# Make executable
chmod +x compile_lab2_1.sh
```

**What this does**:
- Creates automated build script
- `chmod +x` makes it executable (allows `./compile_lab2_1.sh`)
- Script checks all prerequisites before compiling
- Uses colors for clear output (green = success, red = error)

#### **STEP 4: Understand the Compilation Flags**

The compilation command uses several important flags. Let's understand **each one** and **why it matters**:

```bash
arm-linux-gnueabihf-gcc \
  -Wall                    # Show all warnings (helps catch bugs)
  -Wextra                  # Show extra warnings (more thorough checking)
  -O2                      # Optimize for speed (level 2 - balanced)
  -std=c99                 # Use C99 standard (modern C features)
  -I/opt/arm-libs/include  # Where to find header files (iio.h)
  -o lab2_1_hosted         # Output filename
  lab2_1_method3_hosted.c  # Input source file
  -L/opt/arm-libs/lib      # Where to find libraries (libiio.so)
  -liio                    # Link with libiio library (hardware access)
  -lm                      # Link with math library (sqrt, sin, cos, log10)
  -lpthread                # Link with pthread library (threading support)
```

**Detailed Flag Explanations**:

**1. `-Wall` (Warnings: All)**
```c
// Catches common errors like:
int unused_variable;           // Warning: unused variable
if (x = 5)                     // Warning: assignment in condition (should be ==)
printf("%d", y);               // Warning: 'y' may be uninitialized
```
**Why important**: Catches 80% of common bugs at compile time!

**2. `-Wextra` (Warnings: Extra)**
```c
// Catches additional issues like:
int compare(int a, int b) {
    if (a > b) return 1;
    if (a < b) return -1;
    // Warning: control reaches end of non-void function
}
```
**Why important**: More thorough checking, finds edge cases

**3. `-O2` (Optimization level 2)**
```
Optimization levels:
  -O0: No optimization (default, fastest compile, slowest execution)
  -O1: Basic optimization (moderate compile time, faster execution)
  -O2: Moderate optimization (balanced - RECOMMENDED)
  -O3: Aggressive optimization (slower compile, fastest execution, larger binary)
  -Os: Optimize for size (smallest binary)

For PlutoSDR: -O2 is best balance
  - Makes code ~2-3× faster than -O0
  - Doesn't increase binary size too much
  - Good for embedded ARM processors
```

**Example of -O2 optimization**:
```c
// Original code:
for (int i = 0; i < 1000; i++) {
    result += array[i];
}

// After -O2 optimization:
// - Loop unrolling (process 4 elements per iteration)
// - SIMD instructions (ARM NEON if available)
// - Register allocation (keep variables in CPU registers)
// Result: ~3× faster execution
```

**4. `-std=c99` (C99 Standard)**

Enables modern C features used in our code:
```c
// C99 features we use:
// - Inline variable declarations
for (int i = 0; i < 10; i++) { }  // 'int i' declared in loop

// - // comments (not just /* */ comments)
// - stdbool.h for 'bool' type
#include <stdbool.h>
bool detected = true;

// - stdint.h for exact-width integers
#include <stdint.h>
int16_t sample;  // Exactly 16-bit signed integer

// - Variable-length arrays (VLA)
int n = 10;
double array[n];  // Array size determined at runtime
```

**5. `-I/opt/arm-libs/include` (Include directory)**
```c
// Tells compiler where to find header files:
#include <iio.h>  // Looks in: /opt/arm-libs/include/iio.h

// Without -I flag:
// Error: iio.h: No such file or directory
```

**6. `-o lab2_1_hosted` (Output filename)**
```bash
# -o specifies output binary name
# Without -o: creates 'a.out' (default)
# With -o lab2_1_hosted: creates 'lab2_1_hosted'
```

**7. `-L/opt/arm-libs/lib` (Library directory)**
```bash
# Tells linker where to find libraries:
# /opt/arm-libs/lib/libiio.so     ← ARM version
# /usr/lib/libiio.so              ← x86 version (WRONG for PlutoSDR!)

# This is the ARM-compiled version, not x86!
```

**8. `-liio` (Link with libiio)**
```c
// Provides these functions:
iio_create_local_context();
iio_context_find_device();
iio_device_create_buffer();
iio_buffer_refill();
iio_buffer_push();

// Essential for PlutoSDR hardware access
// Without -liio:
// Error: undefined reference to `iio_create_local_context'
```

**9. `-lm` (Link with math library)**
```c
// Provides these functions we use:
sqrt()     // Square root
sin()      // Sine
cos()      // Cosine
atan2()    // Arc tangent (2 arguments)
log10()    // Base-10 logarithm
fabs()     // Absolute value (floating-point)

// Used for I/Q analysis calculations
// Without -lm:
// Error: undefined reference to `sqrt'
```

**10. `-lpthread` (Link with pthread library)**
```c
// Provides POSIX threading support
// Even if we don't explicitly use threads,
// libiio requires it internally

// Without -lpthread:
// Error: undefined reference to `pthread_create'
```

#### **STEP 5: Run the Compilation**

Execute the build script:

```bash
./compile_lab2_1.sh
```

**Expected output**:
```
Compiling LAB 2.1 Method 3 for PlutoSDR
==================================================
Compiler:    arm-linux-gnueabihf-gcc
Source:      lab2_1_method3_hosted.c
Output:      lab2_1_hosted
libiio path: /opt/arm-libs

Compiling...
✓ Compilation successful
Stripping debug symbols...

Binary information:
lab2_1_hosted: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV),
dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 3.2.0,
BuildID[sha1]=f8a3c2d1e9b7..., stripped

-rwxr-xr-x 1 user user 32K Nov 27 15:45 lab2_1_hosted

✓ Binary is ARM architecture

Build complete!
Next step: Deploy to PlutoSDR using deploy_lab2_1.sh
```

**What just happened** (step by step):

1. **Preprocessing**:
   - Expands `#include` directives
   - Processes `#define` macros
   - Includes iio.h, stdio.h, math.h, etc.

2. **Compilation**:
   - Parses C code syntax
   - Type checking (e.g., `int16_t *` vs `double *`)
   - Generates ARM assembly instructions
   - ~796 lines of C → ~5000 lines of ARM assembly

3. **Optimization** (-O2):
   - Loop unrolling for DFT calculations
   - Register allocation (ARM has 16 general-purpose registers)
   - Instruction scheduling (reorder for ARM pipeline efficiency)
   - Dead code elimination (removes unused functions)

4. **Linking**:
   - Combines object code with libraries
   - Resolves function calls: `sqrt()` → address in libm.so
   - Creates final ELF binary (~32 KB)

5. **Stripping**:
   - Removes debug symbols (function names, line numbers)
   - Reduces binary size: 45 KB → 32 KB (~30% smaller)
   - Debug info not needed on PlutoSDR

#### **STEP 6: Verify the Binary**

**Check architecture**:
```bash
file lab2_1_hosted
```

**Must show**:
```
lab2_1_hosted: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV),
dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 3.2.0,
BuildID[sha1]=..., stripped
```

**Key indicators**:
- ✓ `ARM` - Correct architecture
- ✓ `32-bit` - ARM Cortex-A9 is 32-bit
- ✓ `EABI5` - ARM Embedded ABI version 5
- ✓ `dynamically linked` - Uses shared libraries (.so files)
- ✓ `stripped` - Debug symbols removed

**❌ If it shows `x86-64` or `x86`**: You used the wrong compiler! Must use `arm-linux-gnueabihf-gcc`, not `gcc`.

**Check dependencies**:
```bash
arm-linux-gnueabihf-readelf -d lab2_1_hosted | grep NEEDED
```

**Should show**:
```
 0x00000001 (NEEDED)   Shared library: [libiio.so.0]
 0x00000001 (NEEDED)   Shared library: [libm.so.6]
 0x00000001 (NEEDED)   Shared library: [libpthread.so.0]
 0x00000001 (NEEDED)   Shared library: [libc.so.6]
```

**What this means**:
- **libiio.so.0**: Must be deployed to PlutoSDR (custom library)
- **libm.so.6**: Already on PlutoSDR (math library)
- **libpthread.so.0**: Already on PlutoSDR (threading)
- **libc.so.6**: Already on PlutoSDR (standard C library)

**Check binary size**:
```bash
ls -lh lab2_1_hosted
```

**Expected**: 28-35 KB (depending on optimization)

**Size comparison**:
```
Unoptimized (-O0):         45 KB
Optimized (-O2):           32 KB  ← What we built
Optimized + stripped:      32 KB  ← After stripping
Aggressive (-O3):          38 KB  (larger due to inlining)
Size-optimized (-Os):      26 KB  (smallest, but slower)
```

### Common Compilation Errors and Solutions

#### **Error 1: Cross-compiler not found**
```
bash: arm-linux-gnueabihf-gcc: command not found
```

**Solution**: Install ARM cross-compiler:
```bash
sudo apt-get update
sudo apt-get install gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf
```

**Verification**:
```bash
arm-linux-gnueabihf-gcc --version
# Should show: arm-linux-gnueabihf-gcc (Ubuntu/Linaro ...) X.X.X
```

#### **Error 2: iio.h not found**
```
lab2_1_method3_hosted.c:11:10: fatal error: iio.h: No such file or directory
   11 | #include <iio.h>
      |          ^~~~~~~
compilation terminated.
```

**Solution**: Build libiio for ARM (see LAB 0 or LAB 1.1 Method 3):
```bash
# Clone libiio
git clone https://github.com/analogdevicesinc/libiio.git
cd libiio
mkdir build-arm && cd build-arm

# Configure for ARM cross-compilation
cmake .. \
  -DCMAKE_TOOLCHAIN_FILE=../cmake/arm-linux-gnueabihf.cmake \
  -DCMAKE_INSTALL_PREFIX=/opt/arm-libs \
  -DWITH_EXAMPLES=OFF

# Build and install
make -j4
sudo make install
```

**Verification**:
```bash
ls -lh /opt/arm-libs/include/iio.h
ls -lh /opt/arm-libs/lib/libiio.so.0
```

#### **Error 3: Undefined reference to math functions**
```
/usr/bin/arm-linux-gnueabihf-ld: /tmp/ccXXXXXX.o: undefined reference to `sqrt'
/usr/bin/arm-linux-gnueabihf-ld: /tmp/ccXXXXXX.o: undefined reference to `sin'
/usr/bin/arm-linux-gnueabihf-ld: /tmp/ccXXXXXX.o: undefined reference to `cos'
```

**Solution**: Add `-lm` flag to link with math library. Our compile script already includes this.

**Why this happens**: Math functions are in a separate library on Linux (historical reasons).

#### **Error 4: Binary won't run on PlutoSDR**
```
root@pluto:~# ./lab2_1_hosted
-bash: ./lab2_1_hosted: cannot execute binary file: Exec format error
```

**Cause**: You compiled for x86 instead of ARM.

**Diagnosis**:
```bash
# On your PC:
file lab2_1_hosted
# Shows: x86-64  ← WRONG!

# Should show: ARM  ← CORRECT
```

**Solution**: Verify you're using the ARM cross-compiler:
```bash
which arm-linux-gnueabihf-gcc
# Should show: /usr/bin/arm-linux-gnueabihf-gcc

# NOT: /usr/bin/gcc (this is x86 compiler!)
```

Re-compile with correct cross-compiler using `./compile_lab2_1.sh`

#### **Error 5: Permission denied when running script**
```
bash: ./compile_lab2_1.sh: Permission denied
```

**Solution**: Make script executable:
```bash
chmod +x compile_lab2_1.sh
```

#### **Error 6: Warnings about unused variables**
```
lab2_1_method3_hosted.c:945:9: warning: unused variable 'zone_number' [-Wunused-variable]
  945 |     int zone_number;
      |         ^~~~~~~~~~~
```

**Solution**: These are just warnings, not errors. The binary will still work.

**To fix**: Remove or comment out unused variables:
```c
// int zone_number;  // Not needed for this test
```

Or use `(void)` to mark as intentionally unused:
```c
int zone_number;
(void)zone_number;  // Explicitly mark as unused
```

### Compilation Complete!

At this point you should have:
- ✓ `lab2_1_method3_hosted.c` (source code, ~796 lines)
- ✓ `lab2_1_hosted` (ARM binary, ~32 KB)
- ✓ `compile_lab2_1.sh` (build script)

**File listing**:
```bash
ls -lh
# Should show:
# -rw-r--r-- 1 user user  28K lab2_1_method3_hosted.c
# -rwxr-xr-x 1 user user  32K lab2_1_hosted
# -rwxr-xr-x 1 user user 1.5K compile_lab2_1.sh
```

**Verification checklist**:
- [ ] Binary is ARM architecture (`file` command shows "ARM")
- [ ] Binary is dynamically linked (needs libiio.so)
- [ ] Binary size is 28-35 KB
- [ ] Compilation script is executable
- [ ] No compilation errors or warnings

**Next step**: Deploy to PlutoSDR (Part 7)

---