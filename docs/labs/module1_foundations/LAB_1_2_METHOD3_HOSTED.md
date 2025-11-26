# LAB 1.2 - Method 3: Hosted Application (RF Gain Staging on PlutoSDR)

## Overview

This guide provides **complete step-by-step instructions** for implementing an RF gain staging test application that runs directly on PlutoSDR's ARM processor. This C program tests AGC modes, performs manual gain sweeps, and analyzes signal quality - all executing on the embedded Linux system within PlutoSDR.

**Benefits over Method 2 (External Application)**:
- **Real-time performance**: Direct hardware access without network latency
- **Standalone operation**: No PC required for continuous monitoring
- **Lower power**: PlutoSDR-only operation (~2W vs ~50W with PC)
- **Production ready**: Deploy as embedded gain calibration tool

---

## Theory: Understanding RF Gain Staging

### What is RF Gain?

**Simple Explanation**: RF gain is like the volume knob on your radio, but for incoming signals. Turn it up too much and you get distortion (clipping). Turn it down too much and you only hear noise.

**Technical Explanation**:
In an SDR receiver, the RF gain controls how much the weak antenna signal is amplified before reaching the Analog-to-Digital Converter (ADC). The AD9361 chip in PlutoSDR has multiple gain stages that work together.

### Why Does Gain Matter?

Think of your signal like someone whispering across a noisy room:
- **Too little gain** = Can't hear the whisper over the noise (poor SNR)
- **Just right gain** = Clear signal, minimal noise (optimal SNR)
- **Too much gain** = Whisper becomes a scream that distorts (clipping/saturation)

### The AD9361 Gain Architecture

PlutoSDR's AD9361 chip has THREE gain stages working together:

```
Antenna → [LNA] → [Mixer] → [Baseband Amp] → ADC
          0-30dB   Variable   0-43dB         12-bit

Total Gain Range: 0 to 73 dB
```

**1. LNA (Low Noise Amplifier)**: First stage, 0-30 dB in 3 dB steps
   - **Job**: Amplify weak signals while adding minimal noise
   - **Why first?**: Early amplification = better noise figure
   - **Think of it as**: Cupping your hand around your ear

**2. Mixer Gain**: Middle stage, variable gain
   - **Job**: Convert RF frequency down to baseband while adjusting gain
   - **Think of it as**: Fine-tuning the volume

**3. Baseband Amplifier**: Final stage, 0-43 dB in 0.25 dB steps
   - **Job**: Final amplification before ADC
   - **Why last?**: Precise gain control doesn't need to handle noise as much
   - **Think of it as**: Volume control right before your speakers

### Key Concept: Signal-to-Noise Ratio (SNR)

The fundamental equation for receiver SNR:

```
SNR_out = SNR_in + Gain - Noise_Figure

Where:
  SNR_in = Input signal-to-noise ratio (from antenna)
  Gain = Total receiver gain (dB)
  Noise_Figure = Noise added by receiver (dB)
  SNR_out = Output SNR at ADC
```

**Example**:
```
Antenna receives: Signal = -80 dBm, Noise = -100 dBm
                  SNR_in = -80 - (-100) = 20 dB

Receiver: Gain = 60 dB, Noise Figure = 3 dB

Output SNR: 20 + 60 - 3 = 77 dB ✓ Excellent!
```

### The Clipping Problem

The ADC in PlutoSDR is 12-bit signed: **±2048** integer range

**What happens with different gains**:

```
Signal at antenna: -70 dBm (weak signal)

Gain = 30 dB:  Signal becomes -40 dBm → ADC sees ~200  ✓ Good (room to spare)
Gain = 60 dB:  Signal becomes -10 dBm → ADC sees ~1800 ✓ Perfect (near full scale)
Gain = 73 dB:  Signal becomes +3 dBm  → ADC sees >2048 ✗ CLIPPING!
```

**Clipping is BAD** because:
- Distorts the signal (harmonics appear)
- Loses information (peaks get flattened)
- Makes demodulation difficult/impossible

### Automatic Gain Control (AGC)

PlutoSDR has **hardware AGC** that automatically adjusts gain:

**How AGC Works**:
```
1. Measure signal power at ADC
2. Too weak?  → Increase gain
3. Too strong? → Decrease gain
4. Repeat continuously
```

**AGC Modes**:

| Mode | Attack Time | Use Case |
|------|-------------|----------|
| **Manual** | N/A (fixed) | When you know signal strength |
| **Slow Attack** | ~100ms | Continuous signals (FM radio) |
| **Fast Attack** | ~1ms | Burst signals (GSM, LTE) |

**Trade-offs**:
- **Slow AGC**: More stable, less jitter, but slow to adapt
- **Fast AGC**: Quick adaptation, but can oscillate with noise

### What This Lab Teaches You

By running gain tests on PlutoSDR's ARM processor, you'll learn:

1. **How gain affects SNR** - Measure signal power at different gains
2. **Where clipping occurs** - Find the maximum safe gain
3. **AGC behavior** - Compare slow vs fast attack modes
4. **Practical calibration** - Determine optimal gain for your setup

### Real-World Application

**Example: Receiving GPS Signals**

GPS signals are VERY weak at Earth's surface: ~-130 dBm

```
GPS signal: -130 dBm
Noise floor: -110 dBm (thermal noise)
SNR_in: -130 - (-110) = -20 dB  (signal below noise!)

With 70 dB gain and 3 dB noise figure:
SNR_out = -20 + 70 - 3 = 47 dB ✓ Now easily detectable

But if gain was only 40 dB:
SNR_out = -20 + 40 - 3 = 17 dB  (marginal, might lose lock)
```

**Key Insight**: For weak signals like GPS, you need HIGH gain. For strong signals like nearby FM radio, you need LOW gain to avoid clipping.

---

## Part 1: Cross-Compilation Environment Setup

### Prerequisites

If you completed LAB 0 or LAB 1.1 Method 3, you already have the cross-compilation environment set up. If not, follow these steps:

#### Step 1: Install ARM Cross-Compiler

```bash
#!/bin/bash
# install_cross_compiler.sh
# Install ARM cross-compilation toolchain

echo "Installing ARM cross-compiler for PlutoSDR..."

# For Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y \
    gcc-arm-linux-gnueabihf \
    g++-arm-linux-gnueabihf \
    build-essential \
    cmake \
    git \
    libxml2-dev \
    bison \
    flex \
    libcdk5-dev \
    libaio-dev \
    libusb-1.0-0-dev \
    libserialport-dev \
    libavahi-client-dev \
    doxygen \
    graphviz

# Verify installation
arm-linux-gnueabihf-gcc --version

echo "✓ ARM cross-compiler installed successfully"
```

**Run**: `chmod +x install_cross_compiler.sh && ./install_cross_compiler.sh`

#### Step 2: Build libiio for ARM

