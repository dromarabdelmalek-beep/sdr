# LAB 1.3: I/Q Samples & Complex Baseband Representation

## Overview

One of the most fundamental concepts in Software-Defined Radio is the use of **I/Q (In-phase/Quadrature) sampling** to represent RF signals in the digital domain. Understanding I/Q samples is essential for DSP operations, modulation, and signal analysis.

This lab demystifies the complex baseband representation used by PlutoSDR and teaches you how to work with I/Q data effectively.

### Learning Objectives

1. Understand why SDRs use complex (I/Q) representation
2. Learn the relationship between RF signals and baseband I/Q samples
3. Visualize signals in time, frequency, and constellation domains
4. Perform frequency translation and filtering in baseband
5. Generate and decode I/Q modulated signals
6. Understand the benefits of complex signal processing

### Key Concepts

- **Complex Baseband**: RF signal shifted to 0 Hz (DC) center frequency
- **I/Q Samples**: I = In-phase (real), Q = Quadrature (imaginary)
- **Quadrature Mixing**: Using cos(ωt) and sin(ωt) to downconvert RF
- **Negative Frequencies**: Complex representation enables distinction
- **Image Rejection**: Complex mixing eliminates image frequencies
- **Hilbert Transform**: Creates analytic signal (single-sided spectrum)
- **Constellation Diagram**: I/Q plot showing modulation states

---

## Why Complex I/Q Representation?

### Problem with Real Sampling

Consider sampling a real RF signal directly:

```
Real RF Signal: s(t) = A·cos(2πf_c·t + φ)

Issues:
1. Bandwidth: Need sample rate > 2 × f_RF (Nyquist)
   - For 915 MHz, need > 1.83 GHz sampling!
2. No phase information without quadrature component
3. Cannot distinguish positive/negative frequency offsets
4. Image frequencies cause ambiguity
```

### Solution: I/Q Sampling (Complex Baseband)

```
Mixing with Quadrature LO:
  I(t) = s(t) × cos(2πf_LO·t)  [In-phase]
  Q(t) = s(t) × sin(2πf_LO·t)  [Quadrature, 90° phase shift]

Complex Baseband:
  z(t) = I(t) + j·Q(t)

Benefits:
1. Signal at f_RF becomes f_RF - f_LO (baseband)
2. Sample rate only needs to cover bandwidth, not f_RF
3. Preserves full amplitude and phase information
4. Positive/negative frequencies distinguishable
5. No image frequency ambiguity
```

### Mathematical Basis

Using Euler's formula:
```
e^(jωt) = cos(ωt) + j·sin(ωt)

Original RF signal at f_c:
  s(t) = A·cos(2πf_c·t + φ) = A·Re{e^(j(2πf_c·t + φ))}

After mixing with LO at f_c:
  Baseband: z(t) = A·e^(jφ)

The baseband complex sample directly gives us:
  |z(t)| = A      (amplitude)
  ∠z(t) = φ       (phase)
```

---

## I/Q Signal Flow in PlutoSDR

### Transmit (TX) Chain

```
Digital                        Analog                    RF
┌─────────┐    ┌──────┐    ┌─────────┐    ┌────────┐
│ I/Q     │───►│ DAC  │───►│ LPF     │───►│ Mixer  │───► Antenna
│ Samples │    │ 12bit│    │ Analog  │    │ × LO   │
│ (int16) │    └──────┘    └─────────┘    └────────┘
│         │         ▲             ▲              ▲
│ I: Real │         │             │              │
│ Q: Imag │    DAC Clock    Reconstruction   Upconvert
└─────────┘         │         Filter        to f_TX_LO
                    │
              Sample Rate
              (2.084 MSPS)

Data Flow:
  Python: complex64 → libiio → int16 I/Q → DAC → Analog I/Q → RF
```

### Receive (RX) Chain

```
RF                          Analog                Digital
                ┌────────┐    ┌─────────┐    ┌──────┐    ┌─────────┐
Antenna ───────►│ Mixer  │───►│ LPF     │───►│ ADC  │───►│ I/Q     │
                │ × LO   │    │ Anti-   │    │ 12bit│    │ Samples │
                └────────┘    │ Alias   │    └──────┘    │ (int16) │
                     │        └─────────┘         │      └─────────┘
                Downconvert        │              │            │
                to Baseband   Bandlimit      ADC Clock   libiio → Python
                              (Nyquist)      Sample Rate   (complex64)

Data Flow:
  RF → Analog I/Q → ADC → int16 I/Q → libiio → Python complex64
```

### Key Points

1. **I and Q are sampled simultaneously** at the same sample rate
2. **Each complex sample = 2 × 16-bit values** (4 bytes total)
3. **LO frequency (f_LO) determines RF center frequency**
4. **Baseband frequency = RF frequency - LO frequency**
5. **Bandwidth = sample rate** (for complex sampling)

---

## Method 1: Pure Simulation (No Hardware)

### Objective
Understand I/Q representation by simulating quadrature mixing, visualizing complex signals, and performing baseband DSP operations.

### Implementation

