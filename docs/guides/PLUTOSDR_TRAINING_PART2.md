# PlutoSDR Professional Training Course - Part 2

## Advanced Concepts and Applications

---

# 3. Modulation & Waveform Terms

## 3.1 BPSK (Binary Phase Shift Keying)

### Theory
**BPSK** encodes data by shifting carrier phase between two states:
- Binary 0: Phase = 0°
- Binary 1: Phase = 180°

**Constellation:**
```
     Q
     │
  1  │  0
─────┼─────► I
     │
```

**Advantages:**
- Simplest digital modulation
- Maximum distance between symbols (robust)
- Poor spectral efficiency (1 bit/symbol)

### LAB 3.1: BPSK Transmitter and Receiver
**Objective:** Implement complete BPSK modem

**Code:**
```python
#!/usr/bin/env python3
"""
LAB 3.1: BPSK Modulation and Demodulation
"""

import adi
import numpy as np
import matplotlib.pyplot as plt
from scipy import signal

# Parameters
fs = 2.084e6  # Sample rate
symbol_rate = 10e3  # 10 kbaud
samples_per_symbol = int(fs / symbol_rate)
num_symbols = 1000

# Generate random data
data_bits = np.random.randint(0, 2, num_symbols)
print(f"Transmitting {num_symbols} bits: {data_bits[:20]}...")

# BPSK Modulator
def bpsk_modulate(bits, sps):
    """
    BPSK modulation: 0 → -1, 1 → +1
    """
    # Map bits to symbols
    symbols = 2*bits - 1  # 0→-1, 1→+1

    # Upsample (insert zeros)
    upsampled = np.zeros(len(symbols) * sps, dtype=complex)
    upsampled[::sps] = symbols

    # Pulse shaping (Root Raised Cosine)
    # Design RRC filter
    alpha = 0.35  # Roll-off factor
    span = 10  # Filter span in symbols
    rrc_taps = signal.firwin(span*sps, 1/sps, window=('kaiser', 5))
    # Normalize
    rrc_taps = rrc_taps / np.sqrt(np.sum(rrc_taps**2))

    # Filter
    tx_signal = signal.lfilter(rrc_taps, 1, upsampled)

    return tx_signal, rrc_taps

# Modulate
tx_signal, rrc_filter = bpsk_modulate(data_bits, samples_per_symbol)

# Scale for DAC
tx_signal_scaled = tx_signal / np.max(np.abs(tx_signal)) * 0.8
tx_signal_int = (tx_signal_scaled * 2**14).astype(np.int16)

# Transmit via PlutoSDR
sdr = adi.Pluto("ip:192.168.2.1")
sdr.tx_lo = 915e6
sdr.rx_lo = 915e6
sdr.sample_rate = int(fs)
sdr.tx_hardwaregain_chan0 = -20
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 50
sdr.tx_cyclic_buffer = True

# Start TX
sdr.tx(tx_signal_int)
print("Transmitting...")

# Receive
sdr.rx_buffer_size = len(tx_signal_int)
import time
time.sleep(0.5)  # Let TX stabilize

rx_signal = sdr.rx()

# BPSK Demodulator
def bpsk_demodulate(rx_signal, rrc_filter, sps):
    """
    BPSK demodulation with matched filtering and timing recovery
    """
    # Matched filter (same as TX filter)
    mf_out = signal.lfilter(rrc_filter, 1, rx_signal)

    # Simple timing recovery: find peak correlation with preamble
    # (In practice, use Mueller & Muller or Gardner)
    # Here we'll just downsample at correct timing
    # Assume synchronization (skip transient)
    start_idx = len(rrc_filter) // 2

    # Sample at symbol rate
    symbols_rx = mf_out[start_idx::sps]

    # Decision: Re(symbol) > 0 → 1, else → 0
    bits_rx = (np.real(symbols_rx) > 0).astype(int)

    return bits_rx, symbols_rx

# Demodulate
bits_rx, symbols_rx = bpsk_demodulate(rx_signal, rrc_filter, samples_per_symbol)

# Align and compare (find sync)
# Cross-correlate to find alignment
correlation = np.correlate(bits_rx, data_bits, mode='valid')
max_corr_idx = np.argmax(correlation)
bits_aligned = bits_rx[max_corr_idx:max_corr_idx+len(data_bits)]

# Calculate BER
errors = np.sum(bits_aligned != data_bits)
ber = errors / len(data_bits)

print(f"\nReceived {len(bits_rx)} symbols")
print(f"Bit Error Rate: {ber:.6f} ({errors}/{len(data_bits)} errors)")
print(f"RX bits: {bits_aligned[:20]}...")

# Plot
fig, axes = plt.subplots(3, 2, figsize=(14, 10))

# TX time domain
axes[0, 0].plot(np.real(tx_signal[:1000]))
axes[0, 0].set_title('TX Signal (Real part)')
axes[0, 0].set_xlabel('Sample')
axes[0, 0].set_ylabel('Amplitude')
axes[0, 0].grid(True)

# TX spectrum
fft_tx = np.fft.fftshift(np.fft.fft(tx_signal))
freqs = np.fft.fftshift(np.fft.fftfreq(len(tx_signal), 1/fs))
axes[0, 1].plot(freqs/1e3, 20*np.log10(np.abs(fft_tx)))
axes[0, 1].set_title('TX Spectrum')
axes[0, 1].set_xlabel('Frequency (kHz)')
axes[0, 1].set_ylabel('Power (dB)')
axes[0, 1].grid(True)

# RX time domain
axes[1, 0].plot(np.real(rx_signal[:1000]))
axes[1, 0].set_title('RX Signal (Real part)')
axes[1, 0].set_xlabel('Sample')
axes[1, 0].set_ylabel('Amplitude')
axes[1, 0].grid(True)

# RX spectrum
fft_rx = np.fft.fftshift(np.fft.fft(rx_signal))
axes[1, 1].plot(freqs/1e3, 20*np.log10(np.abs(fft_rx)))
axes[1, 1].set_title('RX Spectrum')
axes[1, 1].set_xlabel('Frequency (kHz)')
axes[1, 1].set_ylabel('Power (dB)')
axes[1, 1].grid(True)

# Constellation
axes[2, 0].plot(np.real(symbols_rx[:500]), np.imag(symbols_rx[:500]), '.', alpha=0.5)
axes[2, 0].set_title(f'RX Constellation (BER={ber:.6f})')
axes[2, 0].set_xlabel('I')
axes[2, 0].set_ylabel('Q')
axes[2, 0].axis('equal')
axes[2, 0].grid(True)
axes[2, 0].axhline(0, color='k', linewidth=0.5)
axes[2, 0].axvline(0, color='k', linewidth=0.5)

# Eye diagram
eye_samples = 2 * samples_per_symbol
num_traces = 50
axes[2, 1].set_title('Eye Diagram')
for i in range(num_traces):
    start = i * samples_per_symbol + len(rrc_filter)//2
    if start + eye_samples < len(rx_signal):
        axes[2, 1].plot(np.real(rx_signal[start:start+eye_samples]), alpha=0.3, color='blue')
axes[2, 1].set_xlabel('Sample within 2 symbols')
axes[2, 1].set_ylabel('Amplitude')
axes[2, 1].grid(True)

plt.tight_layout()
plt.savefig('lab3_1_bpsk.png')
plt.show()

sdr.tx_destroy_buffer()

print("\n✓ LAB COMPLETE: BPSK modem implemented")
print("Key points:")
print("  - Constellation has 2 points (180° apart)")
print("  - Eye diagram shows symbol timing quality")
print("  - RRC pulse shaping limits bandwidth")
```

