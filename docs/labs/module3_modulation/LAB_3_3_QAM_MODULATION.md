# LAB 3.3: QAM Modulation (16-QAM, 64-QAM, 256-QAM)

## Overview

This lab explores **QAM (Quadrature Amplitude Modulation)** - a highly spectral-efficient modulation that combines amplitude and phase modulation. QAM is used in modern high-speed communication systems including WiFi, LTE/5G, cable modems, and digital television.

## Learning Objectives

After completing this lab, you will understand:
- QAM fundamentals: combining ASK and PSK
- Square QAM constellations: 16-QAM, 64-QAM, 256-QAM
- Gray coding for rectangular QAM constellations
- Spectral efficiency: up to 8 bits/s/Hz with 256-QAM
- I/Q representation and constellation diagrams
- Average vs. peak power considerations
- BER performance and required SNR
- Adaptive modulation based on channel quality
- PlutoSDR implementation of M-QAM

## Prerequisites

- LAB 0: PlutoSDR Setup
- LAB 1.3: I/Q Modulation
- LAB 2.1-2.3: Sampling and quantization
- LAB 3.1: ASK, FSK, PSK fundamentals
- LAB 3.2: QPSK and 8-PSK
- Understanding of constellation diagrams
- Basic Python or C programming

## Theory

### 1. QAM Fundamentals

**Definition**: QAM modulates both amplitude AND phase.

```
PSK: Only phase changes (constant amplitude)
  s(t) = A·cos(2πfc·t + φ(t))
  A = constant, φ = variable

ASK: Only amplitude changes (constant phase)
  s(t) = A(t)·cos(2πfc·t)
  A = variable, φ = constant

QAM: Both amplitude AND phase change
  s(t) = A(t)·cos(2πfc·t + φ(t))
  Both A and φ variable!
```

**Complex Baseband View**:
```
QAM symbol: s = I + jQ

where I and Q can take multiple levels (not just ±1)

Example 16-QAM:
  I ∈ {-3, -1, +1, +3}  (4 levels)
  Q ∈ {-3, -1, +1, +3}  (4 levels)

  Total: 4×4 = 16 constellation points
  log₂(16) = 4 bits per symbol

This is the "square" or "rectangular" QAM constellation
```

**Why QAM?**
```
Trade-off spectrum efficiency vs. complexity:

Modulation    Bits/symbol    Implementation
---------------------------------------------------
BPSK          1              Simple (phase only)
QPSK          2              Moderate (phase only)
8-PSK         3              Moderate (phase only)
16-QAM        4              Complex (phase + amplitude)
64-QAM        6              Complex (phase + amplitude)
256-QAM       8              Very complex
1024-QAM      10             Extreme (WiFi 6, 5G)

QAM achieves higher data rates than PSK in same bandwidth!
```

### 2. 16-QAM

**Constellation**: 4×4 square grid

```
        Q
        |
    11  |  11
  -3+3j | +1+3j |  11    11
     ●  |  ●    ●  |  ●
  ------+------  ------+------  I
     ●  |  ●    ●  |  ●
  -3-3j | +1-3j |  01    01
    01  |  01
        |

Actually:
        Q (+3)
         |
    1011 | 1001 | 0001 | 0011
      ●  |  ●  |  ●  |  ●   (+3)
    -----+-----+-----+-----
    1010 | 1000 | 0000 | 0010
      ●  |  ●  |  ●  |  ●   (+1)
  ---+-----+-----+-----+-----+--- I
    1110 | 1100 | 0100 | 0110
      ●  |  ●  |  ●  |  ●   (-1)
    -----+-----+-----+-----
    1111 | 1101 | 0101 | 0111
      ●  |  ●  |  ●  |  ●   (-3)
         |
        (-3)

16 points arranged in 4×4 grid
Gray coded: adjacent symbols differ by 1 bit
```

**Symbol Mapping** (Gray coded):
```
I-bits (MSB, MSB-1): Select column
Q-bits (LSB-1, LSB): Select row

Example: bits = 1001
  I-bits: 10 → I = -1 (2nd column from left)
  Q-bits: 01 → Q = +3 (top row)
  Symbol: -1 + j3

Amplitude levels: {-3, -1, +1, +3} (normalized)
Actual levels: ±1, ±3 times base amplitude
```

