# PlutoSDR Firmware Build Scripts Guide

**Version**: 1.0
**Last Updated**: December 3, 2025
**Author**: PlutoSDR Documentation Team

---

## Overview

This guide provides a comprehensive explanation of all scripts in the `scripts/` directory of the PlutoSDR firmware repository. These scripts are essential components of the firmware build system, handling everything from FPGA programming to firmware image creation.

The PlutoSDR firmware build process involves multiple components:
- **FPGA bitstream** (Vivado)
- **First Stage Boot Loader (FSBL)** (Xilinx SDK)
- **U-Boot bootloader** (cross-compiled for ARM)
- **Linux kernel** (Zynq-specific configuration)
- **Root filesystem** (Buildroot)
- **Device tree blobs** (hardware description)

All of these components are orchestrated by the build scripts described in this guide.

---

## Script Directory Structure

```
scripts/
├── 53-adi-plutosdr-usb.rules    # USB udev rules for host PC
├── create_fsbl_project.tcl      # FSBL generation (Xilinx TCL)
├── get_default_envs.sh          # U-Boot environment extraction
├── legal_info_html.sh           # License compliance HTML generator
├── pluto.its                    # FIT image tree source (PlutoSDR)
├── pluto.mk                     # PlutoSDR build configuration
├── run-xsdb.tcl                 # JTAG bootstrap script
├── run.tcl                      # Vivado XSDB script
├── sidekiqz2.its                # FIT image tree source (Sidekiq Z2)
├── sidekiqz2.mk                 # Sidekiq Z2 build configuration
└── target_mtd_info.key          # Flash memory partition layout
```

---

## 1. create_fsbl_project.tcl

**Purpose**: Generates the First Stage Boot Loader (FSBL) for the Zynq-7000 SoC.

**Language**: TCL (Tool Command Language) for Xilinx HSI (Hardware Software Interface)

**When Used**: During the main build process after the FPGA hardware design (HDF/XSA file) is generated.

### How It Works

```tcl
hsi open_hw_design build/system_top.xsa
set cpu_name [lindex [hsi get_cells -filter {IP_TYPE==PROCESSOR}] 0]

setws ./build/sdk
app create -name fsbl -hw build/system_top.xsa -proc $cpu_name -os standalone -lang C -template {Zynq FSBL}
app config -name fsbl -set build-config release
app build -name fsbl
```

**Step-by-Step**:
1. **Opens** the hardware design file (`system_top.xsa`) containing FPGA configuration
2. **Identifies** the ARM processor core (PS7 Cortex-A9)
3. **Creates** an SDK workspace in `build/sdk/`
4. **Generates** FSBL application from Xilinx's template
5. **Configures** for release build (optimized)
6. **Compiles** FSBL to `build/sdk/fsbl/Release/fsbl.elf`

**Why Important**: The FSBL is the first code that runs when PlutoSDR boots. It initializes:
- DDR3 memory controller
- Clock generators (PS and PL)
- I/O peripherals
- Loads U-Boot bootloader into RAM

**Requirements**:
- Xilinx Vitis/SDK installed
- `system_top.xsa` hardware definition file
- ARM cross-compiler toolchain

---

## 2. get_default_envs.sh

**Purpose**: Extracts default U-Boot environment variables from the compiled binary.

**Language**: Bash shell script

**When Used**: After U-Boot is compiled but before creating the final firmware image.

### How It Works

```bash
ENV_OBJ_FILE="env_common.o"
env_obj_file_path=$(find ${path%/scripts*} -not -path "*/spl/*" -name "${ENV_OBJ_FILE}")

cp ${env_obj_file_path} ${ENV_OBJ_FILE_COPY}
${CROSS_COMPILE}objcopy -O binary -j ".rodata.default_environment" ${ENV_OBJ_FILE_COPY}
tr '\0' '\n' < ${ENV_OBJ_FILE_COPY} | sort -u
```

**Step-by-Step**:
1. **Locates** the `env_common.o` object file from U-Boot build
2. **Extracts** the `.rodata.default_environment` section using `objcopy`
3. **Converts** null-terminated strings to newline-separated text
4. **Sorts** and outputs the environment variables

