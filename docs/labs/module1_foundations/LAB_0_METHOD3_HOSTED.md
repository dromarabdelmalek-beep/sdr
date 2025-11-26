# LAB 0 - METHOD 3: Hosted Application (Complete Guide)

## 🔴 METHOD 3: HOSTED APP (C on PlutoSDR ARM CPU)

### Overview

Hosted applications run **directly on PlutoSDR's embedded Linux** using the ARM Cortex-A9 CPU. This approach provides:

- **Standalone operation** (no PC needed)
- **Low latency** (< 1 ms, no network delay)
- **Direct hardware access** (local IIO)
- **Production deployment** ready
- **Lower power consumption**

**Architecture:**
```
PlutoSDR Hardware
┌──────────────────────────────────────┐
│  ┌────────────────────────────────┐  │
│  │  Your C Application            │  │
│  │  (runs on ARM CPU)             │  │
│  └───────────┬────────────────────┘  │
│              │                        │
│              │ Local IIO              │
│              │ (no network)           │
│  ┌───────────▼────────────────────┐  │
│  │  libiio (local backend)        │  │
│  └───────────┬────────────────────┘  │
│              │                        │
│  ┌───────────▼────────────────────┐  │
│  │  IIO Kernel Drivers            │  │
│  └───────────┬────────────────────┘  │
│              │                        │
│  ┌───────────▼────────────────────┐  │
│  │  AD9363 Transceiver            │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

---

## Part 1: Setting Up Cross-Compilation Environment

### Step 1.1: Install Cross-Compiler

The PlutoSDR uses an **ARM Cortex-A9** processor, so we need an ARM cross-compiler.

**On Ubuntu/Debian:**

```bash
#!/bin/bash
# install_cross_compiler.sh

echo "Installing ARM Cross-Compilation Toolchain"
echo "==========================================="

# Update package list
sudo apt-get update

# Install ARM GCC cross-compiler
sudo apt-get install -y \
    gcc-arm-linux-gnueabihf \
    g++-arm-linux-gnueabihf \
    binutils-arm-linux-gnueabihf

# Install build essentials
sudo apt-get install -y \
    build-essential \
    cmake \
    git \
    pkg-config

# Verify installation
echo ""
echo "Verifying installation..."
arm-linux-gnueabihf-gcc --version

if [ $? -eq 0 ]; then
    echo "✓ Cross-compiler installed successfully!"
else
    echo "✗ Installation failed"
    exit 1
fi

echo ""
echo "Environment ready for PlutoSDR cross-compilation"
```

**Make executable and run:**

```bash
chmod +x install_cross_compiler.sh
./install_cross_compiler.sh
```

### Step 1.2: Download and Build libiio

We need to cross-compile **libiio** library for ARM.

```bash
#!/bin/bash
# build_libiio_arm.sh

echo "Building libiio for ARM"
echo "======================="

# Configuration
INSTALL_PREFIX="/opt/arm-libs"
BUILD_DIR="build_arm"

# Clone libiio if not already present
if [ ! -d "libiio" ]; then
    echo "Cloning libiio..."
    git clone https://github.com/analogdevicesinc/libiio.git
fi

cd libiio

# Create build directory
mkdir -p $BUILD_DIR
cd $BUILD_DIR

# Configure with CMake for ARM cross-compilation
echo ""
echo "Configuring libiio for ARM..."

cmake .. \
    -DCMAKE_SYSTEM_NAME=Linux \
    -DCMAKE_SYSTEM_PROCESSOR=arm \
    -DCMAKE_C_COMPILER=arm-linux-gnueabihf-gcc \
    -DCMAKE_CXX_COMPILER=arm-linux-gnueabihf-g++ \
    -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX \
    -DWITH_NETWORK_BACKEND=OFF \
    -DWITH_SERIAL_BACKEND=OFF \
    -DWITH_USB_BACKEND=OFF \
    -DENABLE_IPV6=OFF

if [ $? -ne 0 ]; then
    echo "✗ CMake configuration failed"
    exit 1
fi

# Build
echo ""
echo "Building libiio..."
make -j$(nproc)

if [ $? -ne 0 ]; then
    echo "✗ Build failed"
    exit 1
fi

# Install to local prefix
echo ""
echo "Installing libiio to $INSTALL_PREFIX..."
sudo make install

echo ""
echo "✓ libiio built successfully for ARM!"
echo "Install location: $INSTALL_PREFIX"
```

**Run the script:**

```bash
chmod +x build_libiio_arm.sh
./build_libiio_arm.sh
```

### Step 1.3: Verify Environment

```bash
#!/bin/bash
# verify_environment.sh

echo "Verifying Cross-Compilation Environment"
echo "========================================"

# Check cross-compiler
echo ""
echo "1. Checking ARM cross-compiler..."
if command -v arm-linux-gnueabihf-gcc &> /dev/null; then
    echo "   ✓ arm-linux-gnueabihf-gcc found"
    arm-linux-gnueabihf-gcc --version | head -1
else
    echo "   ✗ arm-linux-gnueabihf-gcc not found"
    exit 1
fi