**Energy Normalization**:
```
To fairly compare modulations, normalize to unit average energy:

Unnormalized 16-QAM: I, Q ∈ {-3, -1, +1, +3}
Average energy: E_avg = (3² + 1² + 1² + 3²)/4 × 2 = 10

Normalized: divide by √10
  I, Q ∈ {-3/√10, -1/√10, +1/√10, +3/√10}
  I, Q ∈ {-0.949, -0.316, +0.316, +0.949}

Now: E_avg = 1.0
```

**Spectral Efficiency**:
```
16-QAM: 4 bits per symbol
  Bit rate: Rb = 4·Rs
  Bandwidth: BW ≈ 2·Rs = Rb/2

  Spectral efficiency: 4 / 2 = 2 bits/s/Hz

Compare to QPSK: 1 bit/s/Hz
16-QAM is 2× more efficient!

Example: 10 MHz bandwidth
  QPSK: 10 Mbps
  16-QAM: 20 Mbps
```

### 3. 64-QAM

**Constellation**: 8×8 square grid

```
64 constellation points in 8×8 arrangement
6 bits per symbol (log₂(64) = 6)

I and Q each have 8 levels:
  I, Q ∈ {-7, -5, -3, -1, +1, +3, +5, +7}

Normalized (E_avg = 1):
  Divide by √42
  I, Q ∈ {±1.08, ±0.77, ±0.46, ±0.15} (approximately)

Spectral efficiency: 6 bits/symbol / 2 = 3 bits/s/Hz
```

**Gray Coding**: More complex for 64-QAM
```
6 bits split: 3 bits for I, 3 bits for Q
Each coordinate uses Gray code for its 8 levels

I-bits (3 MSBs): Select column (0-7)
Q-bits (3 LSBs): Select row (0-7)

Example: 101011
  I-bits: 101 (Gray) → I-level 6 → I = +5
  Q-bits: 011 (Gray) → Q-level 2 → Q = -3
  Symbol: 5 - j3
```

**Applications**:
```
64-QAM widely used in:
  - Cable modems (DOCSIS)
  - WiFi 802.11n/ac/ax (lower MCS indices)
  - LTE (good signal conditions)
  - Digital TV (DVB-C, ATSC)

Good balance: high throughput + reasonable SNR requirement
```

### 4. 256-QAM

**Constellation**: 16×16 square grid

```
256 constellation points
8 bits per symbol (log₂(256) = 8)
One byte per symbol!

I and Q each have 16 levels:
  I, Q ∈ {-15, -13, -11, ..., -1, +1, ..., +13, +15}

Normalized (E_avg = 1):
  Divide by √170
  Levels span approximately ±1.15

Spectral efficiency: 8 / 2 = 4 bits/s/Hz
```

**Very Dense Constellation**:
```
Minimum distance between points: very small!
  For unnormalized: d_min = 2
  For normalized: d_min = 2/√170 = 0.153

Compare to QPSK: d_min = √2 ≈ 1.414

256-QAM points are 9× closer than QPSK!
  → Extremely sensitive to noise
  → Requires very high SNR (>30 dB typically)
```

**Applications**:
```
256-QAM used only in:
  - WiFi 6 (802.11ax) - excellent signal conditions
  - Cable modems - controlled environment
  - 5G mmWave - short range, high SNR
  - Point-to-point links

Not used in:
  - Mobile cellular (too sensitive to fading)
  - Long-range communications
  - Noisy environments
```

### 5. BER Performance

**SNR Requirements** (for BER = 10⁻⁵ in AWGN):

```
Modulation    Eb/N0 (dB)    SNR_dB (approx)    Relative to BPSK
----------------------------------------------------------------
BPSK          9.6           9.6                Baseline
QPSK          9.6           9.6                +0 dB
16-QAM        14.5          20.5               +4.9 dB
64-QAM        18.8          28.6               +9.2 dB
256-QAM       23.0          37.0               +13.4 dB

Higher-order QAM: much higher SNR required!

SNR = Eb/N0 + 10·log₁₀(bits per symbol)
```

**Why Does QAM Need More SNR?**
```
1. Smaller minimum distance:
   Points are closer together
   More likely to confuse adjacent symbols

2. Multiple amplitude levels:
   Need to distinguish not just phase, but also amplitude
   Amplitude more sensitive to fading and AGC errors

3. More constellation points:
   More ways to make errors
   Higher symbol error rate for same noise level
```