**Example Output**:
```
bootcmd=sf probe; sf read ${fit_load_address} 0x100000 ${fit_size}; bootm ${fit_load_address}
bootdelay=3
ethaddr=00:05:f7:90:00:00
ipaddr=192.168.2.1
serverip=192.168.2.10
...
```

**Why Important**:
- Documents default U-Boot configuration
- Used to generate `uboot-env.txt` in the build output
- Critical for boot process and network configuration

**Requirements**:
- U-Boot compiled for PlutoSDR
- ARM cross-compiler toolchain (for objcopy)
- `CROSS_COMPILE` environment variable set

---

## 3. pluto.its (Image Tree Source)

**Purpose**: Defines the structure of the Flattened Image Tree (FIT) that packages all boot components.

**Format**: Device Tree Source (DTS) format

**When Used**: During final firmware image creation (`pluto.itb` generation).

### Structure Overview

The FIT image combines multiple components:

```
pluto.itb (Flattened Image Tree)
├── FPGA Bitstream (system_top.bit)
├── Linux Kernel (zImage)
├── Root Filesystem (rootfs.cpio.gz)
└── Device Trees
    ├── zynq-pluto-sdr.dtb (Rev A)
    ├── zynq-pluto-sdr-revb.dtb (Rev B)
    └── zynq-pluto-sdr-revc.dtb (Rev C)
```

### Key Sections

#### Images Section

```dts
images {
    fpga@1 {
        description = "FPGA";
        data = /incbin/("../build/system_top.bit");
        type = "fpga";
        arch = "arm";
        load = <0xF000000>;
        hash@1 {
            algo = "md5";
        };
    };

    linux_kernel@1 {
        description = "Linux";
        data = /incbin/("../build/zImage");
        type = "kernel";
        load = <0x8000>;
        entry = <0x8000>;
    };

    ramdisk@1 {
        description = "Ramdisk";
        data = /incbin/("../build/rootfs.cpio.gz");
        type = "ramdisk";
        compression = "gzip";
    };
}
```

**Key Points**:
- `load` address: Where component is loaded in RAM
- `entry` address: Where execution starts (for kernel)
- `hash`: MD5 checksums for integrity verification
- `/incbin/`: Includes binary file into the image

#### Configurations Section

```dts
configurations {
    default = "config@0";

    config@0 {
        description = "Linux with fpga RevA";
        fdt = "fdt@1";           # Rev A device tree
        kernel = "linux_kernel@1";
        ramdisk = "ramdisk@1";
        fpga = "fpga@1";
    };

    config@8 {
        description = "Linux with fpga RevC";
        fdt = "fdt@3";           # Rev C device tree
        kernel = "linux_kernel@1";
        ramdisk = "ramdisk@1";
        fpga = "fpga@1";
    };
}
```

**Why Multiple Configurations**:
- Support different PlutoSDR hardware revisions (Rev A, B, C)
- U-Boot selects configuration based on EEPROM data
- Allows single firmware image for all hardware versions

### Memory Layout

```
0x00008000: Kernel entry point
0x0F000000: FPGA bitstream load address
[RAM]: Ramdisk (decompressed at runtime)
```

**Why Important**:
- Single unified firmware image
- Hardware revision detection
- Integrity checking (MD5 hashes)
- Ordered boot sequence (FPGA → Kernel → Ramdisk)

---

## 4. pluto.mk (Build Configuration)

**Purpose**: Defines PlutoSDR-specific build parameters and constants.

**Language**: Makefile syntax

**When Used**: Sourced by the main Makefile during the build process.

### Contents

```make
HDF_URL:=http://github.com/analogdevicesinc/plutosdr-fw/releases/download/${LATEST_TAG}/system_top.hdf
TARGET_DTS_FILES:= zynq-pluto-sdr.dtb zynq-pluto-sdr-revb.dtb zynq-pluto-sdr-revc.dtb
COMPLETE_NAME:=PlutoSDR
ZIP_ARCHIVE_PREFIX:=plutosdr
DEVICE_VID:=0x0456
DEVICE_PID:=0xb673
```

