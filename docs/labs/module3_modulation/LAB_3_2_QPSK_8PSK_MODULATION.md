# LAB 3.2: QPSK and 8-PSK Modulation

## Overview

This lab explores **higher-order Phase Shift Keying (PSK)** modulation schemes: **QPSK (Quadrature Phase Shift Keying)** and **8-PSK**. These modulations transmit multiple bits per symbol, achieving higher spectral efficiency than binary modulation while maintaining constant envelope properties.

## Learning Objectives

After completing this lab, you will understand:
- QPSK (4-PSK): 2 bits per symbol, 4 constellation points
- 8-PSK: 3 bits per symbol, 8 constellation points
- Gray coding for optimal bit-to-symbol mapping
- Spectral efficiency: bits/symbol vs. bandwidth
- Symbol Error Rate (SER) vs. Bit Error Rate (BER)
- I/Q representation of M-PSK signals
- Coherent demodulation with phase recovery
- Trade-offs: data rate vs. noise immunity
- PlutoSDR implementation of QPSK and 8-PSK

## Prerequisites

- LAB 0: PlutoSDR Setup
- LAB 1.3: I/Q Modulation
- LAB 2.1-2.3: Sampling and quantization
- LAB 3.1: ASK, FSK, and PSK fundamentals
- Understanding of complex numbers and constellation diagrams
- Basic Python or C programming

## Theory

### 1. QPSK (Quadrature Phase Shift Keying)

**Definition**: 4 phase states, 2 bits per symbol.

```
QPSK = Quaternary PSK = 4-PSK

4 symbols → log₂(4) = 2 bits/symbol
2× more efficient than BPSK!

Phase states (π/4 offset):
  Symbol 00 → Phase = 45°   = π/4
  Symbol 01 → Phase = 135°  = 3π/4
  Symbol 11 → Phase = 225°  = 5π/4  (or -3π/4)
  Symbol 10 → Phase = 315°  = 7π/4  (or -π/4)

Note: Gray coding! Adjacent symbols differ by only 1 bit.
```

**Complex Baseband Representation**:
```
For unit amplitude (A=1):

Symbol 00: s = e^(jπ/4)     = (1/√2) + j(1/√2)   = 0.707 + j0.707
Symbol 01: s = e^(j3π/4)    = (-1/√2) + j(1/√2)  = -0.707 + j0.707
Symbol 11: s = e^(j5π/4)    = (-1/√2) - j(1/√2)  = -0.707 - j0.707
Symbol 10: s = e^(j7π/4)    = (1/√2) - j(1/√2)   = 0.707 - j0.707

All points lie on unit circle at 45°, 135°, 225°, 315°
```

**QPSK Constellation Diagram**:
```
        Q (Imaginary)
        |
    01  |  00
     ●  |  ●
  ------+------  I (Real)
     ●  |  ●
    11  |  10
        |

Each point represents 2 bits
Minimum phase separation: 90°
Amplitude: constant (1.0)
```

**Why Gray Coding?**
```
Without Gray coding (Natural binary):
  00 → 01 → 10 → 11
  Adjacent symbols can differ by 2 bits!

With Gray coding:
  00 → 01 → 11 → 10
  Adjacent symbols differ by exactly 1 bit

If noise causes error to adjacent symbol:
  - Gray: 1 bit error
  - Natural: up to 2 bit errors

Gray coding minimizes BER for given SER!
```

**QPSK as Two BPSK Signals**:
```
QPSK can be viewed as two independent BPSK signals in quadrature:

I-channel (In-phase):     I = bit₀ (mapped to ±1/√2)
Q-channel (Quadrature):   Q = bit₁ (mapped to ±1/√2)

s(t) = I·cos(2πfc·t) - Q·sin(2πfc·t)

Example: bits = [1, 0] → symbol "10"
  I = +1/√2 (bit 1)
  Q = -1/√2 (bit 0)
  Phase = atan2(-1/√2, +1/√2) = -45° = 315°
```

**Bandwidth**:
```
QPSK transmits 2 bits per symbol:
  Bit rate: Rb = 2·Rs
  Symbol rate: Rs = Rb/2

Bandwidth (null-to-null): BW ≈ 2·Rs = Rb

Compare to BPSK:
  BPSK: BW ≈ 2·Rs = 2·Rb (for same bit rate)
  QPSK: BW ≈ Rb

QPSK is 2× more spectrally efficient than BPSK!

Example: 1 Mbps data rate
  BPSK: Rs = 1 Msps, BW = 2 MHz
  QPSK: Rs = 0.5 Msps, BW = 1 MHz
```

### 2. 8-PSK

**Definition**: 8 phase states, 3 bits per symbol.

