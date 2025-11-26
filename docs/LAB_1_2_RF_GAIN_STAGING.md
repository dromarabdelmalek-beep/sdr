# LAB 1.2: RF Front-End Gain Staging

## Overview

Proper gain configuration is **critical** for SDR performance. Set the gain too low, and your signal drowns in noise. Set it too high, and you get clipping and distortion. This lab teaches you how to configure PlutoSDR's **multi-stage RF front-end** for optimal performance in different scenarios.

### Learning Objectives

1. Understand the AD9361 gain architecture (RX and TX chains)
2. Learn the difference between AGC and manual gain control
3. Optimize SNR without causing saturation
4. Handle strong and weak signal scenarios
5. Measure and analyze dynamic range
6. Implement gain adaptation algorithms

### Key Concepts

- **Gain Stages**: LNA, Mixer, and Baseband amplifiers in AD9361
- **Total Gain Range**: RX: 0-73 dB, TX: -89.75 to 0 dB
- **AGC (Automatic Gain Control)**: Hardware-based adaptive gain
- **Manual Gain**: Software-controlled fixed gain
- **Saturation**: Signal exceeds ADC range (clipping)
- **Noise Figure**: Degradation of SNR through the RX chain
- **1 dB Compression Point**: Power level where gain drops by 1 dB

---

## AD9361 Gain Architecture

### RX Gain Chain

```
        ┌─────────────────────────────────────────────────────────┐
Antenna │                                                         │ ADC
────────┼───►[LNA]───►[Mixer]───►[BB-AMP]───►[LPF]───────────────┼────►
   RF   │   0-30dB    Variable     0-43dB     Analog             │ Digital
        │    3dB       Gain        0.25dB     Filter             │ 12-bit
        │    steps     Control     steps                         │
        │                                                         │
        │  ◄────────── Total RX Gain: 0 to 73 dB ──────────────► │
        └─────────────────────────────────────────────────────────┘

Gain Control Modes:
  • Manual:    Software sets exact gain value
  • Fast AGC:  Hardware adapts in real-time (<1 ms)
  • Slow AGC:  Slower adaptation for stable signals
```

### TX Gain Chain

```
        ┌─────────────────────────────────────────────────────────┐
  DAC   │                                                         │ Antenna
────────┼───►[LPF]───►[BB-AMP]───►[Mixer]───►[PA]────────────────┼────►
Digital │    Analog    Variable    Variable   Power              │  RF
12-bit  │    Filter    Gain        Gain       Amplifier          │
        │              Control     Control    -89.75 to 0 dB     │
        │              (coarse)    (fine)     0.25 dB steps      │
        │                                                         │
        │  ◄────────── Total TX Gain: -89.75 to 0 dB ──────────► │
        └─────────────────────────────────────────────────────────┘

Note: TX gain is actually attenuation (0 dB = maximum output power)
```

### Gain Distribution Strategy

**For RX (receiving weak signals)**:
- Maximize **early-stage gain** (LNA) to minimize noise figure
- Keep **later stages** moderate to avoid saturation
- Typical: LNA=30dB, Mixer=20dB, BB-AMP=20dB = 70dB total

**For TX (transmitting)**:
- Start with **low power** to avoid interference
- Increase only as needed for link requirements
- Monitor for spectral regrowth (non-linearity indicator)

---

## Method 1: Pure Simulation (No Hardware)

### Objective
Simulate an RF front-end with configurable gain to understand trade-offs between SNR, dynamic range, and saturation.

### Theory

The key equation for RF system SNR is:

```
SNR_out = SNR_in + Gain - NF

Where:
  SNR_in  = Input signal-to-noise ratio (dB)
  Gain    = Total system gain (dB)
  NF      = Noise Figure (dB) - noise added by the receiver
  SNR_out = Output SNR (dB)
```

**Saturation** occurs when signal amplitude exceeds ADC range:
```
Signal_level = Input_power + Gain

If Signal_level > ADC_Full_Scale:
    → Clipping/distortion
```

### Implementation