### Parameters Explained

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `HDF_URL` | GitHub release URL | Fallback to download prebuilt FPGA design |
| `TARGET_DTS_FILES` | 3 DTB files | Device trees for Rev A, B, C hardware |
| `COMPLETE_NAME` | "PlutoSDR" | Human-readable device name |
| `ZIP_ARCHIVE_PREFIX` | "plutosdr" | Prefix for release zip files |
| `DEVICE_VID` | 0x0456 | USB Vendor ID (Analog Devices) |
| `DEVICE_PID` | 0xb673 | USB Product ID (PlutoSDR normal mode) |

### USB Product IDs

PlutoSDR uses different PIDs for different modes:
- `0xb673`: Normal SDR operation
- `0xb674`: DFU (Device Firmware Update) mode

**Why Important**:
- Centralizes target-specific configuration
- Allows easy porting to other Zynq platforms (see `sidekiqz2.mk`)
- Defines USB identification for host drivers

---

## 5. 53-adi-plutosdr-usb.rules

**Purpose**: Linux udev rules for automatic PlutoSDR device permissions.

**Location on Host PC**: `/etc/udev/rules.d/`

**When Used**: Installed on development/user machines to access PlutoSDR without root.

### Contents Explained

```bash
# DFU Device (firmware update mode)
SUBSYSTEM=="usb", ATTRS{idVendor}=="0456", ATTRS{idProduct}=="b674", MODE="0664", GROUP="plugdev"

# SDR Device (normal operation mode)
SUBSYSTEM=="usb", ATTRS{idVendor}=="0456", ATTRS{idProduct}=="b673", MODE="0664", GROUP="plugdev"

# Prevent ModemManager from treating PlutoSDR as a modem
SUBSYSTEM=="usb", ATTRS{idVendor}=="0456", ATTRS{idProduct}=="b673", ENV{ID_MM_DEVICE_IGNORE}="1"
```

### Rules Breakdown

| VID:PID | Device Mode | Permissions | Notes |
|---------|-------------|-------------|-------|
| 0456:b674 | DFU (bootloader) | 0664 (rw-rw-r--) | For firmware updates |
| 0456:b673 | Normal SDR | 0664 (rw-rw-r--) | For libiio/SDR apps |
| 2fa2:5a32 | Sidekiq Z2 DFU | 0664 | Alternative device |
| 2fa2:5a02 | Sidekiq Z2 SDR | 0664 | Alternative device |

### Installation

```bash
# Install udev rules (on host PC, not PlutoSDR!)
sudo cp scripts/53-adi-plutosdr-usb.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger

# Add yourself to plugdev group
sudo usermod -a -G plugdev $USER
# Log out and log back in for group changes to take effect
```

### ModemManager Issue

Without `ID_MM_DEVICE_IGNORE`:
- NetworkManager's ModemManager probes USB serial devices
- Sends AT commands to PlutoSDR's serial port
- Interferes with libiio communication

**Why Important**:
- Enables non-root access to PlutoSDR
- Prevents interference from system services
- Required for GNU Radio, Python scripts, and libiio tools

---

## 6. legal_info_html.sh

**Purpose**: Generates comprehensive HTML license report for all open-source components.

**Language**: Bash shell script

**When Used**: During final build phase to create `build/LICENSE.html`.

### How It Works

```bash
MANIFEST=buildroot/output/legal-info/manifest.csv

# Parse manifest.csv
PACKAGE=1      # Column 1: Package name
VERSION=2      # Column 2: Version
LICENSE=3      # Column 3: License type
LICENSE_FILES=4 # Column 4: License file paths
SOURCE_ARCHIVE=5 # Column 5: Source tarball
SOURCE_SITE=6   # Column 6: Download URL
```

**Step-by-Step**:
1. **Reads** Buildroot's legal-info manifest
2. **Sorts** packages alphabetically
3. **Generates** HTML with:
   - Package names and versions
   - License types (GPL, LGPL, MIT, BSD, etc.)
   - Links to license texts
   - Source code locations
