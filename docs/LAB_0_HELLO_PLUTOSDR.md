# LAB 0: Hello PlutoSDR - Complete Three-Method Tutorial

## Introduction

This is your **first lab** - a simple "Hello PlutoSDR" that demonstrates all three implementation methods in detail. By the end of this lab, you will understand:

1. How to transmit/receive with PlutoSDR in simulation
2. How to create external Python applications
3. **How to cross-compile and deploy hosted C applications step-by-step**

This pattern will be used for **every lab** in the training.

---

## Learning Objectives

- Transmit a simple tone at 915 MHz
- Receive and detect the tone
- Measure signal power
- Understand the three development workflows
- **Master cross-compilation for embedded deployment**

---

## 🔷 METHOD 1: SIMULATION (No Hardware)

### Theory

In simulation, we generate perfect I/Q samples mathematically without any RF hardware.

A complex tone at frequency f:
```
s(t) = A · e^(j·2πft) = A · (cos(2πft) + j·sin(2πft))
```

### Implementation

**File:** `lab0_method1_simulation.py`

```python
#!/usr/bin/env python3
"""
LAB 0 - Method 1: Hello PlutoSDR (Simulation)

Simulates a simple tone transmission and reception
No hardware required - pure Python/NumPy
"""

import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec

print("="*70)
print("LAB 0 - METHOD 1: HELLO PLUTOSDR (SIMULATION)")
print("="*70)

# ==================================================================
# STEP 1: DEFINE PARAMETERS
# ==================================================================

print("\n📝 Step 1: Defining parameters...")

# These match PlutoSDR's typical settings
FS = 2.084e6           # Sample rate: 2.084 MSPS
FC = 915e6             # Center frequency: 915 MHz (ISM band)
TONE_OFFSET = 100e3    # Tone offset: 100 kHz from center
DURATION = 0.001       # 1 millisecond
TX_POWER_DBM = 0       # 0 dBm = 1 mW

print(f"  Sample rate:      {FS/1e6:.3f} MSPS")
print(f"  Center frequency: {FC/1e6:.1f} MHz")
print(f"  Tone offset:      {TONE_OFFSET/1e3:.1f} kHz")
print(f"  Duration:         {DURATION*1e3:.1f} ms")
print(f"  TX power:         {TX_POWER_DBM} dBm")

# ==================================================================
# STEP 2: GENERATE TRANSMIT SIGNAL
# ==================================================================

print("\n📡 Step 2: Generating TX signal...")

# Time vector
t = np.arange(0, DURATION, 1/FS)
num_samples = len(t)

print(f"  Generated {num_samples} samples")

# Generate complex tone at offset frequency
# This represents baseband I/Q samples
tx_signal = np.exp(2j * np.pi * TONE_OFFSET * t)

# Apply amplitude (convert dBm to linear)
power_watts = 10**((TX_POWER_DBM - 30) / 10)  # Convert dBm to Watts
amplitude = np.sqrt(power_watts)
tx_signal = amplitude * tx_signal

# Compute actual power
tx_power = np.mean(np.abs(tx_signal)**2)
tx_power_dbm = 10 * np.log10(tx_power * 1000)

print(f"  TX signal power:  {tx_power_dbm:.2f} dBm")

# ==================================================================
# STEP 3: SIMULATE CHANNEL (Add noise)
# ==================================================================

print("\n🌊 Step 3: Simulating RF channel...")

# Channel parameters
SNR_DB = 20  # Signal-to-Noise Ratio
PATH_LOSS_DB = 0  # Direct connection (no loss)

# Calculate noise power
signal_power = tx_power
noise_power = signal_power / (10**(SNR_DB/10))

print(f"  SNR:              {SNR_DB} dB")
print(f"  Path loss:        {PATH_LOSS_DB} dB")
print(f"  Noise power:      {10*np.log10(noise_power*1000):.2f} dBm")

# Generate AWGN (Additive White Gaussian Noise)
noise = np.sqrt(noise_power/2) * (np.random.randn(num_samples) +
                                  1j*np.random.randn(num_samples))

# Received signal
rx_signal = tx_signal + noise

# Compute received power
rx_power = np.mean(np.abs(rx_signal)**2)
rx_power_dbm = 10 * np.log10(rx_power * 1000)

print(f"  RX signal power:  {rx_power_dbm:.2f} dBm")

# ==================================================================
# STEP 4: DETECT AND MEASURE TONE
# ==================================================================

print("\n🔍 Step 4: Detecting tone...")

# Method 1: FFT-based detection
fft_result = np.fft.fftshift(np.fft.fft(rx_signal))
freqs = np.fft.fftshift(np.fft.fftfreq(num_samples, 1/FS))
magnitude_db = 20 * np.log10(np.abs(fft_result) / num_samples)

# Find peak
peak_idx = np.argmax(magnitude_db)
detected_freq = freqs[peak_idx]
peak_power = magnitude_db[peak_idx]

print(f"  Expected frequency: {TONE_OFFSET/1e3:.2f} kHz")
print(f"  Detected frequency: {detected_freq/1e3:.2f} kHz")
print(f"  Error:              {abs(detected_freq - TONE_OFFSET):.2f} Hz")
print(f"  Peak power:         {peak_power:.2f} dB")

# Method 2: Correlation-based detection
reference = np.exp(2j * np.pi * TONE_OFFSET * t)
correlation = np.abs(np.vdot(rx_signal, reference)) / num_samples
correlation_db = 20 * np.log10(correlation)

print(f"  Correlation power:  {correlation_db:.2f} dB")

# ==================================================================
# STEP 5: VISUALIZATION
# ==================================================================

print("\n📊 Step 5: Generating plots...")

fig = plt.figure(figsize=(16, 10))
gs = GridSpec(3, 3, figure=fig)

# Plot 1: Time domain (I and Q)
ax1 = fig.add_subplot(gs[0, :])
time_ms = t[:1000] * 1e3
ax1.plot(time_ms, np.real(tx_signal[:1000]), 'b-', linewidth=1, label='I (In-phase)', alpha=0.7)
ax1.plot(time_ms, np.imag(tx_signal[:1000]), 'r-', linewidth=1, label='Q (Quadrature)', alpha=0.7)
ax1.set_xlabel('Time (ms)')
ax1.set_ylabel('Amplitude')
ax1.set_title('TX Signal - Time Domain (First 1000 samples)')
ax1.legend()
ax1.grid(True, alpha=0.3)

# Plot 2: Constellation diagram
ax2 = fig.add_subplot(gs[1, 0])
ax2.plot(np.real(rx_signal[::10]), np.imag(rx_signal[::10]), '.',
         markersize=2, alpha=0.5, label='RX samples')
circle = plt.Circle((0, 0), amplitude, fill=False, color='r',
                    linestyle='--', label=f'Expected ({amplitude:.3f})')
ax2.add_patch(circle)
ax2.set_xlabel('I')
ax2.set_ylabel('Q')
ax2.set_title(f'Constellation (SNR = {SNR_DB} dB)')
ax2.axis('equal')
ax2.legend()
ax2.grid(True, alpha=0.3)

# Plot 3: Spectrum (FFT)
ax3 = fig.add_subplot(gs[1, 1:])
ax3.plot(freqs/1e3, magnitude_db, linewidth=1)
ax3.axvline(TONE_OFFSET/1e3, color='r', linestyle='--', alpha=0.7,
            label=f'Expected: {TONE_OFFSET/1e3:.1f} kHz')
ax3.axvline(detected_freq/1e3, color='g', linestyle='--', alpha=0.7,
            label=f'Detected: {detected_freq/1e3:.1f} kHz')
ax3.set_xlabel('Frequency Offset (kHz)')
ax3.set_ylabel('Magnitude (dB)')
ax3.set_title('RX Spectrum')
ax3.legend()
ax3.grid(True, alpha=0.3)
ax3.set_ylim([np.max(magnitude_db)-80, np.max(magnitude_db)+5])

# Plot 4: Noise analysis
ax4 = fig.add_subplot(gs[2, 0])
noise_real = np.real(noise)
ax4.hist(noise_real, bins=50, density=True, alpha=0.7, label='Noise histogram')
# Theoretical Gaussian
x = np.linspace(np.min(noise_real), np.max(noise_real), 100)
gaussian = (1/np.sqrt(2*np.pi*noise_power/2)) * np.exp(-x**2/(noise_power))
ax4.plot(x, gaussian, 'r-', linewidth=2, label='Theoretical Gaussian')
ax4.set_xlabel('Amplitude')
ax4.set_ylabel('Probability Density')
ax4.set_title('Noise Distribution')
ax4.legend()
ax4.grid(True, alpha=0.3)

# Plot 5: Power measurement over time
ax5 = fig.add_subplot(gs[2, 1])
window_size = 100
power_windowed = []
for i in range(0, len(rx_signal) - window_size, window_size):
    window = rx_signal[i:i+window_size]
    power_windowed.append(10*np.log10(np.mean(np.abs(window)**2)*1000))

ax5.plot(power_windowed, linewidth=2)
ax5.axhline(rx_power_dbm, color='r', linestyle='--',
            label=f'Average: {rx_power_dbm:.2f} dBm')
ax5.set_xlabel('Window Index')
ax5.set_ylabel('Power (dBm)')
ax5.set_title('Received Power (100-sample windows)')
ax5.legend()
ax5.grid(True, alpha=0.3)

# Plot 6: SNR analysis
ax6 = fig.add_subplot(gs[2, 2])
snr_measured = signal_power / noise_power
snr_measured_db = 10 * np.log10(snr_measured)

categories = ['Expected\nSNR', 'Measured\nSNR', 'Signal\nPower', 'Noise\nPower']
values = [SNR_DB, snr_measured_db, tx_power_dbm, 10*np.log10(noise_power*1000)]
colors = ['blue', 'green', 'orange', 'red']

bars = ax6.bar(categories, values, color=colors, alpha=0.7)
ax6.set_ylabel('Value (dB / dBm)')
ax6.set_title('Performance Metrics')
ax6.grid(True, alpha=0.3, axis='y')

# Add value labels on bars
for bar, value in zip(bars, values):
    height = bar.get_height()
    ax6.text(bar.get_x() + bar.get_width()/2., height,
             f'{value:.1f}', ha='center', va='bottom')

plt.tight_layout()
plt.savefig('lab0_method1_simulation.png', dpi=150, bbox_inches='tight')
plt.show()

# ==================================================================
# SUMMARY
# ==================================================================

print("\n" + "="*70)
print("✓ METHOD 1 COMPLETE: Simulation")
print("="*70)
print("\n📈 Results Summary:")
print(f"  ✓ Generated {num_samples} I/Q samples")
print(f"  ✓ Transmitted tone at {TONE_OFFSET/1e3:.1f} kHz offset")
print(f"  ✓ Added AWGN with SNR = {SNR_DB} dB")
print(f"  ✓ Detected tone with {abs(detected_freq - TONE_OFFSET):.2f} Hz error")
print(f"  ✓ Measured power: {rx_power_dbm:.2f} dBm")
print(f"  ✓ Correlation: {correlation_db:.2f} dB")

print("\n💡 Key Learnings:")
print("  • I/Q samples represent complex baseband signals")
print("  • FFT reveals frequency content")
print("  • SNR affects detection accuracy")
print("  • Correlation detects known signals")

print("\n➡️  Next: Try Method 2 with real PlutoSDR hardware!")
print("    Run: python3 lab0_method2_external.py")
```