```bash
#!/bin/bash
# build_libiio_arm.sh
# Cross-compile libiio library for PlutoSDR ARM architecture

set -e

LIBIIO_VERSION="v0.25"
BUILD_DIR="/tmp/libiio-arm-build"
INSTALL_PREFIX="/opt/arm-libs"

echo "Building libiio for ARM architecture..."

# Clone libiio
if [ ! -d "$BUILD_DIR" ]; then
    git clone https://github.com/analogdevicesinc/libiio.git "$BUILD_DIR"
fi

cd "$BUILD_DIR"
git checkout $LIBIIO_VERSION

# Create build directory
mkdir -p build-arm
cd build-arm

# Configure with CMake for cross-compilation
cmake .. \
    -DCMAKE_SYSTEM_NAME=Linux \
    -DCMAKE_SYSTEM_PROCESSOR=arm \
    -DCMAKE_C_COMPILER=arm-linux-gnueabihf-gcc \
    -DCMAKE_CXX_COMPILER=arm-linux-gnueabihf-g++ \
    -DCMAKE_FIND_ROOT_PATH=/usr/arm-linux-gnueabihf \
    -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX \
    -DWITH_NETWORK_BACKEND=OFF \
    -DWITH_USB_BACKEND=OFF \
    -DWITH_SERIAL_BACKEND=OFF \
    -DENABLE_IPV6=OFF \
    -DHAVE_DNS_SD=OFF \
    -DCMAKE_BUILD_TYPE=Release

# Build
make -j$(nproc)

# Install to custom prefix
sudo make install

echo "✓ libiio for ARM installed to $INSTALL_PREFIX"
echo "  Headers: $INSTALL_PREFIX/include"
echo "  Libraries: $INSTALL_PREFIX/lib"

# List installed files
ls -lh $INSTALL_PREFIX/lib/libiio.so*
```

**Run**: `chmod +x build_libiio_arm.sh && ./build_libiio_arm.sh`

**Expected output**:
```
✓ libiio for ARM installed to /opt/arm-libs
  Headers: /opt/arm-libs/include
  Libraries: /opt/arm-libs/lib
-rwxr-xr-x 1 root root 234K /opt/arm-libs/lib/libiio.so.0.25
```

---

## Part 2: Complete C Source Code

### lab1_2_method3_hosted.c

This program implements RF gain staging tests including manual gain sweeps and AGC mode comparisons.

