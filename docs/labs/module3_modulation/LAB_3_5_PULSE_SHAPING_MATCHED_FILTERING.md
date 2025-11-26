# LAB 3.5: Pulse Shaping and Matched Filtering

## Overview

This lab explores **pulse shaping** and **matched filtering** - techniques that control spectral occupancy, minimize inter-symbol interference (ISI), and optimize receiver performance. These are fundamental to all modern digital communication systems.

## Learning Objectives

After completing this lab, you will understand:
- Inter-Symbol Interference (ISI) and its causes
- Nyquist criterion for zero-ISI
- Raised cosine (RC) and root-raised cosine (RRC) filters
- Roll-off factor and bandwidth trade-offs
- Matched filtering for optimal SNR
- Complete transmit-receive filter chain
- Timing recovery and symbol synchronization
- Practical implementation with PlutoSDR

## Prerequisites

- LAB 0: PlutoSDR Setup
- LAB 2.1-2.3: Sampling and quantization
- LAB 3.1-3.4: Digital modulation fundamentals
- Understanding of convolution and filtering
- Basic Python or C programming

## Theory

### 1. Inter-Symbol Interference (ISI)

**Problem**: Symbols spread in time and interfere with each other.

```
Without pulse shaping (rectangular pulses):
  Symbol 1:  |████|
  Symbol 2:       |████|
  Symbol 3:            |████|

  Frequency domain: sinc(f) → infinite bandwidth!
  Time domain: abrupt transitions → spreads to adjacent symbols

With pulse shaping:
  Symbol 1:   ╱‾‾‾╲
  Symbol 2:        ╱‾‾‾╲
  Symbol 3:             ╱‾‾‾╲

  Smoother transitions → limited bandwidth
  Controlled ISI → can be zero at sampling instants!
```

**ISI causes**:
```
1. Non-ideal pulse shape:
   Rectangular pulses have sinc(f) spectrum (infinite BW)
   Must be filtered → causes time-domain spreading

2. Limited bandwidth:
   Channel filters signal
   High frequencies removed → time-domain spreading

3. Multipath propagation:
   Delayed copies of signal interfere
   Common in wireless channels

4. Poor symbol timing:
   Sampling at wrong instant
   Picks up "tails" from adjacent symbols
```

### 2. Nyquist ISI Criterion

**Nyquist's First Criterion**: Zero ISI at sampling instants.

```
For pulse p(t), zero ISI requires:

  p(kT) = { 1,  k = 0
          { 0,  k = ±1, ±2, ±3, ...

where T = symbol period, k = integer

In frequency domain:
  ∑[n=-∞ to ∞] P(f + n/T) = T   for |f| ≤ 1/(2T)

This means: folded spectrum is flat in Nyquist bandwidth
```

**Ideal Nyquist Pulse** (sinc function):
```
p(t) = sinc(t/T) = sin(πt/T) / (πt/T)

Properties:
  - p(0) = 1
  - p(kT) = 0 for k ≠ 0
  - Bandwidth: 1/(2T) Hz (minimum possible!)

Problem: Infinite time duration!
  Not practical (non-causal, slow decay)
  Sensitive to timing errors
```

### 3. Raised Cosine (RC) Filter

**Definition**: Practical Nyquist pulse with finite bandwidth.

```
Raised Cosine spectrum:
          ⎧ T,                                    |f| ≤ (1-β)/(2T)
  P(f) =  ⎨ T/2·[1 + cos(π·T/β·(|f| - (1-β)/(2T)))], (1-β)/(2T) < |f| ≤ (1+β)/(2T)
          ⎩ 0,                                    |f| > (1+β)/(2T)

where β = roll-off factor (0 ≤ β ≤ 1)

Bandwidth: BW = (1 + β) / (2T)

Time domain (impulse response):
  p(t) = sinc(t/T) · cos(πβt/T) / (1 - 4β²t²/T²)
```