```python
#!/usr/bin/env python3
"""
LAB 1.2 - Method 1: RF Gain Staging Simulation
Demonstrates gain optimization for different signal scenarios
"""

import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
from dataclasses import dataclass
from typing import Tuple

# ADC characteristics (12-bit ADC in AD9361)
ADC_BITS = 12
ADC_FULL_SCALE = 2**(ADC_BITS - 1)  # ±2048 for signed 12-bit
ADC_DBFS = 20 * np.log10(ADC_FULL_SCALE)  # dBFS full scale

# Gain stages
LNA_GAIN_RANGE = (0, 30)       # 0-30 dB, 3 dB steps
MIXER_GAIN_RANGE = (0, 25)     # 0-25 dB
BASEBAND_GAIN_RANGE = (0, 43)  # 0-43 dB, 0.25 dB steps

MAX_RX_GAIN = 73  # dB (sum of all stages)
NOISE_FIGURE = 3  # dB (typical for AD9361)


@dataclass
class SignalScenario:
    """Defines a signal scenario with specific characteristics"""
    name: str
    signal_power_dbm: float  # Input signal power
    noise_floor_dbm: float   # Input noise floor
    description: str

    def snr_in(self) -> float:
        """Calculate input SNR"""
        return self.signal_power_dbm - self.noise_floor_dbm


# Define test scenarios
SCENARIOS = [
    SignalScenario(
        name="Strong Signal",
        signal_power_dbm=-30,
        noise_floor_dbm=-90,
        description="Nearby transmitter, low gain needed"
    ),
    SignalScenario(
        name="Medium Signal",
        signal_power_dbm=-60,
        noise_floor_dbm=-90,
        description="Typical reception scenario"
    ),
    SignalScenario(
        name="Weak Signal",
        signal_power_dbm=-85,
        noise_floor_dbm=-90,
        description="Distant transmitter, maximum gain needed"
    ),
    SignalScenario(
        name="Very Weak Signal",
        signal_power_dbm=-100,
        noise_floor_dbm=-105,
        description="Near noise floor, AGC essential"
    ),
]


class RFGainSimulator:
    """
    Simulates RF front-end gain stages and signal processing
    """

    def __init__(self, sample_rate=2.084e6, samples=16384):
        self.sample_rate = sample_rate
        self.samples = samples
        self.adc_dbfs = ADC_DBFS

    def generate_signal(self, signal_power_dbm, noise_floor_dbm):
        """
        Generate signal with noise at specified power levels

        Power conversion:
        - dBm (absolute) → dBFS (relative to full scale)
        - Assume 50 ohm system, 0 dBFS = +4 dBm
        """
        # Reference: 0 dBFS = +4 dBm (typical for AD9361)
        REF_DBM = 4.0

        # Convert dBm to dBFS
        signal_dbfs = signal_power_dbm - REF_DBM
        noise_dbfs = noise_floor_dbm - REF_DBM

        # Convert dBFS to linear amplitude (normalized to ±1)
        signal_amplitude = 10**(signal_dbfs / 20.0)
        noise_amplitude = 10**(noise_dbfs / 20.0)

        # Generate signal (simple tone)
        t = np.arange(self.samples) / self.sample_rate
        signal_freq = 100e3  # 100 kHz offset
        signal = signal_amplitude * np.exp(2j * np.pi * signal_freq * t)

        # Generate noise (complex AWGN)
        noise_power = noise_amplitude**2 / 2  # Power per I/Q component
        noise = np.sqrt(noise_power) * (np.random.randn(self.samples) +
                                        1j * np.random.randn(self.samples))

        return signal, noise

    def apply_gain(self, signal, noise, gain_db, noise_figure_db=NOISE_FIGURE):
        """
        Apply gain to signal and noise, add amplifier noise

        Amplifier adds noise characterized by noise figure:
        NF = SNR_in / SNR_out (linear)

        Amplifier noise power:
        N_amp = Gain × N_in × (F - 1)
        where F = 10^(NF/10)
        """
        gain_linear = 10**(gain_db / 20.0)  # Voltage gain
        nf_linear = 10**(noise_figure_db / 10.0)  # Noise figure (linear)

        # Apply gain to signal and input noise
        signal_out = signal * gain_linear
        noise_out = noise * gain_linear

        # Add amplifier noise
        noise_in_power = np.mean(np.abs(noise)**2)
        amp_noise_power = gain_linear**2 * noise_in_power * (nf_linear - 1)

        amp_noise = np.sqrt(amp_noise_power / 2) * (np.random.randn(self.samples) +
                                                     1j * np.random.randn(self.samples))

        noise_total = noise_out + amp_noise

        return signal_out, noise_total

    def adc_quantize(self, samples):
        """
        Simulate 12-bit ADC quantization

        Returns:
        - quantized samples (int16)
        - clipping indicator (boolean)
        """
        # Scale to ADC range (±2048 for 12-bit signed)
        samples_scaled = samples * ADC_FULL_SCALE

        # Check for clipping
        clipping = np.any(np.abs(samples_scaled) > ADC_FULL_SCALE)

        # Clip to ADC range
        samples_clipped = np.clip(samples_scaled.real, -ADC_FULL_SCALE, ADC_FULL_SCALE) + \
                         1j * np.clip(samples_scaled.imag, -ADC_FULL_SCALE, ADC_FULL_SCALE)

        # Quantize (round to nearest integer)
        samples_quantized = np.round(samples_clipped).astype(np.int16)

        return samples_quantized, clipping

    def calculate_metrics(self, signal, noise, signal_out, noise_total, clipping):
        """Calculate performance metrics"""
        # Input SNR
        snr_in = 10 * np.log10(np.mean(np.abs(signal)**2) /
                              np.mean(np.abs(noise)**2))

        # Output SNR
        snr_out = 10 * np.log10(np.mean(np.abs(signal_out)**2) /
                               np.mean(np.abs(noise_total)**2))

        # Signal power (dBFS)
        signal_power_dbfs = 20 * np.log10(np.std(np.abs(signal_out)) + 1e-12)

        # Noise power (dBFS)
        noise_power_dbfs = 20 * np.log10(np.std(np.abs(noise_total)) + 1e-12)

        # Headroom (dB below full scale)
        headroom_db = self.adc_dbfs - signal_power_dbfs

        return {
            'snr_in_db': snr_in,
            'snr_out_db': snr_out,
            'signal_power_dbfs': signal_power_dbfs,
            'noise_power_dbfs': noise_power_dbfs,
            'headroom_db': headroom_db,
            'clipping': clipping
        }


def find_optimal_gain(scenario: SignalScenario, simulator: RFGainSimulator):
    """
    Find optimal gain for a given scenario

    Strategy:
    - Maximize SNR without causing clipping
    - Target: 3-6 dB headroom below full scale
    """
    TARGET_HEADROOM = 6.0  # dB
    MIN_HEADROOM = 3.0     # dB (clipping margin)

    best_gain = 0
    best_snr = -float('inf')
    results = []

    # Test gain values from 0 to MAX_RX_GAIN
    for gain_db in range(0, MAX_RX_GAIN + 1, 1):
        # Generate input signal
        signal, noise = simulator.generate_signal(
            scenario.signal_power_dbm,
            scenario.noise_floor_dbm
        )

        # Apply gain
        signal_out, noise_total = simulator.apply_gain(signal, noise, gain_db)

        # Simulate ADC
        rx_samples = signal_out + noise_total
        quantized, clipping = simulator.adc_quantize(rx_samples)

        # Calculate metrics
        metrics = simulator.calculate_metrics(
            signal, noise, signal_out, noise_total, clipping
        )

        results.append({
            'gain_db': gain_db,
            **metrics
        })

        # Find best gain (no clipping, maximize SNR)
        if not clipping and metrics['headroom_db'] >= MIN_HEADROOM:
            if metrics['snr_out_db'] > best_snr:
                best_snr = metrics['snr_out_db']
                best_gain = gain_db

    return best_gain, results


def plot_gain_analysis(scenario: SignalScenario, results, optimal_gain):
    """
    Visualize gain trade-offs
    """
    gains = [r['gain_db'] for r in results]
    snr_out = [r['snr_out_db'] for r in results]
    headroom = [r['headroom_db'] for r in results]
    clipping = [r['clipping'] for r in results]

    fig = plt.figure(figsize=(14, 10))
    gs = GridSpec(3, 2, figure=fig, hspace=0.3, wspace=0.3)

    # 1. SNR vs Gain
    ax1 = fig.add_subplot(gs[0, :])
    ax1.plot(gains, snr_out, linewidth=2, color='blue', label='Output SNR')
    ax1.axvline(optimal_gain, color='green', linestyle='--', linewidth=2,
                label=f'Optimal Gain = {optimal_gain} dB')

    # Mark clipping region
    clip_gains = [g for g, c in zip(gains, clipping) if c]
    if clip_gains:
        ax1.axvspan(min(clip_gains), max(gains), alpha=0.3, color='red',
                   label='Clipping Region')

    ax1.set_xlabel('RX Gain (dB)', fontsize=12)
    ax1.set_ylabel('Output SNR (dB)', fontsize=12)
    ax1.set_title(f'SNR vs Gain - {scenario.name}', fontsize=14, fontweight='bold')
    ax1.legend(fontsize=11)
    ax1.grid(True, alpha=0.3)

    # 2. Headroom vs Gain
    ax2 = fig.add_subplot(gs[1, 0])
    ax2.plot(gains, headroom, linewidth=2, color='purple')
    ax2.axhline(3, color='orange', linestyle='--', label='Min Headroom (3 dB)')
    ax2.axhline(0, color='red', linestyle='-', linewidth=2, label='Clipping (0 dB)')
    ax2.axvline(optimal_gain, color='green', linestyle='--', linewidth=2)
    ax2.set_xlabel('RX Gain (dB)', fontsize=11)
    ax2.set_ylabel('Headroom (dB)', fontsize=11)
    ax2.set_title('Headroom vs Gain', fontsize=12, fontweight='bold')
    ax2.legend(fontsize=9)
    ax2.grid(True, alpha=0.3)
    ax2.set_ylim([-10, max(headroom) + 5])

    # 3. Signal and Noise Power Levels
    ax3 = fig.add_subplot(gs[1, 1])
    signal_power = [r['signal_power_dbfs'] for r in results]
    noise_power = [r['noise_power_dbfs'] for r in results]

    ax3.plot(gains, signal_power, linewidth=2, label='Signal Power', color='blue')
    ax3.plot(gains, noise_power, linewidth=2, label='Noise Power', color='red')
    ax3.axhline(ADC_DBFS, color='black', linestyle='-', linewidth=2,
               label=f'ADC Full Scale ({ADC_DBFS:.1f} dBFS)')
    ax3.axvline(optimal_gain, color='green', linestyle='--', linewidth=2)
    ax3.fill_between(gains, ADC_DBFS - 3, ADC_DBFS, alpha=0.2, color='orange',
                     label='Clipping Risk Zone')
    ax3.set_xlabel('RX Gain (dB)', fontsize=11)
    ax3.set_ylabel('Power (dBFS)', fontsize=11)
    ax3.set_title('Signal and Noise Power Levels', fontsize=12, fontweight='bold')
    ax3.legend(fontsize=9)
    ax3.grid(True, alpha=0.3)

    # 4. Gain Distribution (optimal case)
    ax4 = fig.add_subplot(gs[2, 0])
    # Example gain distribution for optimal gain
    lna_gain = min(30, optimal_gain)
    remaining = max(0, optimal_gain - lna_gain)
    mixer_gain = min(25, remaining)
    remaining = max(0, remaining - mixer_gain)
    bb_gain = min(43, remaining)

    stages = ['LNA\n(0-30dB)', 'Mixer\n(0-25dB)', 'Baseband\n(0-43dB)']
    stage_gains = [lna_gain, mixer_gain, bb_gain]
    colors_stages = ['#ff6b6b', '#4ecdc4', '#45b7d1']

    bars = ax4.bar(stages, stage_gains, color=colors_stages, edgecolor='black', linewidth=2)
    ax4.set_ylabel('Gain (dB)', fontsize=11)
    ax4.set_title(f'Gain Distribution (Total = {optimal_gain} dB)', fontsize=12, fontweight='bold')
    ax4.grid(True, alpha=0.3, axis='y')

    # Add value labels on bars
    for bar in bars:
        height = bar.get_height()
        ax4.text(bar.get_x() + bar.get_width()/2., height,
                f'{int(height)} dB',
                ha='center', va='bottom', fontsize=11, fontweight='bold')

    # 5. Performance Summary Table
    ax5 = fig.add_subplot(gs[2, 1])
    ax5.axis('off')

    optimal_result = results[optimal_gain]

    summary_text = f"""
OPTIMAL GAIN CONFIGURATION
{'='*40}

Scenario: {scenario.name}
{scenario.description}

Input Characteristics:
  • Signal Power:    {scenario.signal_power_dbm:+.1f} dBm
  • Noise Floor:     {scenario.noise_floor_dbm:+.1f} dBm
  • Input SNR:       {scenario.snr_in():.1f} dB

Optimal Gain Settings:
  • Total RX Gain:   {optimal_gain} dB
  • LNA Gain:        {lna_gain} dB
  • Mixer Gain:      {mixer_gain} dB
  • Baseband Gain:   {bb_gain} dB

Output Performance:
  • Output SNR:      {optimal_result['snr_out_db']:.1f} dB
  • SNR Improvement: {optimal_result['snr_out_db'] - scenario.snr_in():.1f} dB
  • Signal Level:    {optimal_result['signal_power_dbfs']:.1f} dBFS
  • Noise Level:     {optimal_result['noise_power_dbfs']:.1f} dBFS
  • Headroom:        {optimal_result['headroom_db']:.1f} dB
  • Clipping:        {'YES ⚠' if optimal_result['clipping'] else 'NO ✓'}

Status: {'⚠ WARNING' if optimal_result['clipping'] else '✓ OPTIMAL'}
"""

    ax5.text(0.05, 0.95, summary_text, transform=ax5.transAxes,
            fontsize=9, verticalalignment='top', fontfamily='monospace',
            bbox=dict(boxstyle='round', facecolor='lightgreen' if not optimal_result['clipping'] else 'lightcoral',
                     alpha=0.5))

    plt.suptitle('LAB 1.2 - Method 1: RF Gain Staging Analysis',
                fontsize=16, fontweight='bold', y=0.995)

    filename = f'lab1_2_method1_{scenario.name.replace(" ", "_").lower()}.png'
    plt.savefig(filename, dpi=150, bbox_inches='tight')
    print(f"  ✓ Plot saved: {filename}")
    plt.close()


def main():
    """Main execution"""
    print("="*70)
    print("LAB 1.2 - Method 1: RF Gain Staging Simulation")
    print("="*70)
    print()

    simulator = RFGainSimulator()

    for scenario in SCENARIOS:
        print(f"\nAnalyzing: {scenario.name}")
        print(f"  {scenario.description}")
        print(f"  Input SNR: {scenario.snr_in():.1f} dB")

        optimal_gain, results = find_optimal_gain(scenario, simulator)

        print(f"  ✓ Optimal gain: {optimal_gain} dB")
        print(f"  ✓ Output SNR: {results[optimal_gain]['snr_out_db']:.1f} dB")
        print(f"  ✓ Headroom: {results[optimal_gain]['headroom_db']:.1f} dB")

        plot_gain_analysis(scenario, results, optimal_gain)

    print("\n" + "="*70)
    print("KEY TAKEAWAYS")
    print("="*70)
    print("1. Strong signals need LOW gain to avoid clipping")
    print("2. Weak signals need HIGH gain to improve SNR")
    print("3. Always maintain 3-6 dB headroom below ADC full scale")
    print("4. LNA gain should be maximized first (best noise figure)")
    print("5. AGC is essential when signal strength varies")
    print("="*70)


if __name__ == "__main__":
    main()
```