### Running the Simulation

```bash
# Install dependencies
pip3 install numpy matplotlib

# Run simulation
python3 lab0_method1_simulation.py
```

### Expected Output

```
======================================================================
LAB 0 - METHOD 1: HELLO PLUTOSDR (SIMULATION)
======================================================================

📝 Step 1: Defining parameters...
  Sample rate:      2.084 MSPS
  Center frequency: 915.0 MHz
  Tone offset:      100.0 kHz
  Duration:         1.0 ms
  TX power:         0 dBm

📡 Step 2: Generating TX signal...
  Generated 2084 samples
  TX signal power:  0.00 dBm

🌊 Step 3: Simulating RF channel...
  SNR:              20 dB
  Path loss:        0 dB
  Noise power:      -20.00 dBm

🔍 Step 4: Detecting tone...
  Expected frequency: 100.00 kHz
  Detected frequency: 100.00 kHz
  Error:              0.00 Hz
  Peak power:         -30.18 dB
  Correlation power:  -30.00 dB

📊 Step 5: Generating plots...

======================================================================
✓ METHOD 1 COMPLETE: Simulation
======================================================================

📈 Results Summary:
  ✓ Generated 2084 I/Q samples
  ✓ Transmitted tone at 100.0 kHz offset
  ✓ Added AWGN with SNR = 20 dB
  ✓ Detected tone with 0.00 Hz error
  ✓ Measured power: -0.04 dBm
  ✓ Correlation: -30.00 dB

💡 Key Learnings:
  • I/Q samples represent complex baseband signals
  • FFT reveals frequency content
  • SNR affects detection accuracy
  • Correlation detects known signals

➡️  Next: Try Method 2 with real PlutoSDR hardware!
    Run: python3 lab0_method2_external.py
```