**Trade-off Example**:
```
Channel: 10 MHz bandwidth, SNR = 25 dB

QPSK (SNR req: 10 dB):
  ✓ Margin: 15 dB (excellent!)
  Data rate: 10 Mbps

16-QAM (SNR req: 21 dB):
  ✓ Margin: 4 dB (acceptable)
  Data rate: 20 Mbps

64-QAM (SNR req: 29 dB):
  ✗ Insufficient SNR! Would have high BER
  Data rate: N/A (won't work reliably)

Choose 16-QAM: 2× QPSK throughput with acceptable margin
```

### 6. Adaptive Modulation

**Link Adaptation**: Change modulation based on channel quality

```
Real-world systems (WiFi, LTE) use adaptive modulation:

SNR Range     Modulation    Data Rate (rel)    Use Case
---------------------------------------------------------------
< 10 dB       BPSK/QPSK     1×                 Poor signal, long range
10-20 dB      16-QAM        2×                 Moderate signal
20-28 dB      64-QAM        3×                 Good signal
28-35 dB      256-QAM       4×                 Excellent signal
> 35 dB       1024-QAM      5×                 Perfect conditions

System constantly monitors channel quality (SNR, BER)
Automatically switches modulation to maximize throughput
```

**MCS (Modulation and Coding Scheme)**:
```
WiFi 802.11ac example:

MCS Index    Modulation    Code Rate    Data Rate (80 MHz)
-------------------------------------------------------------
MCS 0        BPSK          1/2          29.3 Mbps
MCS 1        QPSK          1/2          58.5 Mbps
MCS 2        QPSK          3/4          87.8 Mbps
MCS 3        16-QAM        1/2          117 Mbps
MCS 4        16-QAM        3/4          175.5 Mbps
MCS 5        64-QAM        2/3          234 Mbps
MCS 6        64-QAM        3/4          263.3 Mbps
MCS 7        64-QAM        5/6          292.5 Mbps
MCS 8        256-QAM       3/4          351 Mbps
MCS 9        256-QAM       5/6          390 Mbps

Higher MCS = higher data rate, but needs better signal
```

---

## Part 1: Simulation (Pure Python)

### Implementation 1: 16-QAM Modulator and Demodulator