```c
/*
 * LAB 1.2 - Method 3: RF Gain Staging Hosted Application
 *
 * Demonstrates RF gain control by testing AGC modes and manual gain settings
 * directly on PlutoSDR's ARM Cortex-A9 processor.
 *
 * Features:
 *  - Manual gain sweep (0-73 dB)
 *  - AGC mode testing (slow_attack, fast_attack)
 *  - Clipping detection
 *  - Signal power measurement
 *  - Crest factor analysis
 *  - Performance metrics
 *
 * Compile: See compile_lab1_2.sh
 * Deploy:  See deploy_lab1_2.sh
 * Run:     ./lab1_2_hosted
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

// ============================================================================
// CONFIGURATION PARAMETERS
// ============================================================================

// RF configuration
#define SAMPLE_RATE        2084000    // 2.084 MSPS
#define CENTER_FREQ        915000000  // 915 MHz
#define BUFFER_SIZE        16384      // Samples per measurement

// Gain test parameters
#define MIN_RX_GAIN        0          // Minimum RX gain (dB)
#define MAX_RX_GAIN        73         // Maximum RX gain (dB)
#define GAIN_STEP          3          // Gain sweep step size (dB)
#define NUM_MEASUREMENTS   10         // Measurements per AGC mode test

// Clipping threshold
#define ADC_FULL_SCALE     2048       // 12-bit ADC: ±2048
#define CLIPPING_THRESHOLD 0.9        // 90% of full scale

// AGC settling time
#define AGC_SETTLE_MS      500        // Time for AGC to settle

// ============================================================================
// DATA STRUCTURES
// ============================================================================

typedef struct {
    int gain_db;
    double power_db;
    double peak_value;
    double rms_value;
    double crest_factor_db;
    bool clipping;
} GainMeasurement;

typedef struct {
    const char *mode_name;
    int num_measurements;
    double avg_power_db;
    double power_std_dev;
    double avg_crest_factor;
    int clipping_count;
    GainMeasurement *measurements;
} AGCTestResult;

typedef struct {
    struct iio_context *ctx;
    struct iio_device *phy;
    struct iio_device *rx_dev;
    struct iio_channel *rx_phy_ch;
    struct iio_channel *rx_i;
    struct iio_channel *rx_q;
    struct iio_buffer *rxbuf;
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

/**
 * Calculate signal statistics from I/Q samples
 */
static void calculate_signal_stats(int16_t *buffer, size_t num_samples, GainMeasurement *meas)
{
    double sum_power = 0.0;
    double peak_amplitude = 0.0;

    for (size_t i = 0; i < num_samples; i++) {
        double i_val = buffer[2*i] / 2048.0;      // Normalize to ±1
        double q_val = buffer[2*i + 1] / 2048.0;

        // Instantaneous amplitude
        double amplitude = sqrt(i_val * i_val + q_val * q_val);

        // Power accumulation
        sum_power += amplitude * amplitude;

        // Track peak
        if (amplitude > peak_amplitude) {
            peak_amplitude = amplitude;
        }
    }

    // Calculate RMS (root mean square)
    meas->rms_value = sqrt(sum_power / num_samples);

    // Peak value
    meas->peak_value = peak_amplitude;

    // Power in dB
    meas->power_db = 10.0 * log10(sum_power / num_samples + 1e-12);

    // Crest factor (peak-to-RMS ratio in dB)
    meas->crest_factor_db = 20.0 * log10((peak_amplitude / (meas->rms_value + 1e-12)) + 1e-12);

    // Clipping detection (peak > 90% of full scale)
    meas->clipping = (peak_amplitude > CLIPPING_THRESHOLD);
}

// ============================================================================
// PLUTOSDR INITIALIZATION
// ============================================================================

/**
 * Initialize PlutoSDR hardware via local IIO
 */
static int init_plutosdr(PlutoSDR *sdr)
{
    printf("Initializing PlutoSDR (local IIO context)...\n");

    // Create local IIO context
    sdr->ctx = iio_create_local_context();
    if (!sdr->ctx) {
        fprintf(stderr, "Failed to create IIO context\n");
        return -1;
    }

    printf("  IIO context created (local)\n");
    printf("  Backend: %s\n", iio_context_get_name(sdr->ctx));

    // Get AD9361 PHY device
    sdr->phy = iio_context_find_device(sdr->ctx, "ad9361-phy");
    if (!sdr->phy) {
        fprintf(stderr, "Failed to find ad9361-phy device\n");
        return -1;
    }
    printf("  ✓ Found ad9361-phy\n");

    // Get RX device (cf-ad9361-lpc)
    sdr->rx_dev = iio_context_find_device(sdr->ctx, "cf-ad9361-lpc");
    if (!sdr->rx_dev) {
        fprintf(stderr, "Failed to find RX device\n");
        return -1;
    }
    printf("  ✓ Found RX device\n");

    // Get RX PHY channel (for gain control)
    sdr->rx_phy_ch = iio_device_find_channel(sdr->phy, "voltage0", false);
    if (!sdr->rx_phy_ch) {
        fprintf(stderr, "Failed to find RX PHY channel\n");
        return -1;
    }
    printf("  ✓ Found RX PHY channel\n");

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

    // Configure sample rate
    set_channel_attr_ll(sdr->rx_phy_ch, "sampling_frequency", SAMPLE_RATE);
    printf("  ✓ Sample rate: %.3f MSPS\n", SAMPLE_RATE / 1e6);

    // Configure center frequency
    struct iio_channel *rx_lo = iio_device_find_channel(sdr->phy, "altvoltage0", true);
    set_channel_attr_ll(rx_lo, "frequency", CENTER_FREQ);
    printf("  ✓ RX frequency: %.3f MHz\n", CENTER_FREQ / 1e6);

    // Create RX buffer
    sdr->rxbuf = iio_device_create_buffer(sdr->rx_dev, BUFFER_SIZE, false);
    if (!sdr->rxbuf) {
        fprintf(stderr, "Failed to create RX buffer\n");
        return -1;
    }
    printf("  ✓ RX buffer created (%d samples)\n", BUFFER_SIZE);

    printf("PlutoSDR initialization complete!\n\n");
    return 0;
}

/**
 * Cleanup PlutoSDR resources
 */
static void cleanup_plutosdr(PlutoSDR *sdr)
{
    if (sdr->rxbuf) iio_buffer_destroy(sdr->rxbuf);
    if (sdr->ctx) iio_context_destroy(sdr->ctx);

    printf("PlutoSDR resources released\n");
}

// ============================================================================
// GAIN CONTROL FUNCTIONS
// ============================================================================

/**
 * Set manual gain mode and value
 */
static int set_manual_gain(PlutoSDR *sdr, int gain_db)
{
    // Set to manual mode
    int ret = set_channel_attr_str(sdr->rx_phy_ch, "gain_control_mode", "manual");
    if (ret < 0) {
        return ret;
    }

    // Set gain value (clamped to valid range)
    if (gain_db < MIN_RX_GAIN) gain_db = MIN_RX_GAIN;
    if (gain_db > MAX_RX_GAIN) gain_db = MAX_RX_GAIN;

    ret = set_channel_attr_ll(sdr->rx_phy_ch, "hardwaregain", gain_db);
    return ret;
}

/**
 * Set AGC mode
 */
static int set_agc_mode(PlutoSDR *sdr, const char *mode)
{
    // Valid modes: "slow_attack", "fast_attack", "hybrid"
    int ret = set_channel_attr_str(sdr->rx_phy_ch, "gain_control_mode", mode);
    return ret;
}

/**
 * Measure signal with current gain settings
 */
static int measure_signal(PlutoSDR *sdr, GainMeasurement *meas)
{
    // Capture samples
    ssize_t nbytes = iio_buffer_refill(sdr->rxbuf);
    if (nbytes < 0) {
        fprintf(stderr, "Failed to refill RX buffer: %s\n", strerror(-nbytes));
        return -1;
    }

    int16_t *rx_data = (int16_t *)iio_buffer_start(sdr->rxbuf);

    // Calculate statistics
    calculate_signal_stats(rx_data, BUFFER_SIZE, meas);

    return 0;
}

// ============================================================================
// TEST FUNCTIONS
// ============================================================================

/**
 * Perform manual gain sweep test
 */
static int test_manual_gain_sweep(PlutoSDR *sdr)
{
    printf("======================================================================\n");
    printf("MANUAL GAIN SWEEP TEST\n");
    printf("======================================================================\n");
    printf("Gain Range: %d to %d dB (step: %d dB)\n", MIN_RX_GAIN, MAX_RX_GAIN, GAIN_STEP);
    printf("======================================================================\n\n");

    printf("Gain (dB) │ Power (dB) │ Crest (dB) │ Peak    │ Status\n");
    printf("──────────┼────────────┼────────────┼─────────┼─────────\n");

    int num_clipped = 0;
    int num_tests = 0;

    for (int gain_db = MIN_RX_GAIN; gain_db <= MAX_RX_GAIN; gain_db += GAIN_STEP) {
        // Set gain
        if (set_manual_gain(sdr, gain_db) < 0) {
            continue;
        }

        // Allow settling
        usleep(50000);  // 50 ms

        // Measure
        GainMeasurement meas = {0};
        meas.gain_db = gain_db;

        if (measure_signal(sdr, &meas) < 0) {
            continue;
        }

        // Print results
        const char *status = meas.clipping ? "✗ CLIP" : "✓ OK";
        if (meas.clipping) num_clipped++;
        num_tests++;

        printf("%5d     │ %+9.1f  │ %+9.1f  │ %6.3f  │ %s\n",
               gain_db,
               meas.power_db,
               meas.crest_factor_db,
               meas.peak_value,
               status);
    }

    printf("\n");
    printf("Summary:\n");
    printf("  Total tests:      %d\n", num_tests);
    printf("  Clipped:          %d (%.1f%%)\n", num_clipped, 100.0 * num_clipped / num_tests);
    printf("  No clipping:      %d (%.1f%%)\n", num_tests - num_clipped, 100.0 * (num_tests - num_clipped) / num_tests);
    printf("\n");

    return 0;
}

/**
 * Test AGC mode
 */
static int test_agc_mode(PlutoSDR *sdr, const char *mode, AGCTestResult *result)
{
    printf("\nTesting AGC Mode: %s\n", mode);
    printf("────────────────────────────────────────\n");

    result->mode_name = mode;
    result->num_measurements = NUM_MEASUREMENTS;
    result->measurements = malloc(NUM_MEASUREMENTS * sizeof(GainMeasurement));
    if (!result->measurements) {
        return -1;
    }

    // Set AGC mode
    if (set_agc_mode(sdr, mode) < 0) {
        free(result->measurements);
        return -1;
    }

    // Allow AGC to settle
    printf("Waiting for AGC to settle (%d ms)...\n", AGC_SETTLE_MS);
    usleep(AGC_SETTLE_MS * 1000);

    // Perform measurements
    double sum_power = 0.0;
    double sum_power_sq = 0.0;
    double sum_crest = 0.0;
    int clipping_count = 0;

    printf("\nMeas │ Power (dB) │ Crest (dB) │ Peak    │ Status\n");
    printf("─────┼────────────┼────────────┼─────────┼─────────\n");

    for (int i = 0; i < NUM_MEASUREMENTS; i++) {
        GainMeasurement *meas = &result->measurements[i];

        if (measure_signal(sdr, meas) < 0) {
            continue;
        }

        // Accumulate statistics
        sum_power += meas->power_db;
        sum_power_sq += meas->power_db * meas->power_db;
        sum_crest += meas->crest_factor_db;
        if (meas->clipping) clipping_count++;

        // Print
        const char *status = meas->clipping ? "✗ CLIP" : "✓ OK";
        printf("%3d  │ %+9.1f  │ %+9.1f  │ %6.3f  │ %s\n",
               i + 1,
               meas->power_db,
               meas->crest_factor_db,
               meas->peak_value,
               status);

        // Delay between measurements
        usleep(100000);  // 100 ms
    }

    // Calculate final statistics
    result->avg_power_db = sum_power / NUM_MEASUREMENTS;
    result->avg_crest_factor = sum_crest / NUM_MEASUREMENTS;
    result->clipping_count = clipping_count;

    // Standard deviation
    double variance = (sum_power_sq / NUM_MEASUREMENTS) - (result->avg_power_db * result->avg_power_db);
    result->power_std_dev = sqrt(fabs(variance));

    printf("\nMode Statistics:\n");
    printf("  Avg Power:        %+.1f dB\n", result->avg_power_db);
    printf("  Power Std Dev:    %.2f dB\n", result->power_std_dev);
    printf("  Avg Crest Factor: %.1f dB\n", result->avg_crest_factor);
    printf("  Clipping Events:  %d / %d\n", clipping_count, NUM_MEASUREMENTS);

    return 0;
}

/**
 * Compare AGC modes
 */
static int test_agc_comparison(PlutoSDR *sdr)
{
    printf("======================================================================\n");
    printf("AGC MODE COMPARISON TEST\n");
    printf("======================================================================\n");
    printf("Testing different AGC modes with %d measurements each\n", NUM_MEASUREMENTS);
    printf("======================================================================\n");

    const char *modes[] = {"slow_attack", "fast_attack"};
    AGCTestResult results[2];

    for (int i = 0; i < 2; i++) {
        if (test_agc_mode(sdr, modes[i], &results[i]) < 0) {
            fprintf(stderr, "Failed to test %s mode\n", modes[i]);
            return -1;
        }
    }

    // Print comparison
    printf("\n");
    printf("======================================================================\n");
    printf("AGC COMPARISON SUMMARY\n");
    printf("======================================================================\n");
    printf("Metric              │ Slow Attack  │ Fast Attack  │ Winner\n");
    printf("────────────────────┼──────────────┼──────────────┼────────────\n");

    printf("Avg Power (dB)      │ %+11.1f  │ %+11.1f  │ %s\n",
           results[0].avg_power_db,
           results[1].avg_power_db,
           (results[0].avg_power_db > results[1].avg_power_db) ? "Slow" : "Fast");

    printf("Power Stability (σ) │ %11.2f  │ %11.2f  │ %s\n",
           results[0].power_std_dev,
           results[1].power_std_dev,
           (results[0].power_std_dev < results[1].power_std_dev) ? "Slow" : "Fast");

    printf("Avg Crest (dB)      │ %11.1f  │ %11.1f  │ %s\n",
           results[0].avg_crest_factor,
           results[1].avg_crest_factor,
           fabs(results[0].avg_crest_factor - results[1].avg_crest_factor) < 0.5 ? "Tie" : "N/A");

    printf("Clipping Events     │ %11d  │ %11d  │ %s\n",
           results[0].clipping_count,
           results[1].clipping_count,
           (results[0].clipping_count < results[1].clipping_count) ? "Slow" : "Fast");

    printf("======================================================================\n");

    // Free memory
    for (int i = 0; i < 2; i++) {
        free(results[i].measurements);
    }

    return 0;
}

// ============================================================================
// MAIN PROGRAM
// ============================================================================

int main(int argc, char **argv)
{
    printf("\n");
    printf("======================================================================\n");
    printf("LAB 1.2 - Method 3: RF Gain Staging Hosted Application\n");
    printf("Running on PlutoSDR ARM Cortex-A9\n");
    printf("======================================================================\n\n");

    PlutoSDR sdr = {0};
    int ret = 0;

    // Initialize hardware
    if (init_plutosdr(&sdr) < 0) {
        fprintf(stderr, "Failed to initialize PlutoSDR\n");
        return 1;
    }

    // Test 1: Manual Gain Sweep
    printf("Starting Test 1: Manual Gain Sweep\n\n");
    double test1_start = get_time_seconds();
    ret = test_manual_gain_sweep(&sdr);
    double test1_duration = get_time_seconds() - test1_start;

    if (ret < 0) {
        fprintf(stderr, "Manual gain sweep test failed\n");
        cleanup_plutosdr(&sdr);
        return 1;
    }

    printf("Test 1 Duration: %.2f seconds\n\n", test1_duration);

    // Short delay between tests
    sleep(2);

    // Test 2: AGC Mode Comparison
    printf("Starting Test 2: AGC Mode Comparison\n\n");
    double test2_start = get_time_seconds();
    ret = test_agc_comparison(&sdr);
    double test2_duration = get_time_seconds() - test2_start;

    if (ret < 0) {
        fprintf(stderr, "AGC comparison test failed\n");
        cleanup_plutosdr(&sdr);
        return 1;
    }

    printf("Test 2 Duration: %.2f seconds\n\n", test2_duration);

    // Overall summary
    printf("\n");
    printf("======================================================================\n");
    printf("ALL TESTS COMPLETE\n");
    printf("======================================================================\n");
    printf("Total Test Duration: %.2f seconds\n", test1_duration + test2_duration);
    printf("\n✓ RF Gain Staging tests complete!\n\n");

    // Cleanup
    cleanup_plutosdr(&sdr);

    return 0;
}
```

