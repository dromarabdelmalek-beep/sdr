# PlutoSDR Three-Method Training Approach

## Training Methodology

Each lab is presented in **three progressive implementations**:

### 🔷 Method 1: SIMULATION (Python/NumPy)
- **Location:** Runs entirely on host PC
- **Hardware:** No SDR required
- **Purpose:** Learn fundamental concepts, algorithm development
- **Tools:** Python, NumPy, SciPy, Matplotlib
- **Pros:** Fast iteration, easy debugging, perfect signals
- **Cons:** No real RF effects (noise, interference, multipath)

### 🔶 Method 2: EXTERNAL APP (Python + PlutoSDR via Network)
- **Location:** Application runs on host PC, RF on PlutoSDR
- **Hardware:** PlutoSDR connected via USB/Network
- **Purpose:** Real RF testing with easy development
- **Tools:** pyadi-iio, libiio, SoapySDR, or UHD
- **Pros:** Full PC resources, GUI possible, easy debugging
- **Cons:** Network latency, requires host connection

### 🔴 Method 3: HOSTED APP (C/C++ on PlutoSDR)
- **Location:** Application runs on PlutoSDR ARM CPU
- **Hardware:** Standalone PlutoSDR
- **Purpose:** Embedded deployment, low latency, standalone operation
- **Tools:** Cross-compilation toolchain, libiio local
- **Pros:** Standalone, low latency, no PC needed
- **Cons:** Limited resources, harder debugging

---

## Learning Progression

```
┌─────────────────────────────────────────────────────────────┐
│  LAB PROGRESSION: Simulation → External → Hosted            │
└─────────────────────────────────────────────────────────────┘

Week 1-2: Method 1 (Simulation)
├── Understand theory
├── Develop algorithms
├── Visualize results
└── Perfect implementation

Week 3-4: Method 2 (External App)
├── Real RF testing
├── Handle impairments
├── Optimize for real signals
└── Network integration

Week 5-6: Method 3 (Hosted App)
├── Cross-compile
├── Optimize for embedded
├── Deploy standalone
└── Production ready

Final Projects (Week 7-10)
├── Satellite Communication System
├── IoT Satellite Hub (10k devices)
└── Video Transmission Link
```

---

## Example: Complete Lab with All Three Methods

# LAB 1.3: I/Q Samples and Complex Baseband

## 📚 Theory

**Objective:** Understand I/Q (In-phase/Quadrature) sampling and complex baseband representation.

**Concepts:**
- Complex signal representation: `s(t) = I(t) + j·Q(t)`
- I component: In-phase (0° reference)
- Q component: Quadrature (90° phase shift)
- Why I/Q: Distinguishes positive/negative frequencies
- Complex baseband: Centered at DC, not at RF carrier

**Mathematical Foundation:**
```
Real bandpass signal at fc:
  s(t) = A·cos(2πfct + φ)

Complex baseband equivalent:
  s_bb(t) = A·e^(jφ) = A·cos(φ) + j·A·sin(φ)
           = I(t) + j·Q(t)
```

---

## 🔷 METHOD 1: SIMULATION (Python Only)

**File:** `lab1_3_method1_simulation.py`