```python
import numpy as np
import matplotlib.pyplot as plt

class QAM16Modem:
    """16-QAM modem with Gray coding"""

    def __init__(self, carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20):
        """
        Args:
            carrier_freq: Carrier frequency (Hz)
            symbol_rate: Symbol rate (symbols/s)
            samples_per_symbol: Samples per symbol
        """
        self.fc = carrier_freq
        self.Rs = symbol_rate
        self.Ts = 1 / symbol_rate
        self.sps = samples_per_symbol
        self.fs = symbol_rate * samples_per_symbol

        # Generate 16-QAM constellation (Gray coded)
        # Levels: -3, -1, +1, +3
        levels = np.array([-3, -1, 1, 3])

        # Create all I-Q combinations
        I_vals = np.repeat(levels, 4)
        Q_vals = np.tile(levels, 4)

        # Normalize to unit average energy
        self.constellation = (I_vals + 1j * Q_vals) / np.sqrt(10)

        # Gray code mapping for 16-QAM
        # Bits: [I1, I0, Q1, Q0]
        self.gray_map = {}
        gray_codes = [
            [0,0,0,0], [0,0,0,1], [0,0,1,1], [0,0,1,0],  # I=-3
            [0,1,0,0], [0,1,0,1], [0,1,1,1], [0,1,1,0],  # I=-1
            [1,1,0,0], [1,1,0,1], [1,1,1,1], [1,1,1,0],  # I=+1
            [1,0,0,0], [1,0,0,1], [1,0,1,1], [1,0,1,0]   # I=+3
        ]

        for i, code in enumerate(gray_codes):
            self.gray_map[tuple(code)] = i

        self.gray_demap = {v: k for k, v in self.gray_map.items()}

        print(f"16-QAM Modem Configuration:")
        print(f"  Carrier freq:     {carrier_freq/1e6:.2f} MHz")
        print(f"  Symbol rate:      {symbol_rate/1e3:.0f} ksps")
        print(f"  Bit rate:         {4*symbol_rate/1e3:.0f} kbps (4 bits/symbol)")
        print(f"  Bandwidth:        {2*symbol_rate/1e6:.2f} MHz")
        print(f"  Spectral eff.:    {4*symbol_rate/(2*symbol_rate):.1f} bits/s/Hz")

    def bits_to_symbols(self, bits):
        """Map bits to 16-QAM symbols"""
        # Pad to multiple of 4
        n_pad = (4 - len(bits) % 4) % 4
        if n_pad > 0:
            bits = np.concatenate([bits, np.zeros(n_pad, dtype=int)])

        # Reshape to groups of 4
        bit_groups = bits.reshape(-1, 4)

        # Map to symbols
        symbols = np.zeros(len(bit_groups), dtype=complex)
        for i, group in enumerate(bit_groups):
            idx = self.gray_map[tuple(group)]
            symbols[i] = self.constellation[idx]

        return symbols

    def symbols_to_bits(self, symbol_indices, n_bits):
        """Demap symbols to bits"""
        bits = np.zeros(len(symbol_indices) * 4, dtype=int)

        for i, idx in enumerate(symbol_indices):
            bit_group = self.gray_demap[idx]
            bits[4*i:4*i+4] = bit_group

        return bits[:n_bits]

    def modulate(self, bits):
        """Modulate bits using 16-QAM"""
        symbols = self.bits_to_symbols(bits)

        # Upsample
        symbols_up = np.repeat(symbols, self.sps)

        # Generate carrier
        n_samples = len(symbols_up)
        t = np.arange(n_samples) / self.fs
        carrier = np.exp(2j * np.pi * self.fc * t)

        # Modulate
        signal_complex = symbols_up * carrier
        signal_real = np.real(signal_complex)

        return signal_real, t

    def demodulate(self, signal, n_bits):
        """Demodulate 16-QAM signal"""
        from scipy import signal as sig

        n_samples = len(signal)
        n_symbols = n_samples // self.sps

        # To complex baseband
        t = np.arange(n_samples) / self.fs
        lo = np.exp(-2j * np.pi * self.fc * t)
        baseband = signal * lo * 2

        # Lowpass filter
        b, a = sig.butter(4, self.Rs / (self.fs/2), btype='low')
        baseband_filt = sig.filtfilt(b, a, baseband)

        # Sample at symbol rate
        symbols_rx = baseband_filt[self.sps//2::self.sps][:n_symbols]

        # Decision: nearest neighbor
        symbol_indices = np.zeros(len(symbols_rx), dtype=int)
        for i, sym in enumerate(symbols_rx):
            distances = np.abs(self.constellation - sym)
            symbol_indices[i] = np.argmin(distances)

        # Convert to bits
        bits_rx = self.symbols_to_bits(symbol_indices, n_bits)

        return bits_rx

    def plot_constellation(self, received_symbols=None):
        """Plot 16-QAM constellation"""
        fig, ax = plt.subplots(1, 1, figsize=(10, 10))

        # Ideal constellation
        for i, point in enumerate(self.constellation):
            bits = self.gray_demap[i]
            label = ''.join(map(str, bits))
            ax.plot(point.real, point.imag, 'bo', markersize=12)
            ax.annotate(label, (point.real, point.imag),
                       xytext=(8, 8), textcoords='offset points',
                       fontsize=8, ha='center')

        # Decision boundaries (grid lines)
        decision_levels = np.array([-2, 0, 2]) / np.sqrt(10)
        for level in decision_levels:
            ax.axhline(level, color='k', linestyle='--', alpha=0.2, linewidth=0.5)
            ax.axvline(level, color='k', linestyle='--', alpha=0.2, linewidth=0.5)

        # Received symbols
        if received_symbols is not None:
            ax.plot(received_symbols.real, received_symbols.imag,
                   'r.', alpha=0.3, markersize=3, label='Received')

        ax.set_xlabel('In-phase (I)', fontsize=12)
        ax.set_ylabel('Quadrature (Q)', fontsize=12)
        ax.set_title('16-QAM Constellation (Gray Coded)', fontsize=14, fontweight='bold')
        ax.grid(True, alpha=0.3)
        if received_symbols is not None:
            ax.legend()
        ax.set_xlim([-1.2, 1.2])
        ax.set_ylim([-1.2, 1.2])
        ax.set_aspect('equal')

        plt.tight_layout()
        plt.savefig('16qam_constellation.png', dpi=150, bbox_inches='tight')
        print("✓ Saved 16qam_constellation.png")
        plt.show()


# Test 16-QAM modem
if __name__ == "__main__":
    print("="*70)
    print("16-QAM MODULATION DEMONSTRATION")
    print("="*70)

    modem = QAM16Modem(carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20)

    # Generate random bits
    np.random.seed(42)
    bits = np.random.randint(0, 2, size=80)
    print(f"\nTransmit bits: {bits[:20]}...")

    # Modulate
    signal, t = modem.modulate(bits)

    # Demodulate
    bits_rx = modem.demodulate(signal, len(bits))

    # Check errors
    errors = np.sum(bits != bits_rx)
    print(f"\nDemodulation errors: {errors}/{len(bits)}")
    print(f"BER: {errors/len(bits):.2e}")

    # Plot constellation
    modem.plot_constellation()

    # Plot spectrum
    fig, ax = plt.subplots(1, 1, figsize=(12, 6))

    fft_signal = np.fft.fftshift(np.fft.fft(signal))
    freqs = np.fft.fftshift(np.fft.fftfreq(len(signal), 1/modem.fs))
    spectrum = 20 * np.log10(np.abs(fft_signal) / len(signal) + 1e-12)

    ax.plot(freqs/1e6, spectrum, 'b-', linewidth=1)
    ax.set_xlabel('Frequency (MHz)')
    ax.set_ylabel('Magnitude (dB)')
    ax.set_title('16-QAM Spectrum')
    ax.grid(True, alpha=0.3)
    ax.set_xlim([0, 3])
    ax.axvline(modem.fc/1e6, color='r', linestyle='--', alpha=0.5)

    plt.tight_layout()
    plt.savefig('16qam_spectrum.png', dpi=150, bbox_inches='tight')
    print("✓ Saved 16qam_spectrum.png")
    plt.show()
```

