# CSI Data Processing Pipeline

## Three Dataset Types

The original repo uses a 3-stage processing pipeline to transform raw ESP32 output into ML-ready training samples.

---

### **1. Raw CSI Data** (`01-tvat-raw/`)

**Format**: Direct ESP32 output

**Contains**:
- Full metadata (MAC, RSSI, channel, timestamp, etc.)
- Raw `CSI_DATA` field: `[imaginary₁, real₁, imaginary₂, real₂, ...]`
- 64 subcarrier pairs (128 values total)
- Variable packet count per file

**Example**: `tvat-JJ-1.csv` contains ~5,500 packets

**Sample row**:
```csv
CSI_DATA,AP,C8:F0:9E:F2:C2:EC,-72,11,0,0,0,...,[28 -64 1 0 0 0 -9 -9 -9 -9 ...]
```

---

### **2. Filtered/Denoised Amplitude** (`02-tvat-filtered/`)

**Format**: Processed amplitude values only

**Processing applied**:
1. Extract amplitude: `√(real² + imaginary²)` for each subcarrier
2. **Hampel filter** - removes outliers
3. **Savitzky-Golay filter** - smooths signal

**Result**: Clean CSV with 64 amplitude columns (no metadata)

**Sample format**:
```csv
0,1,2,3,...,63
66.40,1.8,0.0,0.0,...,0.0
123.81,5.8,0.0,0.0,...,0.0
```

Each row = amplitude values for one packet's 64 subcarriers

---

### **3. Segmented Data** (`03-tvat-segments/`)

**Format**: Fixed-size training samples

**Processing**:
1. Drop guard band subcarriers and DC null (keeps 54 useful subcarriers)
2. Split continuous data into **200-packet chunks**
3. Each segment saved as separate CSV file

**Result**: Each file = one training sample `(200 rows × 54 columns)`

**Example**: `tvat-JJ-2-segment-0.csv` = 200 consecutive packets from jumping jacks activity

**Purpose**: Creates uniform input shape for CNN: `(200, 54, 1)` tensor

---

## Pipeline Summary

```
Raw CSI Data
    ↓ Extract amplitude: √(real² + imag²)
Amplitude Time-Series (64 subcarriers)
    ↓ Hampel filter (outlier removal)
    ↓ Savitzky-Golay filter (smoothing)
Denoised Amplitude (64 subcarriers)
    ↓ Drop columns [2,3,4,5,32,59,60,61,62,63]
    ↓ Segment into 200-packet windows
Training Samples (200×54 matrices)
    ↓
CNN Model Input
```

---

## Why Each Stage?

| Stage | Purpose |
|-------|---------|
| **Raw → Amplitude** | Convert complex CSI to usable features (amplitude) |
| **Amplitude → Filtered** | Remove noise and smooth signal for better ML performance |
| **Filtered → Segmented** | Create fixed-size inputs, remove uninformative subcarriers |

---

## Code Reference

Full processing pipeline: [notebooks/CSI_full_training.ipynb](../notebooks/CSI_full_training.ipynb)

Key processing steps:
- **Lines 1-39**: Load raw CSI files
- **Lines 40-74**: Extract amplitude from complex CSI data
- **Lines 75-88**: Apply Hampel + Savitzky-Golay filters
- **Lines 89-100**: Drop guard band/DC null subcarriers
- **Lines 101-122**: Segment into 200-packet windows

---

## For Your Project

You'll follow a similar pipeline but adapt for **presence detection**:

1. **Collect raw CSI** from 3 receivers
2. **Extract amplitude** (or keep both amplitude + phase)
3. **Apply filtering** to reduce noise
4. **Segment into windows** (10-100 packets, based on your needs)
5. **Train autoencoder** on "empty room" baseline
6. **Detect presence** via reconstruction error threshold