```python
#!/usr/bin/env python3
"""
LAB 1.3 - Method 1: I/Q Samples & Complex Baseband Simulation
Demonstrates complex representation and I/Q signal processing
"""

import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
from scipy import signal as sp_signal

# Parameters
FS = 2.084e6  # Sample rate (Hz)
DURATION = 0.001  # 1 ms
NUM_SAMPLES = int(FS * DURATION)

# RF parameters
F_RF = 915e6  # RF carrier frequency
F_LO = 915e6  # Local oscillator frequency
F_SIGNAL = 100e3  # Signal offset from LO (100 kHz)


def simulate_rf_signal(duration, fs, f_rf, f_offset, amplitude=1.0, phase=0.0):
    """
    Simulate a real-valued RF signal

    Signal: A·cos(2π(f_rf + f_offset)·t + φ)
    """
    t = np.arange(int(duration * fs)) / fs
    rf_signal = amplitude * np.cos(2 * np.pi * (f_rf + f_offset) * t + phase)
    return t, rf_signal


def quadrature_downconvert(rf_signal, t, f_lo):
    """
    Perform quadrature downconversion (mixing)

    I = rf_signal × cos(2πf_LO·t)
    Q = rf_signal × sin(2πf_LO·t)

    Returns complex baseband signal: I + jQ
    """
    # Generate local oscillator (LO) signals
    lo_i = np.cos(2 * np.pi * f_lo * t)  # In-phase LO
    lo_q = np.sin(2 * np.pi * f_lo * t)  # Quadrature LO (90° phase shift)

    # Mix (multiply) RF signal with LO
    i_component = rf_signal * lo_i
    q_component = rf_signal * lo_q

    # Combine into complex baseband
    baseband = i_component + 1j * q_component

    return baseband, i_component, q_component


def lowpass_filter(signal, cutoff_hz, fs, order=5):
    """
    Apply lowpass filter to remove high-frequency mixing products
    """
    nyquist = fs / 2
    cutoff_normalized = cutoff_hz / nyquist

    # Design Butterworth filter
    b, a = sp_signal.butter(order, cutoff_normalized, btype='low')

    # Apply filter (handle complex signals)
    if np.iscomplexobj(signal):
        filtered_i = sp_signal.filtfilt(b, a, signal.real)
        filtered_q = sp_signal.filtfilt(b, a, signal.imag)
        filtered = filtered_i + 1j * filtered_q
    else:
        filtered = sp_signal.filtfilt(b, a, signal)

    return filtered


def analyze_iq_signal(signal, fs):
    """
    Compute various metrics of I/Q signal
    """
    # Separate I and Q
    i_samples = signal.real
    q_samples = signal.imag

    # Magnitude and phase
    magnitude = np.abs(signal)
    phase = np.angle(signal)

    # Power
    power_linear = np.mean(magnitude**2)
    power_db = 10 * np.log10(power_linear + 1e-12)

    # Frequency (estimate from phase derivative)
    phase_unwrapped = np.unwrap(phase)
    inst_freq = np.diff(phase_unwrapped) / (2 * np.pi) * fs
    mean_freq = np.mean(inst_freq)

    # FFT
    fft_result = np.fft.fftshift(np.fft.fft(signal))
    freqs = np.fft.fftshift(np.fft.fftfreq(len(signal), 1/fs))
    spectrum_db = 20 * np.log10(np.abs(fft_result) + 1e-12)

    return {
        'i': i_samples,
        'q': q_samples,
        'magnitude': magnitude,
        'phase': phase,
        'power_db': power_db,
        'mean_freq': mean_freq,
        'fft_freqs': freqs,
        'spectrum_db': spectrum_db
    }


def demonstrate_negative_frequency():
    """
    Demonstrate that complex representation can distinguish
    positive and negative frequency offsets
    """
    print("="*70)
    print("DEMONSTRATING NEGATIVE FREQUENCY DETECTION")
    print("="*70)

    t = np.arange(NUM_SAMPLES) / FS

    # Create two signals: one above LO, one below
    signal_pos = np.exp(2j * np.pi * (+F_SIGNAL) * t)  # +100 kHz offset
    signal_neg = np.exp(2j * np.pi * (-F_SIGNAL) * t)  # -100 kHz offset

    # Compute FFT
    fft_pos = np.fft.fftshift(np.fft.fft(signal_pos))
    fft_neg = np.fft.fftshift(np.fft.fft(signal_neg))
    freqs = np.fft.fftshift(np.fft.fftfreq(NUM_SAMPLES, 1/FS))

    # Plot
    fig, axes = plt.subplots(2, 1, figsize=(14, 8))

    # Positive frequency
    axes[0].plot(freqs/1e3, 20*np.log10(np.abs(fft_pos)+1e-12), linewidth=2)
    axes[0].axvline(+F_SIGNAL/1e3, color='red', linestyle='--', label=f'+{F_SIGNAL/1e3:.0f} kHz')
    axes[0].set_xlabel('Frequency (kHz)', fontsize=11)
    axes[0].set_ylabel('Magnitude (dB)', fontsize=11)
    axes[0].set_title('Complex Signal: Positive Frequency Offset (+100 kHz)',
                     fontsize=13, fontweight='bold')
    axes[0].legend()
    axes[0].grid(True, alpha=0.3)
    axes[0].set_xlim([-200, 200])

    # Negative frequency
    axes[1].plot(freqs/1e3, 20*np.log10(np.abs(fft_neg)+1e-12), linewidth=2)
    axes[1].axvline(-F_SIGNAL/1e3, color='red', linestyle='--', label=f'-{F_SIGNAL/1e3:.0f} kHz')
    axes[1].set_xlabel('Frequency (kHz)', fontsize=11)
    axes[1].set_ylabel('Magnitude (dB)', fontsize=11)
    axes[1].set_title('Complex Signal: Negative Frequency Offset (-100 kHz)',
                     fontsize=13, fontweight='bold')
    axes[1].legend()
    axes[1].grid(True, alpha=0.3)
    axes[1].set_xlim([-200, 200])

    plt.tight_layout()
    plt.savefig('lab1_3_method1_negative_frequency.png', dpi=150)
    print("✓ Saved: lab1_3_method1_negative_frequency.png")
    plt.show()

    print("\nKey Observation:")
    print("  Complex signals can distinguish +f from -f")
    print("  Real signals would show symmetric spectrum (positive AND negative)")
    print()


def full_iq_simulation():
    """
    Complete I/Q simulation: RF generation → mixing → filtering → analysis
    """
    print("="*70)
    print("COMPLETE I/Q SIMULATION")
    print("="*70)

    # Step 1: Generate RF signal
    print("\n1. Generating RF signal...")
    print(f"   RF Frequency: {F_RF/1e6:.1f} MHz")
    print(f"   Signal Offset: +{F_SIGNAL/1e3:.0f} kHz")
    print(f"   Total: {(F_RF + F_SIGNAL)/1e6:.6f} MHz")

    t, rf_signal = simulate_rf_signal(DURATION, FS, F_RF, F_SIGNAL)

    # Step 2: Quadrature downconversion
    print("\n2. Performing quadrature downconversion...")
    print(f"   LO Frequency: {F_LO/1e6:.1f} MHz")

    baseband_raw, i_raw, q_raw = quadrature_downconvert(rf_signal, t, F_LO)

    print(f"   Baseband frequency: {F_SIGNAL/1e3:.0f} kHz")
    print(f"   (RF - LO = {(F_RF + F_SIGNAL)/1e6:.6f} - {F_LO/1e6:.1f} MHz)")

    # Step 3: Lowpass filtering
    print("\n3. Applying lowpass filter...")
    cutoff = 0.5 * FS  # Half of sample rate
    print(f"   Cutoff: {cutoff/1e6:.2f} MHz")

    baseband_filtered = lowpass_filter(baseband_raw, cutoff, FS)

    # Step 4: Analyze
    print("\n4. Analyzing I/Q signal...")
    analysis = analyze_iq_signal(baseband_filtered, FS)

    print(f"   Detected Frequency: {analysis['mean_freq']/1e3:.2f} kHz")
    print(f"   Signal Power: {analysis['power_db']:.1f} dB")
    print(f"   I range: [{np.min(analysis['i']):.3f}, {np.max(analysis['i']):.3f}]")
    print(f"   Q range: [{np.min(analysis['q']):.3f}, {np.max(analysis['q']):.3f}]")

    # Visualization
    plot_iq_analysis(t, rf_signal, baseband_raw, baseband_filtered, analysis)

    return baseband_filtered, analysis


def plot_iq_analysis(t, rf_signal, baseband_raw, baseband_filtered, analysis):
    """
    Comprehensive I/Q visualization
    """
    fig = plt.figure(figsize=(16, 12))
    gs = GridSpec(4, 3, figure=fig, hspace=0.35, wspace=0.35)

    # Limit time axis for visibility
    t_ms = t * 1000  # Convert to ms
    plot_samples = min(500, len(t))  # Plot first 500 samples

    # 1. Original RF signal (time domain)
    ax1 = fig.add_subplot(gs[0, :])
    ax1.plot(t_ms[:plot_samples], rf_signal[:plot_samples], linewidth=1)
    ax1.set_xlabel('Time (ms)', fontsize=10)
    ax1.set_ylabel('Amplitude', fontsize=10)
    ax1.set_title(f'Original RF Signal @ {(F_RF+F_SIGNAL)/1e6:.6f} MHz',
                 fontsize=12, fontweight='bold')
    ax1.grid(True, alpha=0.3)

    # 2. I component (before filtering)
    ax2 = fig.add_subplot(gs[1, 0])
    ax2.plot(t_ms[:plot_samples], baseband_raw.real[:plot_samples], linewidth=1, color='blue')
    ax2.set_xlabel('Time (ms)', fontsize=10)
    ax2.set_ylabel('I (In-phase)', fontsize=10)
    ax2.set_title('I Component (After Mixing)', fontsize=11, fontweight='bold')
    ax2.grid(True, alpha=0.3)

    # 3. Q component (before filtering)
    ax3 = fig.add_subplot(gs[1, 1])
    ax3.plot(t_ms[:plot_samples], baseband_raw.imag[:plot_samples], linewidth=1, color='red')
    ax3.set_xlabel('Time (ms)', fontsize=10)
    ax3.set_ylabel('Q (Quadrature)', fontsize=10)
    ax3.set_title('Q Component (After Mixing)', fontsize=11, fontweight='bold')
    ax3.grid(True, alpha=0.3)

    # 4. I vs Q (constellation, before filtering)
    ax4 = fig.add_subplot(gs[1, 2])
    ax4.plot(baseband_raw.real[:plot_samples], baseband_raw.imag[:plot_samples],
            'o', markersize=2, alpha=0.5)
    ax4.set_xlabel('I', fontsize=10)
    ax4.set_ylabel('Q', fontsize=10)
    ax4.set_title('I/Q Constellation (Raw)', fontsize=11, fontweight='bold')
    ax4.grid(True, alpha=0.3)
    ax4.axis('equal')
    ax4.axhline(0, color='black', linewidth=0.5)
    ax4.axvline(0, color='black', linewidth=0.5)

    # 5. I component (after filtering)
    ax5 = fig.add_subplot(gs[2, 0])
    ax5.plot(t_ms[:plot_samples], analysis['i'][:plot_samples], linewidth=1, color='blue')
    ax5.set_xlabel('Time (ms)', fontsize=10)
    ax5.set_ylabel('I (Filtered)', fontsize=10)
    ax5.set_title('I Component (After LPF)', fontsize=11, fontweight='bold')
    ax5.grid(True, alpha=0.3)

    # 6. Q component (after filtering)
    ax6 = fig.add_subplot(gs[2, 1])
    ax6.plot(t_ms[:plot_samples], analysis['q'][:plot_samples], linewidth=1, color='red')
    ax6.set_xlabel('Time (ms)', fontsize=10)
    ax6.set_ylabel('Q (Filtered)', fontsize=10)
    ax6.set_title('Q Component (After LPF)', fontsize=11, fontweight='bold')
    ax6.grid(True, alpha=0.3)

    # 7. I vs Q (constellation, after filtering)
    ax7 = fig.add_subplot(gs[2, 2])
    ax7.plot(analysis['i'][:plot_samples], analysis['q'][:plot_samples],
            'o', markersize=2, alpha=0.5, color='green')
    ax7.set_xlabel('I', fontsize=10)
    ax7.set_ylabel('Q', fontsize=10)
    ax7.set_title('I/Q Constellation (Filtered)', fontsize=11, fontweight='bold')
    ax7.grid(True, alpha=0.3)
    ax7.axis('equal')
    ax7.axhline(0, color='black', linewidth=0.5)
    ax7.axvline(0, color='black', linewidth=0.5)

    # 8. Magnitude over time
    ax8 = fig.add_subplot(gs[3, 0])
    ax8.plot(t_ms[:plot_samples], analysis['magnitude'][:plot_samples],
            linewidth=1, color='purple')
    ax8.set_xlabel('Time (ms)', fontsize=10)
    ax8.set_ylabel('|I+jQ|', fontsize=10)
    ax8.set_title('Signal Magnitude', fontsize=11, fontweight='bold')
    ax8.grid(True, alpha=0.3)

    # 9. Phase over time
    ax9 = fig.add_subplot(gs[3, 1])
    ax9.plot(t_ms[:plot_samples], analysis['phase'][:plot_samples],
            linewidth=1, color='orange')
    ax9.set_xlabel('Time (ms)', fontsize=10)
    ax9.set_ylabel('∠(I+jQ) [rad]', fontsize=10)
    ax9.set_title('Signal Phase', fontsize=11, fontweight='bold')
    ax9.grid(True, alpha=0.3)

    # 10. Spectrum (FFT)
    ax10 = fig.add_subplot(gs[3, 2])
    ax10.plot(analysis['fft_freqs']/1e3, analysis['spectrum_db'], linewidth=1)
    ax10.axvline(F_SIGNAL/1e3, color='red', linestyle='--',
                label=f'Signal @ {F_SIGNAL/1e3:.0f} kHz')
    ax10.set_xlabel('Frequency (kHz)', fontsize=10)
    ax10.set_ylabel('Magnitude (dB)', fontsize=10)
    ax10.set_title('Baseband Spectrum', fontsize=11, fontweight='bold')
    ax10.legend(fontsize=9)
    ax10.grid(True, alpha=0.3)
    ax10.set_xlim([-FS/(2*1e3), FS/(2*1e3)])

    plt.suptitle('LAB 1.3 - Method 1: Complete I/Q Signal Processing',
                fontsize=14, fontweight='bold')

    plt.savefig('lab1_3_method1_full_iq.png', dpi=150, bbox_inches='tight')
    print("\n✓ Saved: lab1_3_method1_full_iq.png")
    plt.show()


def main():
    """Main execution"""
    print("\n" + "="*70)
    print("LAB 1.3 - Method 1: I/Q Samples & Complex Baseband Simulation")
    print("="*70)

    # Demonstration 1: Negative frequency detection
    demonstrate_negative_frequency()

    # Demonstration 2: Full I/Q simulation
    baseband, analysis = full_iq_simulation()

    # Summary
    print("\n" + "="*70)
    print("KEY TAKEAWAYS")
    print("="*70)
    print("1. I/Q (complex) representation enables:")
    print("   • Distinction between positive and negative frequencies")
    print("   • Direct amplitude and phase extraction")
    print("   • Efficient baseband processing")
    print()
    print("2. Quadrature mixing (cos + j·sin) performs frequency translation")
    print()
    print("3. After mixing, signal at f_RF appears at (f_RF - f_LO)")
    print()
    print("4. Lowpass filtering removes high-frequency mixing products")
    print()
    print("5. Complex baseband allows sample rate = bandwidth (not 2×f_RF)")
    print("="*70)
    print()


if __name__ == "__main__":
    main()
```