### Implementation 2: 64-QAM Modulator

```python
class QAM64Modem:
    """64-QAM modem with Gray coding"""

    def __init__(self, carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20):
        """Initialize 64-QAM modem"""
        self.fc = carrier_freq
        self.Rs = symbol_rate
        self.Ts = 1 / symbol_rate
        self.sps = samples_per_symbol
        self.fs = symbol_rate * samples_per_symbol

        # Generate 64-QAM constellation
        # 8 levels: -7, -5, -3, -1, +1, +3, +5, +7
        levels = np.array([-7, -5, -3, -1, 1, 3, 5, 7])

        I_vals = np.repeat(levels, 8)
        Q_vals = np.tile(levels, 8)

        # Normalize: E_avg = sum(I² + Q²)/64 = (4×(7²+5²+3²+1²))/64 = 42
        self.constellation = (I_vals + 1j * Q_vals) / np.sqrt(42)

        # Simplified Gray mapping (full mapping table omitted for brevity)
        # In practice, use lookup tables or algorithmic mapping
        self.bits_per_symbol = 6

        print(f"64-QAM Modem Configuration:")
        print(f"  Carrier freq:     {carrier_freq/1e6:.2f} MHz")
        print(f"  Symbol rate:      {symbol_rate/1e3:.0f} ksps")
        print(f"  Bit rate:         {6*symbol_rate/1e3:.0f} kbps (6 bits/symbol)")
        print(f"  Spectral eff.:    {6*symbol_rate/(2*symbol_rate):.1f} bits/s/Hz")

    def bits_to_symbols(self, bits):
        """Map bits to 64-QAM symbols (simplified natural binary mapping)"""
        # Pad to multiple of 6
        n_pad = (6 - len(bits) % 6) % 6
        if n_pad > 0:
            bits = np.concatenate([bits, np.zeros(n_pad, dtype=int)])

        bit_groups = bits.reshape(-1, 6)
        symbols = np.zeros(len(bit_groups), dtype=complex)

        for i, group in enumerate(bit_groups):
            # Convert bits to index (natural binary for simplicity)
            idx = int(''.join(map(str, group)), 2)
            symbols[i] = self.constellation[idx]

        return symbols

    def modulate(self, bits):
        """Modulate bits using 64-QAM"""
        symbols = self.bits_to_symbols(bits)
        symbols_up = np.repeat(symbols, self.sps)

        n_samples = len(symbols_up)
        t = np.arange(n_samples) / self.fs
        carrier = np.exp(2j * np.pi * self.fc * t)

        signal_complex = symbols_up * carrier
        signal_real = np.real(signal_complex)

        return signal_real, t

    def plot_constellation(self):
        """Plot 64-QAM constellation"""
        fig, ax = plt.subplots(1, 1, figsize=(10, 10))

        ax.plot(self.constellation.real, self.constellation.imag,
               'bo', markersize=8)

        # Decision boundaries
        decision_levels = np.arange(-6, 7, 2) / np.sqrt(42)
        for level in decision_levels:
            ax.axhline(level, color='k', linestyle='--', alpha=0.15, linewidth=0.5)
            ax.axvline(level, color='k', linestyle='--', alpha=0.15, linewidth=0.5)

        ax.set_xlabel('In-phase (I)', fontsize=12)
        ax.set_ylabel('Quadrature (Q)', fontsize=12)
        ax.set_title('64-QAM Constellation (8×8 grid)', fontsize=14, fontweight='bold')
        ax.grid(True, alpha=0.3)
        ax.set_xlim([-1.5, 1.5])
        ax.set_ylim([-1.5, 1.5])
        ax.set_aspect('equal')

        plt.tight_layout()
        plt.savefig('64qam_constellation.png', dpi=150, bbox_inches='tight')
        print("✓ Saved 64qam_constellation.png")
        plt.show()


# Test 64-QAM
if __name__ == "__main__":
    print("\n" + "="*70)
    print("64-QAM MODULATION DEMONSTRATION")
    print("="*70)

    modem = QAM64Modem(carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20)
    modem.plot_constellation()
```