```
8-PSK = Octal PSK

8 symbols → log₂(8) = 3 bits/symbol
1.5× more efficient than QPSK
3× more efficient than BPSK!

Phase states (π/8 offset from I-axis):
  Symbol 000 → 0°    = 0
  Symbol 001 → 45°   = π/4
  Symbol 011 → 90°   = π/2
  Symbol 010 → 135°  = 3π/4
  Symbol 110 → 180°  = π
  Symbol 111 → 225°  = 5π/4
  Symbol 101 → 270°  = 3π/2
  Symbol 100 → 315°  = 7π/4

Gray coding: adjacent symbols differ by 1 bit
```

**8-PSK Constellation**:
```
            Q
            |
       011  |  001
         ●  |  ●
    010   \ | /   000
       ●---\|/---●
      ------+------  I
       ●---/|\---●
    110   / | \   100
         ●  |  ●
       111  |  101
            |

8 equally-spaced points on unit circle
Phase separation: 45° (π/4)
Smaller separation → more sensitive to noise
```

**Complex Baseband** (unit amplitude):
```
For symbol k (k = 0 to 7):
  Phase: θ_k = k·(π/4)  (45° increments)
  s_k = e^(jθ_k) = cos(θ_k) + j·sin(θ_k)

Examples:
  Symbol 000 (k=0): s = e^(j·0)    = 1.000 + j0.000
  Symbol 001 (k=1): s = e^(j·π/4)  = 0.707 + j0.707
  Symbol 011 (k=2): s = e^(j·π/2)  = 0.000 + j1.000
  Symbol 010 (k=3): s = e^(j·3π/4) = -0.707 + j0.707
  ...
```

**Spectral Efficiency**:
```
8-PSK transmits 3 bits per symbol:
  Bit rate: Rb = 3·Rs
  Symbol rate: Rs = Rb/3

Bandwidth: BW ≈ 2·Rs = (2/3)·Rb

Example: 1.5 Mbps data rate
  BPSK: Rs = 1.5 Msps, BW = 3.0 MHz, 0.5 bits/s/Hz
  QPSK: Rs = 0.75 Msps, BW = 1.5 MHz, 1.0 bits/s/Hz
  8-PSK: Rs = 0.5 Msps, BW = 1.0 MHz, 1.5 bits/s/Hz

8-PSK achieves 3× spectral efficiency of BPSK!
```

### 3. Gray Coding

**Gray Code Tables**:

**QPSK Gray Coding**:
```
Quadrant    Phase    Gray Code    Natural Binary
----------------------------------------------------
I (Q1)      45°      00           00
II (Q2)     135°     01           01
III (Q3)    225°     11           10  ← Difference!
IV (Q4)     315°     10           11  ← Difference!

Gray code property: Hamming distance = 1 between adjacent symbols
```

**8-PSK Gray Coding**:
```
Position   Phase    Gray Code    Natural Binary
----------------------------------------------------
0          0°       000          000
1          45°      001          001
2          90°      011          010  ← Difference!
3          135°     010          011  ← Difference!
4          180°     110          100  ← Difference!
5          225°     111          101  ← Difference!
6          270°     101          110  ← Difference!
7          315°     100          111  ← Difference!
```

**Gray Code Generation Algorithm**:
```python
def binary_to_gray(n, bits):
    """Convert binary to Gray code"""
    gray = n ^ (n >> 1)
    return gray

def gray_to_binary(gray, bits):
    """Convert Gray code to binary"""
    binary = gray
    while gray >>= 1:
        binary ^= gray
    return binary

# Example: QPSK
for i in range(4):
    gray = binary_to_gray(i, 2)
    print(f"Binary {i:02b} → Gray {gray:02b}")

Output:
  Binary 00 → Gray 00
  Binary 01 → Gray 01
  Binary 10 → Gray 11
  Binary 11 → Gray 10
```

### 4. Symbol Error Rate vs. Bit Error Rate

**Symbol Error Rate (SER)**: Probability of detecting wrong symbol

**Bit Error Rate (BER)**: Probability of detecting wrong bit

```
Relationship depends on:
  1. Number of bits per symbol (M-ary)
  2. Bit-to-symbol mapping (Gray vs. Natural)

For Gray-coded M-PSK:
  When SER is small, most errors are to adjacent symbols
  Adjacent symbols differ by 1 bit (Gray coding)
  → On average, 1 bit error per symbol error

  BER ≈ SER / log₂(M)

Example: QPSK (M=4, 2 bits/symbol)
  If SER = 10⁻³
  BER ≈ 10⁻³ / 2 = 5×10⁻⁴

Example: 8-PSK (M=8, 3 bits/symbol)
  If SER = 10⁻³
  BER ≈ 10⁻³ / 3 = 3.3×10⁻⁴
```

**Why is this important?**
```
When comparing modulation schemes, specify whether you mean:
  - Bit error rate (application cares about bits)
  - Symbol error rate (modem measures symbols)

Gray coding gives best SER-to-BER conversion!
```

### 5. Performance Comparison

**Eb/N0 Required for BER = 10⁻⁵** (AWGN channel):