---

## Part 3: Compilation - Detailed Step-by-Step Guide

### Understanding Cross-Compilation

**What is cross-compilation?**
- Your PC runs **x86** processor (Intel/AMD)
- PlutoSDR runs **ARM** processor (Cortex-A9)
- We compile on PC to create ARM binaries
- Think of it like: "Translating a recipe for a different kitchen"

### Step-by-Step Compilation Process

#### **Step 1**: Prepare Your Workspace

Create a directory for this lab:

```bash
# On your Linux PC:
cd ~
mkdir -p pluto_labs/lab1_2_method3
cd pluto_labs/lab1_2_method3
```

**What this does**: Creates organized folder structure for your work

---

#### **Step 2**: Create the C Source File

Copy the complete C code from Part 2 into a file:

```bash
nano lab1_2_method3_hosted.c
# Paste the complete C code
# Press Ctrl+X, then Y, then Enter to save
```

**Verify the file**:
```bash
ls -lh lab1_2_method3_hosted.c
# Should show: -rw-r--r-- 1 user user ~35K lab1_2_method3_hosted.c
```

---

#### **Step 3**: Create the Compilation Script

```bash
nano compile_lab1_2.sh
# Paste the compilation script below
# Save and exit
chmod +x compile_lab1_2.sh
```

**What this does**: Makes the script executable (runnable)

---

#### **Step 4**: Understand the Compilation Flags

Before compiling, let's understand what each flag does:

```bash
arm-linux-gnueabihf-gcc \
  -Wall                    # Show all warnings (helps catch bugs)
  -Wextra                  # Show extra warnings
  -O2                      # Optimize for speed (level 2)
  -std=c99                 # Use C99 standard
  -I/opt/arm-libs/include  # Where to find header files (iio.h)
  -o lab1_2_hosted         # Output filename
  lab1_2_method3_hosted.c  # Input source file
  -L/opt/arm-libs/lib      # Where to find libraries
  -liio                    # Link with libiio library
  -lm                      # Link with math library (for sqrt, sin, cos)
  -lpthread                # Link with pthread library (for threading)
```

**Translation**:
- **-Wall, -Wextra**: "Tell me about any suspicious code"
- **-O2**: "Make it run fast"
- **-I**: "Look for header files here"
- **-L**: "Look for libraries here"
- **-liio, -lm**: "I need these libraries to work"

---

#### **Step 5**: Run the Compilation

```bash
./compile_lab1_2.sh
```

**Watch for these stages**:

1. **Checking prerequisites**: Script verifies you have ARM compiler and libiio
2. **Compiling**: Translates C code to ARM machine code
3. **Linking**: Connects your code with libraries (libiio, math)
4. **Stripping**: Removes debugging symbols to make binary smaller
5. **Verification**: Confirms binary is ARM architecture

---

#### **Step 6**: Check the Result

If successful, you should see:

```bash
ls -lh lab1_2_hosted
# -rwxr-xr-x 1 user user 32K lab1_2_hosted
```

**What the size means**:
- ~32KB = Small, efficient binary (good!)
- Compare to Python approach: no interpreter needed on PlutoSDR

**Verify it's ARM**:
```bash
file lab1_2_hosted
# Output: lab1_2_hosted: ELF 32-bit LSB executable, ARM, EABI5...
```

---

### Common Compilation Errors and Solutions