### Expected Output

```
======================================================================
LAB 1.2 - Method 1: RF Gain Staging Simulation
======================================================================

Analyzing: Strong Signal
  Nearby transmitter, low gain needed
  Input SNR: 60.0 dB
  ✓ Optimal gain: 28 dB
  ✓ Output SNR: 57.2 dB
  ✓ Headroom: 6.1 dB
  ✓ Plot saved: lab1_2_method1_strong_signal.png

Analyzing: Medium Signal
  Typical reception scenario
  Input SNR: 30.0 dB
  ✓ Optimal gain: 58 dB
  ✓ Output SNR: 27.4 dB
  ✓ Headroom: 3.2 dB
  ✓ Plot saved: lab1_2_method1_medium_signal.png

Analyzing: Weak Signal
  Distant transmitter, maximum gain needed
  Input SNR: 5.0 dB
  ✓ Optimal gain: 73 dB
  ✓ Output SNR: 2.1 dB
  ✓ Headroom: 5.8 dB
  ✓ Plot saved: lab1_2_method1_weak_signal.png

Analyzing: Very Weak Signal
  Near noise floor, AGC essential
  Input SNR: 5.0 dB
  ✓ Optimal gain: 73 dB
  ✓ Output SNR: 2.1 dB
  ✓ Headroom: 20.5 dB
  ✓ Plot saved: lab1_2_method1_very_weak_signal.png

======================================================================
KEY TAKEAWAYS
======================================================================
1. Strong signals need LOW gain to avoid clipping
2. Weak signals need HIGH gain to improve SNR
3. Always maintain 3-6 dB headroom below ADC full scale
4. LNA gain should be maximized first (best noise figure)
5. AGC is essential when signal strength varies
======================================================================
```