```python
#!/usr/bin/env python3
"""
LAB 1.3 - Method 1: I/Q Simulation
Pure Python simulation to understand I/Q concepts
No hardware required
"""

import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec

print("="*60)
print("LAB 1.3 - METHOD 1: I/Q SIMULATION")
print("="*60)

# Simulation parameters
fs = 2.084e6  # Sample rate (Hz) - matching PlutoSDR
duration = 0.001  # 1 ms
t = np.arange(0, duration, 1/fs)

# Generate complex signals at different frequencies
frequencies = [100e3, -100e3, 250e3]  # Hz offset from center
signals = {}

for freq in frequencies:
    # Complex exponential (natural I/Q representation)
    signal = np.exp(2j * np.pi * freq * t)

    # Extract I and Q
    i_samples = np.real(signal)
    q_samples = np.imag(signal)

    signals[freq] = {
        'complex': signal,
        'i': i_samples,
        'q': q_samples,
        'freq': freq
    }

# Create comprehensive visualization
fig = plt.figure(figsize=(16, 12))
gs = GridSpec(4, 3, figure=fig)

# Plot 1: Time domain I and Q
ax1 = fig.add_subplot(gs[0, :])
ax1.plot(t[:200]*1e6, signals[100e3]['i'][:200], 'b-', linewidth=2, label='I (In-phase)')
ax1.plot(t[:200]*1e6, signals[100e3]['q'][:200], 'r-', linewidth=2, label='Q (Quadrature)')
ax1.set_xlabel('Time (μs)')
ax1.set_ylabel('Amplitude')
ax1.set_title('I and Q Components (100 kHz tone)')
ax1.legend()
ax1.grid(True, alpha=0.3)

# Plot 2: Constellation (I vs Q)
for idx, (freq, sig) in enumerate(signals.items()):
    ax = fig.add_subplot(gs[1, idx])
    ax.plot(sig['i'][:1000], sig['q'][:1000], '.', alpha=0.1, markersize=1)

    # Draw unit circle
    theta = np.linspace(0, 2*np.pi, 100)
    ax.plot(np.cos(theta), np.sin(theta), 'k--', alpha=0.3, linewidth=1)

    ax.set_xlabel('I')
    ax.set_ylabel('Q')
    ax.set_title(f'Constellation: {freq/1e3:+.0f} kHz')
    ax.axis('equal')
    ax.grid(True, alpha=0.3)
    ax.set_xlim([-1.5, 1.5])
    ax.set_ylim([-1.5, 1.5])

# Plot 3: Frequency spectrum
for idx, (freq, sig) in enumerate(signals.items()):
    ax = fig.add_subplot(gs[2, idx])

    # Compute FFT
    fft_result = np.fft.fftshift(np.fft.fft(sig['complex']))
    freqs_axis = np.fft.fftshift(np.fft.fftfreq(len(sig['complex']), 1/fs))
    magnitude_db = 20 * np.log10(np.abs(fft_result) / len(fft_result))

    ax.plot(freqs_axis/1e3, magnitude_db, linewidth=1)
    ax.axvline(freq/1e3, color='r', linestyle='--', alpha=0.7, label=f'Expected: {freq/1e3:.0f} kHz')
    ax.set_xlabel('Frequency (kHz)')
    ax.set_ylabel('Magnitude (dB)')
    ax.set_title(f'Spectrum: {freq/1e3:+.0f} kHz tone')
    ax.grid(True, alpha=0.3)
    ax.legend()
    ax.set_ylim([-80, 0])

# Plot 4: Phase evolution
ax4 = fig.add_subplot(gs[3, 0])
phase_100k = np.angle(signals[100e3]['complex'][:1000])
unwrapped = np.unwrap(phase_100k)
ax4.plot(t[:1000]*1e6, unwrapped, linewidth=2)
ax4.set_xlabel('Time (μs)')
ax4.set_ylabel('Phase (radians)')
ax4.set_title('Phase Evolution (+100 kHz)')
ax4.grid(True, alpha=0.3)

# Plot 5: Real vs Complex spectrum comparison
ax5 = fig.add_subplot(gs[3, 1:])

# Complex signal (I+jQ)
fft_complex = np.fft.fftshift(np.fft.fft(signals[100e3]['complex']))
freqs_axis = np.fft.fftshift(np.fft.fftfreq(len(signals[100e3]['complex']), 1/fs))
mag_complex = 20 * np.log10(np.abs(fft_complex) / len(fft_complex))

# Real signal (I only, no Q)
real_only = signals[100e3]['i']
fft_real = np.fft.fftshift(np.fft.fft(real_only))
mag_real = 20 * np.log10(np.abs(fft_real) / len(fft_real))

ax5.plot(freqs_axis/1e3, mag_complex, 'b-', linewidth=2, label='Complex I/Q (unambiguous)', alpha=0.8)
ax5.plot(freqs_axis/1e3, mag_real, 'r-', linewidth=2, label='Real only (ambiguous)', alpha=0.8)
ax5.axvline(100, color='g', linestyle='--', alpha=0.5, label='+100 kHz')
ax5.axvline(-100, color='orange', linestyle='--', alpha=0.5, label='-100 kHz (mirror)')
ax5.set_xlabel('Frequency (kHz)')
ax5.set_ylabel('Magnitude (dB)')
ax5.set_title('Complex I/Q vs Real-Only Signal')
ax5.legend()
ax5.grid(True, alpha=0.3)
ax5.set_ylim([-80, 0])

plt.tight_layout()
plt.savefig('lab1_3_method1_simulation.png', dpi=150)
plt.show()

# Analysis and measurements
print("\n📊 SIMULATION RESULTS:")
print("-" * 60)

for freq, sig in signals.items():
    # Measure frequency via FFT peak
    fft_result = np.fft.fftshift(np.fft.fft(sig['complex']))
    freqs_axis = np.fft.fftshift(np.fft.fftfreq(len(sig['complex']), 1/fs))
    peak_idx = np.argmax(np.abs(fft_result))
    measured_freq = freqs_axis[peak_idx]

    # Measure rotation rate (phase derivative)
    phase = np.unwrap(np.angle(sig['complex']))
    phase_rate = np.mean(np.diff(phase)) * fs / (2*np.pi)

    print(f"\n{freq/1e3:+7.1f} kHz tone:")
    print(f"  Expected frequency:  {freq/1e3:+7.1f} kHz")
    print(f"  Measured (FFT):      {measured_freq/1e3:+7.1f} kHz")
    print(f"  Measured (phase):    {phase_rate/1e3:+7.1f} kHz")
    print(f"  Error:               {abs(freq - measured_freq):.3f} Hz")

print("\n" + "="*60)
print("✓ METHOD 1 COMPLETE: Simulation")
print("="*60)
print("\nKey Observations:")
print("  1. Positive frequency rotates counter-clockwise")
print("  2. Negative frequency rotates clockwise")
print("  3. Complex I/Q distinguishes +/- frequencies")
print("  4. Real-only signal has mirror image (ambiguity)")
print("  5. Phase continuously increases/decreases")
print("\nNext: Try Method 2 with real PlutoSDR hardware!")
```

