# Real-Time Detection: Raw vs Segmented Data Approach

## Overview
Comparison of two approaches for real-time human presence detection using CSI data and autoencoders.

---

## Approach 1: Raw Data (No Segmentation)

### Pipeline
```
ESP32 → MQTT → Server → Autoencoder → Presence Detection
```

### Characteristics
- **Input**: Single CSI packet (1×54 vector)
- **Latency**: ~10ms per packet
- **Update rate**: 100 Hz

### Pros ✅
- Ultra-low latency (instant processing)
- Simpler pipeline
- Lower memory overhead

### Cons ❌
- **Noisy input**: WiFi interference spikes
- **Less context**: No temporal patterns
- **High variance**: Unstable reconstruction errors
- **Lower accuracy**: 69% vs 93% with segmentation

### Evidence
From [CSI_NoSegment_RF_MLP.ipynb](../notebooks/CSI_NoSegment_RF_MLP.ipynb):
- Raw packets: **69.2% accuracy**
- Segmented data: **93.0% accuracy**
- **24% accuracy drop** without segmentation

---

## Approach 2: Segmented Data (200-packet windows)

### Pipeline
```
ESP32 → MQTT → Buffer (200 packets)
→ Noise Filter (Hampel + Savitzky-Golay)
→ Drop guard bands [2,3,4,5,32,59,60,61,62,63]
→ Segment (200×54 matrix)
→ Autoencoder → Presence Detection
```

### Characteristics
- **Input**: Temporal matrix (200×54)
- **Initial latency**: ~2 seconds (200 packets @ 100Hz)
- **Update rate**: 1 second (with sliding window)

### Pros ✅
- **Clean input**: Noise filtering removes outliers
- **Temporal patterns**: Captures movement over 2 seconds
- **Stable errors**: Averaged over 200 packets
- **Proven accuracy**: 93-94% on similar tasks

### Cons ❌
- Higher latency (~2 sec initial delay)
- More complex pipeline
- Memory overhead (buffer 200 packets)

### Evidence
- [3_act_CSI_CNN_train.ipynb](../notebooks/3_act_CSI_CNN_train.ipynb): **94% accuracy**
- [SVM_LR_RF_3act.ipynb](../notebooks/SVM_LR_RF_3act.ipynb): **93% accuracy**

---

## Why Segmentation Works Better

### 1. Noise Filtering Requires Multiple Samples
```python
# Hampel filter needs 10+ packets to detect outliers
hampel_filtered = hampel(col_series, window_size=10)

# Savitzky-Golay needs 10+ packets for smoothing
sg_filtered = savgol_filter(hampel_filtered, window_length=10, polyorder=3)
```
**Result**: Raw CSI has spikes; filtered CSI shows smooth movement patterns

### 2. Temporal Patterns Emerge Over Time
- **Single packet**: Random amplitudes [12.3, 8.7, 15.2...] - Is this noise or movement?
- **200-packet segment**:
  - Empty room: Amplitudes constant over time
  - Occupied room: Amplitudes vary with movement/breathing

### 3. Richer Feature Space for Autoencoders

| Approach | Features | What Autoencoder Learns |
|----------|----------|------------------------|
| Raw | 54 | Static amplitudes |
| Segmented | 10,800 (200×54) | Temporal-spatial patterns |

- Autoencoder trained on "empty room" learns static patterns
- Occupied room → temporal variations → higher reconstruction error
- More features = clearer separation

### 4. Statistical Stability

**Raw**: `Packet 1: error=0.8 → Occupied? Packet 2: error=0.3 → Empty?` (Hard to threshold)

**Segmented**: `Window 1: error=0.25 → Empty, Window 2: error=0.82 → Occupied` (Clear threshold)

---

## Recommended: Sliding Window Approach

### Strategy
Use segmentation for accuracy, slide window for lower latency

```
Packet Stream (100 packets/sec):
├─ Window 1: [0-199]     → t=2.0s → Empty
├─ Window 2: [100-299]   → t=3.0s → Empty      (50% overlap)
├─ Window 3: [200-399]   → t=4.0s → Occupied
└─ Window 4: [300-499]   → t=5.0s → Occupied
```

### Parameters
- Window size: 200 packets (2 seconds)
- Overlap: 100 packets (50%)
- **Update rate**: 1 second
- Initial delay: 2 seconds

### Algorithm
```python
buffer = []

while True:
    packet = mqtt_receive()
    buffer.append(packet)

    if len(buffer) >= 200:
        segment = np.array(buffer[:200])
        filtered = apply_filters(segment)
        error = autoencoder.reconstruct_error(filtered)

        presence = "Occupied" if error > threshold else "Empty"
        publish_result(presence)

        # Slide window by 100 packets
        buffer = buffer[100:]
```

### Latency Analysis
| Time | Event |
|------|-------|
| t=0s | Begin collecting |
| t=2s | First result |
| t=3s | Second result (1-sec updates start) |
| t=4s | Third result |

---

## Recommended Pipeline for CSI-HD

```
┌─────────────────────────────────────┐
│ ESP32 (1 TX + 3 RX)                │
│ ├─ TX: 100 packets/sec             │
│ ├─ RX1 → MQTT: csi/rx1             │
│ ├─ RX2 → MQTT: csi/rx2             │
│ └─ RX3 → MQTT: csi/rx3             │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│ Server Processing                   │
│ 1. Buffer (200 packets/device)     │
│ 2. Extract amplitude & Filter       │
│ 3. Segment: (200,162) or (3,200,54)│
│ 4. Autoencoder Inference            │
│ 5. Threshold → Occupied/Empty       │
└─────────────────────────────────────┘
              ↓
         Dashboard/Alerts
```

### Training Strategy
1. Collect 10-20 min of "empty room" CSI
2. Train autoencoder: `fit(empty_segments, empty_segments)`
3. Set threshold: `np.percentile(empty_errors, 95)`
4. Inference: `error > threshold → "Occupied"`

---

## Summary

### Use Segmented Data (Recommended)
- ✅ 93-94% accuracy (24% better than raw)
- ✅ Stable reconstruction errors
- ✅ 1-second updates (acceptable for presence detection)
- ✅ Proven in all high-performing models

### Avoid Raw Data
- ❌ 69% accuracy
- ❌ Noisy, unstable errors
- ❌ No temporal pattern extraction

**Key Insight**: Segmentation enables noise filtering, temporal pattern extraction, and 2D feature space - not just "clean data" but fundamentally different features that only exist in time-series windows.

**Trade-off**: 2-second initial delay is acceptable for 24% accuracy improvement.