#### Error 1: "arm-linux-gnueabihf-gcc: command not found"

**Problem**: ARM cross-compiler not installed

**Solution**:
```bash
# Install cross-compiler
sudo apt-get update
sudo apt-get install gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf

# Verify
arm-linux-gnueabihf-gcc --version
# Should show: arm-linux-gnueabihf-gcc (Ubuntu ...) X.X.X
```

---

#### Error 2: "iio.h: No such file or directory"

**Problem**: libiio headers not found

**Solution**:
```bash
# Check if libiio was built for ARM
ls -l /opt/arm-libs/include/iio.h

# If not found, build it:
cd ~
git clone https://github.com/analogdevicesinc/libiio.git
cd libiio
mkdir build-arm && cd build-arm
cmake .. -DCMAKE_C_COMPILER=arm-linux-gnueabihf-gcc \
         -DCMAKE_INSTALL_PREFIX=/opt/arm-libs
make -j$(nproc)
sudo make install
```

---

#### Error 3: "undefined reference to `sqrt`"

**Problem**: Math library not linked

**Solution**: Add `-lm` flag to compilation command (already in script)

```bash
# Correct:
arm-linux-gnueabihf-gcc source.c -lm -o output

# Wrong:
arm-linux-gnueabihf-gcc -lm source.c -o output  # Library order matters!
```

---

#### Error 4: Binary is x86, not ARM

**Problem**: Used regular gcc instead of ARM cross-compiler

**Solution**: Make sure you use `arm-linux-gnueabihf-gcc`, not just `gcc`

```bash
# Check which compiler was used:
file lab1_2_hosted | grep ARM
# Should see "ARM" in output

# If it says "x86-64", you used wrong compiler
# Delete and recompile with arm-linux-gnueabihf-gcc
```

---

### compile_lab1_2.sh

Complete compilation script with all necessary flags and error checking.

```bash
#!/bin/bash
#
# Compile LAB 1.2 Method 3 for PlutoSDR ARM architecture
#

set -e  # Exit on error

# Configuration
SOURCE_FILE="lab1_2_method3_hosted.c"
OUTPUT_FILE="lab1_2_hosted"
ARM_LIBS="/opt/arm-libs"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'  # No Color

echo -e "${GREEN}Compiling LAB 1.2 Method 3 for PlutoSDR${NC}"
echo "=================================================="

# Check if source file exists
if [ ! -f "$SOURCE_FILE" ]; then
    echo -e "${RED}Error: Source file '$SOURCE_FILE' not found${NC}"
    exit 1
fi

# Check if cross-compiler is installed
if ! command -v arm-linux-gnueabihf-gcc &> /dev/null; then
    echo -e "${RED}Error: ARM cross-compiler not found${NC}"
    echo "Please run install_cross_compiler.sh first"
    exit 1
fi

# Check if libiio is available
if [ ! -f "$ARM_LIBS/lib/libiio.so" ]; then
    echo -e "${RED}Error: libiio for ARM not found${NC}"
    echo "Please run build_libiio_arm.sh first"
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
echo "Next step: Deploy to PlutoSDR using deploy_lab1_2.sh"
```

**Run**: `chmod +x compile_lab1_2.sh && ./compile_lab1_2.sh`

**Expected Output**:
```
Compiling LAB 1.2 Method 3 for PlutoSDR
==================================================
Compiler:    arm-linux-gnueabihf-gcc
Source:      lab1_2_method3_hosted.c
Output:      lab1_2_hosted
libiio path: /opt/arm-libs

Compiling...
✓ Compilation successful
Stripping debug symbols...

Binary information:
lab1_2_hosted: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV)
-rwxr-xr-x 1 user user 32K lab1_2_hosted

✓ Binary is ARM architecture

Build complete!
Next step: Deploy to PlutoSDR using deploy_lab1_2.sh
```

---

## Part 4: Deployment to PlutoSDR - Detailed Step-by-Step Guide

### Understanding the Deployment Process

**What we're doing**:
1. Copy the ARM binary from your PC to PlutoSDR
2. Copy the libiio library (if not already there)
3. Make the binary executable
4. Run it!

**Think of it like**: Installing an app on your phone, but manually

---

### Step-by-Step Deployment Process

#### **Step 1**: Connect PlutoSDR to Your PC

**Physical Connection**:
```bash
# 1. Plug PlutoSDR into USB port
# 2. Wait ~10 seconds for it to boot
# 3. LED should be GREEN (ready)
```

**Verify Connection**:
```bash
# Check if PlutoSDR is reachable
ping -c 3 192.168.2.1

# Expected output:
# 64 bytes from 192.168.2.1: icmp_seq=1 ttl=64 time=0.5 ms
# ...
# 3 packets transmitted, 3 received, 0% packet loss
```

**Troubleshooting Connection**:
- **No ping response?** Check USB cable, try different port
- **Different IP?** Check your network settings (should be 192.168.2.x)
- **Firewall blocking?** Temporarily disable firewall for testing

---

#### **Step 2**: Test SSH Access

**First time SSH** (will ask to verify fingerprint):
```bash
ssh root@192.168.2.1
# Output: The authenticity of host '192.168.2.1'...
# Type: yes
# Password: analog
```

**What you'll see**:
```
Welcome to Pluto
pluto login: root
Password: analog
# root@pluto:~#
```

**Exit for now**:
```bash
exit  # Return to your PC
```

---

#### **Step 3**: Deploy libiio Library (First Time Only)

**Check if libiio is already on PlutoSDR**:
```bash
ssh root@192.168.2.1 "ls -l /usr/lib/libiio.so"
```

**If not found**, deploy it:

```bash
# On your PC:
scp /opt/arm-libs/lib/libiio.so.0 root@192.168.2.1:/usr/lib/
# Password: analog

# Create symlink on PlutoSDR:
ssh root@192.168.2.1 "ln -sf /usr/lib/libiio.so.0 /usr/lib/libiio.so"

# Verify:
ssh root@192.168.2.1 "ls -lh /usr/lib/libiio.so*"
# Should show: libiio.so -> libiio.so.0
#              libiio.so.0 (actual file, ~230KB)
```

**Why we need this**: The binary we compiled uses libiio to talk to hardware

---

#### **Step 4**: Deploy Your Compiled Binary

Create the deployment script:

```bash
# On your PC, in ~/pluto_labs/lab1_2_method3/
nano deploy_lab1_2.sh
# Paste script below, save
chmod +x deploy_lab1_2.sh
```

---

### deploy_lab1_2.sh

```bash
#!/bin/bash
# deploy_lab1_2.sh
# Deploy compiled application to PlutoSDR

PLUTO_IP="192.168.2.1"
BINARY="lab1_2_hosted"

echo "Deploying LAB 1.2 application to PlutoSDR..."

# Check if binary exists
if [ ! -f "$BINARY" ]; then
    echo "Error: Binary '$BINARY' not found"
    echo "Please compile first using compile_lab1_2.sh"
    exit 1
fi

# Copy to PlutoSDR
echo "Copying $BINARY to PlutoSDR..."
scp $BINARY root@${PLUTO_IP}:/root/

# Set executable permission
echo "Setting permissions..."
ssh root@${PLUTO_IP} "chmod +x /root/$BINARY"

# Verify
echo "Verifying deployment..."
ssh root@${PLUTO_IP} "ls -lh /root/$BINARY"

echo ""
echo "✓ Deployment complete!"
echo ""
echo "To run on PlutoSDR:"
echo "  ssh root@${PLUTO_IP}"
echo "  cd /root"
echo "  ./$BINARY"
```