**Expected Output:**
```
============================================================
LAB 1.3 - METHOD 1: I/Q SIMULATION
============================================================

📊 SIMULATION RESULTS:
------------------------------------------------------------

+100.0 kHz tone:
  Expected frequency:  +100.0 kHz
  Measured (FFT):      +100.0 kHz
  Measured (phase):    +100.0 kHz
  Error:               0.000 Hz

-100.0 kHz tone:
  Expected frequency:  -100.0 kHz
  Measured (FFT):      -100.0 kHz
  Measured (phase):    -100.0 kHz
  Error:               0.000 Hz

+250.0 kHz tone:
  Expected frequency:  +250.0 kHz
  Measured (FFT):      +250.0 kHz
  Measured (phase):    +250.0 kHz
  Error:               0.000 Hz

============================================================
✓ METHOD 1 COMPLETE: Simulation
============================================================

Key Observations:
  1. Positive frequency rotates counter-clockwise
  2. Negative frequency rotates clockwise
  3. Complex I/Q distinguishes +/- frequencies
  4. Real-only signal has mirror image (ambiguity)
  5. Phase continuously increases/decreases

Next: Try Method 2 with real PlutoSDR hardware!
```

---

## 🔶 METHOD 2: EXTERNAL APP (Python + PlutoSDR)

**File:** `lab1_3_method2_external.py`