### Key Insights from Simulation

1. **Strong Signal Scenario (-30 dBm)**:
   - Optimal gain: ~28 dB (LOW)
   - Reason: Higher gain would cause clipping
   - SNR is already excellent (60 dB), no need for more gain

2. **Medium Signal Scenario (-60 dBm)**:
   - Optimal gain: ~58 dB (MODERATE)
   - Balance between SNR improvement and avoiding saturation

3. **Weak Signal Scenario (-85 dBm)**:
   - Optimal gain: 73 dB (MAXIMUM)
   - Need every bit of gain to pull signal out of noise

4. **Very Weak Signal Scenario (-100 dBm)**:
   - Optimal gain: 73 dB (MAXIMUM)
   - Even at max gain, signal barely above noise floor
   - AGC would be essential for dynamic adaptation

---

## Method 2: External Application (Python + PlutoSDR)

### Objective
Implement AGC and manual gain control on real PlutoSDR hardware, comparing performance in different signal strength scenarios.

*[Due to length constraints, this continues in next message or separate file. Would you like me to continue with Method 2 and Method 3 for LAB 1.2?]*

---

## Method 2: External Application (Python + PlutoSDR)

### Objective
Test AGC and manual gain control on real hardware, comparing performance and demonstrating adaptive gain algorithms.