```
Modulation    Eb/N0 (dB)    Notes
--------------------------------------------------------
BPSK          9.6           Baseline (best)
QPSK          9.6           Same as BPSK! (per bit)
8-PSK         14.0          4.4 dB worse than BPSK
16-PSK        18.5          8.9 dB worse than BPSK

Key insight: QPSK has same BER performance as BPSK
            but 2× spectral efficiency!

Higher-order PSK (8-PSK, 16-PSK):
  - Better spectral efficiency
  - Worse noise performance
  - Need higher SNR for same BER
```

**Why does QPSK = BPSK?**
```
QPSK is two orthogonal BPSK signals:
  - I-channel: independent BPSK
  - Q-channel: independent BPSK

Each bit has same distance to decision boundary as in BPSK:
  d = √2 (for unit-energy symbols)

Same minimum distance → same BER performance!

But: QPSK transmits 2 bits simultaneously → 2× throughput
```

**Constellation Distance Analysis**:
```
Minimum distance between constellation points:

BPSK (2 points): d_min = 2.0 (maximum possible)

QPSK (4 points): d_min = √2 ≈ 1.414

8-PSK (8 points): d_min = 2·sin(π/8) ≈ 0.765

16-PSK (16 points): d_min = 2·sin(π/16) ≈ 0.390

Smaller d_min → more sensitive to noise → higher BER

This is the fundamental trade-off!
```

### 6. Demodulation

**Coherent Demodulation** (required for PSK):

```
1. Carrier Recovery:
   - Estimate carrier phase φ_0
   - Methods: Costas loop, PLL, pilot tones
   - Critical for PSK!

2. Symbol Detection:
   - Multiply by e^(-j2πfc·t - jφ_0)
   - Integrate over symbol period
   - Result: complex sample = I + jQ

3. Decision:
   - Calculate distances to all constellation points
   - Choose nearest point
   - Map symbol to bits (Gray decoding)

QPSK example:
  Received: r = 0.6 + j0.8 (noisy)
  Distances to 4 points:
    d(00) = |r - (0.707+j0.707)| = 0.15  ← Minimum!
    d(01) = |r - (-0.707+j0.707)| = 1.31
    d(11) = |r - (-0.707-j0.707)| = 2.05
    d(10) = |r - (0.707-j0.707)| = 1.51
  Decision: Symbol 00
```

**Simplified QPSK Decision**:
```
Can use sign of I and Q directly:

If I > 0 and Q > 0: Symbol 00
If I < 0 and Q > 0: Symbol 01
If I < 0 and Q < 0: Symbol 11
If I > 0 and Q < 0: Symbol 10

Very efficient! No distance calculations needed.
```

---

## Part 1: Simulation (Pure Python)

### Implementation 1: QPSK Modulator and Demodulator