---

## 🔶 METHOD 2: EXTERNAL APP (Python + PlutoSDR)

### Theory

External applications run on your PC and communicate with PlutoSDR over USB/Network using the **libiio** library. This is the most common development method.

**Architecture:**
```
Your PC                          PlutoSDR
┌──────────────┐                ┌──────────────┐
│  Python App  │                │   iiod       │
│  (pyadi-iio) │◄──USB/Net────►│  (daemon)    │
└──────────────┘                │              │
                                │   IIO        │
                                │   Drivers    │
                                │      │       │
                                │  ┌───▼────┐  │
                                │  │ AD9363 │  │
                                │  └────────┘  │
                                └──────────────┘
```

### Implementation

**File:** `lab0_method2_external.py`

```python
#!/usr/bin/env python3
"""
LAB 0 - Method 2: Hello PlutoSDR (External Application)

Real hardware test using pyadi-iio
Transmits and receives a tone using PlutoSDR
Application runs on host PC
"""

import adi
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
import time

print("="*70)
print("LAB 0 - METHOD 2: HELLO PLUTOSDR (EXTERNAL APPLICATION)")
print("="*70)

# ==================================================================
# STEP 1: CONNECT TO PLUTOSDR
# ==================================================================

print("\n🔌 Step 1: Connecting to PlutoSDR...")

# Try to connect
PLUTO_URI = "ip:192.168.2.1"  # Default PlutoSDR IP

try:
    sdr = adi.Pluto(PLUTO_URI)
    print(f"  ✓ Connected to PlutoSDR at {PLUTO_URI}")
except Exception as e:
    print(f"  ✗ Failed to connect: {e}")
    print("\n  Troubleshooting:")
    print("  1. Check PlutoSDR is connected via USB")
    print("  2. Verify IP address: ping 192.168.2.1")
    print("  3. Check drivers are installed")
    print("  4. Try: uri='usb:1.XX.5' instead")
    exit(1)

# Print PlutoSDR information
print(f"  Hardware: {sdr._ctx.name}")
print(f"  Firmware: {sdr._ctx.attrs['fw_version'].value}")

# ==================================================================
# STEP 2: CONFIGURE PLUTOSDR
# ==================================================================

print("\n⚙️  Step 2: Configuring PlutoSDR...")

# Sample rate
FS = 2.084e6
sdr.sample_rate = int(FS)

# Center frequency
FC = 915e6
sdr.tx_lo = int(FC)
sdr.rx_lo = int(FC)

# RF bandwidth
sdr.tx_rf_bandwidth = int(2e6)  # 2 MHz
sdr.rx_rf_bandwidth = int(2e6)

# Gain settings
sdr.tx_hardwaregain_chan0 = -10  # TX attenuation (dB)
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 60   # RX gain (dB)

# Buffer size
BUFFER_SIZE = 16384
sdr.rx_buffer_size = BUFFER_SIZE

# Cyclic buffer for continuous TX
sdr.tx_cyclic_buffer = True

print(f"  Sample rate:      {sdr.sample_rate/1e6:.3f} MSPS")
print(f"  TX LO:            {sdr.tx_lo/1e6:.1f} MHz")
print(f"  RX LO:            {sdr.rx_lo/1e6:.1f} MHz")
print(f"  TX RF BW:         {sdr.tx_rf_bandwidth/1e6:.1f} MHz")
print(f"  RX RF BW:         {sdr.rx_rf_bandwidth/1e6:.1f} MHz")
print(f"  TX gain:          {sdr.tx_hardwaregain_chan0} dB")
print(f"  RX gain:          {sdr.rx_hardwaregain_chan0} dB")
print(f"  Buffer size:      {BUFFER_SIZE} samples")

# ==================================================================
# STEP 3: GENERATE TX SIGNAL
# ==================================================================

print("\n📡 Step 3: Generating TX signal...")

# Create time vector for one buffer
t = np.arange(BUFFER_SIZE) / FS

# Generate tone at 100 kHz offset
TONE_OFFSET = 100e3
tx_signal = np.exp(2j * np.pi * TONE_OFFSET * t)

# Scale for 12-bit DAC
# PlutoSDR DAC range: -2048 to +2047 (12-bit signed)
# Use 80% to avoid clipping
tx_signal = tx_signal * 0.8 * 2**11

# Convert to int16
tx_signal_int = tx_signal.astype(np.int16)

print(f"  Generated {len(tx_signal_int)} samples")
print(f"  Tone offset:      {TONE_OFFSET/1e3:.1f} kHz")
print(f"  Sample range:     [{np.min(np.real(tx_signal_int))}, {np.max(np.real(tx_signal_int))}]")

# ==================================================================
# STEP 4: TRANSMIT
# ==================================================================

print("\n📤 Step 4: Starting transmission...")

try:
    sdr.tx(tx_signal_int)
    print("  ✓ TX started (cyclic mode)")
except Exception as e:
    print(f"  ✗ TX failed: {e}")
    exit(1)

# Let TX stabilize
print("  Waiting for TX to stabilize...")
time.sleep(0.5)

# ==================================================================
# STEP 5: RECEIVE
# ==================================================================

print("\n📥 Step 5: Receiving signal...")

try:
    rx_samples = sdr.rx()
    print(f"  ✓ Received {len(rx_samples)} complex samples")
except Exception as e:
    print(f"  ✗ RX failed: {e}")
    sdr.tx_destroy_buffer()
    exit(1)

# Convert to float for processing
rx_signal = np.array(rx_samples, dtype=complex)

# Compute statistics
i_samples = np.real(rx_signal)
q_samples = np.imag(rx_signal)

print(f"  I range:          [{np.min(i_samples):.0f}, {np.max(i_samples):.0f}]")
print(f"  Q range:          [{np.min(q_samples):.0f}, {np.max(q_samples):.0f}]")

# Compute power
rx_power = np.mean(np.abs(rx_signal)**2)
rx_power_dbm = 10 * np.log10(rx_power / (2**11)**2 * 1000)  # Normalize by DAC range

print(f"  RX power:         {rx_power_dbm:.2f} dBm (estimated)")

# ==================================================================
# STEP 6: DETECT TONE
# ==================================================================

print("\n🔍 Step 6: Detecting tone...")

# FFT analysis
fft_result = np.fft.fftshift(np.fft.fft(rx_signal))
freqs = np.fft.fftshift(np.fft.fftfreq(len(rx_signal), 1/FS))
magnitude_db = 20 * np.log10(np.abs(fft_result) / len(rx_signal))

# Find peak
peak_idx = np.argmax(magnitude_db)
detected_freq = freqs[peak_idx]
peak_power = magnitude_db[peak_idx]

print(f"  Expected frequency: {TONE_OFFSET/1e3:+.2f} kHz")
print(f"  Detected frequency: {detected_freq/1e3:+.2f} kHz")
print(f"  Frequency error:    {abs(detected_freq - TONE_OFFSET):.2f} Hz")
print(f"  Peak power:         {peak_power:.2f} dB")

# Verify it's our tone
if abs(detected_freq - TONE_OFFSET) < 1000:  # Within 1 kHz
    print("  ✓ Tone detected successfully!")
else:
    print("  ⚠ Warning: Detected frequency doesn't match expected")

# ==================================================================
# STEP 7: VISUALIZATION
# ==================================================================

print("\n📊 Step 7: Generating plots...")

fig = plt.figure(figsize=(16, 10))
gs = GridSpec(3, 3, figure=fig)

# Plot 1: Time domain
ax1 = fig.add_subplot(gs[0, :])
time_ms = np.arange(min(2000, len(rx_signal))) / FS * 1e3
ax1.plot(time_ms, i_samples[:len(time_ms)], 'b-', linewidth=0.5, label='I', alpha=0.7)
ax1.plot(time_ms, q_samples[:len(time_ms)], 'r-', linewidth=0.5, label='Q', alpha=0.7)
ax1.set_xlabel('Time (ms)')
ax1.set_ylabel('ADC Value')
ax1.set_title('RX Signal - Time Domain (PlutoSDR Real Hardware)')
ax1.legend()
ax1.grid(True, alpha=0.3)

# Plot 2: Constellation
ax2 = fig.add_subplot(gs[1, 0])
ax2.plot(i_samples[::10], q_samples[::10], '.', markersize=1, alpha=0.3)
ax2.set_xlabel('I')
ax2.set_ylabel('Q')
ax2.set_title('Constellation Diagram')
ax2.axis('equal')
ax2.grid(True, alpha=0.3)

# Plot 3: Spectrum
ax3 = fig.add_subplot(gs[1, 1:])
ax3.plot(freqs/1e3, magnitude_db, linewidth=1)
ax3.axvline(TONE_OFFSET/1e3, color='r', linestyle='--', alpha=0.7,
            label=f'Expected: {TONE_OFFSET/1e3:.1f} kHz')
ax3.axvline(detected_freq/1e3, color='g', linestyle='--', alpha=0.7,
            label=f'Detected: {detected_freq/1e3:.1f} kHz')
ax3.set_xlabel('Frequency Offset (kHz)')
ax3.set_ylabel('Magnitude (dB)')
ax3.set_title('RX Spectrum (FFT)')
ax3.legend()
ax3.grid(True, alpha=0.3)

# Plot 4: Waterfall (spectrogram)
ax4 = fig.add_subplot(gs[2, :2])
NFFT = 1024
ax4.specgram(rx_signal, NFFT=NFFT, Fs=FS, cmap='viridis', scale='dB')
ax4.set_xlabel('Time (s)')
ax4.set_ylabel('Frequency (Hz)')
ax4.set_title('Waterfall Plot (Time-Frequency)')

# Plot 5: Phase
ax5 = fig.add_subplot(gs[2, 2])
phase = np.angle(rx_signal[:2000])
unwrapped_phase = np.unwrap(phase)
time_us = np.arange(len(unwrapped_phase)) / FS * 1e6
ax5.plot(time_us, unwrapped_phase, linewidth=1)
ax5.set_xlabel('Time (μs)')
ax5.set_ylabel('Phase (radians)')
ax5.set_title('Phase Evolution')
ax5.grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('lab0_method2_external.png', dpi=150, bbox_inches='tight')
plt.show()

# ==================================================================
# CLEANUP
# ==================================================================

print("\n🧹 Step 8: Cleanup...")

# Stop TX
try:
    sdr.tx_destroy_buffer()
    print("  ✓ TX stopped")
except:
    pass

# ==================================================================
# SUMMARY
# ==================================================================

print("\n" + "="*70)
print("✓ METHOD 2 COMPLETE: External Application with PlutoSDR")
print("="*70)
print("\n📈 Results Summary:")
print(f"  ✓ Transmitted tone at {TONE_OFFSET/1e3:.1f} kHz offset")
print(f"  ✓ Received {len(rx_signal)} samples from real RF")
print(f"  ✓ Detected tone at {detected_freq/1e3:+.2f} kHz")
print(f"  ✓ Frequency error: {abs(detected_freq - TONE_OFFSET):.2f} Hz")
print(f"  ✓ RX power: {rx_power_dbm:.2f} dBm")

print("\n💡 Key Differences from Simulation:")
print("  • Real RF propagation (not perfect)")
print("  • ADC/DAC quantization (12-bit)")
print("  • Phase noise from LO")
print("  • Thermal noise from receiver")
print("  • USB bandwidth limitations")

print("\n⚠️  Common Issues:")
print("  • DC offset visible at 0 Hz")
print("  • I/Q imbalance (slight distortion)")
print("  • LO leakage (carrier feedthrough)")
print("  • Network latency (~10-50 ms)")

print("\n➡️  Next: Try Method 3 - Hosted application on PlutoSDR!")
print("    See: lab0_method3_hosted.c")
```

