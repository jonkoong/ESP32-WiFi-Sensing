# Reading CSI Data

## CSI Data Structure

### Metadata Fields
The header contains information about the WiFi packet:
- `type`, `id`, `mac`: Packet identifiers
- `rssi`: Received Signal Strength Indicator (overall signal strength)
- `rate`: Data transmission rate
- `channel`: WiFi channel used
- `bandwidth`: Channel bandwidth (20/40 MHz)
- `local_timestamp`: When packet was received
- `len`, `first_word`: Packet length and initial data

### CSI Data Array

The **`data`** field at the end contains the actual CSI:

```
"[67,48,4,0,0,0,0,0,0,5,0,20,1,20,1,19,0,17,1,16,2,1...]"
```

**Format**: `[Imaginary₁, Real₁, Imaginary₂, Real₂, Imaginary₃, Real₃, ...]`

Each subcarrier is represented by **2 integers**:
1. **Imaginary part** (first)
2. **Real part** (second)

### Converting to Amplitude & Phase

From the complex number (Real, Imaginary):

```python
import numpy as np

# Example: subcarrier 1 = (real=48, imag=67)
real = 48
imag = 67

# Amplitude (signal strength)
amplitude = np.sqrt(real**2 + imag**2)  # = 82.4

# Phase (signal delay/shift in radians)
phase = np.arctan2(imag, real)  # = 0.95 radians
```

### Number of Subcarriers

From your data:
- Array length / 2 = number of subcarriers
- Example shows many pairs, likely **64 or 128 subcarriers** depending on bandwidth

### Subcarrier Types in WiFi CSI

Not all subcarriers are equally useful for machine learning. WiFi divides the frequency spectrum into different types:

#### Types of Subcarriers

| Type | Index Range (64-subcarrier system) | Purpose | Used for CSI? |
|------|-------------------------------------|---------|---------------|
| **Guard Band Subcarriers** | 0-5, 59-63 | Prevent interference with adjacent WiFi channels | ❌ No signal |
| **DC Null** | 32 (center frequency) | Avoids DC offset in hardware | ❌ Zero/unreliable |
| **Pilot Subcarriers** | Fixed positions (e.g., 11, 25, 39, 53) | Synchronization & channel estimation | ✅ Reference only |
| **Data Subcarriers** | Remaining ~48 carriers | Actual WiFi data transmission | ✅ **Most informative** |

#### What Are Guard Bands?

**Guard band subcarriers** are positioned at the **edges** of the frequency spectrum and transmit **no data**. They act as "buffer zones" between adjacent WiFi channels to prevent interference.

**Analogy**: Think of WiFi channels like lanes on a highway:
- **Guard bands** = shoulders (empty space between highways)
- **DC null** = center divider (structural, not for driving)
- **Pilot subcarriers** = lane markers (guide position)
- **Data subcarriers** = actual traffic lanes (carry information)

#### Why Drop Certain Subcarriers for ML?

In the original repo's preprocessing pipeline:
```python
columns_to_drop = [2, 3, 4, 5, 32, 59, 60, 61, 62, 63]  # Guard bands + DC null
```

**Reasons**:
1. **Guard bands (0-5, 59-63)**: Near-zero amplitude, contain no useful information
2. **DC null (32)**: Hardware artifact that distorts the signal
3. **Result**: Keeping only **54 data-rich subcarriers** improves model accuracy and reduces noise

For human detection via CSI, you only care about **how the data subcarriers change** when a person blocks or scatters the WiFi signal path.

### Why This Matters for Your Project

When a **human enters the room**:
- Amplitude values change (signal absorbed/scattered)
- Phase values shift (signal path changes)
- Different subcarriers affected differently
- Your autoencoder will learn the "empty room" pattern and detect deviations

### Your Data Shape

For ESP32 with your proposal (114 subcarriers, 10 time samples):
```python
# Single snapshot
csi_array = [imag₁, real₁, imag₂, real₂, ..., imag₁₁₄, real₁₁₄]  # 228 values

# Convert to complex
csi_complex = [complex(real₁, imag₁), ..., complex(real₁₁₄, imag₁₁₄)]  # 114 complex numbers

# 10 consecutive snapshots → (114, 10) for your model
```

The autoencoder will learn patterns in how these 114 complex numbers evolve over 10 time steps for an empty room.