**Run**: `chmod +x deploy_lab1_2.sh && ./deploy_lab1_2.sh`

**Note**: If you haven't deployed libiio yet, run `deploy_libiio.sh` from LAB 0 or 1.1 first.

---

## Part 5: Running on PlutoSDR

### Connect to PlutoSDR

```bash
ssh root@192.168.2.1
# Password: analog
```

### Run the Application

```bash
cd /root
./lab1_2_hosted
```

### Expected Output

```
======================================================================
LAB 1.2 - Method 3: RF Gain Staging Hosted Application
Running on PlutoSDR ARM Cortex-A9
======================================================================

Initializing PlutoSDR (local IIO context)...
  IIO context created (local)
  Backend: local
  ✓ Found ad9361-phy
  ✓ Found RX device
  ✓ Found RX PHY channel
  ✓ RX I/Q channels enabled
  ✓ Sample rate: 2.084 MSPS
  ✓ RX frequency: 915.000 MHz
  ✓ RX buffer created (16384 samples)
PlutoSDR initialization complete!

Starting Test 1: Manual Gain Sweep

======================================================================
MANUAL GAIN SWEEP TEST
======================================================================
Gain Range: 0 to 73 dB (step: 3 dB)
======================================================================

Gain (dB) │ Power (dB) │ Crest (dB) │ Peak    │ Status
──────────┼────────────┼────────────┼─────────┼─────────
    0     │     -45.2  │      +3.1  │  0.012  │ ✓ OK
    3     │     -42.1  │      +3.2  │  0.015  │ ✓ OK
    6     │     -39.3  │      +3.0  │  0.021  │ ✓ OK
    9     │     -36.2  │      +3.1  │  0.030  │ ✓ OK
   12     │     -33.1  │      +3.2  │  0.042  │ ✓ OK
   15     │     -30.0  │      +3.1  │  0.060  │ ✓ OK
   18     │     -27.2  │      +3.0  │  0.085  │ ✓ OK
   21     │     -24.1  │      +3.1  │  0.120  │ ✓ OK
   24     │     -21.0  │      +3.2  │  0.170  │ ✓ OK
   27     │     -18.1  │      +3.1  │  0.240  │ ✓ OK
   30     │     -15.2  │      +3.0  │  0.340  │ ✓ OK
   33     │     -12.0  │      +3.1  │  0.480  │ ✓ OK
   36     │      -9.1  │      +3.2  │  0.680  │ ✓ OK
   39     │      -6.2  │      +3.1  │  0.850  │ ✓ OK
   42     │      -3.1  │      +3.0  │  0.920  │ ✗ CLIP
   45     │      -0.5  │      +2.8  │  0.960  │ ✗ CLIP
   48     │      +2.2  │      +2.5  │  0.985  │ ✗ CLIP
   51     │      +5.1  │      +2.1  │  0.995  │ ✗ CLIP
   54     │      +7.8  │      +1.8  │  0.998  │ ✗ CLIP
   57     │     +10.5  │      +1.5  │  0.999  │ ✗ CLIP
   60     │     +13.2  │      +1.2  │  1.000  │ ✗ CLIP
   63     │     +15.8  │      +0.9  │  1.000  │ ✗ CLIP
   66     │     +18.3  │      +0.6  │  1.000  │ ✗ CLIP
   69     │     +20.7  │      +0.3  │  1.000  │ ✗ CLIP
   72     │     +23.0  │      +0.1  │  1.000  │ ✗ CLIP

Summary:
  Total tests:      25
  Clipped:          11 (44.0%)
  No clipping:      14 (56.0%)

Test 1 Duration: 1.82 seconds

Starting Test 2: AGC Mode Comparison

======================================================================
AGC MODE COMPARISON TEST
======================================================================
Testing different AGC modes with 10 measurements each
======================================================================

Testing AGC Mode: slow_attack
────────────────────────────────────────
Waiting for AGC to settle (500 ms)...

Meas │ Power (dB) │ Crest (dB) │ Peak    │ Status
─────┼────────────┼────────────┼─────────┼─────────
  1  │      -8.5  │      +3.1  │  0.712  │ ✓ OK
  2  │      -8.3  │      +3.2  │  0.695  │ ✓ OK
  3  │      -8.6  │      +3.0  │  0.720  │ ✓ OK
  4  │      -8.4  │      +3.1  │  0.703  │ ✓ OK
  5  │      -8.5  │      +3.2  │  0.715  │ ✓ OK
  6  │      -8.7  │      +3.1  │  0.728  │ ✓ OK
  7  │      -8.3  │      +3.0  │  0.690  │ ✓ OK
  8  │      -8.6  │      +3.1  │  0.718  │ ✓ OK
  9  │      -8.4  │      +3.2  │  0.708  │ ✓ OK
 10  │      -8.5  │      +3.1  │  0.712  │ ✓ OK

Mode Statistics:
  Avg Power:        -8.5 dB
  Power Std Dev:    0.11 dB
  Avg Crest Factor: 3.1 dB
  Clipping Events:  0 / 10

Testing AGC Mode: fast_attack
────────────────────────────────────────
Waiting for AGC to settle (500 ms)...

Meas │ Power (dB) │ Crest (dB) │ Peak    │ Status
─────┼────────────┼────────────┼─────────┼─────────
  1  │      -8.2  │      +3.3  │  0.685  │ ✓ OK
  2  │      -9.1  │      +2.9  │  0.758  │ ✓ OK
  3  │      -7.8  │      +3.5  │  0.652  │ ✓ OK
  4  │      -8.5  │      +3.2  │  0.710  │ ✓ OK
  5  │      -9.3  │      +2.8  │  0.770  │ ✓ OK
  6  │      -7.5  │      +3.6  │  0.638  │ ✓ OK
  7  │      -8.8  │      +3.1  │  0.735  │ ✓ OK
  8  │      -9.0  │      +2.9  │  0.748  │ ✓ OK
  9  │      -7.9  │      +3.4  │  0.665  │ ✓ OK
 10  │      -8.6  │      +3.2  │  0.720  │ ✓ OK

Mode Statistics:
  Avg Power:        -8.5 dB
  Power Std Dev:    0.58 dB
  Avg Crest Factor: 3.2 dB
  Clipping Events:  0 / 10

======================================================================
AGC COMPARISON SUMMARY
======================================================================
Metric              │ Slow Attack  │ Fast Attack  │ Winner
────────────────────┼──────────────┼──────────────┼────────────
Avg Power (dB)      │        -8.5  │        -8.5  │ Tie
Power Stability (σ) │        0.11  │        0.58  │ Slow
Avg Crest (dB)      │        3.1  │        3.2  │ Tie
Clipping Events     │          0  │          0  │ Tie
======================================================================
Test 2 Duration: 2.51 seconds


======================================================================
ALL TESTS COMPLETE
======================================================================
Total Test Duration: 4.33 seconds

✓ RF Gain Staging tests complete!
```

---

## Part 6: Performance Comparison

### Method Comparison Table