**Roll-off Factor β**:
```
β = 0 (Ideal Nyquist, rectangular spectrum):
  - Minimum bandwidth: Rs/2
  - Infinite time duration
  - Very sensitive to timing errors
  - Impractical!

β = 0.35 (WiFi, LTE):
  - Bandwidth: 0.675·Rs
  - Faster time decay
  - Moderate timing sensitivity
  - Good compromise

β = 1 (100% excess bandwidth):
  - Bandwidth: Rs (2× minimum)
  - Fast time decay
  - Robust to timing errors
  - Easier implementation

Typical values: β = 0.2 to 0.5
```

**Raised Cosine Properties**:
```
Advantages:
  ✓ Zero ISI at sampling instants
  ✓ Finite bandwidth (unlike sinc)
  ✓ Faster time decay than sinc
  ✓ Adjustable via β

Disadvantages:
  ✗ Still infinite time duration (in theory)
  ✗ Truncation needed (causes small ISI)
  ✗ Full RC must be in transmitter AND receiver
```

### 4. Root-Raised Cosine (RRC) Filter

**Split Filtering**: RC = RRC (TX) ⊗ RRC (RX)

```
Problem with full RC at TX:
  - Received signal has poor SNR before filtering
  - Noise not optimally filtered

Solution: Split RC between TX and RX
  - RRC at transmitter (pulse shaping)
  - RRC at receiver (matched filter)
  - Cascade: RRC ⊗ RRC = RC (zero ISI!)
  - Plus: matched filtering maximizes SNR!
```

**RRC Frequency Response**:
```
  H_RRC(f) = √(H_RC(f))

Root-Raised Cosine spectrum:
          ⎧ √T,                                    |f| ≤ (1-β)/(2T)
  H(f) =  ⎨ √[T/2·[1 + cos(π·T/β·(|f| - (1-β)/(2T)))]],  (1-β)/(2T) < |f| ≤ (1+β)/(2T)
          ⎩ 0,                                     |f| > (1+β)/(2T)

Time domain (impulse response):
  h(t) = (sin(π(1-β)t/T) + 4βt/T·cos(π(1+β)t/T)) / (π·t/T·(1 - (4βt/T)²))
```

**Complete TX-RX Chain**:
```
Transmitter:
  Bits → Symbols → Upsample → RRC Filter → DAC → RF

Receiver:
  RF → ADC → RRC Filter → Downsample → Symbols → Bits
                 ↓
            (Matched filter)

Cascade:
  h_tx(t) ⊗ h_rx(t) = h_RC(t)

Result at sampling instant:
  - Zero ISI (from RC pulse)
  - Maximum SNR (from matched filtering)
```

### 5. Matched Filtering

**Matched Filter Theorem**: Maximize SNR in AWGN.

```
For signal s(t) in AWGN, optimal receiver filter:
  h_matched(t) = s*(-t)  (time-reversed, conjugated)

For real pulses:
  h_matched(t) = s(T-t)  (time-reversed)

SNR gain: proportional to signal energy
  SNR_out / SNR_in = E_signal / N_0
```

**Why RRC is Matched to RRC**:
```
TX pulse: p_tx(t) = h_RRC(t)
RX filter: h_rx(t) = h_RRC(-t) = h_RRC(t)  (RRC is symmetric!)

This is a matched filter!
  → Maximizes SNR before symbol decision
  → Plus: gives RC pulse shape (zero ISI)

Best of both worlds:
  ✓ Zero ISI (from RC)
  ✓ Maximum SNR (from matched filtering)
```

### 6. Filter Design Parameters

**Span (length)**:
```
RRC filter is infinite in theory, truncate in practice:

Filter span: N_symbols × T
  Common: N = 6 to 12 symbol periods

Taps: N_taps = N_symbols × SPS + 1
  where SPS = samples per symbol

Example:
  Span: 8 symbols
  SPS: 8
  Taps: 8×8 + 1 = 65 taps

Longer span:
  ✓ Better frequency response
  ✓ Less ISI
  ✗ More computation
  ✗ More latency
```