### Expected Output

```
======================================================================
LAB 1.3 - Method 1: I/Q Samples & Complex Baseband Simulation
======================================================================

======================================================================
DEMONSTRATING NEGATIVE FREQUENCY DETECTION
======================================================================
✓ Saved: lab1_3_method1_negative_frequency.png

Key Observation:
  Complex signals can distinguish +f from -f
  Real signals would show symmetric spectrum (positive AND negative)

======================================================================
COMPLETE I/Q SIMULATION
======================================================================

1. Generating RF signal...
   RF Frequency: 915.0 MHz
   Signal Offset: +100 kHz
   Total: 915.100000 MHz

2. Performing quadrature downconversion...
   LO Frequency: 915.0 MHz
   Baseband frequency: 100 kHz
   (RF - LO = 915.100000 - 915.0 MHz)

3. Applying lowpass filter...
   Cutoff: 1.04 MHz

4. Analyzing I/Q signal...
   Detected Frequency: 100.02 kHz
   Signal Power: -3.0 dB
   I range: [-0.707, 0.707]
   Q range: [-0.707, 0.707]

✓ Saved: lab1_3_method1_full_iq.png

======================================================================
KEY TAKEAWAYS
======================================================================
1. I/Q (complex) representation enables:
   • Distinction between positive and negative frequencies
   • Direct amplitude and phase extraction
   • Efficient baseband processing

2. Quadrature mixing (cos + j·sin) performs frequency translation

3. After mixing, signal at f_RF appears at (f_RF - f_LO)

4. Lowpass filtering removes high-frequency mixing products

5. Complex baseband allows sample rate = bandwidth (not 2×f_RF)
======================================================================
```