### Implementation 3: 256-QAM Modulator

```python
class QAM256Modem:
    """256-QAM modem"""

    def __init__(self, carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20):
        """Initialize 256-QAM modem"""
        self.fc = carrier_freq
        self.Rs = symbol_rate
        self.sps = samples_per_symbol
        self.fs = symbol_rate * samples_per_symbol

        # Generate 256-QAM constellation
        # 16 levels: -15, -13, ..., -1, +1, ..., +13, +15
        levels = np.arange(-15, 16, 2)

        I_vals = np.repeat(levels, 16)
        Q_vals = np.tile(levels, 16)

        # Normalize: E_avg = sum((2k-1)²)/16 for k=1..16
        # = (1²+3²+5²+...+15²)/16 = 170
        self.constellation = (I_vals + 1j * Q_vals) / np.sqrt(170)

        self.bits_per_symbol = 8

        print(f"256-QAM Modem Configuration:")
        print(f"  Carrier freq:     {carrier_freq/1e6:.2f} MHz")
        print(f"  Symbol rate:      {symbol_rate/1e3:.0f} ksps")
        print(f"  Bit rate:         {8*symbol_rate/1e3:.0f} kbps (8 bits/symbol = 1 byte!)")
        print(f"  Spectral eff.:    {8*symbol_rate/(2*symbol_rate):.1f} bits/s/Hz")

    def plot_constellation(self):
        """Plot 256-QAM constellation"""
        fig, ax = plt.subplots(1, 1, figsize=(10, 10))

        ax.plot(self.constellation.real, self.constellation.imag,
               'bo', markersize=4)

        # Decision boundaries (very fine grid)
        decision_levels = np.arange(-14, 15, 2) / np.sqrt(170)
        for level in decision_levels:
            ax.axhline(level, color='k', linestyle='--', alpha=0.1, linewidth=0.3)
            ax.axvline(level, color='k', linestyle='--', alpha=0.1, linewidth=0.3)

        ax.set_xlabel('In-phase (I)', fontsize=12)
        ax.set_ylabel('Quadrature (Q)', fontsize=12)
        ax.set_title('256-QAM Constellation (16×16 grid - VERY DENSE!)',
                    fontsize=14, fontweight='bold')
        ax.grid(True, alpha=0.3)
        ax.set_xlim([-1.5, 1.5])
        ax.set_ylim([-1.5, 1.5])
        ax.set_aspect('equal')

        # Annotate density
        ax.text(0, -1.35, '256 points!', ha='center', fontsize=12,
               bbox=dict(boxstyle='round', facecolor='yellow', alpha=0.7))

        plt.tight_layout()
        plt.savefig('256qam_constellation.png', dpi=150, bbox_inches='tight')
        print("✓ Saved 256qam_constellation.png")
        plt.show()


# Test 256-QAM
if __name__ == "__main__":
    print("\n" + "="*70)
    print("256-QAM MODULATION DEMONSTRATION")
    print("="*70)

    modem = QAM256Modem(carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20)
    modem.plot_constellation()
```