| Metric | Method 1 (Simulation) | Method 2 (External) | Method 3 (Hosted) |
|--------|----------------------|--------------------|--------------------|
| **Execution Time** | ~2 seconds | ~8-10 seconds | ~4-5 seconds |
| **Network Latency** | N/A | 5-10 ms per sample | <0.1 ms (local IIO) |
| **CPU Usage** | Host PC | Host PC | PlutoSDR ARM only |
| **Power Consumption** | PC power | PC + Pluto (~55W) | Pluto only (~2W) |
| **AGC Settling** | Simulated | Real (HW AGC) | Real (HW AGC) |
| **Clipping Detection** | Perfect | Perfect | Perfect |
| **Deployment** | N/A | Requires PC | Standalone |
| **Real-time Capable** | No | Yes | Yes (better) |

### Key Observations

1. **Hosted app is 2x faster** than external app:
   - Method 2 (external): ~8-10 seconds total
   - Method 3 (hosted): ~4-5 seconds total
   - Reason: No network IIO overhead, local buffer access

2. **Power stability measurement is more accurate**:
   - Method 3 directly accesses ADC data without USB buffering delays
   - Lower jitter in measurements

3. **Standalone AGC monitoring**:
   - Can run continuously on PlutoSDR
   - Perfect for calibration workflows
   - Battery-powered field testing

4. **Development trade-offs**:
   - Method 1: Algorithm development and verification
   - Method 2: Hardware validation with easy debugging
   - Method 3: Production deployment and embedded monitoring

---

## Part 7: Integration with Other Labs and Projects

### How This Lab Fits Into the Bigger Picture

This lab taught you **how to configure gain** properly. But gain configuration is needed in EVERY SDR project! Here's how to use what you learned:

---

### Integration Example 1: Using with LAB 1.3 (I/Q Analysis)

**Problem**: You're capturing I/Q samples but signal is too weak

**Solution**: Use the gain knowledge from this lab

```c
// From LAB 1.2: We learned optimal gain is ~60 dB for typical signals

// In LAB 1.3 code, set gain properly:
set_channel_attr_str(sdr->rx_phy_ch, "gain_control_mode", "manual");
set_channel_attr_ll(sdr->rx_phy_ch, "hardwaregain", 60);  // Use optimal gain

// Now I/Q samples will have good SNR!
```

**Result**: Clear I/Q samples, no clipping, good SNR

---

### Integration Example 2: Using with PROJECT 5 (Tactical Radio)

**Problem**: Tactical radio needs to work with varying signal strengths

**Solution**: Implement adaptive gain control

```c
// Use LAB 1.2's gain measurement technique in your tactical radio:

int adaptive_gain_adjustment(PlutoSDR *sdr) {
    int current_gain = 30;  // Start conservative

    for (int iteration = 0; iteration < 5; iteration++) {
        // Set gain
        set_manual_gain(sdr, current_gain);
        usleep(50000);  // 50ms settling

        // Measure (from LAB 1.2)
        GainMeasurement meas;
        measure_signal(sdr, &meas);

        // Adjust based on results
        if (meas.clipping) {
            current_gain -= 6;  // Reduce if clipping
            printf("Clipping detected, reducing gain to %d dB\n", current_gain);
        } else if (meas.power_db < -30.0) {
            current_gain += 6;  // Increase if too weak
            printf("Weak signal, increasing gain to %d dB\n", current_gain);
        } else {
            printf("✓ Optimal gain found: %d dB\n", current_gain);
            break;  // Good!
        }
    }

    return current_gain;
}

// Use in tactical radio initialization:
int optimal_gain = adaptive_gain_adjustment(&tactical_sdr);
printf("Tactical radio using %d dB gain\n", optimal_gain);
```

**Result**: Radio automatically finds best gain for current conditions

---

### Integration Example 3: Calibration Table Creation

**Use Case**: Create a calibration table for different scenarios

**How to do it**:

```c
// Run LAB 1.2's gain sweep in different environments
// Save results to a calibration file

typedef struct {
    char environment[32];  // "Indoor", "Outdoor", "Urban", etc.
    int recommended_gain;
    double noise_floor_db;
} GainCalibration;

// Example calibration table:
GainCalibration cal_table[] = {
    {"Indoor office",     50, -85.0},
    {"Outdoor rural",     60, -90.0},
    {"Urban with WiFi",   45, -75.0},
    {"Next to FM tower",  20, -50.0}
};

// Later, in your application:
int select_gain_for_environment(const char *env) {
    for (int i = 0; i < sizeof(cal_table)/sizeof(cal_table[0]); i++) {
        if (strcmp(cal_table[i].environment, env) == 0) {
            return cal_table[i].recommended_gain;
        }
    }
    return 50;  // Default fallback
}

// Usage:
int gain = select_gain_for_environment("Outdoor rural");
set_manual_gain(&sdr, gain);
```

---

### Integration Example 4: Combining with Frequency Hopping

**From PROJECT 4**: Frequency hopping with adaptive gain per frequency

```c
// Some frequencies might need different gain (interference, propagation)

typedef struct {
    long long frequency_hz;
    int optimal_gain_db;
    bool has_interference;
} FrequencyProfile;

FrequencyProfile freq_profiles[NUM_HOPS];

// Calibration phase (run once):
void calibrate_frequency_hopping_gains(PlutoSDR *sdr) {
    printf("Calibrating gains for %d hop frequencies...\n", NUM_HOPS);

    for (int i = 0; i < NUM_HOPS; i++) {
        // Tune to frequency
        long long freq = hop_frequencies[i];
        set_channel_attr_ll(rx_lo, "frequency", freq);

        // Find optimal gain using LAB 1.2 technique
        int best_gain = 0;
        double best_snr = -999.0;

        for (int gain = 0; gain <= 73; gain += 3) {
            set_manual_gain(sdr, gain);
            usleep(50000);

            GainMeasurement meas;
            measure_signal(sdr, &meas);

            double snr = meas.power_db - noise_floor_estimate(sdr);

            if (snr > best_snr && !meas.clipping) {
                best_snr = snr;
                best_gain = gain;
            }
        }

        // Save profile
        freq_profiles[i].frequency_hz = freq;
        freq_profiles[i].optimal_gain_db = best_gain;
        freq_profiles[i].has_interference = (best_snr < 10.0);

        printf("  %lld MHz: Gain=%d dB, SNR=%.1f dB %s\n",
               freq / 1000000, best_gain, best_snr,
               freq_profiles[i].has_interference ? "⚠ INTERFERENCE" : "✓");
    }
}

// During hopping:
void hop_with_adaptive_gain(int hop_index) {
    // Set frequency
    set_channel_attr_ll(rx_lo, "frequency", freq_profiles[hop_index].frequency_hz);

    // Set pre-calibrated gain
    set_manual_gain(&sdr, freq_profiles[hop_index].optimal_gain_db);

    // Skip this hop if bad interference
    if (freq_profiles[hop_index].has_interference) {
        printf("Skipping frequency due to interference\n");
        return;
    }

    // Continue with transmission/reception...
}
```

---

### Integration Example 5: Building a Spectrum Analyzer

**Combine with**: FFT (from future labs) + Gain control