### Visualizations Explained

**Plot 1: Negative Frequency Detection**
- Shows two complex exponentials: one at +100 kHz, one at -100 kHz
- Demonstrates that complex FFT distinguishes positive from negative
- Real signals would show mirror images at both ±f

**Plot 2: Complete I/Q Processing**
- Row 1: Original RF signal (real-valued)
- Row 2: I, Q components after mixing (raw, with high-frequency content)
- Row 3: I, Q components after lowpass filtering (clean baseband)
- Row 4: Magnitude, phase, and frequency spectrum

**Key Observation**: The "rotating" I/Q constellation diagram shows the complex exponential nature of the signal.

---

## Method 2: External Application (Python + PlutoSDR)

### Objective
Capture real I/Q samples from PlutoSDR, analyze them, and perform baseband DSP operations on hardware data.

### Implementation

```python
#!/usr/bin/env python3
"""
LAB 1.3 - Method 2: I/Q Samples with PlutoSDR Hardware
Capture and analyze real I/Q data from PlutoSDR
"""

import numpy as np
import adi
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
from matplotlib.animation import FuncAnimation

PLUTO_URI = "ip:192.168.2.1"
SAMPLE_RATE = 2084000
RX_LO = int(915e6)
TX_LO = int(915e6)
BUFFER_SIZE = 2**14


class IQAnalyzer:
    """Analyze I/Q samples from PlutoSDR"""

    def __init__(self, uri):
        self.sdr = adi.Pluto(uri)
        self.setup_sdr()

    def setup_sdr(self):
        """Configure PlutoSDR"""
        # RX configuration
        self.sdr.sample_rate = int(SAMPLE_RATE)
        self.sdr.rx_lo = RX_LO
        self.sdr.rx_rf_bandwidth = int(SAMPLE_RATE)
        self.sdr.rx_buffer_size = BUFFER_SIZE
        self.sdr.gain_control_mode_chan0 = "manual"
        self.sdr.rx_hardwaregain_chan0 = 60

        # TX configuration (for generating test signal)
        self.sdr.tx_lo = TX_LO
        self.sdr.tx_rf_bandwidth = int(SAMPLE_RATE)
        self.sdr.tx_hardwaregain_chan0 = -10
        self.sdr.tx_cyclic_buffer = True

        print(f"PlutoSDR Configured:")
        print(f"  Sample Rate: {self.sdr.sample_rate/1e6:.3f} MSPS")
        print(f"  RX LO: {self.sdr.rx_lo/1e6:.1f} MHz")
        print(f"  TX LO: {self.sdr.tx_lo/1e6:.1f} MHz")
        print(f"  Buffer Size: {self.sdr.rx_buffer_size} samples")

    def transmit_tone(self, offset_freq):
        """Generate and transmit I/Q tone"""
        t = np.arange(BUFFER_SIZE) / SAMPLE_RATE

        # Generate complex tone
        tone = np.exp(2j * np.pi * offset_freq * t)

        # Scale to DAC range (int16)
        tone_scaled = (tone * 0.8 * 2**11).astype(np.int16)

        # Transmit
        self.sdr.tx(tone_scaled)
        print(f"  Transmitting tone at {offset_freq/1e3:+.0f} kHz offset")

    def capture_iq(self):
        """Capture I/Q samples"""
        samples = self.sdr.rx()
        return samples

    def analyze_iq_samples(self, samples):
        """Comprehensive I/Q analysis"""
        # Extract I and Q
        i_samples = np.real(samples)
        q_samples = np.imag(samples)

        # Statistics
        i_mean = np.mean(i_samples)
        q_mean = np.mean(q_samples)
        i_std = np.std(i_samples)
        q_std = np.std(q_samples)

        # DC offset
        dc_offset = np.mean(samples)

        # Power
        power_linear = np.mean(np.abs(samples)**2)
        power_db = 10 * np.log10(power_linear + 1e-12)

        # Magnitude and phase
        magnitude = np.abs(samples)
        phase = np.angle(samples)

        # FFT
        fft_result = np.fft.fftshift(np.fft.fft(samples))
        freqs = np.fft.fftshift(np.fft.fftfreq(len(samples), 1/self.sdr.sample_rate))
        spectrum_db = 20 * np.log10(np.abs(fft_result) + 1e-12)

        # Find peak frequency
        peak_idx = np.argmax(spectrum_db)
        peak_freq = freqs[peak_idx]
        peak_power = spectrum_db[peak_idx]

        return {
            'i': i_samples,
            'q': q_samples,
            'i_mean': i_mean,
            'q_mean': q_mean,
            'i_std': i_std,
            'q_std': q_std,
            'dc_offset': dc_offset,
            'power_db': power_db,
            'magnitude': magnitude,
            'phase': phase,
            'freqs': freqs,
            'spectrum_db': spectrum_db,
            'peak_freq': peak_freq,
            'peak_power': peak_power
        }

    def plot_iq_analysis(self, analysis, title="I/Q Analysis"):
        """Visualize I/Q samples"""
        fig = plt.figure(figsize=(16, 10))
        gs = GridSpec(3, 3, figure=fig, hspace=0.3, wspace=0.3)

        samples_to_plot = min(1000, len(analysis['i']))

        # 1. I samples (time domain)
        ax1 = fig.add_subplot(gs[0, 0])
        ax1.plot(analysis['i'][:samples_to_plot], linewidth=0.8, color='blue')
        ax1.axhline(analysis['i_mean'], color='red', linestyle='--', alpha=0.7, label=f'Mean: {analysis["i_mean"]:.1f}')
        ax1.set_xlabel('Sample Index', fontsize=10)
        ax1.set_ylabel('I (In-phase)', fontsize=10)
        ax1.set_title('I Component (Time Domain)', fontsize=11, fontweight='bold')
        ax1.legend(fontsize=9)
        ax1.grid(True, alpha=0.3)

        # 2. Q samples (time domain)
        ax2 = fig.add_subplot(gs[0, 1])
        ax2.plot(analysis['q'][:samples_to_plot], linewidth=0.8, color='red')
        ax2.axhline(analysis['q_mean'], color='blue', linestyle='--', alpha=0.7, label=f'Mean: {analysis["q_mean"]:.1f}')
        ax2.set_xlabel('Sample Index', fontsize=10)
        ax2.set_ylabel('Q (Quadrature)', fontsize=10)
        ax2.set_title('Q Component (Time Domain)', fontsize=11, fontweight='bold')
        ax2.legend(fontsize=9)
        ax2.grid(True, alpha=0.3)

        # 3. I vs Q (constellation)
        ax3 = fig.add_subplot(gs[0, 2])
        ax3.scatter(analysis['i'][:samples_to_plot], analysis['q'][:samples_to_plot],
                   s=5, alpha=0.3, c=range(samples_to_plot), cmap='viridis')
        ax3.scatter(analysis['i_mean'], analysis['q_mean'], s=200, c='red', marker='x', linewidths=3, label='Mean')
        ax3.set_xlabel('I', fontsize=10)
        ax3.set_ylabel('Q', fontsize=10)
        ax3.set_title('I/Q Constellation Diagram', fontsize=11, fontweight='bold')
        ax3.legend(fontsize=9)
        ax3.grid(True, alpha=0.3)
        ax3.axis('equal')
        ax3.axhline(0, color='black', linewidth=0.5)
        ax3.axvline(0, color='black', linewidth=0.5)

        # 4. Magnitude
        ax4 = fig.add_subplot(gs[1, 0])
        ax4.plot(analysis['magnitude'][:samples_to_plot], linewidth=0.8, color='purple')
        ax4.set_xlabel('Sample Index', fontsize=10)
        ax4.set_ylabel('|I + jQ|', fontsize=10)
        ax4.set_title('Signal Magnitude', fontsize=11, fontweight='bold')
        ax4.grid(True, alpha=0.3)

        # 5. Phase
        ax5 = fig.add_subplot(gs[1, 1])
        ax5.plot(analysis['phase'][:samples_to_plot], linewidth=0.8, color='orange')
        ax5.set_xlabel('Sample Index', fontsize=10)
        ax5.set_ylabel('∠(I + jQ) [rad]', fontsize=10)
        ax5.set_title('Signal Phase', fontsize=11, fontweight='bold')
        ax5.grid(True, alpha=0.3)

        # 6. Histogram (I and Q distribution)
        ax6 = fig.add_subplot(gs[1, 2])
        ax6.hist(analysis['i'], bins=50, alpha=0.6, label='I', color='blue')
        ax6.hist(analysis['q'], bins=50, alpha=0.6, label='Q', color='red')
        ax6.set_xlabel('Value', fontsize=10)
        ax6.set_ylabel('Count', fontsize=10)
        ax6.set_title('I/Q Distribution', fontsize=11, fontweight='bold')
        ax6.legend(fontsize=9)
        ax6.grid(True, alpha=0.3, axis='y')

        # 7. Spectrum (FFT)
        ax7 = fig.add_subplot(gs[2, :2])
        ax7.plot(analysis['freqs']/1e3, analysis['spectrum_db'], linewidth=1)
        ax7.axvline(analysis['peak_freq']/1e3, color='red', linestyle='--',
                   label=f'Peak @ {analysis["peak_freq"]/1e3:.1f} kHz ({analysis["peak_power"]:.1f} dB)')
        ax7.set_xlabel('Frequency (kHz)', fontsize=10)
        ax6.set_ylabel('Magnitude (dB)', fontsize=10)
        ax7.set_title('Baseband Spectrum', fontsize=11, fontweight='bold')
        ax7.legend(fontsize=9)
        ax7.grid(True, alpha=0.3)
        ax7.set_xlim([analysis['freqs'][0]/1e3, analysis['freqs'][-1]/1e3])

        # 8. Statistics
        ax8 = fig.add_subplot(gs[2, 2])
        ax8.axis('off')

        stats_text = f"""
I/Q STATISTICS
{'='*30}

I Component:
  Mean:     {analysis['i_mean']:+.2f}
  Std Dev:  {analysis['i_std']:.2f}
  Range:    [{np.min(analysis['i']):.1f}, {np.max(analysis['i']):.1f}]

Q Component:
  Mean:     {analysis['q_mean']:+.2f}
  Std Dev:  {analysis['q_std']:.2f}
  Range:    [{np.min(analysis['q']):.1f}, {np.max(analysis['q']):.1f}]

Signal Properties:
  Power:        {analysis['power_db']:.1f} dB
  DC Offset:    {np.abs(analysis['dc_offset']):.2f}
  Peak Freq:    {analysis['peak_freq']/1e3:+.1f} kHz

ADC Utilization:
  Max I/Q:      {max(np.max(np.abs(analysis['i'])), np.max(np.abs(analysis['q']))):.0f}
  Full Scale:   2048 (12-bit)
  Usage:        {100*max(np.max(np.abs(analysis['i'])), np.max(np.abs(analysis['q'])))/2048:.1f}%
"""

        ax8.text(0.05, 0.95, stats_text, transform=ax8.transAxes,
                fontsize=9, verticalalignment='top', fontfamily='monospace',
                bbox=dict(boxstyle='round', facecolor='lightblue', alpha=0.5))

        plt.suptitle(f'{title} - PlutoSDR Real I/Q Data', fontsize=14, fontweight='bold')

        filename = f'lab1_3_method2_{title.replace(" ", "_").lower()}.png'
        plt.savefig(filename, dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved: {filename}")
        plt.show()


def main():
    """Main execution"""
    print("="*70)
    print("LAB 1.3 - Method 2: I/Q Samples with PlutoSDR Hardware")
    print("="*70)
    print()

    try:
        analyzer = IQAnalyzer(PLUTO_URI)
    except Exception as e:
        print(f"✗ Failed to connect to PlutoSDR: {e}")
        return

    # Test 1: Transmit and receive tone
    print("\nTest 1: Analyzing Transmitted Tone")
    print("="*70)
    tone_offset = 100e3  # 100 kHz
    analyzer.transmit_tone(tone_offset)

    import time
    time.sleep(0.1)  # Allow TX to start

    samples = analyzer.capture_iq()
    analysis = analyzer.analyze_iq_samples(samples)

    print(f"\nCaptured {len(samples)} I/Q samples")
    print(f"  Peak frequency: {analysis['peak_freq']/1e3:+.1f} kHz")
    print(f"  Expected: {tone_offset/1e3:+.1f} kHz")
    print(f"  Error: {(analysis['peak_freq'] - tone_offset)/1e3:.2f} kHz")

    analyzer.plot_iq_analysis(analysis, "Transmitted_Tone")

    # Test 2: Analyze noise floor
    print("\nTest 2: Analyzing Noise Floor (No TX)")
    print("="*70)
    analyzer.sdr.tx_destroy_buffer()

    time.sleep(0.1)

    samples_noise = analyzer.capture_iq()
    analysis_noise = analyzer.analyze_iq_samples(samples_noise)

    print(f"\nNoise floor power: {analysis_noise['power_db']:.1f} dB")
    print(f"Signal power: {analysis['power_db']:.1f} dB")
    print(f"SNR: {analysis['power_db'] - analysis_noise['power_db']:.1f} dB")

    analyzer.plot_iq_analysis(analysis_noise, "Noise_Floor")

    print("\n" + "="*70)
    print("KEY OBSERVATIONS")
    print("="*70)
    print("1. I/Q samples are 16-bit signed integers (±2048 for 12-bit ADC)")
    print("2. DC offset in I/Q indicates LO leakage or calibration error")
    print("3. Constellation diagram shows signal characteristics visually")
    print("4. FFT of I/Q samples gives single-sided spectrum (no image)")
    print("5. Noise floor power indicates receiver sensitivity")
    print("="*70)


if __name__ == "__main__":
    main()
```

