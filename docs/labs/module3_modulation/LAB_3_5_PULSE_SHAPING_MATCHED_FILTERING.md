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

## Method 3: Hosted Application (C Implementation)

This method implements pulse shaping and matched filtering in **production-ready C code** for the PlutoSDR's ARM Cortex-A9 processor, providing real-time performance and deep understanding of the complete transmit-receive filter chain.

---

### **Part 3: Theory Deep Dive** 📚

This section provides comprehensive mathematical foundations and practical insights into pulse shaping, matched filtering, and timing recovery for digital communication systems.

---

#### **3.1 Mathematical Foundation of Pulse Shaping**

**Why Pulse Shaping?**

Rectangular pulses used in basic modulation have infinite bandwidth:

```
Rectangular pulse:
  p(t) = { 1,  0 ≤ t < T
         { 0,  otherwise

Fourier Transform:
  P(f) = T · sinc(πfT) = T · sin(πfT)/(πfT)

Bandwidth: Infinite (sinc has slow 1/f decay)
Power Spectral Density: |P(f)|² ∝ sinc²(πfT)

Problem:
  - 99% of power in BW ≈ 20/T Hz (very wide!)
  - Adjacent channel interference
  - Regulatory violations (FCC/ETSI limits)
```

**Nyquist's Solution**: Use pulses that satisfy:

```
p(kT) = { 1,  k = 0
        { 0,  k = ±1, ±2, ...

In frequency domain (Nyquist Criterion):
  ∑[n=-∞→∞] P(f + n/T) = T,  for |f| ≤ 1/(2T)

This ensures zero ISI at symbol-spaced sampling instants!
```

---

#### **3.2 Raised Cosine (RC) Filter**

**Time Domain**:

```
         sin(πt/T)        cos(βπt/T)
p(t) = ------------ · -------------------
          πt/T       1 - (2βt/T)²

where:
  T = symbol period
  β = roll-off factor (0 ≤ β ≤ 1)
```

**Frequency Domain**:

```
         ⎧ T,                            |f| ≤ (1-β)/(2T)
         ⎪
P(f) =   ⎨ T/2 · [1 + cos(πT/(β)·       (1-β)/(2T) < |f| ≤ (1+β)/(2T)
         ⎪      (|f| - (1-β)/(2T)))]
         ⎪
         ⎩ 0,                            |f| > (1+β)/(2T)

Bandwidth: BW = (1 + β)/(2T) Hz

Roll-off factor β controls trade-off:
  β = 0:  BW = 1/(2T)  (minimum, but infinite time extent)
  β = 0.5: BW = 0.75/T  (balanced)
  β = 1:  BW = 1/T     (wide, but compact in time)
```

**Key Properties**:

1. **Zero ISI**: p(kT) = δ(k) for all β
2. **Finite bandwidth**: BW = (1+β)/(2T) Hz
3. **Smooth spectrum**: No discontinuities
4. **Time decay**: p(t) ∝ 1/t³ for t → ∞ (much faster than sinc's 1/t)

**Roll-off Factor Trade-offs**:

| β | Bandwidth | Time Decay | Sensitivity to Timing | Use Case |
|---|-----------|------------|----------------------|----------|
| 0.0 | 1/(2T) (min) | Slow (1/t) | Very high | Theoretical only |
| 0.25 | 0.625/T | Medium | Moderate | Satellite, DSL |
| 0.35 | 0.675/T | Medium-fast | Moderate | LTE, WiFi |
| 0.5 | 0.75/T | Fast | Low | General-purpose |
| 1.0 | 1/T (max) | Very fast (1/t³) | Very low | Low SNR channels |

---

#### **3.3 Root-Raised Cosine (RRC) Filter**

**Problem with RC**: If we use RC at TX only, receiver has no matched filter.

**Solution**: Split RC between TX and RRC:

```
RC(f) = RRC_TX(f) · RRC_RX(f)

where:
  RRC_TX(f) = √(RC(f))
  RRC_RX(f) = √(RC(f))

Convolution in time:
  p_RC(t) = p_RRC(t) ⊗ p_RRC(t)
```

**RRC Time Domain** (no closed form, must compute numerically):

```
              sin(π(1-β)t/T) + (4βt/T)·cos(π(1+β)t/T)
p_RRC(t) = -----------------------------------------------
                    (πt/T) · (1 - (4βt/T)²)

Special cases (handle singularities):

At t = 0:
  p_RRC(0) = (1 + β(4/π - 1))

At t = ±T/(4β):
  p_RRC(±T/(4β)) = (β/√2) · [(1+2/π)sin(π/(4β)) + (1-2/π)cos(π/(4β))]
```

**Why RRC is Optimal**:

1. **Matched filtering**: RRC_RX is matched to RRC_TX → maximizes SNR
2. **Zero ISI**: Combined response is RC → zero ISI at sampling instants
3. **Bandwidth efficiency**: Uses minimum bandwidth for given β
4. **Practical**: Finite time extent (truncate at ±6T with <1% error)

**Matched Filter Theorem** (North, 1943):

```
For signal s(t) in AWGN with PSD N₀/2:

Optimal filter: h(t) = s*(T - t)  (time-reversed conjugate)

SNR at output: SNR_out = (2·E_s) / N₀

where E_s = ∫|s(t)|² dt is signal energy

Proof sketch:
  - Output y(T) = ∫s(τ)h(T-τ)dτ + noise
  - SNR = |y(T)|² / σ²
  - Cauchy-Schwarz inequality → maximum when h(t) ∝ s*(T-t)
```

---

#### **3.4 Practical RRC Filter Design**

**Sampling Requirements**:

```
RRC filter taps: h[n] = p_RRC(n·T_s)

where:
  T_s = sample period
  T = symbol period
  sps = T/T_s = samples per symbol (oversampling factor)

Typical values:
  sps = 4:  Good for most systems
  sps = 8:  Better timing recovery
  sps = 2:  Minimum (Nyquist rate)
```

**Filter Length**:

```
Time span: [-span·T, +span·T]

Number of taps: N = 2·span·sps + 1

Typical values:
  span = 6:  61 dB sidelobe suppression
  span = 8:  73 dB sidelobe suppression
  span = 10: 82 dB sidelobe suppression

Trade-off:
  - Larger span: Better spectral purity, more delay
  - Smaller span: Less delay, worse spectrum
```

**Truncation Effects**:

```
Truncating RRC to finite length causes:

1. Spectral ripple:
   Δ|P(f)| ≈ exp(-2π·span)

   span = 6:  ripple ≈ -40 dB
   span = 8:  ripple ≈ -50 dB

2. Time-domain ringing:
   Gibbs phenomenon at truncation points

3. ISI residue:
   Small non-zero values at p(kT) for k ≠ 0

   Magnitude: ISI ≈ 1/span² (typically < -30 dB for span=6)

Solution: Window the filter (Hamming, Kaiser) to reduce ripple
```

**Windowing**:

```
Hamming window:
  w[n] = 0.54 - 0.46·cos(2πn/(N-1))

Kaiser window (adjustable):
  w[n] = I₀(β·√(1-(2n/(N-1)-1)²)) / I₀(β)

  where I₀ is modified Bessel function of first kind

  β controls trade-off:
    β = 5:  moderate sidelobe suppression (-50 dB)
    β = 8:  high suppression (-70 dB)

Windowed RRC:
  h_win[n] = h_RRC[n] · w[n]

Trade-off:
  ✅ Reduced spectral ripple
  ❌ Slightly wider main lobe
  ❌ Small ISI increase
```

---

#### **3.5 Complete Transmitter Chain**

**Block Diagram**:

```
       Upsampling      RRC Filter      Modulation
Bits ──────────> ───────────────> ────────────> RF
       (×sps)      (pulse shape)   (I/Q mixer)

Step 1: Upsample
  - Insert (sps-1) zeros between symbols
  - Symbol stream: [s₀, s₁, s₂, ...]
  - Upsampled: [s₀, 0, 0, 0, s₁, 0, 0, 0, s₂, ...]

Step 2: RRC Filter
  - Convolve upsampled signal with RRC
  - Smooths transitions between symbols
  - Limits bandwidth to (1+β)/(2T)

Step 3: Modulation (already complex baseband)
  - For QPSK: symbols are already ±1±j
  - Just scale to desired power and upconvert to RF
```

**Mathematical Expression**:

```
Symbol sequence: s[k], k = 0, 1, 2, ...

Upsampled: x[n] = s[n/sps]  if n is multiple of sps
                 = 0         otherwise

Pulse-shaped: y[n] = ∑[k] s[k] · h_RRC[n - k·sps]

Continuous-time: s(t) = ∑[k] s[k] · p_RRC(t - kT)
```

**Energy Normalization**:

```
Goal: Ensure E[|s(t)|²] = 1 (unit average power)

RRC filter energy: E_h = ∑[n] |h[n]|²

Normalization: h_norm[n] = h[n] / √(E_h · sps)

Why "sps" factor?
  - Upsampling by sps reduces power by sps
  - Need to compensate to maintain unit power

Verification:
  E[|y[n]|²] = (E[|s[k]|²] / sps) · E_h
             = 1  (if E[|s[k]|²] = 1 and normalized correctly)
```

---

#### **3.6 Complete Receiver Chain**

**Block Diagram**:

```
       Demodulation    Matched Filter   Symbol Sync   Decimation
RF ───────────────> ───────────────> ─────────────> ──────────> Symbols
      (I/Q mixer)      (RRC RX)       (timing recov)  (×1/sps)

Step 1: Demodulation
  - Downconvert RF to baseband
  - I/Q demodulation

Step 2: Matched Filter (RRC_RX)
  - Same as TX: h_RRC[n]
  - Maximizes SNR
  - Combined TX+RX gives RC response

Step 3: Symbol Synchronization
  - Find optimal sampling instant
  - Compensate for timing offset

Step 4: Decimation
  - Sample at symbol rate (every sps samples)
  - Discard intermediate samples
```

**Timing Offset Problem**:

```
TX clock:  |―T―|―T―|―T―|  (symbol period T)
RX clock:  |―T―|―T―|―T―|  (same T, but phase offset τ)

Sampling phase: τ ∈ [0, T)

Eye diagram shows why timing matters:
  - Center of eye: τ = T/2 (optimal, maximum SNR)
  - Edge of eye:   τ = 0 or T (worst, maximum ISI)

Timing error causes:
  1. ISI: sampling at non-zero crossings of p(t)
  2. SNR loss: suboptimal matched filter output
  3. BER degradation: both effects combine
```

**Timing Recovery Algorithms**:

**1. Gardner Algorithm** (popular, NDA = Non-Data-Aided):

```
Timing Error Detector (TED):
  e[k] = real(y[k]) · (real(y[k+1]) - real(y[k-1]))

  where:
    y[k] = matched filter output at time k·T_s
    e[k] > 0: sampling too early
    e[k] < 0: sampling too late
    e[k] ≈ 0: optimal timing

Loop Filter (PI controller):
  τ[k+1] = τ[k] + K_p·e[k] + K_i·∑e[k]

  where:
    K_p: proportional gain (fast response)
    K_i: integral gain (removes steady-state error)

Typical gains:
  K_p = 0.01 to 0.1
  K_i = K_p² / 4 (for critical damping)
```

**2. Mueller & Müller Algorithm** (DD = Decision-Directed):

```
TED:
  e[k] = real(y[k]) · real(ŝ[k-1]) - real(y[k-1]) · real(ŝ[k])

  where:
    ŝ[k] = decision on symbol k (±1 for BPSK, ±1±j for QPSK)

Advantage: Works at symbol rate (no interpolation needed)
Disadvantage: Requires accurate decisions (high SNR)
```

**3. Early-Late Gate** (simple, popular in hardware):

```
TED:
  e[k] = |y[k+Δ]|² - |y[k-Δ]|²

  where:
    Δ = T/(2·sps) (half symbol period in samples)
    y[k±Δ] = samples straddling optimal point

Advantage: Very simple, no multiplications
Disadvantage: Requires high SNR
```

---

#### **3.7 Timing Recovery Mathematics**

**Phase-Locked Loop (PLL) Model**:

```
           ┌─────┐      ┌────────┐      ┌─────┐
y[n] ──>│ TED │──> e[k] │ Loop   │──> τ[k] │ NCO │──> sample points
           └─────┘      │ Filter │      └─────┘
                         └────────┘         │
                               ▲            │
                               └────────────┘

TED: Timing Error Detector
NCO: Numerically Controlled Oscillator (generates sampling instants)

Loop dynamics:
  τ[k+1] = τ[k] + μ·e[k]  (first-order)

  or

  τ[k+1] = τ[k] + μ₁·e[k] + μ₂·∑e[k]  (second-order)

Natural frequency: ω_n = √(μ₁·μ₂)
Damping ratio: ζ = (μ₁ + μ₂) / (2·√(μ₁·μ₂))

Critical damping: ζ = 1  →  μ₂ = μ₁²/4
```

**Timing Jitter Analysis**:

```
Residual timing error: σ_τ²

For Gardner TED at high SNR:
  σ_τ² ≈ (1 / (4·SNR)) · (1 / N_avg)

  where:
    SNR = symbol SNR
    N_avg = averaging length of loop filter

BER degradation due to timing jitter:
  BER_timing ≈ BER_ideal · (1 + (2π·σ_τ/T)²·SNR)

Example:
  σ_τ/T = 1%  (1% of symbol period)
  SNR = 15 dB
  → BER degradation ≈ 0.5 dB (acceptable)
```

**Fractional Delay Implementation**:

```
Problem: Optimal sampling instant is not at integer sample index

Solution 1: Polyphase filterbank
  - Pre-compute M fractional-delayed versions of matched filter
  - Select appropriate filter based on τ[k]
  - M = 32 typical (resolution: T/32)

Solution 2: Lagrange interpolation
  - Interpolate between samples
  - Order 3 (cubic) gives good accuracy

  y(t) = ∑[k=0 to 3] y[k] · L_k(t)

  where L_k(t) are Lagrange basis polynomials

Solution 3: Farrow structure
  - Efficient implementation of Lagrange
  - Reuses computations
  - Lower complexity than polyphase
```

---

#### **3.8 Performance Metrics**

**1. Output SNR (after matched filtering)**:

```
Input SNR (before RX filter): SNR_in
Output SNR (after RX filter):

  SNR_out = SNR_in · (E_s / N₀) · T_s

where:
  E_s = symbol energy
  N₀ = noise PSD

Matched filter gain:
  G_MF = ∑[n] |h[n]|² = E_h

For normalized RRC: G_MF = 1

Processing gain from matched filter:
  G_proc = 10·log₁₀(sps) dB

  sps = 4:  G_proc = 6 dB
  sps = 8:  G_proc = 9 dB
```

**2. ISI Measurement**:

```
After TX RRC + RX RRC → total response is RC

ISI at sampling instants:
  ISI_k = p_RC(kT) / p_RC(0),  k ≠ 0

For ideal RC with β = 0.35:
  ISI_1 ≈ -40 dB  (first neighbor)
  ISI_2 ≈ -60 dB  (second neighbor)

Truncation to span=6 adds:
  ISI_residual ≈ -30 dB

Total ISI: ISI_total = 10·log₁₀(∑[k≠0] ISI_k²)

  Typical: ISI_total ≈ -25 to -35 dB (negligible)
```

**3. Spectral Occupancy**:

```
99% power bandwidth:
  BW_99 ≈ (1 + β) / T · k

  where k ≈ 1.1 to 1.2 (depends on truncation)

Out-of-band rejection (OOBR):
  OOBR = 10·log₁₀(P_out / P_total)

  where:
    P_out = power outside allocated band
    P_total = total power

For RRC with span=6:
  OOBR ≈ -40 to -45 dB (good)

For RRC with span=10:
  OOBR ≈ -50 to -55 dB (excellent)
```

**4. Timing Recovery Performance**:

```
Acquisition time: t_acq ≈ (3 to 5) / (μ·R_s)

  where:
    μ = loop gain
    R_s = symbol rate

Tracking range: Δf_max ≈ μ·R_s / (2π)

Jitter bandwidth: BW_jitter ≈ μ·R_s / (2π)

Example:
  R_s = 1 Msps
  μ = 0.01
  → t_acq ≈ 3-5 ms
  → Δf_max ≈ 1.6 kHz
  → BW_jitter ≈ 1.6 kHz
```

---

#### **3.9 PlutoSDR-Specific Considerations**

**1. AD9361 Digital Filters**:

```
AD9361 has built-in FIR filters:

TX path:
  - TFIR (Transmit FIR): 128 taps
  - Can implement custom RRC
  - Decimation by 1, 2, or 4

RX path:
  - RFIR (Receive FIR): 128 taps
  - Can implement custom RRC
  - Interpolation by 1, 2, or 4

Advantage: Offload filtering to FPGA (zero CPU cost)
Disadvantage: Fixed coefficients until reconfigured

Access via libiio:
  iio_channel_attr_write_longlong(chn, "filter_fir_en", 1);
  iio_device_attr_write_raw(dev, "filter_fir_config", config, len);
```

**2. Sample Rate Constraints**:

```
AD9361 RX/TX sample rate: 2.5 MHz to 61.44 MHz

For symbol rate R_s with sps oversampling:
  F_s = R_s · sps

  → R_s_max = 61.44 MHz / sps

  sps = 4:  R_s_max = 15.36 Msps
  sps = 8:  R_s_max = 7.68 Msps

Recommended for low CPU load:
  R_s ≤ 1 Msps  (allows processing on ARM)
```

**3. Memory Constraints**:

```
PlutoSDR available RAM: ~300 MB

Filter taps storage:
  span = 10, sps = 4:  N = 81 taps × 8 bytes = 648 bytes (negligible)

Buffer storage (critical):
  1 second @ 1 Msps: 1M samples × 8 bytes (complex float) = 8 MB

  Circular buffer recommended: 100 ms max (~800 KB)
```

**4. Fixed-Point vs Floating-Point**:

```
ARM Cortex-A9 has NEON SIMD:
  - 32-bit float: 4 operations/cycle (using NEON)
  - 16-bit fixed: 8 operations/cycle

Recommendation:
  - Use float for ease of development
  - NEON optimization gives 4× speedup automatically
  - Fixed-point only if needed for extreme performance

Precision requirements:
  - RRC coefficients: 16-bit sufficient (SQNR > 90 dB)
  - Samples: 12-bit (AD9361 native) or 16-bit
  - Timing error: 32-bit float (avoid quantization)
```

---

#### **3.10 Design Trade-offs Summary**

| Parameter | Low Value | High Value | Sweet Spot |
|-----------|-----------|------------|------------|
| **Roll-off β** | Narrow BW, slow decay | Wide BW, fast decay | β = 0.35 |
| **Span** | Low latency, more ISI | High latency, less ISI | span = 6-8 |
| **sps** | Low sampling rate | Better timing recovery | sps = 4 |
| **Loop gain μ** | Slow tracking, low jitter | Fast tracking, high jitter | μ = 0.01-0.05 |
| **Filter length** | Low CPU, more ripple | High CPU, clean spectrum | N = 49-65 |

**Typical Configuration for PlutoSDR**:

```
Symbol rate: R_s = 500 ksps
Oversampling: sps = 4
Sample rate: F_s = 2 Msps
Roll-off: β = 0.35
Span: span = 6
Filter taps: N = 2·6·4 + 1 = 49
Timing loop gain: μ = 0.02
```

**Performance Expectations**:

```
BER (QPSK at 15 dB SNR):
  - Ideal: 5.3×10⁻⁶
  - With RRC filtering: 6.1×10⁻⁶ (0.6 dB loss)
  - With timing recovery: 7.5×10⁻⁶ (0.5 dB additional loss)
  - Total degradation: ~1.1 dB (acceptable)

Spectrum:
  - 99% power BW: 675 kHz (= 0.675·R_s for β=0.35)
  - OOBR @ ±1 MHz: -45 dB

CPU load (ARM Cortex-A9 @ 667 MHz):
  - RRC filtering: ~5%
  - Timing recovery: ~2%
  - Total: ~7% (comfortable margin)
```

---

### **Summary of Part 3**

You now understand:

✅ **Mathematical foundations**: Nyquist criterion, RC, and RRC filters
✅ **Filter design**: Length, truncation, windowing, and normalization
✅ **Transmit chain**: Upsampling, pulse shaping, and energy normalization
✅ **Receive chain**: Matched filtering, timing recovery, and decimation
✅ **Timing algorithms**: Gardner, Mueller & Müller, Early-Late Gate
✅ **Performance metrics**: SNR, ISI, spectral occupancy, timing jitter
✅ **PlutoSDR specifics**: AD9361 constraints, CPU/memory trade-offs

**Next**: Part 4 will provide complete production-ready C implementation!

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