# Check libiio headers
echo ""
echo "2. Checking libiio headers..."
if [ -f "/opt/arm-libs/include/iio.h" ]; then
    echo "   ✓ libiio headers found at /opt/arm-libs/include/"
else
    echo "   ✗ libiio headers not found"
    echo "   Run: ./build_libiio_arm.sh"
    exit 1
fi

# Check libiio library
echo ""
echo "3. Checking libiio library..."
if [ -f "/opt/arm-libs/lib/libiio.so" ]; then
    echo "   ✓ libiio.so found at /opt/arm-libs/lib/"
    ls -lh /opt/arm-libs/lib/libiio.so*
else
    echo "   ✗ libiio.so not found"
    exit 1
fi

echo ""
echo "========================================"
echo "✓ Environment ready for cross-compilation!"
echo ""
echo "Next steps:"
echo "  1. Write your C application"
echo "  2. Compile with compile_for_pluto.sh"
echo "  3. Deploy to PlutoSDR"
```

---

## Part 2: Writing the Hosted Application

### Step 2.1: Complete Source Code

**File:** `lab0_method3_hosted.c`

```c
/*
 * LAB 0 - Method 3: Hello PlutoSDR (Hosted Application)
 *
 * This application runs directly on PlutoSDR's ARM CPU
 * Demonstrates:
 * - Local IIO context creation
 * - AD9361 configuration
 * - TX/RX buffer management
 * - Tone generation and detection
 * - Low-latency operation
 *
 * Compile:
 *   See compile_for_pluto.sh
 *
 * Deploy:
 *   scp lab0_hosted root@192.168.2.1:/root/
 *
 * Run:
 *   ssh root@192.168.2.1
 *   ./lab0_hosted
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
#include <iio.h>
#include <unistd.h>
#include <signal.h>
#include <time.h>

// ===================================================================
// CONFIGURATION
// ===================================================================

#define SAMPLE_RATE     2084000     // 2.084 MSPS
#define CENTER_FREQ     915000000   // 915 MHz
#define TONE_OFFSET     100000      // 100 kHz
#define BUFFER_SIZE     16384       // Samples per buffer
#define TX_GAIN         -10         // dB attenuation
#define RX_GAIN         60          // dB gain

// ===================================================================
// GLOBAL VARIABLES
// ===================================================================

static volatile int keep_running = 1;

// IIO context and devices
struct iio_context *ctx = NULL;
struct iio_device *phy = NULL;
struct iio_device *tx_dev = NULL;
struct iio_device *rx_dev = NULL;

// Buffers
struct iio_buffer *txbuf = NULL;
struct iio_buffer *rxbuf = NULL;

// ===================================================================
// SIGNAL HANDLER
// ===================================================================

void signal_handler(int signum)
{
    printf("\n\nReceived signal %d, shutting down...\n", signum);
    keep_running = 0;
}

// ===================================================================
// HELPER FUNCTIONS
// ===================================================================

void print_separator(const char *title)
{
    printf("\n");
    printf("======================================================================\n");
    printf("%s\n", title);
    printf("======================================================================\n");
}

void print_step(const char *step, const char *description)
{
    printf("\n%s %s\n", step, description);
    printf("----------------------------------------------------------------------\n");
}

// ===================================================================
// IIO SETUP FUNCTIONS
// ===================================================================

int setup_iio_context(void)
{
    print_step("🔌 STEP 1:", "Creating local IIO context...");

    // Create local context (no network)
    ctx = iio_create_local_context();
    if (!ctx) {
        fprintf(stderr, "✗ Failed to create local IIO context\n");
        fprintf(stderr, "  Make sure you're running on PlutoSDR\n");
        return -1;
    }

    printf("✓ Local IIO context created\n");
    printf("  Devices found: %u\n", iio_context_get_devices_count(ctx));

    // List all devices
    for (unsigned int i = 0; i < iio_context_get_devices_count(ctx); i++) {
        struct iio_device *dev = iio_context_get_device(ctx, i);
        printf("  [%u] %s (%u channels)\n",
               i,
               iio_device_get_name(dev),
               iio_device_get_channels_count(dev));
    }

    return 0;
}

int setup_devices(void)
{
    print_step("📡 STEP 2:", "Setting up IIO devices...");

    // Find PHY device (AD9361)
    phy = iio_context_find_device(ctx, "ad9361-phy");
    if (!phy) {
        fprintf(stderr, "✗ Failed to find ad9361-phy device\n");
        return -1;
    }
    printf("✓ Found ad9361-phy\n");

    // Find TX device
    tx_dev = iio_context_find_device(ctx, "cf-ad9361-dds-core-lpc");
    if (!tx_dev) {
        fprintf(stderr, "✗ Failed to find TX device\n");
        return -1;
    }
    printf("✓ Found TX device: %s\n", iio_device_get_name(tx_dev));

    // Find RX device
    rx_dev = iio_context_find_device(ctx, "cf-ad9361-lpc");
    if (!rx_dev) {
        fprintf(stderr, "✗ Failed to find RX device\n");
        return -1;
    }
    printf("✓ Found RX device: %s\n", iio_device_get_name(rx_dev));

    return 0;
}

int configure_ad9361(void)
{
    print_step("⚙️  STEP 3:", "Configuring AD9361 transceiver...");

    // Configure TX LO frequency
    struct iio_channel *tx_lo = iio_device_find_channel(phy, "altvoltage1", true);
    if (!tx_lo) {
        fprintf(stderr, "✗ Failed to find TX LO channel\n");
        return -1;
    }
    iio_channel_attr_write_longlong(tx_lo, "frequency", CENTER_FREQ);
    printf("✓ TX LO:           %ld MHz\n", CENTER_FREQ / 1000000);

    // Configure RX LO frequency
    struct iio_channel *rx_lo = iio_device_find_channel(phy, "altvoltage0", true);
    if (!rx_lo) {
        fprintf(stderr, "✗ Failed to find RX LO channel\n");
        return -1;
    }
    iio_channel_attr_write_longlong(rx_lo, "frequency", CENTER_FREQ);
    printf("✓ RX LO:           %ld MHz\n", CENTER_FREQ / 1000000);

    // Configure sample rate (TX)
    struct iio_channel *tx_ch = iio_device_find_channel(phy, "voltage0", true);
    if (!tx_ch) {
        fprintf(stderr, "✗ Failed to find TX channel\n");
        return -1;
    }
    iio_channel_attr_write_longlong(tx_ch, "sampling_frequency", SAMPLE_RATE);
    printf("✓ Sample rate:     %.3f MSPS\n", SAMPLE_RATE / 1e6);

    // Configure TX gain
    iio_channel_attr_write_longlong(tx_ch, "hardwaregain", TX_GAIN);
    printf("✓ TX gain:         %d dB\n", TX_GAIN);

    // Configure RX gain
    struct iio_channel *rx_ch = iio_device_find_channel(phy, "voltage0", false);
    if (!rx_ch) {
        fprintf(stderr, "✗ Failed to find RX channel\n");
        return -1;
    }

    iio_channel_attr_write(rx_ch, "gain_control_mode", "manual");
    iio_channel_attr_write_longlong(rx_ch, "hardwaregain", RX_GAIN);
    printf("✓ RX gain:         %d dB\n", RX_GAIN);

    return 0;
}

int setup_buffers(void)
{
    print_step("📦 STEP 4:", "Creating TX/RX buffers...");

    // Enable TX channels (I and Q)
    struct iio_channel *tx_i = iio_device_find_channel(tx_dev, "voltage0", true);
    struct iio_channel *tx_q = iio_device_find_channel(tx_dev, "voltage1", true);

    if (!tx_i || !tx_q) {
        fprintf(stderr, "✗ Failed to find TX I/Q channels\n");
        return -1;
    }

    iio_channel_enable(tx_i);
    iio_channel_enable(tx_q);
    printf("✓ TX I/Q channels enabled\n");

    // Enable RX channels (I and Q)
    struct iio_channel *rx_i = iio_device_find_channel(rx_dev, "voltage0", false);
    struct iio_channel *rx_q = iio_device_find_channel(rx_dev, "voltage1", false);

    if (!rx_i || !rx_q) {
        fprintf(stderr, "✗ Failed to find RX I/Q channels\n");
        return -1;
    }

    iio_channel_enable(rx_i);
    iio_channel_enable(rx_q);
    printf("✓ RX I/Q channels enabled\n");

    // Create TX buffer (cyclic for continuous transmission)
    txbuf = iio_device_create_buffer(tx_dev, BUFFER_SIZE, true);
    if (!txbuf) {
        fprintf(stderr, "✗ Failed to create TX buffer\n");
        return -1;
    }
    printf("✓ TX buffer created: %u samples (cyclic)\n", BUFFER_SIZE);

    // Create RX buffer
    rxbuf = iio_device_create_buffer(rx_dev, BUFFER_SIZE, false);
    if (!rxbuf) {
        fprintf(stderr, "✗ Failed to create RX buffer\n");
        return -1;
    }
    printf("✓ RX buffer created: %u samples\n", BUFFER_SIZE);

    return 0;
}

// ===================================================================
// SIGNAL GENERATION
// ===================================================================

void generate_tone(int16_t *buffer, size_t num_samples, double tone_freq, double fs)
{
    printf("\n📻 Generating %zu-sample tone at %.1f kHz offset...\n",
           num_samples, tone_freq / 1e3);

    for (size_t i = 0; i < num_samples; i++) {
        double t = (double)i / fs;
        double phase = 2.0 * M_PI * tone_freq * t;

        // Generate complex tone: e^(j*2πft)
        // Scale to 80% of 12-bit range to avoid clipping
        double amplitude = 0.8 * 2048.0;

        double i_val = amplitude * cos(phase);
        double q_val = amplitude * sin(phase);

        // Interleaved I/Q samples
        buffer[2*i]     = (int16_t)i_val;  // I
        buffer[2*i + 1] = (int16_t)q_val;  // Q
    }

    printf("✓ Tone generated\n");
    printf("  First sample: I=%d, Q=%d\n", buffer[0], buffer[1]);
    printf("  Last sample:  I=%d, Q=%d\n",
           buffer[2*(num_samples-1)], buffer[2*(num_samples-1)+1]);
}

// ===================================================================
// SIGNAL ANALYSIS
// ===================================================================

void analyze_received_signal(int16_t *buffer, size_t num_samples, double fs)
{
    print_step("🔍 STEP 6:", "Analyzing received signal...");

    // Calculate statistics
    double i_sum = 0, q_sum = 0;
    double power_sum = 0;
    int16_t i_min = 32767, i_max = -32768;
    int16_t q_min = 32767, q_max = -32768;

    for (size_t i = 0; i < num_samples; i++) {
        int16_t i_val = buffer[2*i];
        int16_t q_val = buffer[2*i + 1];

        i_sum += i_val;
        q_sum += q_val;

        // Track min/max
        if (i_val < i_min) i_min = i_val;
        if (i_val > i_max) i_max = i_val;
        if (q_val < q_min) q_min = q_val;
        if (q_val > q_max) q_max = q_val;

        // Power
        power_sum += (double)i_val * i_val + (double)q_val * q_val;
    }

    double i_mean = i_sum / num_samples;
    double q_mean = q_sum / num_samples;
    double avg_power = power_sum / num_samples;
    double avg_power_db = 10.0 * log10(avg_power);

    printf("\n📊 Signal Statistics:\n");
    printf("  Samples:       %zu\n", num_samples);
    printf("  I range:       [%d, %d]\n", i_min, i_max);
    printf("  Q range:       [%d, %d]\n", q_min, q_max);
    printf("  I mean:        %.2f\n", i_mean);
    printf("  Q mean:        %.2f\n", q_mean);
    printf("  Average power: %.2f dB\n", avg_power_db);

    // Simple tone detection by correlation
    printf("\n🎯 Tone Detection:\n");

    double corr_real = 0, corr_imag = 0;

    for (size_t i = 0; i < num_samples; i++) {
        double t = (double)i / fs;
        double phase = 2.0 * M_PI * TONE_OFFSET * t;

        double ref_i = cos(phase);
        double ref_q = sin(phase);

        // Correlate
        corr_real += buffer[2*i] * ref_i + buffer[2*i+1] * ref_q;
        corr_imag += buffer[2*i+1] * ref_i - buffer[2*i] * ref_q;
    }

    corr_real /= num_samples;
    corr_imag /= num_samples;

    double correlation = sqrt(corr_real*corr_real + corr_imag*corr_imag);
    double correlation_db = 20.0 * log10(correlation);

    printf("  Expected freq: %.1f kHz\n", TONE_OFFSET / 1e3);
    printf("  Correlation:   %.2f dB\n", correlation_db);

    if (correlation_db > -40) {
        printf("  ✓ Tone detected!\n");
    } else {
        printf("  ⚠ Weak or no tone detected\n");
    }

    // Print first few samples
    printf("\n📝 First 10 I/Q pairs:\n");
    for (int i = 0; i < 10 && i < (int)num_samples; i++) {
        printf("  [%d] I=%6d, Q=%6d\n", i, buffer[2*i], buffer[2*i+1]);
    }
}

// ===================================================================
// MAIN FUNCTION
// ===================================================================

int main(int argc, char **argv)
{
    int ret = 0;
    struct timespec start_time, end_time;
    double elapsed_time;

    // Setup signal handler
    signal(SIGINT, signal_handler);
    signal(SIGTERM, signal_handler);

    print_separator("LAB 0 - METHOD 3: HELLO PLUTOSDR (HOSTED APPLICATION)");

    printf("\nThis application runs directly on PlutoSDR's ARM CPU\n");
    printf("It demonstrates low-latency local IIO access\n");

    clock_gettime(CLOCK_MONOTONIC, &start_time);

    // Initialize IIO
    if (setup_iio_context() < 0) {
        ret = -1;
        goto cleanup;
    }

    if (setup_devices() < 0) {
        ret = -1;
        goto cleanup;
    }

    if (configure_ad9361() < 0) {
        ret = -1;
        goto cleanup;
    }

    if (setup_buffers() < 0) {
        ret = -1;
        goto cleanup;
    }

    // Generate TX signal
    print_step("📡 STEP 5:", "Transmitting tone...");

    int16_t *tx_data = (int16_t *)iio_buffer_start(txbuf);
    generate_tone(tx_data, BUFFER_SIZE, TONE_OFFSET, SAMPLE_RATE);

    // Push TX buffer
    ssize_t nbytes_tx = iio_buffer_push(txbuf);
    if (nbytes_tx < 0) {
        fprintf(stderr, "✗ Failed to push TX buffer: %s\n", strerror(-nbytes_tx));
        ret = -1;
        goto cleanup;
    }

    printf("✓ TX buffer pushed: %zd bytes\n", nbytes_tx);
    printf("  Continuous transmission started (cyclic buffer)\n");

    // Wait for TX to stabilize
    printf("\n⏳ Waiting 500ms for TX to stabilize...\n");
    usleep(500000);

    // Receive
    ssize_t nbytes_rx = iio_buffer_refill(rxbuf);
    if (nbytes_rx < 0) {
        fprintf(stderr, "✗ Failed to refill RX buffer: %s\n", strerror(-nbytes_rx));
        ret = -1;
        goto cleanup;
    }

    printf("✓ RX buffer filled: %zd bytes\n", nbytes_rx);

    // Analyze received signal
    int16_t *rx_data = (int16_t *)iio_buffer_start(rxbuf);
    analyze_received_signal(rx_data, BUFFER_SIZE, SAMPLE_RATE);

    // Measure execution time
    clock_gettime(CLOCK_MONOTONIC, &end_time);
    elapsed_time = (end_time.tv_sec - start_time.tv_sec) +
                   (end_time.tv_nsec - start_time.tv_nsec) / 1e9;

    // Summary
    print_separator("✓ METHOD 3 COMPLETE: Hosted Application");

    printf("\n📈 Performance Summary:\n");
    printf("  Total execution time:  %.3f seconds\n", elapsed_time);
    printf("  TX/RX latency:         < 1 ms (local IIO)\n");
    printf("  Memory usage:          ~%zu KB\n", (BUFFER_SIZE * 4) / 1024);
    printf("  CPU:                   ARM Cortex-A9 @ 666 MHz\n");

    printf("\n💡 Key Advantages:\n");
    printf("  ✓ No network latency\n");
    printf("  ✓ Standalone operation\n");
    printf("  ✓ Direct hardware access\n");
    printf("  ✓ Low power consumption\n");
    printf("  ✓ Production ready\n");

    printf("\n⚠️  Limitations:\n");
    printf("  • Limited CPU power (666 MHz ARM)\n");
    printf("  • Limited RAM (512 MB)\n");
    printf("  • No GUI (console only)\n");
    printf("  • Harder to debug than PC\n");

cleanup:
    // Cleanup
    if (txbuf) iio_buffer_destroy(txbuf);
    if (rxbuf) iio_buffer_destroy(rxbuf);
    if (ctx) iio_context_destroy(ctx);

    printf("\n🧹 Cleanup complete\n");
    print_separator("END");

    return ret;
}
```

---

## Part 3: Compilation

### Step 3.1: Compilation Script

**File:** `compile_for_pluto.sh`

```bash
#!/bin/bash
################################################################################
# PlutoSDR Cross-Compilation Script
#
# This script compiles C applications for PlutoSDR's ARM processor
#
# Usage:
#   ./compile_for_pluto.sh <source_file.c> [output_name]
#
# Example:
#   ./compile_for_pluto.sh lab0_method3_hosted.c lab0_hosted
################################################################################

set -e  # Exit on error

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Configuration
CROSS_COMPILE="arm-linux-gnueabihf-"
CC="${CROSS_COMPILE}gcc"
STRIP="${CROSS_COMPILE}strip"
READELF="${CROSS_COMPILE}readelf"

# Paths
LIBIIO_PREFIX="/opt/arm-libs"
INCLUDE_PATH="${LIBIIO_PREFIX}/include"
LIB_PATH="${LIBIIO_PREFIX}/lib"

echo -e "${BLUE}======================================================================${NC}"
echo -e "${BLUE}PlutoSDR Cross-Compilation Script${NC}"
echo -e "${BLUE}======================================================================${NC}"

# Check arguments
if [ $# -lt 1 ]; then
    echo -e "${RED}✗ Error: No source file specified${NC}"
    echo ""
    echo "Usage: $0 <source_file.c> [output_name]"
    echo ""
    echo "Example:"
    echo "  $0 lab0_method3_hosted.c lab0_hosted"
    exit 1
fi

SOURCE_FILE="$1"
if [ $# -ge 2 ]; then
    OUTPUT_NAME="$2"
else
    # Default output name (remove .c extension)
    OUTPUT_NAME="${SOURCE_FILE%.c}"
fi

# Verify source file exists
if [ ! -f "$SOURCE_FILE" ]; then
    echo -e "${RED}✗ Error: Source file not found: $SOURCE_FILE${NC}"
    exit 1
fi

echo -e "\n${YELLOW}Configuration:${NC}"
echo "  Source file:    $SOURCE_FILE"
echo "  Output binary:  $OUTPUT_NAME"
echo "  Cross-compiler: $CC"

# Check cross-compiler
echo -e "\n${YELLOW}Step 1: Verifying cross-compiler...${NC}"
if ! command -v $CC &> /dev/null; then
    echo -e "${RED}✗ Error: Cross-compiler not found: $CC${NC}"
    echo ""
    echo "Install with:"
    echo "  sudo apt-get install gcc-arm-linux-gnueabihf"
    exit 1
fi

echo -e "${GREEN}✓ Cross-compiler found${NC}"
$CC --version | head -1

# Check libiio
echo -e "\n${YELLOW}Step 2: Verifying libiio...${NC}"
if [ ! -f "${INCLUDE_PATH}/iio.h" ]; then
    echo -e "${RED}✗ Error: libiio headers not found at ${INCLUDE_PATH}${NC}"
    echo ""
    echo "Build libiio with:"
    echo "  ./build_libiio_arm.sh"
    exit 1
fi

if [ ! -f "${LIB_PATH}/libiio.so" ]; then
    echo -e "${RED}✗ Error: libiio library not found at ${LIB_PATH}${NC}"
    exit 1
fi

echo -e "${GREEN}✓ libiio found${NC}"
echo "  Headers: ${INCLUDE_PATH}/iio.h"
echo "  Library: ${LIB_PATH}/libiio.so"

# Compile
echo -e "\n${YELLOW}Step 3: Compiling...${NC}"

CFLAGS="-Wall -Wextra -O2 -I${INCLUDE_PATH}"
LDFLAGS="-L${LIB_PATH} -liio -lm -lpthread"

echo "  CFLAGS:  $CFLAGS"
echo "  LDFLAGS: $LDFLAGS"

$CC $CFLAGS -o $OUTPUT_NAME $SOURCE_FILE $LDFLAGS

if [ $? -ne 0 ]; then
    echo -e "${RED}✗ Compilation failed${NC}"
    exit 1
fi

echo -e "${GREEN}✓ Compilation successful${NC}"

# Verify binary
echo -e "\n${YELLOW}Step 4: Verifying binary...${NC}"

# Check file type
FILE_OUTPUT=$(file $OUTPUT_NAME)
echo "  $FILE_OUTPUT"

if [[ $FILE_OUTPUT != *"ARM"* ]]; then
    echo -e "${YELLOW}⚠ Warning: Binary doesn't appear to be ARM${NC}"
fi

# Check architecture
ARCH=$($READELF -h $OUTPUT_NAME | grep Machine | awk '{print $2, $3, $4}')
echo "  Architecture: $ARCH"

# Show file size
FILE_SIZE=$(du -h $OUTPUT_NAME | cut -f1)
echo "  Size: $FILE_SIZE"

# Strip symbols to reduce size
echo -e "\n${YELLOW}Step 5: Stripping symbols...${NC}"
SIZE_BEFORE=$(stat -c%s $OUTPUT_NAME)
$STRIP $OUTPUT_NAME
SIZE_AFTER=$(stat -c%s $OUTPUT_NAME)
SIZE_SAVED=$((SIZE_BEFORE - SIZE_AFTER))

echo -e "${GREEN}✓ Stripped${NC}"
echo "  Before: $(numfmt --to=iec-i --suffix=B $SIZE_BEFORE)"
echo "  After:  $(numfmt --to=iec-i --suffix=B $SIZE_AFTER)"
echo "  Saved:  $(numfmt --to=iec-i --suffix=B $SIZE_SAVED) ($(( SIZE_SAVED * 100 / SIZE_BEFORE ))%)"

# Summary
echo -e "\n${BLUE}======================================================================${NC}"
echo -e "${GREEN}✓ Build Complete${NC}"
echo -e "${BLUE}======================================================================${NC}"

echo -e "\n${YELLOW}📦 Output:${NC}"
echo "  Binary: $OUTPUT_NAME"
echo "  Size:   $(du -h $OUTPUT_NAME | cut -f1)"

echo -e "\n${YELLOW}📤 Deployment:${NC}"
echo "  Deploy to PlutoSDR:"
echo "    ${GREEN}scp $OUTPUT_NAME root@192.168.2.1:/root/${NC}"
echo ""
echo "  Run on PlutoSDR:"
echo "    ${GREEN}ssh root@192.168.2.1${NC}"
echo "    ${GREEN}./root/$OUTPUT_NAME${NC}"

echo -e "\n${YELLOW}🔍 Optional Tests:${NC}"
echo "  List dependencies:"
echo "    ${BLUE}arm-linux-gnueabihf-readelf -d $OUTPUT_NAME${NC}"
echo ""
echo "  Check symbols:"
echo "    ${BLUE}arm-linux-gnueabihf-nm $OUTPUT_NAME${NC}"

echo ""
```

**Make executable:**

```bash
chmod +x compile_for_pluto.sh
```

### Step 3.2: Compile the Application

```bash
./compile_for_pluto.sh lab0_method3_hosted.c lab0_hosted
```

**Expected Output:**

```
======================================================================
PlutoSDR Cross-Compilation Script
======================================================================

Configuration:
  Source file:    lab0_method3_hosted.c
  Output binary:  lab0_hosted
  Cross-compiler: arm-linux-gnueabihf-gcc

Step 1: Verifying cross-compiler...
✓ Cross-compiler found
arm-linux-gnueabihf-gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0

Step 2: Verifying libiio...
✓ libiio found
  Headers: /opt/arm-libs/include/iio.h
  Library: /opt/arm-libs/lib/libiio.so

Step 3: Compiling...
  CFLAGS:  -Wall -Wextra -O2 -I/opt/arm-libs/include
  LDFLAGS: -L/opt/arm-libs/lib -liio -lm -lpthread
✓ Compilation successful

Step 4: Verifying binary...
  lab0_hosted: ELF 32-bit LSB executable, ARM, EABI5 version 1
  Architecture: ARM
  Size: 24K

Step 5: Stripping symbols...
✓ Stripped
  Before: 23.7K
  After:  14.2K
  Saved:  9.5K (40%)

======================================================================
✓ Build Complete
======================================================================

📦 Output:
  Binary: lab0_hosted
  Size:   14K

📤 Deployment:
  Deploy to PlutoSDR:
    scp lab0_hosted root@192.168.2.1:/root/

  Run on PlutoSDR:
    ssh root@192.168.2.1
    ./root/lab0_hosted

🔍 Optional Tests:
  List dependencies:
    arm-linux-gnueabihf-readelf -d lab0_hosted

  Check symbols:
    arm-linux-gnueabihf-nm lab0_hosted
```

---

## Part 4: Deployment to PlutoSDR

### Step 4.1: Copy libiio Library

Since we cross-compiled libiio, we need to copy it to PlutoSDR:

```bash
#!/bin/bash
# deploy_libiio.sh

echo "Deploying libiio to PlutoSDR"
echo "============================="

PLUTO_IP="192.168.2.1"
LIB_PATH="/opt/arm-libs/lib"

# Copy libiio library
echo "Copying libiio.so..."
scp ${LIB_PATH}/libiio.so* root@${PLUTO_IP}:/usr/lib/

if [ $? -eq 0 ]; then
    echo "✓ libiio deployed successfully"
else
    echo "✗ Deployment failed"
    exit 1
fi

# Set library path on PlutoSDR
echo ""
echo "Updating library cache on PlutoSDR..."
ssh root@${PLUTO_IP} 'ldconfig'

echo "✓ Complete"
```

Run once:
```bash
chmod +x deploy_libiio.sh
./deploy_libiio.sh
```

### Step 4.2: Deploy Application

**Automated deployment script:**

**File:** `deploy_to_pluto.sh`

```bash
#!/bin/bash
################################################################################
# Deploy Application to PlutoSDR
################################################################################

PLUTO_IP="192.168.2.1"
PLUTO_USER="root"
PLUTO_PASS="analog"  # Default password

# Check if binary specified
if [ $# -lt 1 ]; then
    echo "Usage: $0 <binary_name>"
    exit 1
fi

BINARY="$1"

if [ ! -f "$BINARY" ]; then
    echo "✗ Error: Binary not found: $BINARY"
    exit 1
fi

echo "Deploying to PlutoSDR"
echo "====================="
echo "  Binary: $BINARY"
echo "  Target: ${PLUTO_USER}@${PLUTO_IP}"
echo ""

# Copy binary
echo "📤 Copying binary..."
scp $BINARY ${PLUTO_USER}@${PLUTO_IP}:/root/

if [ $? -ne 0 ]; then
    echo "✗ SCP failed"
    echo ""
    echo "Troubleshooting:"
    echo "  1. Check PlutoSDR is connected: ping ${PLUTO_IP}"
    echo "  2. Verify SSH access: ssh ${PLUTO_USER}@${PLUTO_IP}"
    echo "  3. Default password: ${PLUTO_PASS}"
    exit 1
fi

# Make executable
echo "🔧 Setting executable permission..."
ssh ${PLUTO_USER}@${PLUTO_IP} "chmod +x /root/$BINARY"

# Verify
echo "✓ Deployment complete!"
echo ""
echo "To run:"
echo "  ssh ${PLUTO_USER}@${PLUTO_IP}"
echo "  /root/$BINARY"
echo ""
echo "Or run remotely:"
echo "  ssh ${PLUTO_USER}@${PLUTO_IP} '/root/$BINARY'"
```

**Deploy:**

```bash
chmod +x deploy_to_pluto.sh
./deploy_to_pluto.sh lab0_hosted
```

---

## Part 5: Running on PlutoSDR

### Step 5.1: SSH into PlutoSDR

```bash
ssh root@192.168.2.1
# Password: analog (default)
```

### Step 5.2: Run the Application

```bash
cd /root
./lab0_hosted
```

### Expected Output on PlutoSDR

```
======================================================================
LAB 0 - METHOD 3: HELLO PLUTOSDR (HOSTED APPLICATION)
======================================================================

This application runs directly on PlutoSDR's ARM CPU
It demonstrates low-latency local IIO access

🔌 STEP 1: Creating local IIO context...
----------------------------------------------------------------------
✓ Local IIO context created
  Devices found: 3
  [0] ad9361-phy (15 channels)
  [1] cf-ad9361-dds-core-lpc (4 channels)
  [2] cf-ad9361-lpc (4 channels)

📡 STEP 2: Setting up IIO devices...
----------------------------------------------------------------------
✓ Found ad9361-phy
✓ Found TX device: cf-ad9361-dds-core-lpc
✓ Found RX device: cf-ad9361-lpc

⚙️  STEP 3: Configuring AD9361 transceiver...
----------------------------------------------------------------------
✓ TX LO:           915 MHz
✓ RX LO:           915 MHz
✓ Sample rate:     2.084 MSPS
✓ TX gain:         -10 dB
✓ RX gain:         60 dB

📦 STEP 4: Creating TX/RX buffers...
----------------------------------------------------------------------
✓ TX I/Q channels enabled
✓ RX I/Q channels enabled
✓ TX buffer created: 16384 samples (cyclic)
✓ RX buffer created: 16384 samples

📡 STEP 5: Transmitting tone...
----------------------------------------------------------------------

📻 Generating 16384-sample tone at 100.0 kHz offset...
✓ Tone generated
  First sample: I=1638, Q=0
  Last sample:  I=1580, Q=349
✓ TX buffer pushed: 65536 bytes
  Continuous transmission started (cyclic buffer)

⏳ Waiting 500ms for TX to stabilize...
✓ RX buffer filled: 65536 bytes

🔍 STEP 6: Analyzing received signal...
----------------------------------------------------------------------

📊 Signal Statistics:
  Samples:       16384
  I range:       [-967, 982]
  Q range:       [-958, 971]
  I mean:        -2.13
  Q mean:        1.87
  Average power: 65.82 dB

🎯 Tone Detection:
  Expected freq: 100.0 kHz
  Correlation:   -25.34 dB
  ✓ Tone detected!

📝 First 10 I/Q pairs:
  [0] I=   245, Q=   -12
  [1] I=   487, Q=   231
  [2] I=   698, Q=   463
  [3] I=   871, Q=   675
  [4] I=  1001, Q=   859
  [5] I=  1083, Q=  1009
  [6] I=  1117, Q=  1120
  [7] I=  1101, Q=  1187
  [8] I=  1037, Q=  1206
  [9] I=   928, Q=  1176

======================================================================
✓ METHOD 3 COMPLETE: Hosted Application
======================================================================

📈 Performance Summary:
  Total execution time:  0.587 seconds
  TX/RX latency:         < 1 ms (local IIO)
  Memory usage:          ~64 KB
  CPU:                   ARM Cortex-A9 @ 666 MHz

💡 Key Advantages:
  ✓ No network latency
  ✓ Standalone operation
  ✓ Direct hardware access
  ✓ Low power consumption
  ✓ Production ready

⚠️  Limitations:
  • Limited CPU power (666 MHz ARM)
  • Limited RAM (512 MB)
  • No GUI (console only)
  • Harder to debug than PC

🧹 Cleanup complete

======================================================================
END
======================================================================
```

---

## Part 6: Comparison and Summary

### Performance Comparison

| Metric | Method 1<br/>(Simulation) | Method 2<br/>(External) | Method 3<br/>(Hosted) |
|--------|--------------------------|------------------------|---------------------|
| **Hardware Required** | None | PlutoSDR + PC | PlutoSDR only |
| **Latency** | N/A | 10-50 ms | < 1 ms |
| **CPU Power** | Full PC | Full PC | ARM 666 MHz |
| **Network Needed** | No | Yes (USB/Ethernet) | No |
| **Development Speed** | Fast | Fast | Medium |
| **Debugging** | Easy | Easy | Harder |
| **Deployment** | N/A | Easy (Python) | Medium (compile) |
| **Production Ready** | No | No | Yes |
| **Power Consumption** | High (PC) | High (PC+PlutoSDR) | Low (PlutoSDR) |
| **GUI Possible** | Yes | Yes | No |
| **Standalone** | N/A | No | Yes |

### When to Use Each Method

**Use Method 1 (Simulation):**
- ✅ Learning SDR concepts
- ✅ Algorithm development
- ✅ No hardware available
- ✅ Need perfect signals
- ✅ Fast prototyping

**Use Method 2 (External App):**
- ✅ Real RF testing
- ✅ Need visualization
- ✅ Development phase
- ✅ Debugging complex systems
- ✅ PC resources needed

**Use Method 3 (Hosted App):**
- ✅ Production deployment
- ✅ Standalone operation needed
- ✅ Low latency critical
- ✅ Battery powered
- ✅ No PC in field

---

## Complete Workflow Summary

```
┌─────────────────────────────────────────────────────────┐
│  LAB 0: Complete Three-Method Workflow                  │
└─────────────────────────────────────────────────────────┘

Week 1: Method 1 - Simulation
├── Learn theory (I/Q, FFT, modulation)
├── Develop algorithms
├── Perfect understanding
└── No hardware needed

Week 2: Method 2 - External App
├── Real RF testing
├── Verify algorithms with hardware
├── Develop on PC
└── Easy debugging

Week 3: Method 3 - Hosted App
├── Cross-compile for ARM
├── Deploy to PlutoSDR
├── Test standalone
└── Production ready

Final: Choose deployment method based on requirements
```

---

**Next Lab:** LAB 1 will use this same three-method approach for more advanced concepts!

**All scripts provided:**
- ✅ install_cross_compiler.sh
- ✅ build_libiio_arm.sh
- ✅ verify_environment.sh
- ✅ compile_for_pluto.sh
- ✅ deploy_libiio.sh
- ✅ deploy_to_pluto.sh

**Complete source code:**
- ✅ lab0_method1_simulation.py (Method 1)
- ✅ lab0_method2_external.py (Method 2)
- ✅ lab0_method3_hosted.c (Method 3)

🎉 **You now understand all three development methods for PlutoSDR!**