```python
#!/usr/bin/env python3
"""
LAB 1.3 - Method 2: External Application with PlutoSDR
Application runs on host PC, uses PlutoSDR for real RF
Demonstrates I/Q with actual hardware
"""

import adi
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.gridspec import GridSpec
import time

print("="*60)
print("LAB 1.3 - METHOD 2: EXTERNAL APP WITH PLUTOSDR")
print("="*60)

# Connect to PlutoSDR
print("\n🔌 Connecting to PlutoSDR...")
try:
    sdr = adi.Pluto("ip:192.168.2.1")
    print("✓ Connected to PlutoSDR")
except Exception as e:
    print(f"✗ Failed to connect: {e}")
    print("  Make sure PlutoSDR is connected and accessible at 192.168.2.1")
    exit(1)

# Configure PlutoSDR
print("\n⚙️  Configuring PlutoSDR...")
sdr.sample_rate = int(2.084e6)
sdr.rx_lo = int(915e6)  # 915 MHz center frequency
sdr.tx_lo = int(915e6)
sdr.rx_rf_bandwidth = int(2e6)
sdr.tx_rf_bandwidth = int(2e6)
sdr.rx_buffer_size = 16384
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 60  # RX gain in dB
sdr.tx_hardwaregain_chan0 = -10  # TX attenuation in dB
sdr.tx_cyclic_buffer = True

print(f"  Sample rate:  {sdr.sample_rate/1e6:.3f} MSPS")
print(f"  Center freq:  {sdr.rx_lo/1e6:.1f} MHz")
print(f"  RX gain:      {sdr.rx_hardwaregain_chan0} dB")

# Generate test signals to transmit
print("\n📡 Generating TX signal (three tones)...")
fs = sdr.sample_rate
duration = 0.01  # 10 ms
t = np.arange(0, duration, 1/fs)

# Three tones at different offsets
tone1 = 0.3 * np.exp(2j * np.pi * 100e3 * t)   # +100 kHz
tone2 = 0.3 * np.exp(2j * np.pi * -150e3 * t)  # -150 kHz
tone3 = 0.3 * np.exp(2j * np.pi * 300e3 * t)   # +300 kHz

tx_signal = tone1 + tone2 + tone3

# Scale for DAC (16-bit signed integer)
tx_signal_scaled = tx_signal / np.max(np.abs(tx_signal)) * 0.8
tx_signal_int = (tx_signal_scaled * 2**14).astype(np.int16)

print(f"  TX signal length: {len(tx_signal_int)} samples")
print(f"  Frequencies: +100, -150, +300 kHz")

# Transmit
print("\n📤 Starting transmission...")
sdr.tx(tx_signal_int)
time.sleep(0.5)  # Let TX stabilize

# Receive
print("📥 Receiving signal...")
rx_samples = sdr.rx()
print(f"  Received {len(rx_samples)} complex samples")

# Extract I and Q
i_samples = np.real(rx_samples)
q_samples = np.imag(rx_samples)

print(f"  I range: [{np.min(i_samples):.0f}, {np.max(i_samples):.0f}]")
print(f"  Q range: [{np.min(q_samples):.0f}, {np.max(q_samples):.0f}]")

# Analysis
print("\n🔬 Analyzing received signal...")

# FFT
fft_result = np.fft.fftshift(np.fft.fft(rx_samples))
freqs = np.fft.fftshift(np.fft.fftfreq(len(rx_samples), 1/fs))
magnitude_db = 20 * np.log10(np.abs(fft_result) / len(rx_samples))

# Find peaks
threshold = np.max(magnitude_db) - 20  # 20 dB below max
peaks_idx = np.where(magnitude_db > threshold)[0]
peak_freqs = freqs[peaks_idx]

# Cluster peaks
detected_tones = []
if len(peak_freqs) > 0:
    detected_tones.append(peak_freqs[0])
    for freq in peak_freqs[1:]:
        if abs(freq - detected_tones[-1]) > 10e3:  # 10 kHz separation
            detected_tones.append(freq)

print(f"  Detected {len(detected_tones)} tones:")
for freq in detected_tones[:5]:  # Show first 5
    print(f"    {freq/1e3:+7.1f} kHz")

# Visualization
fig = plt.figure(figsize=(16, 12))
gs = GridSpec(3, 3, figure=fig)

# Plot 1: Time domain I/Q
ax1 = fig.add_subplot(gs[0, :])
time_axis = np.arange(len(rx_samples[:2000])) / fs * 1e6
ax1.plot(time_axis, i_samples[:2000], 'b-', linewidth=1, label='I (In-phase)', alpha=0.8)
ax1.plot(time_axis, q_samples[:2000], 'r-', linewidth=1, label='Q (Quadrature)', alpha=0.8)
ax1.set_xlabel('Time (μs)')
ax1.set_ylabel('ADC Value')
ax1.set_title('Received I/Q Time Series (PlutoSDR Real Hardware)')
ax1.legend()
ax1.grid(True, alpha=0.3)

# Plot 2: Constellation
ax2 = fig.add_subplot(gs[1, 0])
ax2.plot(i_samples, q_samples, '.', alpha=0.01, markersize=1, color='blue')
ax2.set_xlabel('I')
ax2.set_ylabel('Q')
ax2.set_title('Constellation Diagram')
ax2.axis('equal')
ax2.grid(True, alpha=0.3)

# Plot 3: Spectrum
ax3 = fig.add_subplot(gs[1, 1:])
ax3.plot(freqs/1e3, magnitude_db, linewidth=1)
ax3.axhline(threshold, color='r', linestyle='--', alpha=0.5, label='Detection threshold')
for tone_freq in [100e3, -150e3, 300e3]:
    ax3.axvline(tone_freq/1e3, color='g', linestyle='--', alpha=0.5)
ax3.set_xlabel('Frequency Offset (kHz)')
ax3.set_ylabel('Magnitude (dB)')
ax3.set_title('Received Spectrum (Three Transmitted Tones)')
ax3.legend()
ax3.grid(True, alpha=0.3)
ax3.set_ylim([np.max(magnitude_db)-80, np.max(magnitude_db)+5])

# Plot 4: Waterfall (spectrogram)
ax4 = fig.add_subplot(gs[2, :2])
NFFT = 1024
ax4.specgram(rx_samples, NFFT=NFFT, Fs=fs, cmap='viridis',
             scale='dB', mode='magnitude')
ax4.set_xlabel('Time (s)')
ax4.set_ylabel('Frequency (Hz)')
ax4.set_title('Waterfall Plot (Time-Frequency)')

# Plot 5: Phase evolution
ax5 = fig.add_subplot(gs[2, 2])
phase = np.angle(rx_samples[:2000])
unwrapped_phase = np.unwrap(phase)
ax5.plot(time_axis, unwrapped_phase, linewidth=1)
ax5.set_xlabel('Time (μs)')
ax5.set_ylabel('Phase (radians)')
ax5.set_title('Phase Evolution')
ax5.grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('lab1_3_method2_external.png', dpi=150)
plt.show()

# Stop TX
sdr.tx_destroy_buffer()

print("\n" + "="*60)
print("✓ METHOD 2 COMPLETE: External App with PlutoSDR")
print("="*60)
print("\nKey Observations:")
print("  1. Real I/Q samples from AD9363 ADC (12-bit)")
print("  2. Quantization noise visible in constellation")
print("  3. RF impairments (noise, phase noise) present")
print("  4. TX-RX loopback demonstrates full chain")
print("  5. Network latency: ~10-50 ms for buffer transfer")
print("\nNext: Try Method 3 - hosted app on PlutoSDR!")
```

**Expected Output:**
```
============================================================
LAB 1.3 - METHOD 2: EXTERNAL APP WITH PLUTOSDR
============================================================

🔌 Connecting to PlutoSDR...
✓ Connected to PlutoSDR

⚙️  Configuring PlutoSDR...
  Sample rate:  2.084 MSPS
  Center freq:  915.0 MHz
  RX gain:      60 dB

📡 Generating TX signal (three tones)...
  TX signal length: 20840 samples
  Frequencies: +100, -150, +300 kHz

📤 Starting transmission...
📥 Receiving signal...
  Received 16384 complex samples
  I range: [-2048, 2047]
  Q range: [-2048, 2047]

🔬 Analyzing received signal...
  Detected 3 tones:
    -150.0 kHz
    +100.1 kHz
    +300.2 kHz

============================================================
✓ METHOD 2 COMPLETE: External App with PlutoSDR
============================================================

Key Observations:
  1. Real I/Q samples from AD9363 ADC (12-bit)
  2. Quantization noise visible in constellation
  3. RF impairments (noise, phase noise) present
  4. TX-RX loopback demonstrates full chain
  5. Network latency: ~10-50 ms for buffer transfer

Next: Try Method 3 - hosted app on PlutoSDR!
```