4. **Outputs** to `build/LICENSE.html`

### Example Output

```html
<h2>Package: busybox - Version: 1.35.0</h2>
<p>License: GPL-2.0</p>
<p>License files: COPYING</p>
<p>Source: busybox-1.35.0.tar.bz2</p>
<p>Source site: https://busybox.net/downloads/</p>
```

**Why Important**:
- **GPL Compliance**: Required for distributing GPL software
- **Legal Documentation**: Tracks all third-party licenses
- **Transparency**: Users can see all software components
- **Source Availability**: Links to source code for GPL components

**Included in**: `build/legal-info-vX.XX.tar.gz` (~475 MB!)

---

## 7. run-xsdb.tcl (JTAG Bootstrap)

**Purpose**: Programs PlutoSDR via JTAG for initial flash programming or recovery.

**Language**: TCL for Xilinx System Debugger (XSDB)

**When Used**:
- **First-time programming** of blank PlutoSDR
- **Recovery** from corrupted flash
- **Development** without writing to flash

### How It Works

```tcl
connect              # Connect to JTAG adapter
target 2            # Select ARM core (target 1 is FPGA)

rst                 # Reset the system

source ps7_init.tcl # Initialize processing system
ps7_init            # Configure clocks, DDR, peripherals
ps7_post_config     # Final initialization

dow u-boot.elf      # Download u-boot to RAM
con                 # Continue execution
disconnect
```

### Execution Flow

```
JTAG Adapter
    ↓
[1] FPGA (optional: fpga -f system_top.bit)
    ↓
[2] ARM Cortex-A9 (target 2)
    ↓
[3] Initialize PS (ps7_init.tcl)
    ↓
[4] Load U-Boot to RAM
    ↓
[5] Execute U-Boot
```

### Usage Example

```bash
# Navigate to build directory
cd plutosdr-fw/build/

# Launch XSDB console
xsdb

# Run bootstrap script
xsdb% source ../scripts/run-xsdb.tcl
```

**Interactive U-Boot via JTAG**:
```
U-Boot 2019.01-00099-g90f0e2a9d7 (Dec 03 2025 - 10:30:00 +0000)

zynq-uboot> # You now have U-Boot shell!
zynq-uboot> sf probe           # Test SPI flash
zynq-uboot> mmc info            # Check SD card
```

### Use Cases

**1. Initial Flash Programming**:
```
xsdb> source run-xsdb.tcl
U-Boot> run dfu_ram
[On host PC]
$ dfu-util -D pluto.dfu -a firmware.dfu
```

**2. Development Testing**:
- Test new U-Boot builds without flashing
- Iterate quickly during bootloader development
- Safe testing (RAM only, no flash writes)

**3. Flash Recovery**:
- Recover from bad U-Boot flash
- Reprogram corrupted firmware
- Factory reset

**Why Important**:
- Essential for manufacturing
- Critical recovery tool
- Development workflow enabler

**Requirements**:
- JTAG adapter (e.g., Xilinx Platform Cable)
- JTAG connection to PlutoSDR (requires hardware modification)
- Xilinx XSDB tool (part of Vivado/Vitis)

---

## 8. target_mtd_info.key

**Purpose**: Documents the QSPI flash memory partition layout.

**Format**: Text file (reference only, not executed)

**When Used**: Documentation for understanding PlutoSDR flash structure.

### Flash Partition Layout

```
dev:    size   erasesize  name
mtd0: 00100000 00001000 "qspi-fsbl-uboot"    (1 MB)
mtd1: 00020000 00001000 "qspi-uboot-env"     (128 KB)
mtd2: 000e0000 00001000 "qspi-nvmfs"         (896 KB)
mtd3: 01e00000 00001000 "qspi-linux"         (30 MB)
```

### Partition Details