```python
import numpy as np
import matplotlib.pyplot as plt

class QPSKModem:
    """QPSK (4-PSK) modem with Gray coding"""

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

        # QPSK constellation (Gray coded, π/4 offset)
        # Symbols: 00, 01, 11, 10 at phases π/4, 3π/4, 5π/4, 7π/4
        self.constellation = np.array([
            np.exp(1j * np.pi / 4),      # 00: 45°
            np.exp(1j * 3*np.pi / 4),    # 01: 135°
            np.exp(1j * 5*np.pi / 4),    # 11: 225°
            np.exp(1j * 7*np.pi / 4)     # 10: 315°
        ])

        # Gray code mapping: bits → symbol index
        self.gray_map = {
            (0, 0): 0,  # 00 → index 0
            (0, 1): 1,  # 01 → index 1
            (1, 1): 2,  # 11 → index 2
            (1, 0): 3   # 10 → index 3
        }

        # Reverse mapping: symbol index → bits
        self.gray_demap = {v: k for k, v in self.gray_map.items()}

        print(f"QPSK Modem Configuration:")
        print(f"  Carrier freq:     {carrier_freq/1e6:.2f} MHz")
        print(f"  Symbol rate:      {symbol_rate/1e3:.0f} ksps")
        print(f"  Bit rate:         {2*symbol_rate/1e3:.0f} kbps (2 bits/symbol)")
        print(f"  Bandwidth:        {2*symbol_rate/1e6:.2f} MHz")
        print(f"  Spectral eff.:    {2*symbol_rate/(2*symbol_rate):.1f} bits/s/Hz")

    def bits_to_symbols(self, bits):
        """
        Map bits to QPSK symbols using Gray coding

        Args:
            bits: Binary array (length must be even)

        Returns:
            Complex symbols
        """
        if len(bits) % 2 != 0:
            raise ValueError("Bit length must be even for QPSK")

        # Reshape to pairs
        bit_pairs = bits.reshape(-1, 2)

        # Map each pair to symbol
        symbols = np.zeros(len(bit_pairs), dtype=complex)
        for i, pair in enumerate(bit_pairs):
            idx = self.gray_map[tuple(pair)]
            symbols[i] = self.constellation[idx]

        return symbols

    def symbols_to_bits(self, symbol_indices):
        """
        Demap symbol indices to bits using Gray decoding

        Args:
            symbol_indices: Array of symbol indices (0-3)

        Returns:
            Binary array
        """
        bits = np.zeros(len(symbol_indices) * 2, dtype=int)

        for i, idx in enumerate(symbol_indices):
            bit_pair = self.gray_demap[idx]
            bits[2*i:2*i+2] = bit_pair

        return bits

    def modulate(self, bits):
        """
        Modulate bits using QPSK

        Args:
            bits: Binary data

        Returns:
            Modulated signal (real samples), time vector
        """
        # Convert bits to symbols
        symbols = self.bits_to_symbols(bits)

        # Upsample symbols
        n_samples = len(symbols) * self.sps
        symbols_up = np.zeros(n_samples, dtype=complex)
        symbols_up[::self.sps] = symbols

        # Pulse shape (rectangular for now)
        symbols_up = np.repeat(symbols, self.sps)

        # Generate carrier and modulate
        t = np.arange(n_samples) / self.fs
        carrier = np.exp(2j * np.pi * self.fc * t)

        # Complex baseband to RF
        signal_complex = symbols_up * carrier
        signal_real = np.real(signal_complex)

        return signal_real, t

    def demodulate(self, signal):
        """
        Demodulate QPSK signal

        Args:
            signal: Received signal

        Returns:
            Detected bits
        """
        from scipy import signal as sig

        n_samples = len(signal)
        n_symbols = n_samples // self.sps

        # Convert to complex baseband
        t = np.arange(n_samples) / self.fs
        lo = np.exp(-2j * np.pi * self.fc * t)
        baseband = signal * lo * 2  # Factor of 2 from mixing

        # Lowpass filter
        b, a = sig.butter(4, self.Rs / (self.fs/2), btype='low')
        baseband_filt = sig.filtfilt(b, a, baseband)

        # Sample at symbol rate
        symbols_rx = baseband_filt[self.sps//2::self.sps][:n_symbols]

        # Decision: find nearest constellation point
        symbol_indices = np.zeros(len(symbols_rx), dtype=int)
        for i, sym in enumerate(symbols_rx):
            distances = np.abs(self.constellation - sym)
            symbol_indices[i] = np.argmin(distances)

        # Convert symbols to bits
        bits_rx = self.symbols_to_bits(symbol_indices)

        return bits_rx

    def plot_constellation(self, received_symbols=None):
        """
        Plot QPSK constellation

        Args:
            received_symbols: Optional received symbols to overlay
        """
        fig, ax = plt.subplots(1, 1, figsize=(8, 8))

        # Ideal constellation
        for i, point in enumerate(self.constellation):
            bits = self.gray_demap[i]
            label = f"{bits[0]}{bits[1]}"
            ax.plot(point.real, point.imag, 'bo', markersize=15, label=label)
            ax.annotate(label, (point.real, point.imag),
                       xytext=(10, 10), textcoords='offset points',
                       fontsize=12, fontweight='bold')

        # Decision boundaries
        ax.axhline(0, color='k', linestyle='--', alpha=0.3)
        ax.axvline(0, color='k', linestyle='--', alpha=0.3)

        # Received symbols (if provided)
        if received_symbols is not None:
            ax.plot(received_symbols.real, received_symbols.imag,
                   'r.', alpha=0.5, markersize=3, label='Received')

        # Unit circle
        circle = plt.Circle((0, 0), 1, fill=False, color='gray',
                           linestyle=':', alpha=0.5)
        ax.add_patch(circle)

        ax.set_xlabel('In-phase (I)', fontsize=12)
        ax.set_ylabel('Quadrature (Q)', fontsize=12)
        ax.set_title('QPSK Constellation (Gray Coded)', fontsize=14, fontweight='bold')
        ax.grid(True, alpha=0.3)
        ax.legend(loc='upper right')
        ax.set_xlim([-1.5, 1.5])
        ax.set_ylim([-1.5, 1.5])
        ax.set_aspect('equal')

        plt.tight_layout()
        plt.savefig('qpsk_constellation.png', dpi=150, bbox_inches='tight')
        print("✓ Saved qpsk_constellation.png")
        plt.show()


# Test QPSK modem
if __name__ == "__main__":
    print("="*70)
    print("QPSK MODULATION DEMONSTRATION")
    print("="*70)

    modem = QPSKModem(carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20)

    # Generate random bits (even length)
    np.random.seed(42)
    bits = np.random.randint(0, 2, size=40)
    print(f"\nTransmit bits: {bits[:20]}...")

    # Modulate
    signal, t = modem.modulate(bits)

    # Demodulate
    bits_rx = modem.demodulate(signal)

    # Check errors
    errors = np.sum(bits != bits_rx)
    print(f"\nDemodulation errors: {errors}/{len(bits)}")
    print(f"BER: {errors/len(bits):.2e}")

    # Plot constellation
    symbols_tx = modem.bits_to_symbols(bits)
    modem.plot_constellation()

    # Plot modulated signal
    fig, axes = plt.subplots(3, 1, figsize=(12, 9))

    # Bits
    t_bits = np.arange(len(bits)//2) * modem.Ts
    bit_symbols = [''.join(map(str, bits[i:i+2])) for i in range(0, len(bits), 2)]
    axes[0].step(t_bits*1e6, range(len(bit_symbols)), 'b-', linewidth=2, where='post')
    axes[0].set_xlabel('Time (µs)')
    axes[0].set_ylabel('Symbol')
    axes[0].set_title('QPSK Symbols (2 bits each)')
    axes[0].grid(True, alpha=0.3)
    axes[0].set_yticks(range(min(10, len(bit_symbols))))
    axes[0].set_yticklabels(bit_symbols[:10])

    # Modulated signal
    n_show = 5 * modem.sps
    axes[1].plot(t[:n_show]*1e6, signal[:n_show], 'r-', linewidth=1)
    axes[1].set_xlabel('Time (µs)')
    axes[1].set_ylabel('Amplitude')
    axes[1].set_title('QPSK Modulated Signal')
    axes[1].grid(True, alpha=0.3)

    # Spectrum
    fft_signal = np.fft.fftshift(np.fft.fft(signal))
    freqs = np.fft.fftshift(np.fft.fftfreq(len(signal), 1/modem.fs))
    spectrum = 20 * np.log10(np.abs(fft_signal) / len(signal) + 1e-12)

    axes[2].plot(freqs/1e6, spectrum, 'b-', linewidth=1)
    axes[2].set_xlabel('Frequency (MHz)')
    axes[2].set_ylabel('Magnitude (dB)')
    axes[2].set_title('QPSK Spectrum')
    axes[2].grid(True, alpha=0.3)
    axes[2].set_xlim([0, 3])
    axes[2].axvline(modem.fc/1e6, color='r', linestyle='--', alpha=0.5, label='Carrier')
    axes[2].legend()

    plt.tight_layout()
    plt.savefig('qpsk_modulation.png', dpi=150, bbox_inches='tight')
    print("✓ Saved qpsk_modulation.png")
    plt.show()
```