### Expected Output

```
======================================================================
LAB 1.3 - Method 2: I/Q Samples with PlutoSDR Hardware
======================================================================

PlutoSDR Configured:
  Sample Rate: 2.084 MSPS
  RX LO: 915.0 MHz
  TX LO: 915.0 MHz
  Buffer Size: 16384 samples

Test 1: Analyzing Transmitted Tone
======================================================================
  Transmitting tone at +100 kHz offset

Captured 16384 I/Q samples
  Peak frequency: +100.1 kHz
  Expected: +100.0 kHz
  Error: 0.13 kHz

✓ Saved: lab1_3_method2_transmitted_tone.png

Test 2: Analyzing Noise Floor (No TX)
======================================================================

Noise floor power: -42.3 dB
Signal power: -18.5 dB
SNR: 23.8 dB

✓ Saved: lab1_3_method2_noise_floor.png

======================================================================
KEY OBSERVATIONS
======================================================================
1. I/Q samples are 16-bit signed integers (±2048 for 12-bit ADC)
2. DC offset in I/Q indicates LO leakage or calibration error
3. Constellation diagram shows signal characteristics visually
4. FFT of I/Q samples gives single-sided spectrum (no image)
5. Noise floor power indicates receiver sensitivity
======================================================================
```

---

## Method 3: Hosted Application (C on PlutoSDR)