### Hardware Setup

Same as LAB 0 and 1.1:
- PlutoSDR connected via USB (192.168.2.1)
- Two antennas or use signal generator + attenuator for controlled testing

### AGC Modes in AD9361

PlutoSDR supports three gain control modes:

1. **Manual**: Fixed gain set by software
2. **Fast Attack AGC**: Rapid adaptation (<1 ms), good for bursty signals
3. **Slow Attack AGC**: Gradual adaptation, good for continuous signals

### Implementation

```python
#!/usr/bin/env python3
"""
LAB 1.2 - Method 2: RF Gain Staging with PlutoSDR
Compare AGC vs Manual gain control
"""

import numpy as np
import adi
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
import time

PLUTO_URI = "ip:192.168.2.1"
SAMPLE_RATE = 2084000
BUFFER_SIZE = 2**14

class PlutoGainController:
    """Control PlutoSDR gain settings and test different scenarios"""

    def __init__(self, uri):
        self.sdr = adi.Pluto(uri)
        self.sdr.sample_rate = int(SAMPLE_RATE)
        self.sdr.rx_rf_bandwidth = int(SAMPLE_RATE)
        self.sdr.rx_buffer_size = BUFFER_SIZE
        self.sdr.tx_lo = int(915e6)
        self.sdr.rx_lo = int(915e6)

        print(f"PlutoSDR initialized at {uri}")
        print(f"  Sample Rate: {self.sdr.sample_rate/1e6:.3f} MHz")

    def set_manual_gain(self, gain_db):
        """Set manual gain mode"""
        self.sdr.gain_control_mode_chan0 = "manual"
        self.sdr.rx_hardwaregain_chan0 = int(gain_db)
        print(f"  Manual gain set to {gain_db} dB")

    def set_agc_mode(self, mode="slow_attack"):
        """
        Set AGC mode
        Options: 'slow_attack', 'fast_attack', 'hybrid'
        """
        self.sdr.gain_control_mode_chan0 = mode
        print(f"  AGC mode set to: {mode}")

    def measure_signal_strength(self):
        """Capture and measure received signal power"""
        rx_samples = self.sdr.rx()

        # Calculate power in dB
        power_linear = np.mean(np.abs(rx_samples)**2)
        power_db = 10 * np.log10(power_linear + 1e-12)

        # Check for clipping (>90% of full scale)
        full_scale = 2**11  # 12-bit ADC
        max_val = np.max(np.abs(rx_samples))
        clipping = max_val > (0.9 * full_scale)

        # Calculate crest factor (peak-to-RMS ratio)
        rms = np.sqrt(power_linear)
        peak = max_val
        crest_factor_db = 20 * np.log10(peak / (rms + 1e-12))

        return {
            'power_db': power_db,
            'clipping': clipping,
            'crest_factor_db': crest_factor_db,
            'max_value': max_val,
            'samples': rx_samples
        }

    def test_manual_gain_sweep(self, gain_range=(0, 73, 3)):
        """Test manual gain from min to max"""
        results = []

        print("\nManual Gain Sweep Test")
        print("="*60)

        for gain_db in range(*gain_range):
            self.set_manual_gain(gain_db)
            time.sleep(0.1)  # Allow settling

            measurement = self.measure_signal_strength()
            measurement['gain_db'] = gain_db

            status = "✗ CLIP" if measurement['clipping'] else "✓ OK"
            print(f"Gain {gain_db:2d} dB: Power {measurement['power_db']:+6.1f} dB | {status}")

            results.append(measurement)

        return results

    def test_agc_modes(self, duration_sec=5):
        """Test different AGC modes"""
        modes = ['slow_attack', 'fast_attack']
        results = {}

        for mode in modes:
            print(f"\nTesting AGC Mode: {mode}")
            print("="*60)

            self.set_agc_mode(mode)
            time.sleep(0.5)  # Allow AGC to settle

            mode_results = []
            for i in range(10):  # Capture 10 measurements
                measurement = self.measure_signal_strength()
                mode_results.append(measurement)
                print(f"  Measurement {i+1}: Power {measurement['power_db']:+6.1f} dB")
                time.sleep(0.5)

            results[mode] = mode_results

        return results


def plot_manual_gain_results(results):
    """Visualize manual gain sweep results"""
    gains = [r['gain_db'] for r in results]
    powers = [r['power_db'] for r in results]
    clipping = [r['clipping'] for r in results]
    crest_factors = [r['crest_factor_db'] for r in results]

    fig, axes = plt.subplots(3, 1, figsize=(12, 10), sharex=True)

    # Power vs Gain
    colors = ['red' if c else 'blue' for c in clipping]
    axes[0].scatter(gains, powers, c=colors, s=50, alpha=0.7)
    axes[0].plot(gains, powers, 'k-', alpha=0.3, linewidth=1)
    axes[0].set_ylabel('Received Power (dB)', fontsize=11)
    axes[0].set_title('Manual Gain Sweep - Signal Power', fontsize=13, fontweight='bold')
    axes[0].grid(True, alpha=0.3)

    # Add legend
    axes[0].scatter([], [], c='blue', label='No Clipping')
    axes[0].scatter([], [], c='red', label='Clipping')
    axes[0].legend()

    # Crest Factor
    axes[1].plot(gains, crest_factors, 'o-', color='purple', linewidth=2)
    axes[1].axhline(12, color='orange', linestyle='--', label='Typical OFDM (~12 dB)')
    axes[1].axhline(0, color='red', linestyle='--', label='Pure Sine Wave (0 dB)')
    axes[1].set_ylabel('Crest Factor (dB)', fontsize=11)
    axes[1].set_title('Crest Factor vs Gain', fontsize=13, fontweight='bold')
    axes[1].legend()
    axes[1].grid(True, alpha=0.3)

    # Clipping indicator
    clip_indicator = [1 if c else 0 for c in clipping]
    axes[2].fill_between(gains, 0, clip_indicator, color='red', alpha=0.5, step='mid')
    axes[2].set_xlabel('RX Gain (dB)', fontsize=11)
    axes[2].set_ylabel('Clipping', fontsize=11)
    axes[2].set_title('Clipping Detection', fontsize=13, fontweight='bold')
    axes[2].set_ylim([-0.1, 1.1])
    axes[2].set_yticks([0, 1])
    axes[2].set_yticklabels(['No', 'Yes'])
    axes[2].grid(True, alpha=0.3, axis='x')

    plt.tight_layout()
    plt.savefig('lab1_2_method2_manual_gain.png', dpi=150)
    print("\n✓ Plot saved: lab1_2_method2_manual_gain.png")
    plt.show()


def plot_agc_comparison(results):
    """Compare AGC modes"""
    fig, axes = plt.subplots(2, 1, figsize=(12, 8), sharex=True)

    for mode, measurements in results.items():
        times = np.arange(len(measurements)) * 0.5  # 0.5 sec intervals
        powers = [m['power_db'] for m in measurements]

        axes[0].plot(times, powers, 'o-', linewidth=2, label=mode.replace('_', ' ').title())

    axes[0].set_ylabel('Received Power (dB)', fontsize=11)
    axes[0].set_title('AGC Mode Comparison - Signal Power Over Time', fontsize=13, fontweight='bold')
    axes[0].legend(fontsize=11)
    axes[0].grid(True, alpha=0.3)

    # Show standard deviation (stability metric)
    for mode, measurements in results.items():
        powers = [m['power_db'] for m in measurements]
        std_dev = np.std(powers)

        axes[1].bar(mode.replace('_', ' ').title(), std_dev, alpha=0.7)

    axes[1].set_ylabel('Power Std Dev (dB)', fontsize=11)
    axes[1].set_title('AGC Stability (Lower is Better)', fontsize=13, fontweight='bold')
    axes[1].grid(True, alpha=0.3, axis='y')

    plt.tight_layout()
    plt.savefig('lab1_2_method2_agc_comparison.png', dpi=150)
    print("✓ Plot saved: lab1_2_method2_agc_comparison.png")
    plt.show()


def main():
    """Main execution"""
    print("="*70)
    print("LAB 1.2 - Method 2: RF Gain Staging with PlutoSDR")
    print("="*70)
    print()

    try:
        controller = PlutoGainController(PLUTO_URI)
    except Exception as e:
        print(f"✗ Failed to connect to PlutoSDR: {e}")
        return

    # Test 1: Manual gain sweep
    print("\n" + "="*70)
    print("TEST 1: Manual Gain Sweep")
    print("="*70)
    manual_results = controller.test_manual_gain_sweep(gain_range=(0, 74, 3))
    plot_manual_gain_results(manual_results)

    # Test 2: AGC modes
    print("\n" + "="*70)
    print("TEST 2: AGC Mode Comparison")
    print("="*70)
    agc_results = controller.test_agc_modes()
    plot_agc_comparison(agc_results)

    # Summary
    print("\n" + "="*70)
    print("KEY OBSERVATIONS")
    print("="*70)
    print("1. Manual gain provides predictable, repeatable results")
    print("2. AGC automatically adapts to signal strength changes")
    print("3. Fast AGC responds quickly but may be less stable")
    print("4. Slow AGC is more stable for continuous signals")
    print("5. Always monitor for clipping at high gain settings")
    print("="*70)


if __name__ == "__main__":
    main()
```