### Implementation 2: 8-PSK Modulator and Demodulator

```python
class PSK8Modem:
    """8-PSK modem with Gray coding"""

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

        # 8-PSK constellation (Gray coded)
        # Starting at 0° and incrementing by 45° (π/4)
        phases = np.arange(8) * (np.pi / 4)
        self.constellation = np.exp(1j * phases)

        # Gray code mapping for 8-PSK
        # Optimized to minimize bit errors between adjacent symbols
        self.gray_map = {
            (0, 0, 0): 0,  # 000 → 0°
            (0, 0, 1): 1,  # 001 → 45°
            (0, 1, 1): 2,  # 011 → 90°
            (0, 1, 0): 3,  # 010 → 135°
            (1, 1, 0): 4,  # 110 → 180°
            (1, 1, 1): 5,  # 111 → 225°
            (1, 0, 1): 6,  # 101 → 270°
            (1, 0, 0): 7   # 100 → 315°
        }

        self.gray_demap = {v: k for k, v in self.gray_map.items()}

        print(f"8-PSK Modem Configuration:")
        print(f"  Carrier freq:     {carrier_freq/1e6:.2f} MHz")
        print(f"  Symbol rate:      {symbol_rate/1e3:.0f} ksps")
        print(f"  Bit rate:         {3*symbol_rate/1e3:.0f} kbps (3 bits/symbol)")
        print(f"  Bandwidth:        {2*symbol_rate/1e6:.2f} MHz")
        print(f"  Spectral eff.:    {3*symbol_rate/(2*symbol_rate):.1f} bits/s/Hz")

    def bits_to_symbols(self, bits):
        """Map bits to 8-PSK symbols using Gray coding"""
        # Pad to multiple of 3
        n_pad = (3 - len(bits) % 3) % 3
        if n_pad > 0:
            bits = np.concatenate([bits, np.zeros(n_pad, dtype=int)])

        # Reshape to triplets
        bit_triplets = bits.reshape(-1, 3)

        # Map each triplet to symbol
        symbols = np.zeros(len(bit_triplets), dtype=complex)
        for i, triplet in enumerate(bit_triplets):
            idx = self.gray_map[tuple(triplet)]
            symbols[i] = self.constellation[idx]

        return symbols

    def symbols_to_bits(self, symbol_indices, n_bits):
        """Demap symbol indices to bits"""
        bits = np.zeros(len(symbol_indices) * 3, dtype=int)

        for i, idx in enumerate(symbol_indices):
            bit_triplet = self.gray_demap[idx]
            bits[3*i:3*i+3] = bit_triplet

        # Remove padding
        return bits[:n_bits]

    def modulate(self, bits):
        """Modulate bits using 8-PSK"""
        # Convert bits to symbols
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
        """Demodulate 8-PSK signal"""
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

        # Decision
        symbol_indices = np.zeros(len(symbols_rx), dtype=int)
        for i, sym in enumerate(symbols_rx):
            distances = np.abs(self.constellation - sym)
            symbol_indices[i] = np.argmin(distances)

        # Convert to bits
        bits_rx = self.symbols_to_bits(symbol_indices, n_bits)

        return bits_rx

    def plot_constellation(self, received_symbols=None):
        """Plot 8-PSK constellation"""
        fig, ax = plt.subplots(1, 1, figsize=(8, 8))

        # Ideal constellation
        for i, point in enumerate(self.constellation):
            bits = self.gray_demap[i]
            label = f"{bits[0]}{bits[1]}{bits[2]}"
            ax.plot(point.real, point.imag, 'bo', markersize=15)

            # Position labels outside circle
            angle = np.angle(point)
            offset_x = 0.2 * np.cos(angle)
            offset_y = 0.2 * np.sin(angle)
            ax.annotate(label, (point.real, point.imag),
                       xytext=(offset_x, offset_y), textcoords='offset points',
                       fontsize=10, fontweight='bold',
                       ha='center', va='center')

        # Received symbols
        if received_symbols is not None:
            ax.plot(received_symbols.real, received_symbols.imag,
                   'r.', alpha=0.5, markersize=3, label='Received')

        # Decision boundaries (lines between constellation points)
        for i in range(8):
            angle = (i + 0.5) * (np.pi / 4)
            ax.plot([0, 1.3*np.cos(angle)], [0, 1.3*np.sin(angle)],
                   'k--', alpha=0.2, linewidth=0.5)

        # Unit circle
        circle = plt.Circle((0, 0), 1, fill=False, color='gray',
                           linestyle=':', alpha=0.5)
        ax.add_patch(circle)

        ax.set_xlabel('In-phase (I)', fontsize=12)
        ax.set_ylabel('Quadrature (Q)', fontsize=12)
        ax.set_title('8-PSK Constellation (Gray Coded)', fontsize=14, fontweight='bold')
        ax.grid(True, alpha=0.3)
        if received_symbols is not None:
            ax.legend(loc='upper right')
        ax.set_xlim([-1.5, 1.5])
        ax.set_ylim([-1.5, 1.5])
        ax.set_aspect('equal')

        plt.tight_layout()
        plt.savefig('8psk_constellation.png', dpi=150, bbox_inches='tight')
        print("✓ Saved 8psk_constellation.png")
        plt.show()


# Test 8-PSK modem
if __name__ == "__main__":
    print("\n" + "="*70)
    print("8-PSK MODULATION DEMONSTRATION")
    print("="*70)

    modem = PSK8Modem(carrier_freq=1e6, symbol_rate=100e3, samples_per_symbol=20)

    # Generate random bits (multiple of 3)
    np.random.seed(42)
    bits = np.random.randint(0, 2, size=60)
    print(f"\nTransmit bits: {bits[:21]}...")

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

    # Plot spectrum comparison
    fig, ax = plt.subplots(1, 1, figsize=(12, 6))

    fft_signal = np.fft.fftshift(np.fft.fft(signal))
    freqs = np.fft.fftshift(np.fft.fftfreq(len(signal), 1/modem.fs))
    spectrum = 20 * np.log10(np.abs(fft_signal) / len(signal) + 1e-12)

    ax.plot(freqs/1e6, spectrum, 'b-', linewidth=1)
    ax.set_xlabel('Frequency (MHz)')
    ax.set_ylabel('Magnitude (dB)')
    ax.set_title('8-PSK Spectrum')
    ax.grid(True, alpha=0.3)
    ax.set_xlim([0, 3])
    ax.axvline(modem.fc/1e6, color='r', linestyle='--', alpha=0.5, label='Carrier')
    ax.legend()

    plt.tight_layout()
    plt.savefig('8psk_spectrum.png', dpi=150, bbox_inches='tight')
    print("✓ Saved 8psk_spectrum.png")
    plt.show()
```