| Partition | Size | Purpose | Contents |
|-----------|------|---------|----------|
| `mtd0` | 1 MB | Bootloaders | FSBL + U-Boot (boot.bin) |
| `mtd1` | 128 KB | Environment | U-Boot environment variables |
| `mtd2` | 896 KB | NVMFS | Non-volatile storage (calibration, config) |
| `mtd3` | 30 MB | Firmware | FIT image (kernel + ramdisk + FPGA) |

### Memory Map Visualization

```
0x00000000 ┌────────────────────┐
           │  mtd0: boot.bin    │ ← FSBL + U-Boot (1 MB)
0x00100000 ├────────────────────┤
           │  mtd1: uboot-env   │ ← Environment (128 KB)
0x00120000 ├────────────────────┤
           │  mtd2: nvmfs       │ ← Calibration data (896 KB)
0x00200000 ├────────────────────┤
           │                    │
           │  mtd3: pluto.itb   │ ← Linux + FPGA + rootfs (30 MB)
           │                    │
0x02000000 └────────────────────┘ Total: 32 MB QSPI flash
```

### Accessing on PlutoSDR

```bash
# SSH into PlutoSDR
ssh root@192.168.2.1

# View partition info
cat /proc/mtd

# Dump partition contents (be careful!)
dd if=/dev/mtd0 of=/tmp/boot.bin bs=1M count=1

# Mount NVMFS partition (calibration data)
mount -t jffs2 /dev/mtdblock2 /mnt
ls /mnt/
```

### Partition Usage

**mtd0 (boot.bin)**:
- Updated by `boot.dfu` or `boot.frm`
- Contains FSBL.elf + U-Boot
- Critical for boot (corrupted = brick)

**mtd1 (uboot-env)**:
- Updated by `uboot-env.dfu`
- Stores network config, boot command
- Can be reset to defaults via U-Boot

**mtd2 (nvmfs - JFFS2 filesystem)**:
- AD9361 calibration data
- User configuration files
- Network settings
- SSH keys

**mtd3 (pluto.itb)**:
- Updated by `pluto.dfu` or `pluto.frm`
- Largest partition
- Complete Linux system

**Why Important**:
- Understanding firmware update targets
- Recovery procedures
- Debugging boot issues
- Custom firmware development

---

## 9. sidekiqz2.its / sidekiqz2.mk

**Purpose**: Support for Sidekiq Z2 SDR platform (alternative to PlutoSDR).

**Note**: Similar to PlutoSDR variants but for a different hardware platform.

### Key Differences

| Parameter | PlutoSDR | Sidekiq Z2 |
|-----------|----------|------------|
| USB VID:PID | 0456:b673 | 2fa2:5a02 |
| Device Tree | zynq-pluto-sdr.dtb | (Sidekiq-specific) |
| FPGA Design | AD9361 + Zynq-7010 | Similar RF + Zynq |

**Why Included**: Demonstrates how to port firmware to similar Zynq-based SDR platforms.

---

## Firmware Build Flow

Here's how all these scripts work together during `make`:

```
┌─────────────────────────────────────────────────────────────┐
│  1. FPGA Build (Vivado)                                     │
│     Generates: system_top.bit, system_top.xsa               │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  2. FSBL Build (create_fsbl_project.tcl)                    │
│     Input: system_top.xsa                                   │
│     Output: fsbl.elf                                        │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  3. U-Boot Build (Cross-compile ARM)                        │
│     Toolchain: arm-linux-gnueabihf-gcc                      │
│     Output: u-boot.elf                                      │
│     Script: get_default_envs.sh → uboot-env.txt             │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  4. Linux Kernel Build (Cross-compile ARM)                  │
│     Config: zynq_pluto_defconfig                            │
│     Output: zImage, zynq-pluto-sdr*.dtb                     │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  5. Buildroot Rootfs Build                                  │
│     Config: pluto.mk (target-specific)                      │
│     Output: rootfs.cpio.gz                                  │
│     Script: legal_info_html.sh → LICENSE.html               │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  6. Create Boot Image (Xilinx bootgen)                      │
│     Input: fsbl.elf + u-boot.elf                            │
│     Output: boot.bin                                        │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  7. Create FIT Image (mkimage)                              │
│     Input: pluto.its + all binaries                         │
│     Output: pluto.itb                                       │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  8. Create DFU/FRM Images                                   │
│     boot.bin → boot.dfu / boot.frm                          │
│     pluto.itb → pluto.dfu / pluto.frm                       │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  9. Package Release ZIP                                     │
│     plutosdr-fw-vX.XX.zip (all images + docs)               │
└─────────────────────────────────────────────────────────────┘
```