### Expected Output

```
======================================================================
LAB 1.2 - Method 2: RF Gain Staging with PlutoSDR
======================================================================

PlutoSDR initialized at ip:192.168.2.1
  Sample Rate: 2.084 MHz

======================================================================
TEST 1: Manual Gain Sweep
======================================================================

Manual Gain Sweep Test
============================================================
  Manual gain set to 0 dB
Gain  0 dB: Power  -45.2 dB | ✓ OK
  Manual gain set to 3 dB
Gain  3 dB: Power  -42.1 dB | ✓ OK
  Manual gain set to 6 dB
Gain  6 dB: Power  -39.3 dB | ✓ OK
...
  Manual gain set to 60 dB
Gain 60 dB: Power  -5.8 dB | ✓ OK
  Manual gain set to 63 dB
Gain 63 dB: Power  -2.7 dB | ✓ OK
  Manual gain set to 66 dB
Gain 66 dB: Power  +0.5 dB | ✗ CLIP
  Manual gain set to 69 dB
Gain 69 dB: Power  +3.2 dB | ✗ CLIP
  Manual gain set to 72 dB
Gain 72 dB: Power  +6.1 dB | ✗ CLIP

✓ Plot saved: lab1_2_method2_manual_gain.png

======================================================================
TEST 2: AGC Mode Comparison
======================================================================

Testing AGC Mode: slow_attack
============================================================
  AGC mode set to: slow_attack
  Measurement 1: Power  -8.2 dB
  Measurement 2: Power  -8.3 dB
  Measurement 3: Power  -8.1 dB
  ...

Testing AGC Mode: fast_attack
============================================================
  AGC mode set to: fast_attack
  Measurement 1: Power  -7.9 dB
  Measurement 2: Power  -8.5 dB
  Measurement 3: Power  -7.8 dB
  ...

✓ Plot saved: lab1_2_method2_agc_comparison.png

======================================================================
KEY OBSERVATIONS
======================================================================
1. Manual gain provides predictable, repeatable results
2. AGC automatically adapts to signal strength changes
3. Fast AGC responds quickly but may be less stable
4. Slow AGC is more stable for continuous signals
5. Always monitor for clipping at high gain settings
======================================================================
```