### Implementation 3: Performance Comparison (BPSK vs QPSK vs 8-PSK)

```python
def compare_psk_modulations():
    """Compare BPSK, QPSK, and 8-PSK"""

    print("\n" + "="*70)
    print("PSK MODULATION COMPARISON")
    print("="*70)

    # Import BPSK from LAB 3.1
    from LAB_3_1_ASK_FSK_PSK_MODULATION import PSKModem as BPSKModem

    # Create modems
    fc = 1e6
    Rs = 100e3
    sps = 20

    bpsk = BPSKModem(fc, Rs, sps)
    qpsk = QPSKModem(fc, Rs, sps)
    psk8 = PSK8Modem(fc, Rs, sps)

    # Generate test bits
    np.random.seed(42)
    bits_bpsk = np.random.randint(0, 2, size=60)
    bits_qpsk = bits_bpsk[:60]  # Even length
    bits_8psk = bits_bpsk[:60]  # Multiple of 3

    # Modulate
    sig_bpsk, _ = bpsk.modulate(bits_bpsk)
    sig_qpsk, _ = qpsk.modulate(bits_qpsk)
    sig_8psk, _ = psk8.modulate(bits_8psk)

    # Plot constellation comparison
    fig, axes = plt.subplots(1, 3, figsize=(15, 5))

    # BPSK constellation
    axes[0].plot([-1, 1], [0, 0], 'bo', markersize=15)
    axes[0].annotate('1', (-1, 0), xytext=(0, 15), textcoords='offset points',
                    ha='center', fontsize=12, fontweight='bold')
    axes[0].annotate('0', (1, 0), xytext=(0, 15), textcoords='offset points',
                    ha='center', fontsize=12, fontweight='bold')
    axes[0].axhline(0, color='k', linewidth=0.5)
    axes[0].axvline(0, color='k', linewidth=0.5, linestyle='--')
    axes[0].set_xlabel('I')
    axes[0].set_ylabel('Q')
    axes[0].set_title('BPSK (1 bit/symbol)')
    axes[0].grid(True, alpha=0.3)
    axes[0].set_xlim([-1.5, 1.5])
    axes[0].set_ylim([-1.5, 1.5])
    axes[0].set_aspect('equal')

    # QPSK constellation
    for i, point in enumerate(qpsk.constellation):
        axes[1].plot(point.real, point.imag, 'go', markersize=15)
        bits = qpsk.gray_demap[i]
        label = f"{bits[0]}{bits[1]}"
        axes[1].annotate(label, (point.real, point.imag),
                        xytext=(15, 15), textcoords='offset points',
                        fontsize=12, fontweight='bold')
    axes[1].axhline(0, color='k', linewidth=0.5, linestyle='--')
    axes[1].axvline(0, color='k', linewidth=0.5, linestyle='--')
    axes[1].set_xlabel('I')
    axes[1].set_ylabel('Q')
    axes[1].set_title('QPSK (2 bits/symbol)')
    axes[1].grid(True, alpha=0.3)
    axes[1].set_xlim([-1.5, 1.5])
    axes[1].set_ylim([-1.5, 1.5])
    axes[1].set_aspect('equal')

    # 8-PSK constellation
    for i, point in enumerate(psk8.constellation):
        axes[2].plot(point.real, point.imag, 'ro', markersize=15)
        bits = psk8.gray_demap[i]
        label = f"{bits[0]}{bits[1]}{bits[2]}"
        angle = np.angle(point)
        offset_x = 25 * np.cos(angle)
        offset_y = 25 * np.sin(angle)
        axes[2].annotate(label, (point.real, point.imag),
                        xytext=(offset_x, offset_y), textcoords='offset points',
                        fontsize=10, fontweight='bold', ha='center')
    for i in range(8):
        angle = (i + 0.5) * (np.pi / 4)
        axes[2].plot([0, 1.2*np.cos(angle)], [0, 1.2*np.sin(angle)],
                    'k--', alpha=0.2, linewidth=0.5)
    axes[2].set_xlabel('I')
    axes[2].set_ylabel('Q')
    axes[2].set_title('8-PSK (3 bits/symbol)')
    axes[2].grid(True, alpha=0.3)
    axes[2].set_xlim([-1.5, 1.5])
    axes[2].set_ylim([-1.5, 1.5])
    axes[2].set_aspect('equal')

    plt.tight_layout()
    plt.savefig('psk_comparison_constellations.png', dpi=150, bbox_inches='tight')
    print("\n✓ Saved psk_comparison_constellations.png")
    plt.show()

    # Summary table
    print("\n" + "="*70)
    print("PSK MODULATION COMPARISON TABLE")
    print("="*70)
    print(f"{'Parameter':<25} {'BPSK':<15} {'QPSK':<15} {'8-PSK':<15}")
    print("-" * 70)
    print(f"{'Bits per symbol':<25} {1:<15} {2:<15} {3:<15}")
    print(f"{'Constellation points':<25} {2:<15} {4:<15} {8:<15}")
    print(f"{'Phase separation':<25} {'180°':<15} {'90°':<15} {'45°':<15}")
    print(f"{'Min. distance (d_min)':<25} {'2.000':<15} {'1.414':<15} {'0.765':<15}")
    print(f"{'Spectral efficiency':<25} {'0.5':<15} {'1.0':<15} {'1.5':<15}")
    print(f"{'Eb/N0 @ BER=10⁻⁵':<25} {'9.6 dB':<15} {'9.6 dB':<15} {'14.0 dB':<15}")
    print(f"{'Relative performance':<25} {'Baseline':<15} {'Same':<15} {'+4.4 dB':<15}")
    print("="*70)


if __name__ == "__main__":
    compare_psk_modulations()
```