**Expected Results:**
- Constellation shows two distinct clouds at I = ±1, Q = 0
- Eye diagram opens at sampling instant
- BER should be very low (<10⁻³) with good SNR
- **Conclusion:** BPSK provides robust 1 bit/symbol transmission

---

## 3.2 QPSK (Quadrature Phase Shift Keying)

### Theory
**QPSK** encodes 2 bits per symbol using 4 phase states:
- 00: 45° (1+j)
- 01: 135° (-1+j)
- 10: -45° (1-j)
- 11: -135° (-1-j)

**Constellation:**
```
      Q
      │
  01  │  00
──────┼──────► I
  11  │  10
      │
```

**Advantages:**
- 2 bits/symbol (double BPSK efficiency)
- Same error performance as BPSK (with same energy/bit)
- Widely used (WiFi, satellite, LTE)

### LAB 3.2: QPSK Modem
**Objective:** Implement QPSK with Gray coding

**Code:**
```python
#!/usr/bin/env python3
"""
LAB 3.2: QPSK Modulation with Gray Coding
"""

import adi
import numpy as np
import matplotlib.pyplot as plt

# Parameters
fs = 2.084e6
symbol_rate = 20e3  # 20 kbaud
samples_per_symbol = int(fs / symbol_rate)

# Generate random data
num_bits = 2000  # Must be even for QPSK
data_bits = np.random.randint(0, 2, num_bits)

# QPSK Modulator with Gray coding
def qpsk_modulate(bits, sps):
    """
    QPSK with Gray coding:
    00 → +1+1j (45°)
    01 → -1+1j (135°)
    11 → -1-1j (225°)
    10 → +1-1j (315°)
    """
    # Group bits into pairs
    bit_pairs = bits.reshape(-1, 2)

    # Gray coding constellation map
    gray_map = {
        (0, 0): (1+1j),
        (0, 1): (-1+1j),
        (1, 1): (-1-1j),
        (1, 0): (1-1j)
    }

    # Map to symbols
    symbols = np.array([gray_map[tuple(pair)] for pair in bit_pairs])

    # Normalize
    symbols = symbols / np.sqrt(2)

    # Upsample
    upsampled = np.zeros(len(symbols) * sps, dtype=complex)
    upsampled[::sps] = symbols

    # Pulse shaping (simple rectangular for this demo)
    # In practice, use RRC
    tx_signal = np.repeat(symbols, sps)

    return tx_signal, symbols

tx_signal, tx_symbols = qpsk_modulate(data_bits, samples_per_symbol)

# Transmit
sdr = adi.Pluto("ip:192.168.2.1")
sdr.tx_lo = 915e6
sdr.rx_lo = 915e6
sdr.sample_rate = int(fs)
sdr.tx_hardwaregain_chan0 = -10
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 50
sdr.tx_cyclic_buffer = True

# Scale and transmit
tx_scaled = (tx_signal * 0.8 * 2**14).astype(np.int16)
sdr.tx(tx_scaled)

# Receive
import time
time.sleep(0.5)
sdr.rx_buffer_size = len(tx_scaled)
rx_signal = sdr.rx()

# QPSK Demodulator
def qpsk_demodulate(rx_signal, sps):
    """
    QPSK demodulation with hard decision
    """
    # Downsample (simple method)
    rx_symbols = rx_signal[::sps]

    # Normalize
    rx_symbols = rx_symbols / np.mean(np.abs(rx_symbols)) * (1/np.sqrt(2))

    # Hard decision with Gray coding
    bits_rx = []
    for sym in rx_symbols:
        i_bit = 1 if np.real(sym) < 0 else 0
        q_bit = 0 if np.imag(sym) > 0 else 1
        bits_rx.extend([i_bit ^ q_bit, q_bit])  # Gray decode

    return np.array(bits_rx), rx_symbols

bits_rx, rx_symbols = qpsk_demodulate(rx_signal, samples_per_symbol)

# Calculate BER
# Find alignment
correlation = np.correlate(bits_rx, data_bits, mode='valid')
max_idx = np.argmax(correlation)
bits_aligned = bits_rx[max_idx:max_idx+len(data_bits)]

errors = np.sum(bits_aligned != data_bits)
ber = errors / len(data_bits)

print(f"QPSK BER: {ber:.6f} ({errors} errors)")

# Plot
fig, axes = plt.subplots(2, 2, figsize=(12, 10))

# TX Constellation
axes[0, 0].plot(np.real(tx_symbols), np.imag(tx_symbols), 'o', markersize=10, alpha=0.5)
axes[0, 0].set_title('TX Constellation (Ideal QPSK)')
axes[0, 0].set_xlabel('I')
axes[0, 0].set_ylabel('Q')
axes[0, 0].axis('equal')
axes[0, 0].grid(True)
axes[0, 0].axhline(0, color='k', linewidth=0.5)
axes[0, 0].axvline(0, color='k', linewidth=0.5)

# Add Gray code labels
labels = ['00', '01', '11', '10']
positions = [(1, 1), (-1, 1), (-1, -1), (1, -1)]
for label, pos in zip(labels, positions):
    axes[0, 0].text(pos[0]*1.2/np.sqrt(2), pos[1]*1.2/np.sqrt(2), label,
                    ha='center', va='center', fontsize=12, color='red')

# RX Constellation
axes[0, 1].plot(np.real(rx_symbols), np.imag(rx_symbols), '.', alpha=0.3)
axes[0, 1].set_title(f'RX Constellation (BER={ber:.6f})')
axes[0, 1].set_xlabel('I')
axes[0, 1].set_ylabel('Q')
axes[0, 1].axis('equal')
axes[0, 1].grid(True)
axes[0, 1].axhline(0, color='k', linewidth=0.5)
axes[0, 1].axvline(0, color='k', linewidth=0.5)

# Decision boundaries
axes[0, 1].axhline(0, color='r', linestyle='--', alpha=0.5)
axes[0, 1].axvline(0, color='r', linestyle='--', alpha=0.5)

# Spectrum
fft_tx = np.fft.fftshift(np.fft.fft(tx_signal))
freqs = np.fft.fftshift(np.fft.fftfreq(len(tx_signal), 1/fs))
axes[1, 0].plot(freqs/1e3, 20*np.log10(np.abs(fft_tx)))
axes[1, 0].set_title('TX Spectrum')
axes[1, 0].set_xlabel('Frequency (kHz)')
axes[1, 0].set_ylabel('Power (dB)')
axes[1, 0].grid(True)

# Error vector magnitude
evm = rx_symbols[:len(tx_symbols)] - tx_symbols
evm_magnitude = np.abs(evm)
axes[1, 1].hist(evm_magnitude, bins=50)
axes[1, 1].set_title(f'Error Vector Magnitude (EVM)')
axes[1, 1].set_xlabel('EVM')
axes[1, 1].set_ylabel('Count')
axes[1, 1].grid(True)

mean_evm = np.mean(evm_magnitude)
axes[1, 1].axvline(mean_evm, color='r', linestyle='--', label=f'Mean: {mean_evm:.3f}')
axes[1, 1].legend()

plt.tight_layout()
plt.savefig('lab3_2_qpsk.png')
plt.show()

sdr.tx_destroy_buffer()

print("\n✓ LAB COMPLETE: QPSK demonstrated")
print(f"  - 2 bits/symbol (vs 1 for BPSK)")
print(f"  - Gray coding minimizes bit errors")
print(f"  - Symbol rate: {symbol_rate/1e3:.0f} kbaud")
print(f"  - Bit rate: {2*symbol_rate/1e3:.0f} kbps")
```