---

## 🔴 METHOD 3: HOSTED APP (C on PlutoSDR)

**File:** `lab1_3_method3_hosted.c`

```c
/*
 * LAB 1.3 - Method 3: Hosted Application
 *
 * This application runs directly on PlutoSDR ARM CPU
 * Demonstrates local IIO access for low-latency I/Q processing
 *
 * Compile with cross-compiler:
 *   arm-linux-gnueabihf-gcc -o lab1_3_hosted lab1_3_method3_hosted.c -liio -lm
 *
 * Deploy to PlutoSDR:
 *   scp lab1_3_hosted root@192.168.2.1:/root/
 *   ssh root@192.168.2.1
 *   ./lab1_3_hosted
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
#include <iio.h>
#include <unistd.h>

#define SAMPLE_COUNT 16384
#define TONE_FREQ_1  100000.0   // 100 kHz
#define TONE_FREQ_2 -150000.0   // -150 kHz
#define TONE_FREQ_3  300000.0   // 300 kHz

/* Function prototypes */
void analyze_iq_samples(int16_t *buffer, size_t count, double fs);
void compute_fft_peaks(int16_t *buffer, size_t count, double fs);
void print_iq_statistics(int16_t *buffer, size_t count);

int main(int argc, char **argv)
{
    struct iio_context *ctx;
    struct iio_device *phy, *tx_dev, *rx_dev;
    struct iio_channel *rx0_i, *rx0_q, *tx0_i, *tx0_q;
    struct iio_buffer *rxbuf, *txbuf;
    int16_t *tx_buffer, *rx_buffer;
    double fs = 2084000.0;  // 2.084 MSPS
    size_t i;

    printf("============================================================\n");
    printf("LAB 1.3 - METHOD 3: HOSTED APP ON PLUTOSDR\n");
    printf("============================================================\n\n");

    /* Create local IIO context (no network) */
    printf("🔌 Creating local IIO context...\n");
    ctx = iio_create_local_context();
    if (!ctx) {
        fprintf(stderr, "✗ Failed to create IIO context\n");
        return -1;
    }
    printf("✓ Local context created (%u devices)\n",
           iio_context_get_devices_count(ctx));

    /* Get devices */
    phy = iio_context_find_device(ctx, "ad9361-phy");
    rx_dev = iio_context_find_device(ctx, "cf-ad9361-lpc");
    tx_dev = iio_context_find_device(ctx, "cf-ad9361-dds-core-lpc");

    if (!phy || !rx_dev || !tx_dev) {
        fprintf(stderr, "✗ Failed to find IIO devices\n");
        goto error;
    }

    printf("\n⚙️  Configuring AD9361...\n");

    /* Configure PHY */
    iio_channel_attr_write_longlong(
        iio_device_find_channel(phy, "altvoltage0", true),
        "frequency", 915000000);  // RX LO = 915 MHz

    iio_channel_attr_write_longlong(
        iio_device_find_channel(phy, "altvoltage1", true),
        "frequency", 915000000);  // TX LO = 915 MHz

    iio_channel_attr_write_longlong(
        iio_device_find_channel(phy, "voltage0", false),
        "sampling_frequency", (long long)fs);

    iio_channel_attr_write(
        iio_device_find_channel(phy, "voltage0", false),
        "gain_control_mode", "manual");

    iio_channel_attr_write_longlong(
        iio_device_find_channel(phy, "voltage0", false),
        "hardwaregain", 60);  // RX gain

    iio_channel_attr_write_longlong(
        iio_device_find_channel(phy, "voltage0", true),
        "hardwaregain", -10);  // TX attenuation

    printf("  Sample rate:  %.3f MSPS\n", fs/1e6);
    printf("  Center freq:  915.0 MHz\n");
    printf("  RX gain:      60 dB\n");

    /* Setup TX channels */
    tx0_i = iio_device_find_channel(tx_dev, "voltage0", true);
    tx0_q = iio_device_find_channel(tx_dev, "voltage1", true);
    iio_channel_enable(tx0_i);
    iio_channel_enable(tx0_q);

    /* Setup RX channels */
    rx0_i = iio_device_find_channel(rx_dev, "voltage0", false);
    rx0_q = iio_device_find_channel(rx_dev, "voltage1", false);
    iio_channel_enable(rx0_i);
    iio_channel_enable(rx0_q);

    /* Create buffers */
    printf("\n📡 Creating TX/RX buffers...\n");
    txbuf = iio_device_create_buffer(tx_dev, SAMPLE_COUNT, true);
    rxbuf = iio_device_create_buffer(rx_dev, SAMPLE_COUNT, false);

    if (!txbuf || !rxbuf) {
        fprintf(stderr, "✗ Failed to create buffers\n");
        goto error;
    }

    /* Generate TX signal (three tones) */
    printf("📤 Generating TX signal (three tones)...\n");
    printf("  Tone 1: %+.1f kHz\n", TONE_FREQ_1/1e3);
    printf("  Tone 2: %+.1f kHz\n", TONE_FREQ_2/1e3);
    printf("  Tone 3: %+.1f kHz\n", TONE_FREQ_3/1e3);

    tx_buffer = (int16_t *)iio_buffer_start(txbuf);

    for (i = 0; i < SAMPLE_COUNT; i++) {
        double t = (double)i / fs;

        // Sum of three complex exponentials
        double i_val = 0.3 * cos(2*M_PI*TONE_FREQ_1*t) +
                       0.3 * cos(2*M_PI*TONE_FREQ_2*t) +
                       0.3 * cos(2*M_PI*TONE_FREQ_3*t);

        double q_val = 0.3 * sin(2*M_PI*TONE_FREQ_1*t) +
                       0.3 * sin(2*M_PI*TONE_FREQ_2*t) +
                       0.3 * sin(2*M_PI*TONE_FREQ_3*t);

        // Scale to 12-bit DAC range
        tx_buffer[2*i]   = (int16_t)(i_val * 2048);  // I
        tx_buffer[2*i+1] = (int16_t)(q_val * 2048);  // Q
    }

    /* Push TX buffer */
    ssize_t nbytes_tx = iio_buffer_push(txbuf);
    if (nbytes_tx < 0) {
        fprintf(stderr, "✗ TX push failed\n");
        goto error;
    }
    printf("  Pushed %zd bytes to TX\n", nbytes_tx);

    /* Wait for TX to stabilize */
    usleep(500000);  // 500 ms

    /* Receive */
    printf("\n📥 Receiving signal...\n");
    ssize_t nbytes_rx = iio_buffer_refill(rxbuf);
    if (nbytes_rx < 0) {
        fprintf(stderr, "✗ RX refill failed\n");
        goto error;
    }
    printf("  Received %zd bytes\n", nbytes_rx);

    rx_buffer = (int16_t *)iio_buffer_start(rxbuf);

    /* Analyze I/Q samples */
    printf("\n🔬 Analyzing I/Q samples...\n");
    analyze_iq_samples(rx_buffer, SAMPLE_COUNT, fs);

    printf("\n📊 I/Q Statistics:\n");
    print_iq_statistics(rx_buffer, SAMPLE_COUNT);

    printf("\n🔍 Detecting frequency peaks...\n");
    compute_fft_peaks(rx_buffer, SAMPLE_COUNT, fs);

    /* Cleanup */
    iio_buffer_destroy(rxbuf);
    iio_buffer_destroy(txbuf);

    printf("\n");
    printf("============================================================\n");
    printf("✓ METHOD 3 COMPLETE: Hosted App\n");
    printf("============================================================\n");
    printf("\nKey Advantages of Hosted App:\n");
    printf("  1. No network latency (local IIO access)\n");
    printf("  2. Standalone operation (no PC needed)\n");
    printf("  3. Low-latency processing (< 1 ms)\n");
    printf("  4. Embedded deployment ready\n");
    printf("  5. Direct hardware access\n");
    printf("\nLimitations:\n");
    printf("  - Limited CPU: ARM Cortex-A9 @ 666 MHz\n");
    printf("  - Limited RAM: 512 MB DDR3\n");
    printf("  - No GUI (console only)\n");
    printf("  - Harder debugging than PC\n");

error:
    iio_context_destroy(ctx);
    return 0;
}

void analyze_iq_samples(int16_t *buffer, size_t count, double fs)
{
    double i_sum = 0, q_sum = 0;
    double i_sq_sum = 0, q_sq_sum = 0;
    size_t i;

    for (i = 0; i < count; i++) {
        double i_val = (double)buffer[2*i];
        double q_val = (double)buffer[2*i+1];

        i_sum += i_val;
        q_sum += q_val;
        i_sq_sum += i_val * i_val;
        q_sq_sum += q_val * q_val;
    }

    double i_mean = i_sum / count;
    double q_mean = q_sum / count;
    double i_rms = sqrt(i_sq_sum / count);
    double q_rms = sqrt(q_sq_sum / count);
    double power = (i_sq_sum + q_sq_sum) / count;
    double power_db = 10 * log10(power);

    printf("  I mean: %.2f, RMS: %.2f\n", i_mean, i_rms);
    printf("  Q mean: %.2f, RMS: %.2f\n", q_mean, q_rms);
    printf("  Total power: %.2f dB\n", power_db);
}

void print_iq_statistics(int16_t *buffer, size_t count)
{
    int16_t i_min = 32767, i_max = -32768;
    int16_t q_min = 32767, q_max = -32768;
    size_t i;

    for (i = 0; i < count; i++) {
        if (buffer[2*i] < i_min) i_min = buffer[2*i];
        if (buffer[2*i] > i_max) i_max = buffer[2*i];
        if (buffer[2*i+1] < q_min) q_min = buffer[2*i+1];
        if (buffer[2*i+1] > q_max) q_max = buffer[2*i+1];
    }

    printf("  I range: [%d, %d]\n", i_min, i_max);
    printf("  Q range: [%d, %d]\n", q_min, q_max);

    /* Print first 10 samples */
    printf("  First 10 I/Q pairs:\n");
    for (i = 0; i < 10 && i < count; i++) {
        printf("    [%zu] I=%6d, Q=%6d\n", i, buffer[2*i], buffer[2*i+1]);
    }
}

void compute_fft_peaks(int16_t *buffer, size_t count, double fs)
{
    /* Simple peak detection in frequency domain */
    /* For full FFT, would need FFTW or similar library */
    /* Here we do correlation-based detection */

    double test_freqs[] = {TONE_FREQ_1, TONE_FREQ_2, TONE_FREQ_3};
    int num_freqs = sizeof(test_freqs) / sizeof(test_freqs[0]);
    int f;

    printf("  Correlating with expected frequencies:\n");

    for (f = 0; f < num_freqs; f++) {
        double freq = test_freqs[f];
        double corr_i = 0, corr_q = 0;
        size_t i;

        /* Correlate with expected frequency */
        for (i = 0; i < count; i++) {
            double t = (double)i / fs;
            double ref_i = cos(2*M_PI*freq*t);
            double ref_q = sin(2*M_PI*freq*t);

            corr_i += buffer[2*i] * ref_i + buffer[2*i+1] * ref_q;
            corr_q += buffer[2*i+1] * ref_i - buffer[2*i] * ref_q;
        }

        double magnitude = sqrt(corr_i*corr_i + corr_q*corr_q) / count;
        double magnitude_db = 20 * log10(magnitude);

        printf("    %+7.1f kHz: %.2f dB\n", freq/1e3, magnitude_db);
    }
}
```