---

## Part 2: PlutoSDR Hardware Implementation

```python
import adi
import numpy as np
import matplotlib.pyplot as plt

class PlutoPSKTest:
    """Test PSK modulations on PlutoSDR"""

    def __init__(self, uri="ip:192.168.2.1"):
        self.sdr = adi.Pluto(uri)
        self.sdr.sample_rate = int(1e6)
        self.sdr.tx_lo = int(915e6)
        self.sdr.rx_lo = int(915e6)
        self.sdr.tx_cyclic_buffer = True
        self.sdr.tx_hardwaregain_chan0 = -30
        self.sdr.rx_hardwaregain_chan0 = 40

        print("PlutoSDR PSK Test")
        print(f"Center frequency: 915 MHz")
        print(f"Sample rate: 1 MSPS")

    def test_qpsk(self, symbol_rate=10e3, n_symbols=1000):
        """Test QPSK transmission and reception"""

        # Generate QPSK symbols
        bits = np.random.randint(0, 2, size=n_symbols*2)
        sps = int(self.sdr.sample_rate / symbol_rate)

        # Create QPSK modem
        modem = QPSKModem(0, symbol_rate, sps)  # Baseband (fc=0)
        symbols = modem.bits_to_symbols(bits)

        # Upsample for transmission
        tx_samples = np.repeat(symbols, sps) * 0.5  # Scale to prevent clipping

        # Transmit
        self.sdr.tx(tx_samples)

        # Receive
        rx_samples = self.sdr.rx()

        # Plot received constellation
        # Downsample to symbol rate (simple - should add matched filter)
        rx_symbols = rx_samples[sps//2::sps][:n_symbols]

        # Plot
        fig, axes = plt.subplots(1, 2, figsize=(12, 6))

        # Transmitted constellation
        axes[0].plot(symbols.real, symbols.imag, 'b.', alpha=0.5, markersize=5)
        axes[0].set_xlabel('I')
        axes[0].set_ylabel('Q')
        axes[0].set_title('Transmitted QPSK Constellation')
        axes[0].grid(True, alpha=0.3)
        axes[0].set_aspect('equal')
        axes[0].set_xlim([-1.5, 1.5])
        axes[0].set_ylim([-1.5, 1.5])

        # Received constellation
        # Normalize
        rx_norm = rx_symbols / np.mean(np.abs(rx_symbols))
        axes[1].plot(rx_norm.real, rx_norm.imag, 'r.', alpha=0.5, markersize=5)

        # Plot ideal points
        for point in modem.constellation:
            axes[1].plot(point.real, point.imag, 'bo', markersize=15, alpha=0.7)

        axes[1].set_xlabel('I')
        axes[1].set_ylabel('Q')
        axes[1].set_title('Received QPSK Constellation (PlutoSDR)')
        axes[1].grid(True, alpha=0.3)
        axes[1].set_aspect('equal')
        axes[1].set_xlim([-1.5, 1.5])
        axes[1].set_ylim([-1.5, 1.5])

        plt.tight_layout()
        plt.savefig('pluto_qpsk_constellation.png', dpi=150, bbox_inches='tight')
        print("\n✓ Saved pluto_qpsk_constellation.png")
        plt.show()

        print(f"\nQPSK PlutoSDR Test:")
        print(f"  TX symbols: {len(symbols)}")
        print(f"  RX samples: {len(rx_samples)}")
        print(f"  Constellation: Check plot for quality")

        return tx_samples, rx_samples


# Run test
if __name__ == "__main__":
    tester = PlutoPSKTest()
    tx, rx = tester.test_qpsk(symbol_rate=10e3, n_symbols=1000)
```

---

## Summary

In this lab, you learned:

✅ **QPSK (4-PSK)**:
   - 2 bits per symbol, 4 constellation points
   - Same BER as BPSK, but 2× spectral efficiency
   - Gray coding minimizes bit errors

✅ **8-PSK**:
   - 3 bits per symbol, 8 constellation points
   - 3× spectral efficiency of BPSK
   - 4.4 dB worse BER performance (more noise sensitive)

✅ **Gray Coding**:
   - Adjacent symbols differ by 1 bit
   - Minimizes BER for given SER
   - Critical for M-ary modulations

✅ **Performance Trade-offs**:
   - Higher-order PSK: better spectral efficiency
   - But: smaller phase separation, worse BER
   - QPSK is "sweet spot": 2× efficiency, same BER as BPSK

✅ **Symbol vs. Bit Errors**:
   - BER ≈ SER / log₂(M) for Gray coding
   - Important for comparing modulations

---

## Next Steps

Continue to:
- **LAB 3.3**: QAM Modulation (16/64/256-QAM)
- **LAB 3.4**: BER Testing and Eye Diagrams
- **LAB 3.5**: Pulse Shaping and Matched Filtering