---

## Practical Usage Examples

### Example 1: Full Firmware Build

```bash
# Setup environment
export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2021.2/settings64.sh
export CROSS_COMPILE=arm-linux-gnueabihf-

# Clone and build
git clone --recursive https://github.com/analogdevicesinc/plutosdr-fw.git
cd plutosdr-fw
make

# Results in build/:
# - pluto.frm (for USB mass storage update)
# - pluto.dfu (for DFU mode update)
# - boot.frm, boot.dfu (bootloader updates)
```

### Example 2: Update Only Linux Kernel

```bash
# Modify kernel config
make linux-menuconfig

# Rebuild just Linux components
make linux-rebuild

# Create new firmware image
make

# Flash via USB mass storage
cp build/pluto.frm /media/$USER/PlutoSDR/
sync
# PlutoSDR reboots automatically
```

### Example 3: JTAG Recovery

```bash
# PlutoSDR won't boot - flash corrupted
# Use JTAG to recover

cd plutosdr-fw/build/
xsdb

# In XSDB console:
xsdb% source ../scripts/run-xsdb.tcl

# U-Boot starts, now flash firmware:
zynq-uboot> run dfu_ram

# On host PC:
dfu-util -D pluto.dfu -a firmware.dfu -R
```

### Example 4: Custom U-Boot Environment

```bash
# Extract current environment
./scripts/get_default_envs.sh > my_custom_env.txt

# Edit environment
nano my_custom_env.txt
# Change: ipaddr=192.168.3.1

# Create new environment binary
mkenvimage -s 131072 -o uboot-env.bin my_custom_env.txt

# Flash to PlutoSDR
dfu-util -D uboot-env.bin -a uboot-env.dfu
```

---

## Script Dependencies

### Required Tools

| Script | Dependencies |
|--------|--------------|
| `create_fsbl_project.tcl` | Xilinx Vitis/SDK, HSI |
| `get_default_envs.sh` | arm-linux-gnueabihf-objcopy, bash |
| `pluto.its` | mkimage (u-boot-tools) |
| `legal_info_html.sh` | bash, Buildroot manifest |
| `run-xsdb.tcl` | Xilinx XSDB, JTAG hardware |

### Environment Variables

```bash
# Required for ARM cross-compilation
export CROSS_COMPILE=arm-linux-gnueabihf-

# Required for FPGA/FSBL build
export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2021.2/settings64.sh
source $VIVADO_SETTINGS
```

---

## Troubleshooting

### Issue 1: FSBL Creation Fails

**Error**:
```
ERROR: [Hsi 55-2053] app create: Application fsbl already exists
```

**Solution**:
```bash
rm -rf build/sdk/fsbl
make
```

### Issue 2: DFU Update Not Working

**Error**: `dfu-util: Cannot open DFU device 0456:b673`

**Solution**:
```bash
# Install udev rules
sudo cp scripts/53-adi-plutosdr-usb.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger

# Add yourself to plugdev group
sudo usermod -a -G plugdev $USER
# Log out and back in
```

### Issue 3: JTAG Connection Failed

**Error**: `XSDB% connect - Error: Connection failed`

**Solution**:
```bash
# Check JTAG cable
lsusb | grep Xilinx

# Check permissions
sudo chmod 666 /dev/bus/usb/XXX/YYY

# Verify hardware
xsdb% connect
xsdb% targets
# Should see: 1 (target) Zynq
```

### Issue 4: FIT Image Too Large

**Error**: `pluto.itb exceeds partition size (0x1E00000)`