### Cross-Compilation Script

**File:** `compile_hosted.sh`

```bash
#!/bin/bash
# Cross-compilation script for PlutoSDR hosted applications

echo "============================================================"
echo "PlutoSDR Hosted App Cross-Compilation"
echo "============================================================"

# Configuration
APP_NAME="lab1_3_hosted"
SOURCE_FILE="lab1_3_method3_hosted.c"
CROSS_COMPILE="arm-linux-gnueabihf-"
PLUTO_IP="192.168.2.1"

# Check for cross-compiler
if ! command -v ${CROSS_COMPILE}gcc &> /dev/null; then
    echo "✗ Cross-compiler not found: ${CROSS_COMPILE}gcc"
    echo ""
    echo "Install cross-compiler:"
    echo "  Ubuntu/Debian: sudo apt install gcc-arm-linux-gnueabihf"
    echo "  Or download from: https://releases.linaro.org/components/toolchain/binaries/"
    exit 1
fi

echo "✓ Cross-compiler found: $(${CROSS_COMPILE}gcc --version | head -1)"

# Compile
echo ""
echo "📦 Compiling $SOURCE_FILE..."
${CROSS_COMPILE}gcc -o $APP_NAME $SOURCE_FILE -liio -lm -Wall -O2

if [ $? -ne 0 ]; then
    echo "✗ Compilation failed"
    exit 1
fi

echo "✓ Compilation successful"

# Check binary
file $APP_NAME
${CROSS_COMPILE}readelf -h $APP_NAME | grep Machine

# Strip symbols (reduce size)
echo ""
echo "🔧 Stripping symbols..."
${CROSS_COMPILE}strip $APP_NAME
ls -lh $APP_NAME

# Deploy to PlutoSDR
echo ""
read -p "Deploy to PlutoSDR at $PLUTO_IP? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo "📤 Deploying to PlutoSDR..."
    scp $APP_NAME root@$PLUTO_IP:/root/

    if [ $? -eq 0 ]; then
        echo "✓ Deployed successfully"
        echo ""
        echo "To run on PlutoSDR:"
        echo "  ssh root@$PLUTO_IP"
        echo "  ./$APP_NAME"
    else
        echo "✗ Deployment failed"
    fi
fi

echo ""
echo "============================================================"
echo "✓ Build complete"
echo "============================================================"
```