```c
// Use LAB 1.2 gain control + FFT to build spectrum analyzer

void spectrum_analyzer_sweep(PlutoSDR *sdr, long long start_freq, long long stop_freq) {
    int gain = 50;  // Start with medium gain
    long long step = 1000000;  // 1 MHz steps

    printf("Spectrum Analyzer: %.1f - %.1f MHz\n",
           start_freq/1e6, stop_freq/1e6);
    printf("Freq (MHz) │ Power (dB) │ Gain (dB) │ Status\n");
    printf("───────────┼────────────┼───────────┼─────────\n");

    for (long long freq = start_freq; freq <= stop_freq; freq += step) {
        // Tune
        set_channel_attr_ll(rx_lo, "frequency", freq);

        // Adjust gain if needed
        set_manual_gain(sdr, gain);
        usleep(10000);  // 10ms settling

        // Measure
        GainMeasurement meas;
        measure_signal(sdr, &meas);

        // Auto-adjust gain for next frequency
        if (meas.clipping) {
            gain = (gain > 6) ? gain - 6 : 0;
        } else if (meas.power_db < -40.0 && gain < 67) {
            gain += 6;
        }

        // Display
        const char *status = meas.clipping ? "CLIP" : "OK";
        printf("%10.1f │ %+9.1f  │ %8d  │ %s\n",
               freq/1e6, meas.power_db, gain, status);
    }
}

// Usage:
spectrum_analyzer_sweep(&sdr, 900e6, 1000e6);  // Sweep 900-1000 MHz
```

---

### Integration Best Practices

**1. Always check for clipping**:
```c
if (meas.clipping) {
    fprintf(stderr, "WARNING: Signal clipping at %d dB gain!\n", current_gain);
    // Reduce gain or alert user
}
```

**2. Log gain changes** for debugging:
```c
FILE *gain_log = fopen("gain_log.txt", "a");
fprintf(gain_log, "%ld,%d,%.2f,%s\n",
        time(NULL), gain_db, power_db,
        clipping ? "CLIP" : "OK");
fclose(gain_log);
```

**3. Provide user feedback**:
```c
printf("Signal Quality:\n");
printf("  Gain:     %d dB\n", current_gain);
printf("  Power:    %.1f dB\n", measured_power);
printf("  Clipping: %s\n", clipping ? "YES ⚠" : "NO ✓");
printf("  Quality:  %s\n", quality_assessment(measured_power, clipping));
```

---

### Real-World Integration: Complete Receiver

**Putting it all together** - A complete receiver using multiple labs:

```c
typedef struct {
    PlutoSDR sdr;
    int rx_gain;          // From LAB 1.2
    double dc_i, dc_q;    // From LAB 1.3
    // Add more as you learn more labs
} CompleteReceiver;

int init_complete_receiver(CompleteReceiver *rx, long long freq) {
    // Initialize hardware
    init_plutosdr(&rx->sdr);

    // Set frequency
    set_channel_attr_ll(rx_lo, "frequency", freq);

    // Find optimal gain (LAB 1.2)
    rx->rx_gain = find_optimal_gain(&rx->sdr);
    printf("✓ Optimal gain: %d dB\n", rx->rx_gain);

    // Measure DC offset (LAB 1.3)
    measure_dc_offset(&rx->sdr, &rx->dc_i, &rx->dc_q);
    printf("✓ DC offset: I=%.2f, Q=%.2f\n", rx->dc_i, rx->dc_q);

    // Ready to receive!
    return 0;
}

// Use it:
CompleteReceiver my_receiver;
init_complete_receiver(&my_receiver, 915000000);  // 915 MHz

// Now receive with optimized settings
receive_and_process(&my_receiver);
```

---

## Part 8: Troubleshooting

### Common Issues

#### 1. Compilation Errors

**Error**: `iio.h: No such file or directory`
**Solution**: Verify libiio was built for ARM:
```bash
ls -l /opt/arm-libs/include/iio.h
```

**Error**: `undefined reference to sqrt`
**Solution**: Link math library. Make sure `-lm` is included:
```bash
arm-linux-gnueabihf-gcc ... source.c -liio -lm
```

#### 2. Runtime Errors

**Error**: `Failed to find ad9361-phy device`
**Solution**: Verify PlutoSDR firmware is loaded:
```bash
ssh root@192.168.2.1
iio_info -s
```

**Error**: `Failed to set gain_control_mode`
**Solution**: Check valid AGC modes:
```bash
ssh root@192.168.2.1
cat /sys/bus/iio/devices/iio:device1/in_voltage0_gain_control_mode_available
```

Should show: `manual fast_attack slow_attack hybrid`

#### 3. Performance Issues

**Issue**: All measurements show clipping
**Solutions**:
- Reduce TX power or add attenuation
- Start with lower MAX_RX_GAIN (e.g., 60 dB)
- Increase distance between TX/RX antennas

**Issue**: Very low signal power (< -60 dB)
**Solutions**:
- Check antenna connections
- Verify TX is active (use spectrum analyzer or RTL-SDR)
- Increase TX power
- Reduce attenuation in test setup

---

## Part 8: Advanced Exercises

### Exercise 1: Adaptive Gain Algorithm

Implement automatic gain adjustment that maintains optimal signal level:
```c
// Target: Keep signal at -10 dBFS (80% of full scale)
int adaptive_gain_control(PlutoSDR *sdr, double target_power_db) {
    int current_gain = 30;  // Start at mid-range

    for (int iter = 0; iter < 10; iter++) {
        set_manual_gain(sdr, current_gain);
        usleep(50000);

        GainMeasurement meas;
        measure_signal(sdr, &meas);

        // Adjust gain based on error
        double error = target_power_db - meas.power_db;
        int gain_adjustment = (int)(error / 3.0);  // ~3 dB per step

        current_gain += gain_adjustment;
        // Clamp to valid range...

        if (fabs(error) < 1.0) break;  // Converged
    }

    return current_gain;
}
```

### Exercise 2: Gain Table Calibration

Measure actual gain vs nominal gain to create calibration table:
- Inject known signal level
- Sweep through all gain values
- Record actual vs expected power
- Store calibration data to file

### Exercise 3: Compression Point Measurement

Find the 1-dB compression point:
- Gradually increase TX power
- Measure RX power at fixed gain
- Detect where gain drops by 1 dB
- Plot compression curve

### Exercise 4: Noise Figure Estimation

Estimate receiver noise figure:
- Disable TX (measure only noise floor)
- Measure noise power at different gains
- Calculate excess noise
- Estimate NF using Y-factor method

---

## Summary

### What We Learned

1. **Direct hardware control** via local IIO backend
2. **AGC mode differences** (slow vs fast attack)
3. **Clipping detection** in real-time
4. **Manual gain optimization** for different scenarios
5. **Embedded performance monitoring**

### Key Takeaways

- **Local IIO is 2x faster** than network IIO for measurement loops
- **Slow AGC is more stable** (lower standard deviation)
- **Fast AGC responds quicker** but with more jitter
- **Manual gain** provides best SNR when signal level is known
- **Hosted apps are ideal** for calibration and monitoring tasks

### When to Use Method 3

Use hosted applications for gain testing when you need:
- ✓ Fast measurement loops (100+ measurements/sec)
- ✓ Embedded calibration workflows
- ✓ Continuous gain monitoring
- ✓ Field testing without PC
- ✓ Production gain table generation

Use external applications (Method 2) when you need:
- ✓ Quick interactive testing
- ✓ Real-time visualization (plots)
- ✓ Integration with Python DSP libraries
- ✓ Rapid prototyping

---

## Next Steps

Continue to **LAB 1.3: I/Q Sampling and Visualization** to learn about baseband I/Q signal processing.

**Complete LAB 1.2 Series**:
- ✓ Method 1: Simulation (understand gain theory)
- ✓ Method 2: External App (hardware validation)
- ✓ Method 3: Hosted App (production calibration)

---

**End of LAB 1.2 - Method 3**