**Expected Results:**
- 4 constellation points at 45°, 135°, 225°, 315°
- Gray coding ensures adjacent symbols differ by 1 bit
- Double the data rate of BPSK for same bandwidth
- **Conclusion:** QPSK doubles spectral efficiency

---

## 3.3 QAM (Quadrature Amplitude Modulation)

### Theory
**QAM** varies both amplitude and phase:
- 16-QAM: 16 symbols = 4 bits/symbol
- 64-QAM: 64 symbols = 6 bits/symbol
- 256-QAM: 256 symbols = 8 bits/symbol

**Trade-off:**
- Higher QAM = more bits/symbol (better efficiency)
- But requires higher SNR (less robust)

### LAB 3.3: 16-QAM Implementation
**Objective:** Implement and visualize 16-QAM

**Code:**
```python
#!/usr/bin/env python3
"""
LAB 3.3: 16-QAM Modulation
"""

import numpy as np
import matplotlib.pyplot as plt
import adi

# 16-QAM constellation (Gray-coded)
qam16_constellation = np.array([
    -3-3j, -3-1j, -3+3j, -3+1j,  # 0000, 0001, 0010, 0011
    -1-3j, -1-1j, -1+3j, -1+1j,  # 0100, 0101, 0110, 0111
    +3-3j, +3-1j, +3+3j, +3+1j,  # 1000, 1001, 1010, 1011
    +1-3j, +1-1j, +1+3j, +1+1j,  # 1100, 1101, 1110, 1111
]) / np.sqrt(10)  # Normalize to average power = 1

# Generate random data
num_symbols = 500
symbol_indices = np.random.randint(0, 16, num_symbols)
tx_symbols = qam16_constellation[symbol_indices]

# Upsample for transmission
fs = 2.084e6
symbol_rate = 50e3
sps = int(fs / symbol_rate)
tx_signal = np.repeat(tx_symbols, sps)

# Transmit via PlutoSDR
sdr = adi.Pluto("ip:192.168.2.1")
sdr.sample_rate = int(fs)
sdr.tx_lo = 915e6
sdr.rx_lo = 915e6
sdr.tx_hardwaregain_chan0 = -10
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 60
sdr.tx_cyclic_buffer = True

tx_scaled = (tx_signal * 0.7 * 2**14).astype(np.int16)
sdr.tx(tx_scaled)

import time
time.sleep(0.5)

sdr.rx_buffer_size = len(tx_scaled)
rx_signal = sdr.rx()

# Demodulate
rx_symbols = rx_signal[::sps]
rx_symbols_norm = rx_symbols / np.mean(np.abs(rx_symbols)) * np.sqrt(10)

# Hard decision (find nearest constellation point)
def qam16_demod(rx_syms):
    detected = []
    for sym in rx_syms:
        distances = np.abs(qam16_constellation - sym)
        detected.append(np.argmin(distances))
    return np.array(detected)

detected_indices = qam16_demod(rx_symbols_norm)

# Calculate SER (Symbol Error Rate)
# Align sequences first
min_len = min(len(detected_indices), len(symbol_indices))
errors = np.sum(detected_indices[:min_len] != symbol_indices[:min_len])
ser = errors / min_len

print(f"16-QAM Symbol Error Rate: {ser:.6f} ({errors}/{min_len} errors)")

# Plot
fig, axes = plt.subplots(2, 2, figsize=(12, 10))

# Ideal constellation
axes[0, 0].plot(np.real(qam16_constellation), np.imag(qam16_constellation),
                'ro', markersize=12, label='Ideal')
axes[0, 0].set_title('Ideal 16-QAM Constellation')
axes[0, 0].set_xlabel('I')
axes[0, 0].set_ylabel('Q')
axes[0, 0].axis('equal')
axes[0, 0].grid(True)
axes[0, 0].legend()

# Add labels
for i, point in enumerate(qam16_constellation):
    label = format(i, '04b')  # Binary representation
    axes[0, 0].text(np.real(point), np.imag(point)+0.08, label,
                    ha='center', fontsize=8)

# Received constellation
axes[0, 1].plot(np.real(rx_symbols_norm), np.imag(rx_symbols_norm), '.', alpha=0.5)
axes[0, 1].plot(np.real(qam16_constellation), np.imag(qam16_constellation),
                'rx', markersize=12, label='Ideal')
axes[0, 1].set_title(f'RX Constellation (SER={ser:.4f})')
axes[0, 1].set_xlabel('I')
axes[0, 1].set_ylabel('Q')
axes[0, 1].axis('equal')
axes[0, 1].grid(True)
axes[0, 1].legend()

# I/Q time series
axes[1, 0].plot(np.real(rx_signal[:5000]), label='I', alpha=0.7)
axes[1, 0].plot(np.imag(rx_signal[:5000]), label='Q', alpha=0.7)
axes[1, 0].set_title('Received I/Q Time Series')
axes[1, 0].set_xlabel('Sample')
axes[1, 0].set_ylabel('Amplitude')
axes[1, 0].legend()
axes[1, 0].grid(True)

# Spectrum
fft_rx = np.fft.fftshift(np.fft.fft(rx_signal))
freqs = np.fft.fftshift(np.fft.fftfreq(len(rx_signal), 1/fs))
axes[1, 1].plot(freqs/1e3, 20*np.log10(np.abs(fft_rx)))
axes[1, 1].set_title('RX Spectrum')
axes[1, 1].set_xlabel('Frequency (kHz)')
axes[1, 1].set_ylabel('Power (dB)')
axes[1, 1].grid(True)

plt.tight_layout()
plt.savefig('lab3_3_16qam.png')
plt.show()

sdr.tx_destroy_buffer()

print("\n✓ LAB COMPLETE: 16-QAM demonstrated")
print("  - 16 constellation points = 4 bits/symbol")
print("  - Requires higher SNR than QPSK")
print("  - Used in WiFi, LTE, 5G")
```