**SciPy Implementation**:
```python
from scipy.signal import firwin, remez

# Method 1: Least-squares (firls equivalent)
h_rrc = signal.firwin(
    numtaps,
    cutoff,
    window='hamming',
    pass_zero=False
)

# Method 2: CommPy library (recommended for RRC)
from commpy.filters import rrcosfilter

taps, h_rrc = rrcosfilter(
    N=num_taps,       # Filter length
    alpha=beta,       # Roll-off factor
    Ts=1.0,          # Symbol period (normalized)
    Fs=sps           # Samples per symbol
)
```

### 7. Bandwidth Calculation

**Symbol rate vs. Bandwidth**:
```
Rectangular pulse:
  BW_null = 2·Rs   (null-to-null)

Raised Cosine (β):
  BW = Rs·(1 + β)

Examples:
  Rs = 1 Msps

  β = 0:    BW = 1.0 MHz (minimum Nyquist)
  β = 0.35: BW = 1.35 MHz (typical WiFi/LTE)
  β = 0.5:  BW = 1.5 MHz
  β = 1.0:  BW = 2.0 MHz (100% excess BW)

Spectral efficiency:
  η = Rs / BW = 1 / (1 + β) symbols/s/Hz

  β = 0:    η = 1.0 (maximum)
  β = 0.35: η = 0.74
  β = 0.5:  η = 0.67
  β = 1.0:  η = 0.5
```

---

## Part 1: Simulation (Pure Python)

### Implementation 1: Raised Cosine Filter

```python
import numpy as np
import matplotlib.pyplot as plt
from scipy import signal

class RaisedCosineFilter:
    """Raised Cosine (RC) pulse shaping filter"""

    def __init__(self, beta=0.35, span=8, sps=8):
        """
        Args:
            beta: Roll-off factor (0 to 1)
            span: Filter span in symbol periods
            sps: Samples per symbol
        """
        self.beta = beta
        self.span = span
        self.sps = sps
        self.taps = span * sps + 1

        # Generate filter
        self.h = self.design_rc_filter()

        print(f"Raised Cosine Filter:")
        print(f"  Roll-off (β):  {beta}")
        print(f"  Span:          {span} symbols")
        print(f"  SPS:           {sps}")
        print(f"  Taps:          {self.taps}")
        print(f"  Bandwidth:     {1 + beta:.2f} × Rs")

    def design_rc_filter(self):
        """Design Raised Cosine filter"""
        # Time vector
        t = np.arange(-self.span/2, self.span/2, 1/self.sps)

        # Raised Cosine pulse
        h = np.zeros(len(t))

        for i, ti in enumerate(t):
            if abs(ti) < 1e-10:
                # At t=0
                h[i] = 1.0
            elif abs(abs(ti) - 1/(2*self.beta)) < 1e-10:
                # At t = ±1/(2β)
                h[i] = (np.pi/4) * np.sinc(1/(2*self.beta))
            else:
                # General case
                h[i] = np.sinc(ti) * np.cos(np.pi*self.beta*ti) / (1 - (2*self.beta*ti)**2)

        # Normalize
        h = h / np.sum(h)

        return h

    def plot_impulse_response(self):
        """Plot filter impulse response"""
        fig, axes = plt.subplots(2, 1, figsize=(12, 8))

        t = np.arange(len(self.h)) / self.sps - self.span/2

        # Time domain
        axes[0].plot(t, self.h, 'b-', linewidth=2)
        axes[0].stem(t[::self.sps], self.h[::self.sps], linefmt='r-',
                    markerfmt='ro', basefmt='r-', label='Symbol instants')
        axes[0].set_xlabel('Time (symbol periods)')
        axes[0].set_ylabel('Amplitude')
        axes[0].set_title(f'Raised Cosine Pulse (β = {self.beta})')
        axes[0].grid(True, alpha=0.3)
        axes[0].legend()

        # Frequency domain
        fft_h = np.fft.fftshift(np.fft.fft(self.h, 2048))
        freqs = np.fft.fftshift(np.fft.fftfreq(2048, 1/self.sps))
        magnitude = 20 * np.log10(np.abs(fft_h) + 1e-12)

        axes[1].plot(freqs, magnitude, 'b-', linewidth=2)
        axes[1].axvline((1-self.beta)/2, color='g', linestyle='--',
                       alpha=0.5, label=f'(1-β)/(2T)')
        axes[1].axvline((1+self.beta)/2, color='r', linestyle='--',
                       alpha=0.5, label=f'(1+β)/(2T)')
        axes[1].set_xlabel('Normalized Frequency (×Rs)')
        axes[1].set_ylabel('Magnitude (dB)')
        axes[1].set_title('Raised Cosine Frequency Response')
        axes[1].grid(True, alpha=0.3)
        axes[1].legend()
        axes[1].set_xlim([0, 1.5])
        axes[1].set_ylim([-60, 5])

        plt.tight_layout()
        plt.savefig(f'rc_filter_beta{self.beta}.png', dpi=150, bbox_inches='tight')
        print(f"\n✓ Saved rc_filter_beta{self.beta}.png")
        plt.show()


# Test RC filter
if __name__ == "__main__":
    print("="*70)
    print("RAISED COSINE FILTER DEMONSTRATION")
    print("="*70)

    for beta in [0.0, 0.35, 1.0]:
        print(f"\n{'='*70}")
        rc = RaisedCosineFilter(beta=beta, span=8, sps=8)
        rc.plot_impulse_response()
```