### Implementation 4: QAM Comparison

```python
def compare_qam_modulations():
    """Compare all QAM schemes"""

    print("\n" + "="*70)
    print("QAM MODULATION COMPARISON")
    print("="*70)

    # Create modems
    qam16 = QAM16Modem(1e6, 100e3, 20)
    qam64 = QAM64Modem(1e6, 100e3, 20)
    qam256 = QAM256Modem(1e6, 100e3, 20)

    # Plot side-by-side constellations
    fig, axes = plt.subplots(1, 3, figsize=(18, 6))

    # 16-QAM
    axes[0].plot(qam16.constellation.real, qam16.constellation.imag, 'bo', markersize=10)
    axes[0].set_title('16-QAM\n(4×4 grid, 4 bits/sym)', fontsize=12, fontweight='bold')
    axes[0].set_xlabel('I')
    axes[0].set_ylabel('Q')
    axes[0].grid(True, alpha=0.3)
    axes[0].set_xlim([-1.2, 1.2])
    axes[0].set_ylim([-1.2, 1.2])
    axes[0].set_aspect('equal')

    # 64-QAM
    axes[1].plot(qam64.constellation.real, qam64.constellation.imag, 'go', markersize=6)
    axes[1].set_title('64-QAM\n(8×8 grid, 6 bits/sym)', fontsize=12, fontweight='bold')
    axes[1].set_xlabel('I')
    axes[1].set_ylabel('Q')
    axes[1].grid(True, alpha=0.3)
    axes[1].set_xlim([-1.2, 1.2])
    axes[1].set_ylim([-1.2, 1.2])
    axes[1].set_aspect('equal')

    # 256-QAM
    axes[2].plot(qam256.constellation.real, qam256.constellation.imag, 'ro', markersize=3)
    axes[2].set_title('256-QAM\n(16×16 grid, 8 bits/sym)', fontsize=12, fontweight='bold')
    axes[2].set_xlabel('I')
    axes[2].set_ylabel('Q')
    axes[2].grid(True, alpha=0.3)
    axes[2].set_xlim([-1.2, 1.2])
    axes[2].set_ylim([-1.2, 1.2])
    axes[2].set_aspect('equal')

    plt.tight_layout()
    plt.savefig('qam_comparison_constellations.png', dpi=150, bbox_inches='tight')
    print("\n✓ Saved qam_comparison_constellations.png")
    plt.show()

    # Comparison table
    print("\n" + "="*70)
    print("QAM PERFORMANCE COMPARISON")
    print("="*70)
    print(f"{'Parameter':<25} {'16-QAM':<15} {'64-QAM':<15} {'256-QAM':<15}")
    print("-" * 70)
    print(f"{'Bits per symbol':<25} {4:<15} {6:<15} {8:<15}")
    print(f"{'Constellation points':<25} {'16 (4×4)':<15} {'64 (8×8)':<15} {'256 (16×16)':<15}")
    print(f"{'Spectral efficiency':<25} {'2.0':<15} {'3.0':<15} {'4.0':<15}")
    print(f"{'Min. distance (norm)':<25} {'0.632':<15} {'0.308':<15} {'0.153':<15}")
    print(f"{'Eb/N0 @ BER=10⁻⁵':<25} {'14.5 dB':<15} {'18.8 dB':<15} {'23.0 dB':<15}")
    print(f"{'SNR requirement':<25} {'~21 dB':<15} {'~29 dB':<15} {'~37 dB':<15}")
    print(f"{'Typical application':<25} {'WiFi, LTE':<15} {'Cable, WiFi':<15} {'WiFi 6':<15}")
    print("="*70)


if __name__ == "__main__":
    compare_qam_modulations()
```

