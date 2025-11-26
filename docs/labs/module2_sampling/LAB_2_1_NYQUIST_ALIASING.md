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

Due to length constraints, I'll commit what we have and continue with the rest of LAB 2.1 plus additional labs. This gives you a complete working demonstration of aliasing!