**Make it executable:**
```bash
chmod +x compile_hosted.sh
```

**Usage:**
```bash
./compile_hosted.sh
```

**Expected Output:**
```
============================================================
PlutoSDR Hosted App Cross-Compilation
============================================================
✓ Cross-compiler found: arm-linux-gnueabihf-gcc (GNU) 7.3.1

📦 Compiling lab1_3_method3_hosted.c...
✓ Compilation successful
lab1_3_hosted: ELF 32-bit LSB executable, ARM, version 1 (SYSV)
  Machine:                           ARM

🔧 Stripping symbols...
-rwxr-xr-x 1 user user 12K lab1_3_hosted

Deploy to PlutoSDR at 192.168.2.1? (y/n) y
📤 Deploying to PlutoSDR...
lab1_3_hosted                 100%   12KB  12.0KB/s   00:01
✓ Deployed successfully

To run on PlutoSDR:
  ssh root@$PLUTO_IP
  ./lab1_3_hosted

============================================================
✓ Build complete
============================================================
```

**Running on PlutoSDR:**
```bash
ssh root@192.168.2.1
# Password: analog

cd /root
./lab1_3_hosted
```

**Expected Output on PlutoSDR:**
```
============================================================
LAB 1.3 - METHOD 3: HOSTED APP ON PLUTOSDR
============================================================

🔌 Creating local IIO context...
✓ Local context created (3 devices)

⚙️  Configuring AD9361...
  Sample rate:  2.084 MSPS
  Center freq:  915.0 MHz
  RX gain:      60 dB

📡 Creating TX/RX buffers...
📤 Generating TX signal (three tones)...
  Tone 1: +100.0 kHz
  Tone 2: -150.0 kHz
  Tone 3: +300.0 kHz
  Pushed 65536 bytes to TX

📥 Receiving signal...
  Received 65536 bytes

🔬 Analyzing I/Q samples...
  I mean: -2.45, RMS: 756.32
  Q mean: 1.23, RMS: 751.89
  Total power: 66.32 dB

📊 I/Q Statistics:
  I range: [-1842, 1856]
  Q range: [-1831, 1849]
  First 10 I/Q pairs:
    [0] I=   245, Q=   -12
    [1] I=   491, Q=   243
    [2] I=   712, Q=   489
    [3] I=   898, Q=   721
    [4] I=  1042, Q=   927
    [5] I=  1138, Q=  1095
    [6] I=  1182, Q=  1218
    [7] I=  1171, Q=  1291
    [8] I=  1106, Q=  1310
    [9] I=   991, Q=  1274

🔍 Detecting frequency peaks...
  Correlating with expected frequencies:
    +100.0 kHz: -23.45 dB
    -150.0 kHz: -24.12 dB
    +300.0 kHz: -22.89 dB

============================================================
✓ METHOD 3 COMPLETE: Hosted App
============================================================

Key Advantages of Hosted App:
  1. No network latency (local IIO access)
  2. Standalone operation (no PC needed)
  3. Low-latency processing (< 1 ms)
  4. Embedded deployment ready
  5. Direct hardware access

Limitations:
  - Limited CPU: ARM Cortex-A9 @ 666 MHz
  - Limited RAM: 512 MB DDR3
  - No GUI (console only)
  - Harder debugging than PC
```