### Implementation 2: Root-Raised Cosine Filter

```python
class RootRaisedCosineFilter:
    """Root-Raised Cosine (RRC) pulse shaping filter"""

    def __init__(self, beta=0.35, span=8, sps=8):
        """
        Args:
            beta: Roll-off factor
            span: Filter span in symbols
            sps: Samples per symbol
        """
        self.beta = beta
        self.span = span
        self.sps = sps
        self.taps = span * sps + 1

        self.h = self.design_rrc_filter()

        print(f"Root-Raised Cosine Filter:")
        print(f"  Roll-off (β):  {beta}")
        print(f"  Taps:          {self.taps}")

    def design_rrc_filter(self):
        """Design RRC filter"""
        t = np.arange(-self.span/2, self.span/2, 1/self.sps)
        h = np.zeros(len(t))

        for i, ti in enumerate(t):
            if abs(ti) < 1e-10:
                # At t=0
                h[i] = 1.0 - self.beta + 4*self.beta/np.pi
            elif abs(abs(ti) - 1/(4*self.beta)) < 1e-10:
                # At t = ±1/(4β)
                h[i] = (self.beta/np.sqrt(2)) * (
                    (1 + 2/np.pi) * np.sin(np.pi/(4*self.beta)) +
                    (1 - 2/np.pi) * np.cos(np.pi/(4*self.beta))
                )
            else:
                # General case
                numerator = np.sin(np.pi*ti*(1-self.beta)) + \
                           4*self.beta*ti*np.cos(np.pi*ti*(1+self.beta))
                denominator = np.pi*ti*(1 - (4*self.beta*ti)**2)
                h[i] = numerator / denominator

        # Normalize energy
        h = h / np.sqrt(np.sum(h**2))

        return h

    def apply_filter(self, symbols_up):
        """Apply RRC filter to upsampled symbols"""
        # Convolve with RRC filter
        return signal.convolve(symbols_up, self.h, mode='same')


# Test RRC filter
if __name__ == "__main__":
    print("\n" + "="*70)
    print("ROOT-RAISED COSINE FILTER DEMONSTRATION")
    print("="*70)

    rrc = RootRaisedCosineFilter(beta=0.35, span=8, sps=8)

    # Plot RRC and verify RRC ⊗ RRC = RC
    fig, axes = plt.subplots(2, 1, figsize=(12, 8))

    t = np.arange(len(rrc.h)) / rrc.sps - rrc.span/2

    # RRC impulse response
    axes[0].plot(t, rrc.h, 'b-', linewidth=2, label='RRC')
    axes[0].set_xlabel('Time (symbol periods)')
    axes[0].set_ylabel('Amplitude')
    axes[0].set_title('Root-Raised Cosine Filter')
    axes[0].grid(True, alpha=0.3)
    axes[0].legend()

    # RRC ⊗ RRC = RC
    rc_from_rrc = signal.convolve(rrc.h, rrc.h, mode='same')
    rc_from_rrc = rc_from_rrc / np.max(rc_from_rrc)  # Normalize

    axes[1].plot(t, rc_from_rrc, 'r-', linewidth=2, label='RRC ⊗ RRC')
    axes[1].stem(t[::rrc.sps], rc_from_rrc[::rrc.sps],
                linefmt='g-', markerfmt='go', basefmt='g-',
                label='Zero crossings')
    axes[1].set_xlabel('Time (symbol periods)')
    axes[1].set_ylabel('Amplitude')
    axes[1].set_title('RRC ⊗ RRC = Raised Cosine (Zero ISI)')
    axes[1].grid(True, alpha=0.3)
    axes[1].legend()

    plt.tight_layout()
    plt.savefig('rrc_cascade.png', dpi=150, bbox_inches='tight')
    print("\n✓ Saved rrc_cascade.png")
    plt.show()
```