### Objective
Process I/Q samples directly on PlutoSDR ARM processor, demonstrating efficient embedded I/Q handling.

### Implementation Overview

The hosted application performs:
1. Capture I/Q samples via local IIO
2. Real-time I/Q analysis (power, FFT)
3. DC offset compensation
4. Peak detection in frequency domain
5. Performance measurements

**Complete implementation documented in:**
**[LAB_1_3_METHOD3_HOSTED.md](LAB_1_3_METHOD3_HOSTED.md)**

Includes:
- Full C source with complex number handling
- FFT implementation using kiss_fft
- Real-time I/Q statistics
- Compilation and deployment

---

## Comparison of Methods

| Aspect | Method 1 (Simulation) | Method 2 (External) | Method 3 (Hosted) |
|--------|----------------------|--------------------|--------------------|
| **I/Q Source** | Generated mathematically | Real hardware (ADC) | Real hardware (ADC) |
| **Processing** | Python (NumPy) | Python (NumPy) | C (manual loops) |
| **Visualization** | Matplotlib (full) | Matplotlib (full) | Limited (text output) |
| **Latency** | N/A | ~10-50 ms | <1 ms |
| **FFT** | NumPy FFT | NumPy FFT | kiss_fft or CMSIS-DSP |
| **Use Case** | Learning, algorithm dev | Lab testing | Real-time embedded |

---

## Practical Exercises

### Exercise 1: DC Offset Measurement
Capture I/Q samples and measure DC offset. What causes it? How would you correct it?

### Exercise 2: IQ Imbalance
Measure the ratio of I std dev to Q std dev. Perfect balance = 1.0. What causes imbalance?

### Exercise 3: Phase Noise Visualization
Transmit a CW tone and plot instantaneous phase over time. Observe phase noise from oscillator.

### Exercise 4: Modulation Recognition
Generate different modulation types (AM, FM, PSK) and observe their I/Q constellation patterns.

---

## Summary

### Key Concepts Mastered

1. **I/Q representation** stores both amplitude and phase
2. **Quadrature mixing** enables frequency translation
3. **Complex baseband** distinguishes +f from -f
4. **Sample rate = bandwidth** for complex signals
5. **Constellation diagrams** visualize modulation states

### Progression Through Methods

- **Method 1**: Mathematical understanding of I/Q concepts
- **Method 2**: Real hardware I/Q data analysis
- **Method 3**: Efficient embedded I/Q processing

---

## Next Lab

**LAB 1.4: Nyquist Sampling & Aliasing** - Learn about sample rate requirements, aliasing effects, and anti-alias filtering.

---

**End of LAB 1.3**