---

## Part 2: PlutoSDR Hardware Implementation

```python
import adi
import numpy as np
import matplotlib.pyplot as plt

class PlutoQAMTest:
    """Test QAM modulations on PlutoSDR"""

    def __init__(self, uri="ip:192.168.2.1"):
        self.sdr = adi.Pluto(uri)
        self.sdr.sample_rate = int(2e6)
        self.sdr.tx_lo = int(915e6)
        self.sdr.rx_lo = int(915e6)
        self.sdr.tx_cyclic_buffer = True
        self.sdr.tx_hardwaregain_chan0 = -30
        self.sdr.rx_hardwaregain_chan0 = 40

        print("PlutoSDR QAM Test")
        print(f"Sample rate: 2 MSPS")

    def test_16qam(self, symbol_rate=50e3, n_symbols=1000):
        """Test 16-QAM on PlutoSDR"""

        # Generate 16-QAM symbols
        bits = np.random.randint(0, 2, size=n_symbols*4)
        sps = int(self.sdr.sample_rate / symbol_rate)

        # Create modem
        modem = QAM16Modem(0, symbol_rate, sps)
        symbols = modem.bits_to_symbols(bits)

        # Upsample
        tx_samples = np.repeat(symbols, sps) * 0.5

        # Transmit
        self.sdr.tx(tx_samples)

        # Receive
        rx_samples = self.sdr.rx()

        # Downsample
        rx_symbols = rx_samples[sps//2::sps][:n_symbols]

        # Plot constellation
        fig, axes = plt.subplots(1, 2, figsize=(14, 7))

        # TX
        axes[0].plot(symbols.real, symbols.imag, 'b.', alpha=0.5, markersize=5)
        for point in modem.constellation:
            axes[0].plot(point.real, point.imag, 'ro', markersize=12, alpha=0.7)
        axes[0].set_title('Transmitted 16-QAM')
        axes[0].set_xlabel('I')
        axes[0].set_ylabel('Q')
        axes[0].grid(True, alpha=0.3)
        axes[0].set_aspect('equal')
        axes[0].set_xlim([-1.2, 1.2])
        axes[0].set_ylim([-1.2, 1.2])

        # RX
        rx_norm = rx_symbols / np.mean(np.abs(rx_symbols))
        axes[1].plot(rx_norm.real, rx_norm.imag, 'g.', alpha=0.3, markersize=3)
        for point in modem.constellation:
            axes[1].plot(point.real, point.imag, 'ro', markersize=12, alpha=0.7)
        axes[1].set_title('Received 16-QAM (PlutoSDR)')
        axes[1].set_xlabel('I')
        axes[1].set_ylabel('Q')
        axes[1].grid(True, alpha=0.3)
        axes[1].set_aspect('equal')
        axes[1].set_xlim([-1.2, 1.2])
        axes[1].set_ylim([-1.2, 1.2])

        plt.tight_layout()
        plt.savefig('pluto_16qam_constellation.png', dpi=150, bbox_inches='tight')
        print("\n✓ Saved pluto_16qam_constellation.png")
        plt.show()


# Run test
if __name__ == "__main__":
    tester = PlutoQAMTest()
    tester.test_16qam()
```

---

## Summary

In this lab, you learned:

✅ **QAM Fundamentals**:
   - Combines amplitude and phase modulation
   - Square constellations: M = 4, 16, 64, 256, ...
   - Higher data rates than PSK in same bandwidth

✅ **16-QAM**: 4 bits/symbol, 2 bits/s/Hz
   - Good balance: efficiency vs. robustness
   - Widely used in WiFi, LTE, cable modems

✅ **64-QAM**: 6 bits/symbol, 3 bits/s/Hz
   - 3× efficiency of BPSK
   - Requires ~29 dB SNR

✅ **256-QAM**: 8 bits/symbol, 4 bits/s/Hz
   - One byte per symbol!
   - Requires >35 dB SNR (excellent conditions only)

✅ **Adaptive Modulation**:
   - Real systems switch between modulations
   - Based on channel quality (SNR, BER)
   - Maximize throughput while maintaining reliability

---

## Next Steps

Continue to:
- **LAB 3.4**: BER Testing and Eye Diagrams
- **LAB 3.5**: Pulse Shaping and Matched Filtering