---

## 📊 Comparison of Three Methods

| Aspect | Method 1: Simulation | Method 2: External App | Method 3: Hosted App |
|--------|---------------------|----------------------|---------------------|
| **Development Speed** | ⭐⭐⭐⭐⭐ Fast | ⭐⭐⭐⭐ Fast | ⭐⭐ Slower |
| **Debugging** | ⭐⭐⭐⭐⭐ Easy | ⭐⭐⭐⭐ Easy | ⭐⭐ Harder |
| **RF Realism** | ❌ No RF | ⭐⭐⭐⭐⭐ Real RF | ⭐⭐⭐⭐⭐ Real RF |
| **Latency** | N/A | ~10-50 ms | < 1 ms |
| **Standalone** | ❌ No | ❌ Needs PC | ⭐⭐⭐⭐⭐ Yes |
| **Processing Power** | ⭐⭐⭐⭐⭐ Full PC | ⭐⭐⭐⭐⭐ Full PC | ⭐⭐ ARM Limited |
| **GUI Possible** | ⭐⭐⭐⭐⭐ Yes | ⭐⭐⭐⭐⭐ Yes | ❌ No |
| **Deployment** | N/A | Easy (Python) | Medium (Cross-compile) |
| **Best For** | Learning, algorithm dev | Testing, visualization | Production, embedded |

---

## 🎯 When to Use Each Method

### Use Method 1 (Simulation) When:
- Learning new concepts
- Developing algorithms
- No hardware available
- Need perfect controlled signals
- Rapid prototyping

### Use Method 2 (External App) When:
- Testing with real RF
- Need visualization/GUI
- Development phase
- Debugging complex systems
- Have host PC available

### Use Method 3 (Hosted App) When:
- Deploying production system
- Need standalone operation
- Latency critical (< 1 ms)
- Battery powered / embedded
- No host PC in field

---

## 📝 Lab Completion Checklist

- [ ] Complete Method 1: Understand I/Q theory from simulation
- [ ] Complete Method 2: Test with real PlutoSDR hardware
- [ ] Complete Method 3: Deploy hosted application
- [ ] Compare results between all three methods
- [ ] Understand trade-offs of each approach

---

**Next Lab:** We'll apply this three-method approach to LAB 2.1 (Decimation) and beyond!