**Expected Results:**
- 16 distinct constellation points in 4×4 grid
- Higher sensitivity to noise than QPSK
- 4 bits/symbol encoding
- **Conclusion:** QAM trades robustness for spectral efficiency

---

## 3.4 OFDM (Orthogonal Frequency Division Multiplexing)

### Theory
**OFDM** divides data across many narrow subcarriers:
- Each subcarrier is modulated (BPSK/QPSK/QAM)
- Subcarriers are orthogonal (don't interfere)
- Robust to multipath fading
- Used in WiFi, LTE, 5G, DAB

**Implementation:**
- TX: IFFT converts frequency domain to time domain
- RX: FFT converts time domain back to frequency domain

### LAB 3.4: Simple OFDM System
**Objective:** Implement basic OFDM transceiver

**Code:**
```python
#!/usr/bin/env python3
"""
LAB 3.4: OFDM Transmitter and Receiver
"""

import numpy as np
import matplotlib.pyplot as plt
import adi

# OFDM Parameters
N_fft = 64  # FFT size
N_data = 48  # Number of data subcarriers
N_pilot = 4  # Number of pilot subcarriers
CP_len = 16  # Cyclic prefix length
num_symbols = 20  # Number of OFDM symbols

# Subcarrier allocation
data_carriers = np.array([i for i in range(1, N_fft) if i not in [N_fft//2] and
                          i not in range(N_fft//2-2, N_fft//2+3)])[:N_data]
pilot_carriers = np.array([11, 25, 39, 53])  # Pilot positions

print(f"Data carriers: {len(data_carriers)}")
print(f"Pilot carriers: {len(pilot_carriers)}")

# Generate random QPSK symbols for data
qpsk_symbols = (2*np.random.randint(0, 2, (num_symbols, N_data)) - 1) + \
               1j*(2*np.random.randint(0, 2, (num_symbols, N_data)) - 1)
qpsk_symbols = qpsk_symbols / np.sqrt(2)

# Pilot symbols (known sequence)
pilot_value = 1+1j

# OFDM Transmitter
def ofdm_modulate(data_symbols, data_carr, pilot_carr, N, CP):
    """
    OFDM modulation with pilots and cyclic prefix
    """
    ofdm_time_signal = []

    for ofdm_sym in data_symbols:
        # Initialize frequency domain symbol
        freq_symbol = np.zeros(N, dtype=complex)

        # Insert data
        freq_symbol[data_carr] = ofdm_sym

        # Insert pilots
        freq_symbol[pilot_carr] = pilot_value

        # IFFT to time domain
        time_symbol = np.fft.ifft(freq_symbol)

        # Add cyclic prefix
        time_symbol_cp = np.concatenate([time_symbol[-CP:], time_symbol])

        ofdm_time_signal.append(time_symbol_cp)

    return np.concatenate(ofdm_time_signal)

tx_signal = ofdm_modulate(qpsk_symbols, data_carriers, pilot_carriers, N_fft, CP_len)

# Transmit via PlutoSDR
sdr = adi.Pluto("ip:192.168.2.1")
fs = 2.084e6
sdr.sample_rate = int(fs)
sdr.tx_lo = 915e6
sdr.rx_lo = 915e6
sdr.tx_hardwaregain_chan0 = -10
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 60
sdr.tx_cyclic_buffer = True

tx_scaled = (tx_signal * 0.7 * 2**14).astype(np.int16)
sdr.tx(tx_scaled)

import time
time.sleep(0.5)

sdr.rx_buffer_size = len(tx_scaled)
rx_signal = sdr.rx()

# OFDM Receiver
def ofdm_demodulate(rx_sig, data_carr, pilot_carr, N, CP, num_syms):
    """
    OFDM demodulation with pilot-based equalization
    """
    symbol_len = N + CP
    data_symbols_rx = []
    pilots_rx = []

    for i in range(num_syms):
        # Extract one OFDM symbol
        start = i * symbol_len
        ofdm_sym = rx_sig[start:start+symbol_len]

        if len(ofdm_sym) < symbol_len:
            break

        # Remove cyclic prefix
        ofdm_sym_no_cp = ofdm_sym[CP:]

        # FFT to frequency domain
        freq_sym = np.fft.fft(ofdm_sym_no_cp)

        # Extract pilots (for channel estimation)
        pilots = freq_sym[pilot_carr]
        pilots_rx.append(pilots)

        # Simple channel estimation: use pilot to estimate gain/phase
        if len(pilots) > 0:
            channel_est = np.mean(pilots) / pilot_value
        else:
            channel_est = 1.0

        # Equalize data carriers
        data_rx = freq_sym[data_carr] / channel_est

        data_symbols_rx.append(data_rx)

    return np.array(data_symbols_rx), np.array(pilots_rx)

data_rx, pilots_rx = ofdm_demodulate(rx_signal, data_carriers, pilot_carriers,
                                      N_fft, CP_len, num_symbols)

# Calculate EVM
min_len = min(len(data_rx), len(qpsk_symbols))
evm = np.abs(data_rx[:min_len] - qpsk_symbols[:min_len])
mean_evm = np.mean(evm)

print(f"\nOFDM Demodulation:")
print(f"  Mean EVM: {mean_evm:.4f}")
print(f"  Number of OFDM symbols: {len(data_rx)}")

# Plot
fig = plt.figure(figsize=(14, 10))
gs = fig.add_gridspec(3, 2)

# OFDM time signal
ax1 = fig.add_subplot(gs[0, :])
ax1.plot(np.real(tx_signal[:1000]), label='I')
ax1.plot(np.imag(tx_signal[:1000]), label='Q', alpha=0.7)
ax1.set_title('OFDM Time Domain Signal')
ax1.set_xlabel('Sample')
ax1.set_ylabel('Amplitude')
ax1.legend()
ax1.grid(True)

# Spectrum
ax2 = fig.add_subplot(gs[1, 0])
fft_tx = np.fft.fftshift(np.fft.fft(tx_signal))
freqs = np.fft.fftshift(np.fft.fftfreq(len(tx_signal), 1/fs))
ax2.plot(freqs/1e3, 20*np.log10(np.abs(fft_tx)))
ax2.set_title('OFDM Spectrum (Multicarrier)')
ax2.set_xlabel('Frequency (kHz)')
ax2.set_ylabel('Power (dB)')
ax2.grid(True)

# Subcarrier allocation
ax3 = fig.add_subplot(gs[1, 1])
subcarrier_power = np.zeros(N_fft)
subcarrier_power[data_carriers] = 1
subcarrier_power[pilot_carriers] = 1.5
ax3.stem(subcarrier_power)
ax3.set_title('Subcarrier Allocation (Data=1, Pilot=1.5)')
ax3.set_xlabel('Subcarrier Index')
ax3.set_ylabel('Power')
ax3.grid(True)

# TX constellation
ax4 = fig.add_subplot(gs[2, 0])
ax4.plot(np.real(qpsk_symbols.flatten()), np.imag(qpsk_symbols.flatten()),
         '.', alpha=0.5, label='TX')
ax4.set_title('TX Constellation (QPSK per subcarrier)')
ax4.set_xlabel('I')
ax4.set_ylabel('Q')
ax4.axis('equal')
ax4.grid(True)
ax4.legend()

# RX constellation
ax5 = fig.add_subplot(gs[2, 1])
ax5.plot(np.real(data_rx.flatten()), np.imag(data_rx.flatten()),
         '.', alpha=0.3, label='RX')
ax5.set_title(f'RX Constellation (EVM={mean_evm:.4f})')
ax5.set_xlabel('I')
ax5.set_ylabel('Q')
ax5.axis('equal')
ax5.grid(True)
ax5.legend()

plt.tight_layout()
plt.savefig('lab3_4_ofdm.png')
plt.show()

sdr.tx_destroy_buffer()

print("\n✓ LAB COMPLETE: OFDM system demonstrated")
print("Key concepts:")
print("  - Multiple narrow subcarriers (orthogonal)")
print("  - Cyclic prefix prevents ISI")
print("  - Pilots for channel estimation")
print("  - Used in WiFi, LTE, 5G")
```

**Expected Results:**
- Spectrum shows multiple subcarriers
- Constellation per subcarrier is QPSK
- Cyclic prefix reduces inter-symbol interference
- **Conclusion:** OFDM enables high data rates with multipath robustness

---

# 4. Coding, FEC, and Framing

## 4.1 Frame Structure and Preamble

### Theory
**Frame structure:**
```
[Preamble][Header][Payload][CRC/FEC]
```

**Preamble** is a known sequence for:
- Frame detection
- Timing synchronization
- Frequency offset estimation
- AGC settling

**Common preambles:**
- Barker codes (good autocorrelation)
- Zadoff-Chu sequences (constant amplitude)
- PN sequences (pseudorandom)

### LAB 4.1: Frame Detection with Cross-Correlation
**Objective:** Detect packets using preamble correlation

**Code:**
```python
#!/usr/bin/env python3
"""
LAB 4.1: Frame Detection with Preamble
"""

import numpy as np
import matplotlib.pyplot as plt
import adi

# Preamble: Barker code 13 (excellent autocorrelation)
barker13 = np.array([1, 1, 1, 1, 1, -1, -1, 1, 1, -1, 1, -1, 1])

# Upsample preamble
sps = 8  # Samples per symbol
preamble = np.repeat(barker13, sps)

# Generate payload (random BPSK)
payload_bits = np.random.randint(0, 2, 100)
payload_symbols = 2*payload_bits - 1
payload = np.repeat(payload_symbols, sps)

# Complete frame
frame = np.concatenate([preamble, payload])

# Add silence before and after
silence = np.zeros(200)
tx_signal = np.concatenate([silence, frame, silence, frame, silence])

# Transmit
sdr = adi.Pluto("ip:192.168.2.1")
fs = 2.084e6
sdr.sample_rate = int(fs)
sdr.tx_lo = 915e6
sdr.rx_lo = 915e6
sdr.tx_hardwaregain_chan0 = -10
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 60
sdr.tx_cyclic_buffer = True

tx_scaled = (tx_signal * 0.8 * 2**14).astype(np.int16)
sdr.tx(tx_scaled)

import time
time.sleep(0.5)

sdr.rx_buffer_size = len(tx_scaled)
rx_signal = sdr.rx()

# Frame Detection: Cross-correlate with preamble
correlation = np.correlate(np.real(rx_signal), preamble, mode='same')
correlation_norm = correlation / np.max(np.abs(correlation))

# Detect peaks (frames)
threshold = 0.6
peaks = np.where(correlation_norm > threshold)[0]

# Find distinct frames (cluster peaks)
frame_starts = []
if len(peaks) > 0:
    frame_starts.append(peaks[0])
    for peak in peaks[1:]:
        if peak - frame_starts[-1] > len(preamble):
            frame_starts.append(peak)

print(f"Detected {len(frame_starts)} frames at positions: {frame_starts}")

# Plot
fig, axes = plt.subplots(4, 1, figsize=(14, 10))

# Transmitted signal
axes[0].plot(np.real(tx_signal), linewidth=0.5)
axes[0].set_title('Transmitted Signal (2 frames with silence)')
axes[0].set_xlabel('Sample')
axes[0].set_ylabel('Amplitude')
axes[0].grid(True)

# Received signal
axes[1].plot(np.real(rx_signal), linewidth=0.5)
axes[1].set_title('Received Signal')
axes[1].set_xlabel('Sample')
axes[1].set_ylabel('Amplitude')
axes[1].grid(True)

# Cross-correlation
axes[2].plot(correlation_norm)
axes[2].axhline(threshold, color='r', linestyle='--', label=f'Threshold={threshold}')
axes[2].plot(frame_starts, [correlation_norm[i] for i in frame_starts],
             'go', markersize=10, label='Detected frames')
axes[2].set_title('Cross-Correlation with Preamble')
axes[2].set_xlabel('Sample')
axes[2].set_ylabel('Normalized Correlation')
axes[2].legend()
axes[2].grid(True)

# Preamble autocorrelation
preamble_autocorr = np.correlate(preamble, preamble, mode='full')
lags = np.arange(-len(preamble)+1, len(preamble))
axes[3].stem(lags, preamble_autocorr)
axes[3].set_title('Preamble Autocorrelation (Barker-13)')
axes[3].set_xlabel('Lag (samples)')
axes[3].set_ylabel('Autocorrelation')
axes[3].grid(True)

plt.tight_layout()
plt.savefig('lab4_1_frame_detection.png')
plt.show()

sdr.tx_destroy_buffer()

print("\n✓ LAB COMPLETE: Frame detection demonstrated")
print("Key points:")
print("  - Preamble has sharp autocorrelation peak")
print("  - Cross-correlation detects frame start")
print("  - Barker codes minimize false alarms")
```

**Expected Results:**
- Sharp peaks in correlation at frame locations
- Low sidelobes (Barker code property)
- Reliable frame detection with simple threshold
- **Conclusion:** Good preamble enables robust frame detection

---

*Continuing with the remaining sections...*

This comprehensive training manual continues with sections on:
- CRC and error detection
- FEC (Convolutional, LDPC, Turbo codes)
- Hardware concepts (ADC/DAC, FPGA, DDS)
- Synchronization (carrier, timing recovery)
- Channel models
- Radar applications
- Satellite communications

Would you like me to continue creating the remaining sections (5-14) covering these advanced topics with complete lab exercises?