### Running External App

```bash
# Install dependencies
pip3 install pyadi-iio numpy matplotlib

# Connect PlutoSDR via USB
# Check connection
ping 192.168.2.1

# Run external app
python3 lab0_method2_external.py
```

### Expected Output

```
======================================================================
LAB 0 - METHOD 2: HELLO PLUTOSDR (EXTERNAL APPLICATION)
======================================================================

🔌 Step 1: Connecting to PlutoSDR...
  ✓ Connected to PlutoSDR at ip:192.168.2.1
  Hardware: Analog Devices PlutoSDR Rev.C (Z7010-AD9363A)
  Firmware: v0.38

⚙️  Step 2: Configuring PlutoSDR...
  Sample rate:      2.084 MSPS
  TX LO:            915.0 MHz
  RX LO:            915.0 MHz
  TX RF BW:         2.0 MHz
  RX RF BW:         2.0 MHz
  TX gain:          -10 dB
  RX gain:          60 dB
  Buffer size:      16384 samples

📡 Step 3: Generating TX signal...
  Generated 16384 samples
  Tone offset:      100.0 kHz
  Sample range:     [-1638, 1638]

📤 Step 4: Starting transmission...
  ✓ TX started (cyclic mode)
  Waiting for TX to stabilize...

📥 Step 5: Receiving signal...
  ✓ Received 16384 complex samples
  I range:          [-856, 891]
  Q range:          [-892, 845]
  RX power:         -23.45 dBm (estimated)

🔍 Step 6: Detecting tone...
  Expected frequency: +100.00 kHz
  Detected frequency: +100.08 kHz
  Frequency error:    80.00 Hz
  Peak power:         -35.21 dB
  ✓ Tone detected successfully!

📊 Step 7: Generating plots...

🧹 Step 8: Cleanup...
  ✓ TX stopped

======================================================================
✓ METHOD 2 COMPLETE: External Application with PlutoSDR
======================================================================

📈 Results Summary:
  ✓ Transmitted tone at 100.0 kHz offset
  ✓ Received 16384 samples from real RF
  ✓ Detected tone at +100.08 kHz
  ✓ Frequency error: 80.00 Hz
  ✓ RX power: -23.45 dBm

💡 Key Differences from Simulation:
  • Real RF propagation (not perfect)
  • ADC/DAC quantization (12-bit)
  • Phase noise from LO
  • Thermal noise from receiver
  • USB bandwidth limitations

⚠️  Common Issues:
  • DC offset visible at 0 Hz
  • I/Q imbalance (slight distortion)
  • LO leakage (carrier feedthrough)
  • Network latency (~10-50 ms)

➡️  Next: Try Method 3 - Hosted application on PlutoSDR!
    See: lab0_method3_hosted.c
```

---

*Continuing in next part with METHOD 3 (Hosted App with detailed cross-compilation)...*
