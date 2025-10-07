# Collecting CSI Data

## Understanding Packets

### What Are "Packets"?

The ESP32 **doesn't sample at a fixed rate like 100Hz**. Instead, it captures CSI data **whenever a WiFi packet is received** from the transmitter. This is **event-driven**, not periodic sampling.

**Each packet = one WiFi frame** captured by the receiver, containing:
- Metadata (RSSI, timestamp, channel, MAC address, etc.)
- **CSI_DATA** array (64 subcarriers of complex values)

---

## Packet Collection Rate

### Default Behavior: Variable Rate

Without configuration, packet arrival times are **irregular** and depend on:
- WiFi traffic in the environment
- Transmitter sending frequency
- Hardware processing overhead

**Example timestamps** (in microseconds):
```
8346021 → 8346426 (0.4ms gap)
8346426 → 8375326 (29ms gap)
8375326 → 8397104 (22ms gap)
```

Typical rates vary but often range **~50-100 packets/second**.

---

## Configuring Fixed Transmission Rate

### Yes! You Can Fix the Packet Rate

The ESP32 firmware provides a **configurable transmission rate** via the `PACKET_RATE` parameter.

### Configuration Steps

When building the ESP32 firmware:

```bash
cd esp32-csi-tool/active_sta
idf.py menuconfig
```

Navigate to:
```
ESP32 CSI Tool Config → Packet TX Rate
```

**Configuration Options**:
- **Default**: 100 packets/second
- **Range**: 1-1000 packets/second
- **Unit**: Packets per second (Hz)

**Example values**:
- `50` = 50 packets/sec (20ms intervals)
- `100` = 100 packets/sec (10ms intervals) - **Recommended**
- `200` = 200 packets/sec (5ms intervals)

---

### How It Works

**Transmitter (active_sta)** sends packets at controlled intervals.

**Implementation** ([sockets_component.h](../esp32-csi-tool/_components/sockets_component.h)):
```c
#if defined CONFIG_PACKET_RATE && (CONFIG_PACKET_RATE > 0)
    double wait_duration = (1000.0 / CONFIG_PACKET_RATE) - lag;
    int w = floor(wait_duration);
    vTaskDelay(w);  // Delay to maintain target rate
#else
    vTaskDelay(10); // Default: ~100 packets/second
#endif
```

**Logic**:
1. Calculate delay between packets: `1000ms / PACKET_RATE`
2. Compensate for processing lag
3. Maintain consistent transmission rate

**Receiver (active_ap)** captures CSI from each received packet.

---

## Data Segmentation

### What Does "200-Packet Chunk" Mean?

**Segmentation** groups **consecutive WiFi packets** into fixed-size training samples.

**Example with 200-packet chunks**:
```
Continuous stream:  [Packet₁] [Packet₂] [Packet₃] ... [Packet₅₅₀₀]
                           ↓
Split into chunks:  [Chunk 1: Packets 1-200]
                    [Chunk 2: Packets 201-400]
                    [Chunk 3: Packets 401-600]
                    ...
```

**Each chunk becomes one training sample** with shape `(200 packets × 54 subcarriers)`

---

### Why 200 Packets?

In the original repo's activity recognition model:

- **Temporal context**: Captures movement patterns over ~2-4 seconds (at 50-100 packets/sec)
- **Sufficient data**: Enough samples to detect activity patterns
- **Computational balance**: Not too large (memory) or too small (context loss)

**Calculation at 100 packets/sec**:
- 200 packets = 2 seconds of continuous data
- Each segment = one training example

---

## For Your Presence Detection Project

### Recommended Configuration

**Packet Rate**: 100 packets/second (10ms intervals)
- Provides consistent temporal spacing
- Balances data rate with serial bandwidth
- Standard rate used in research papers

**Window Size**: Adjust based on your needs
- **Original proposal**: 10 consecutive samples → 100ms windows
- **Alternative**: 50-100 samples → 0.5-1 second windows (better for stationary detection)

**Example calculation with 100 packets/sec**:
| Window Size | Time Duration | Inference Rate |
|-------------|---------------|----------------|
| 10 packets  | 100ms         | 10 Hz          |
| 50 packets  | 500ms         | 2 Hz           |
| 100 packets | 1 second      | 1 Hz           |
| 200 packets | 2 seconds     | 0.5 Hz         |

---

### Benefits of Fixed Packet Rate

**Original repo approach**:
- Variable packet rate → segments of 200 packets (time varies)
- Time duration per segment is inconsistent

**Your fixed-rate approach**:
- Fixed packet rate → consistent time windows
- 10 packets always = 100ms (predictable)
- **More consistent temporal patterns** for autoencoder training

This makes your CSI patterns **more predictable and easier to learn**! 🎯

---

## Important Considerations

### Serial Baud Rate

Higher packet rates require higher serial baud rates to avoid buffer overflow.

**Recommended settings** (configured via `idf.py menuconfig`):
```
Serial flasher config > Custom baud rate value > 921600
Component config > Common ESP32-related > UART console baud rate > 921600
```

**Baud rate requirements**:
- 100 packets/sec: **921600 baud** (sufficient)
- 200+ packets/sec: May require **1152000 or 1500000 baud**

### Monitoring Actual Rate

Use the measurement utility to verify actual collection rate:

```bash
idf.py monitor | python ../python_utils/serial_measure_rate.py
```

This displays:
- Packets per second (real-time)
- Average packet rate
- Rolling average (last 10 seconds)

---

## Data Collection Workflow

### Step 1: Configure ESP32

```bash
cd esp32-csi-tool/active_sta
idf.py menuconfig
# Set: PACKET_RATE = 100
# Set: SHOULD_COLLECT_CSI = y
# Set: SEND_CSI_TO_SERIAL = y
idf.py flash
```

### Step 2: Collect Data

**To file**:
```bash
idf.py monitor | grep "CSI_DATA" > my-experiment.csv
```

**With timestamp**:
```bash
idf.py monitor | python ../python_utils/serial_append_time.py > my-experiment.csv
```

**Verify rate while collecting**:
```bash
idf.py monitor | python ../python_utils/serial_measure_rate.py
```

### Step 3: Process Data

See [reading-CSI-data.md](reading-CSI-data.md) for parsing the CSI_DATA field.

**Pipeline**:
1. Parse raw CSV → extract amplitude/phase
2. Apply noise filtering (Hampel + Savitzky-Golay)
3. Segment into fixed windows (e.g., 100 packets)
4. Train autoencoder on "empty room" baseline
5. Use reconstruction error for presence detection

---

## Summary

✅ **Packet rate is configurable** (1-1000 packets/sec)
✅ **Recommended**: 100 packets/sec for balanced performance
✅ **Fixed rate** enables consistent temporal windows
✅ **Monitor actual rate** during collection to verify settings
✅ **Adjust window size** based on detection requirements (10-200 packets)