### Implementation 3: Complete TX-RX Chain with Pulse Shaping

```python
class PulseShapedModem:
    """Complete modem with RRC pulse shaping and matched filtering"""

    def __init__(self, modulation='QPSK', beta=0.35, span=8, sps=8):
        """
        Args:
            modulation: Modulation type
            beta: Roll-off factor
            span: Filter span
            sps: Samples per symbol
        """
        self.modulation = modulation
        self.beta = beta
        self.sps = sps

        # Create RRC filter
        self.rrc = RootRaisedCosineFilter(beta, span, sps)

        # Create modulation mapper
        if modulation == 'BPSK':
            from LAB_3_1_ASK_FSK_PSK_MODULATION import PSKModem as BPSKModem
            self.modem = BPSKModem(0, 1, sps)
        elif modulation == 'QPSK':
            from LAB_3_2_QPSK_8PSK_MODULATION import QPSKModem
            self.modem = QPSKModem(0, 1, sps)
        else:
            raise ValueError(f"Unknown modulation: {modulation}")

        print(f"\nPulse-Shaped {modulation} Modem:")
        print(f"  RRC β = {beta}")
        print(f"  Filter span: {span} symbols")

    def transmit(self, bits):
        """Transmit with pulse shaping"""
        # Map bits to symbols
        symbols = self.modem.bits_to_symbols(bits)

        # Upsample (insert zeros)
        symbols_up = np.zeros(len(symbols) * self.sps, dtype=complex)
        symbols_up[::self.sps] = symbols

        # Apply RRC filter (pulse shaping)
        tx_signal = signal.convolve(symbols_up, self.rrc.h, mode='same')

        return tx_signal

    def receive(self, rx_signal, n_bits):
        """Receive with matched filtering"""
        # Apply RRC matched filter
        mf_output = signal.convolve(rx_signal, self.rrc.h, mode='same')

        # Downsample at symbol rate
        n_symbols = len(mf_output) // self.sps
        symbols_rx = mf_output[self.sps//2::self.sps][:n_symbols]

        # Decode symbols
        symbol_indices = np.zeros(len(symbols_rx), dtype=int)
        for i, sym in enumerate(symbols_rx):
            distances = np.abs(self.modem.constellation - sym)
            symbol_indices[i] = np.argmin(distances)

        # Convert to bits
        bits_rx = self.modem.symbols_to_bits(symbol_indices, n_bits)

        return bits_rx

    def plot_spectrum_comparison(self, bits):
        """Compare spectra with and without pulse shaping"""
        # Without pulse shaping
        symbols = self.modem.bits_to_symbols(bits)
        symbols_up = np.zeros(len(symbols) * self.sps, dtype=complex)
        symbols_up[::self.sps] = symbols

        # With pulse shaping
        tx_shaped = self.transmit(bits)

        # Compute spectra
        fft_unshaped = np.fft.fftshift(np.fft.fft(symbols_up, 4096))
        fft_shaped = np.fft.fftshift(np.fft.fft(tx_shaped, 4096))

        freqs = np.fft.fftshift(np.fft.fftfreq(4096, 1/self.sps))

        spectrum_unshaped = 20 * np.log10(np.abs(fft_unshaped) + 1e-12)
        spectrum_shaped = 20 * np.log10(np.abs(fft_shaped) + 1e-12)

        # Plot
        fig, ax = plt.subplots(1, 1, figsize=(12, 7))

        ax.plot(freqs, spectrum_unshaped, 'r-', linewidth=1.5, alpha=0.7,
               label='Without pulse shaping (rectangular)')
        ax.plot(freqs, spectrum_shaped, 'b-', linewidth=2,
               label=f'With RRC pulse shaping (β={self.beta})')

        ax.set_xlabel('Normalized Frequency (×Rs)')
        ax.set_ylabel('Magnitude (dB)')
        ax.set_title(f'{self.modulation} Spectrum: Pulse Shaping Effect')
        ax.grid(True, alpha=0.3)
        ax.legend()
        ax.set_xlim([-2, 2])
        ax.set_ylim([-80, 0])

        # Mark bandwidth
        bw = (1 + self.beta) / 2
        ax.axvline(-bw, color='g', linestyle='--', alpha=0.5)
        ax.axvline(bw, color='g', linestyle='--', alpha=0.5,
                  label=f'Bandwidth: {1+self.beta:.2f}×Rs')

        plt.tight_layout()
        plt.savefig('pulse_shaping_spectrum_comparison.png', dpi=150, bbox_inches='tight')
        print("\n✓ Saved pulse_shaping_spectrum_comparison.png")
        plt.show()


# Test pulse-shaped modem
if __name__ == "__main__":
    print("\n" + "="*70)
    print("PULSE-SHAPED MODEM DEMONSTRATION")
    print("="*70)

    modem = PulseShapedModem(modulation='QPSK', beta=0.35, span=8, sps=8)

    # Generate test bits
    np.random.seed(42)
    bits = np.random.randint(0, 2, size=200)

    # Test transmission
    tx_signal = modem.transmit(bits)

    # Add noise
    snr_db = 15
    tx_power = np.mean(np.abs(tx_signal)**2)
    snr_linear = 10**(snr_db/10)
    noise_power = tx_power / snr_linear
    noise = np.sqrt(noise_power/2) * (np.random.randn(len(tx_signal)) +
                                      1j*np.random.randn(len(tx_signal)))
    rx_signal = tx_signal + noise

    # Receive
    bits_rx = modem.receive(rx_signal, len(bits))

    # Check errors
    errors = np.sum(bits != bits_rx)
    print(f"\nTransmission Test:")
    print(f"  Bits sent: {len(bits)}")
    print(f"  Errors:    {errors}")
    print(f"  BER:       {errors/len(bits):.2e}")

    # Plot spectrum comparison
    modem.plot_spectrum_comparison(bits)
```

---

## Summary

In this lab, you learned:

✅ **Inter-Symbol Interference (ISI)**:
   - Causes: bandwidth limits, multipath, timing
   - Nyquist criterion for zero ISI

✅ **Raised Cosine (RC) Filter**:
   - Practical Nyquist pulse
   - Roll-off factor β controls bandwidth
   - Zero ISI at sampling instants

✅ **Root-Raised Cosine (RRC)**:
   - Split filtering: TX and RX
   - RRC ⊗ RRC = RC
   - Matched filtering + zero ISI

✅ **Pulse Shaping Benefits**:
   - Reduced bandwidth
   - Controlled ISI
   - Better spectral efficiency
   - Reduced interference

✅ **Practical Implementation**:
   - Filter design parameters
   - Complete TX-RX chain
   - Bandwidth vs. β trade-off

---

## Next Steps

Continue to:
- **LAB 4.1**: FIR and IIR Filter Design
- **LAB 4.2**: Windowing and Spectral Leakage
- **LAB 4.3**: FFT Analysis and Spectrograms