**Solution**:
```bash
# Check sizes
ls -lh build/zImage build/rootfs.cpio.gz

# Reduce rootfs size in Buildroot
make menuconfig
# Target packages → Deselect unused packages

# Rebuild
make clean && make
```

---

## Advanced Customization

### Creating Custom Hardware Variants

To support a custom Zynq board:

1. **Create device tree** (`zynq-mycustom-sdr.dts`)
2. **Create build config** (`scripts/mycustom.mk`):
   ```make
   TARGET_DTS_FILES:= zynq-mycustom-sdr.dtb
   COMPLETE_NAME:=MyCustomSDR
   ZIP_ARCHIVE_PREFIX:=mycustom-sdr
   DEVICE_VID:=0x????
   DEVICE_PID:=0x????
   ```
3. **Create FIT image tree** (`scripts/mycustom.its`)
4. **Update Makefile** to include new target

### Modifying Flash Layout

To change partition sizes, edit:
- U-Boot device tree (`arch/arm/dts/zynq-pluto-sdr.dts`)
- Update `target_mtd_info.key` documentation
- Adjust `pluto.its` load addresses if needed

**Warning**: Changing flash layout requires careful testing to avoid bricking!

---

## Security Considerations

### Verified Boot

PlutoSDR firmware supports optional secure boot:
- FSBL can verify U-Boot signature
- U-Boot can verify FIT image signatures
- Requires RSA keys configured in FSBL

### Flash Encryption

Zynq-7000 supports AES encryption of flash contents:
- Configure in Vivado (eFUSE)
- Requires one-time eFUSE programming
- **Warning**: Irreversible!

### Network Security

Default U-Boot environment allows:
- TFTP boot (development feature)
- DFU over USB

For production:
- Disable unused boot methods
- Change default IP addresses
- Use signed firmware images

---

## Reference Documentation

### Official Xilinx Documentation

- [UG585: Zynq-7000 Technical Reference Manual](https://www.xilinx.com/support/documentation/user_guides/ug585-Zynq-7000-TRM.pdf)
- [UG821: Zynq-7000 Software Developers Guide](https://www.xilinx.com/support/documentation/user_guides/ug821-zynq-7000-swdev.pdf)
- [UG1144: PetaLinux Tools Reference Guide](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2021_2/ug1144-petalinux-tools-reference-guide.pdf)

### PlutoSDR-Specific Documentation

- [PlutoSDR Wiki](https://wiki.analog.com/university/tools/pluto)
- [Building the PlutoSDR Firmware](https://wiki.analog.com/university/tools/pluto/building_the_image)
- [PlutoSDR GitHub Repository](https://github.com/analogdevicesinc/plutosdr-fw)

### U-Boot Documentation

- [U-Boot FIT Image Documentation](https://source.denx.de/u-boot/u-boot/-/blob/master/doc/uImage.FIT/howto.txt)
- [U-Boot Environment Variables](https://docs.u-boot.org/en/latest/usage/environment.html)

---

## Conclusion

The PlutoSDR firmware build scripts provide a complete toolchain for:
- **FPGA programming** (Vivado)
- **Bootloader creation** (FSBL + U-Boot)
- **Linux kernel compilation**
- **Root filesystem generation** (Buildroot)
- **Firmware packaging** (FIT images)
- **License compliance** (GPL/LGPL tracking)
- **Development tools** (JTAG bootstrap)

Understanding these scripts is essential for:
- Custom firmware modifications
- Hardware bring-up
- Recovery procedures
- Production manufacturing

For further assistance:
- Check the [PlutoSDR forum](https://ez.analog.com/adieducation/university-program/)
- Review the [main firmware repository](https://github.com/analogdevicesinc/plutosdr-fw)
- See our comprehensive [lab documentation](../README.md)

---

**Version History**:
- v1.0 (December 3, 2025): Initial comprehensive documentation

**Related Guides**:
- [PlutoSDR Architecture](PLUTOSDR_ARCHITECTURE.md)
- [PlutoSDR Custom Applications](PLUTOSDR_CUSTOM_APPS.md)
- [Three Method Approach](THREE_METHOD_APPROACH.md)