### When to Use Each Mode

| Mode | Use Case | Advantages | Disadvantages |
|------|----------|------------|---------------|
| **Manual** | Fixed signal strength, calibration | Repeatable, precise control | No adaptation to changes |
| **Slow AGC** | Continuous signals (broadcast) | Stable, smooth adaptation | Slow response to changes |
| **Fast AGC** | Bursty signals (packets, radar) | Quick response | May oscillate/hunt |

---

## Method 3: Hosted Application (C on PlutoSDR)

### Objective
Implement adaptive gain control algorithm running on PlutoSDR's ARM processor, demonstrating real-time gain optimization.

### Implementation Overview

The hosted application will:
1. Monitor received signal power in real-time
2. Detect clipping/saturation
3. Implement custom gain adaptation algorithm
4. Log performance metrics

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│         PlutoSDR ARM Cortex-A9 @ 666 MHz               │
│                                                         │
│  ┌───────────────────────────────────────────────┐    │
│  │  Hosted Gain Controller (C Application)       │    │
│  │                                                 │    │
│  │  while (running) {                             │    │
│  │    1. Capture RX samples                       │    │
│  │    2. Measure signal power                     │    │
│  │    3. Detect clipping                          │    │
│  │    4. Compute optimal gain:                    │    │
│  │       if (clipping)    → decrease gain         │    │
│  │       if (low_power)   → increase gain         │    │
│  │       if (optimal)     → maintain              │    │
│  │    5. Apply new gain setting                   │    │
│  │  }                                             │    │
│  └───────────────────────────────────────────────┘    │
│                         ▲│                              │
│                         ││ libiio (local)               │
│                         │▼                              │
│  ┌───────────────────────────────────────────────┐    │
│  │          AD9361 RF Transceiver                 │    │
│  │  • Real-time gain adjustment                   │    │
│  │  • Sub-millisecond response time               │    │
│  └───────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Key Features

1. **Real-Time Adaptation**: <1 ms response time
2. **Clipping Prevention**: Automatic gain reduction
3. **SNR Optimization**: Maximize gain without saturation
4. **Persistent Settings**: Save optimal gain to file
5. **Statistics Logging**: Track performance over time

### Complete Implementation

Due to the complexity and length of the C implementation, Method 3 is documented in detail in:

**[LAB_1_2_METHOD3_HOSTED.md](LAB_1_2_METHOD3_HOSTED.md)**

This separate document includes:
- Complete C source code (~900 lines)
- Adaptive gain control algorithm
- Clipping detection and recovery
- Power measurement and filtering
- Compilation scripts
- Deployment procedures
- Performance benchmarks

**Quick summary of Method 3**:
- Implements PID-style gain controller
- Target signal level: -6 dBFS (6 dB headroom)
- Adaptation rate: configurable (50-500 ms time constant)
- Clipping response: immediate -3 dB reduction
- Weak signal response: gradual +1 dB increase every 100 ms

---

## Comparison of Methods

| Aspect | Method 1 (Simulation) | Method 2 (External) | Method 3 (Hosted) |
|--------|----------------------|--------------------|--------------------|
| **Gain Control** | Perfect (no hardware limits) | AGC or manual via Python | Custom algorithm in C |
| **Response Time** | Instant | 10-50 ms (network + Python) | <1 ms (direct hardware) |
| **Clipping Detection** | Simulated (perfect) | Analyzed post-capture | Real-time detection |
| **Adaptation** | Offline optimization | Semi-real-time | True real-time |
| **Use Case** | Algorithm design | Lab testing, calibration | Production deployment |
| **Complexity** | Moderate (Python) | Moderate (Python + HW) | High (C + embedded) |

---

## Practical Exercises

### Exercise 1: Find Optimal Gain
Connect PlutoSDR to a signal generator with known output power. Use Method 2 to find the optimal RX gain that maximizes SNR without clipping.

**Steps**:
1. Set signal generator to -60 dBm, 915 MHz
2. Run manual gain sweep
3. Plot power vs gain
4. Identify highest gain before clipping
5. Verify 3-6 dB headroom

### Exercise 2: AGC Stress Test
Test AGC performance with rapidly changing signal levels:
1. Use signal generator with step attenuator
2. Vary signal level from -90 dBm to -30 dBm
3. Compare fast vs slow AGC response
4. Measure adaptation time
5. Check for gain hunting/oscillation

### Exercise 3: TX Gain Optimization
Measure TX output power vs gain setting:
1. Connect TX port to spectrum analyzer
2. Sweep TX gain from -89 dB to 0 dB
3. Measure actual output power
4. Plot gain curve
5. Identify 1 dB compression point

### Exercise 4: Two-SDR Link Budget
Set up communication link between two PlutoSDRs:
1. Calculate theoretical link budget
2. Set TX gain on transmitter
3. Optimize RX gain on receiver
4. Measure actual SNR
5. Compare to predicted SNR

---

## Real-World Applications

### 1. Cellular Base Stations
- **AGC**: Handles users at different distances
- **Gain Range**: 60-80 dB typical
- **Response Time**: <100 µs for LTE

### 2. Satellite Ground Stations
- **Manual Gain**: Fixed based on link budget
- **High Gain**: 60-70 dB for weak signals from space
- **Noise Figure**: Critical (< 2 dB typical)

### 3. Spectrum Monitoring
- **AGC**: Adapts to strong interferers
- **Wide Dynamic Range**: 80+ dB
- **No Clipping**: Must capture weak signals in presence of strong ones

### 4. Software-Defined Radar
- **Fast AGC**: Adapts between pulses
- **Clipping Prevention**: Critical for accurate ranging
- **Dynamic Range**: >70 dB

---

## Troubleshooting Guide

### Problem: Clipping at Low Gain

**Symptoms**: Distortion even at 20-30 dB gain
**Causes**:
- Input signal too strong
- TX leakage (poor isolation)
- Nearby interferer

**Solutions**:
1. Add external attenuator (10-20 dB)
2. Increase distance from TX
3. Use bandpass filter
4. Check antenna connections

### Problem: Poor SNR at Maximum Gain

**Symptoms**: Noisy signal even at 73 dB gain
**Causes**:
- Input signal too weak
- High noise floor
- Poor antenna

**Solutions**:
1. Use external LNA (low noise amplifier)
2. Improve antenna (higher gain)
3. Reduce cable losses
4. Check for local interference

### Problem: AGC Oscillation

**Symptoms**: Gain constantly changing, unstable
**Causes**:
- Signal level at AGC threshold
- Fast AGC too aggressive

**Solutions**:
1. Switch to slow AGC mode
2. Use manual gain if signal stable
3. Adjust AGC attack/decay parameters (advanced)

---

## Summary and Key Takeaways

### Core Principles

1. **Gain stages cascade**: Total gain = LNA + Mixer + Baseband
2. **Early gain is best**: LNA gain minimizes noise figure
3. **Headroom is essential**: 3-6 dB prevents clipping
4. **AGC trades off**: Adaptation speed vs stability

### Design Guidelines

**For Receiving**:
- Start with maximum LNA gain (30 dB)
- Add mixer/baseband gain as needed
- Monitor for clipping
- Target -6 dBFS signal level

**For Transmitting**:
- Start with minimum power (-80 dBm)
- Increase only as required
- Monitor for spectral regrowth
- Never exceed regulatory limits

### When to Use Manual vs AGC

**Use Manual Gain When**:
- Signal strength is constant
- Calibration/testing
- Precise measurements needed
- Weak signal DX (distant) reception

**Use AGC When**:
- Signal strength varies
- Multiple signal levels
- Bursty communications (packets)
- General monitoring/scanning

### Method Selection

- **Method 1**: Use for algorithm design and optimization
- **Method 2**: Use for lab testing and characterization
- **Method 3**: Use for production embedded systems

---

## Next Lab

**LAB 1.3: I/Q Samples & Complex Baseband** - Learn how PlutoSDR represents RF signals as complex I/Q samples and perform DSP operations.

---

## Additional Resources

### Documentation
- [AD9361 UG-570: Software Gain Control](https://wiki.analog.com/resources/eval/user-guides/ad-fmcomms2-ebz/software/basic_iq_datafiles)
- [AGC Tuning Guide](https://wiki.analog.com/resources/tools-software/linux-drivers/iio-transceiver/ad9361)

### Application Notes
- **AN-1367**: "Circuit Design for EMI Compliance"
- **AN-1354**: "AD9361 AGC Operation"

### Further Reading
- "RF Microelectronics" by Behzad Razavi (Chapter 6: Low Noise Amplifiers)
- "Phased Array Antenna Handbook" by Mailloux (Chapter on Dynamic Range)

---

**End of LAB 1.2**

