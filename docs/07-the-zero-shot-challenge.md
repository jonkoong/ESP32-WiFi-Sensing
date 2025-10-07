# The Zero-Shot Challenge

## The Deployment Problem

Current CSI-based human detection systems are **environment-specific**. A model trained in one location won't work in another without retraining because WiFi signals depend on:

- Room geometry/size
- Furniture placement
- Wall materials
- Transmitter-receiver positions

**The challenge:** You need to collect extensive data and train a model at each deployment location before the system becomes useful. This makes rapid deployment impractical.

## What is Zero-Shot Learning?

**Zero-shot learning** = A model that can perform tasks without seeing training examples for that specific task or environment.

In the context of CSI-HD: deploying an ESP32 system at a new location and having it work immediately (or with minimal calibration) without collecting hours of labeled training data.

## Zero-Shot Approaches for CSI-HD

### 1. Transfer Learning with Domain Adaptation

**Concept:** Train on multiple diverse environments to learn **environment-invariant features** rather than location-specific patterns.

**Process:**
- Collect data from 5-10+ different rooms/buildings upfront
- Train model to recognize motion patterns that generalize across environments
- Focus on relative changes (e.g., motion dynamics) rather than absolute CSI values
- Fine-tune with minimal (<10 minutes) calibration data at new location

**Pros:**
- Can achieve good accuracy with minimal on-site data
- Learns robust, generalizable features

**Cons:**
- Requires extensive upfront data collection across diverse environments
- Still needs some calibration data at deployment site

---

### 2. Self-Supervised Anomaly Detection (Most Promising)

**Concept:** Train the autoencoder on-site using only "empty room" data, making it location-specific automatically.

**Process:**
1. Deploy ESP32 at new location
2. **Auto-calibrate for 30-60 minutes** collecting baseline CSI when room is empty
3. Train autoencoder on-site on just this empty room data
4. Any deviation from baseline = presence detected
5. System is ready to use

**Pros:**
- **Near zero-shot**: only needs unlabeled empty room data (no manual labeling required)
- Naturally adapts to each unique environment
- Aligns with your existing autoencoder approach
- Simple deployment workflow

**Cons:**
- Requires guaranteed empty room during calibration period
- May need periodic recalibration if environment changes (furniture moved, etc.)

**Implementation:**
```python
# On-site calibration phase (30-60 min)
empty_data = collect_csi_data(duration=3600)  # 1 hour
autoencoder.train(empty_data)
autoencoder.save("location_specific_model.h5")

# Deployment phase
while True:
    current_csi = collect_segment()
    reconstruction_error = autoencoder.reconstruct_error(current_csi)

    if reconstruction_error > threshold:
        status = "Occupied"
    else:
        status = "Empty"
```

---

### 3. Normalized Feature Engineering

**Concept:** Extract features that are less dependent on absolute CSI values and more on patterns.

**Features to use:**
- **Relative changes** instead of absolute CSI values
- Variance ratios between subcarriers
- Temporal derivatives (rate of change)
- Correlation patterns across subcarriers
- Frequency domain features

**Process:**
```python
# Instead of using raw amplitude
amplitude = calculate_amplitude(csi)

# Use normalized features
variance_ratio = np.var(amplitude, axis=1) / np.mean(np.var(amplitude, axis=1))
temporal_derivative = np.diff(amplitude, axis=0)
subcarrier_correlation = np.corrcoef(amplitude.T)
```

**Pros:**
- Makes features less location-dependent
- Can improve generalization across environments

**Cons:**
- Still requires some training data
- May lose discriminative information by normalizing

---

### 4. Meta-Learning (Complex)

**Concept:** Train a model that "learns how to adapt quickly" to new environments with minimal data.

**Approach:**
- Use MAML (Model-Agnostic Meta-Learning)
- Train on tasks from 10+ different environments
- Model learns to adapt to new environment with just a few examples

**Pros:**
- Theoretically can adapt very quickly to new locations
- State-of-the-art approach for few-shot learning

**Cons:**
- Very complex to implement
- Requires collecting data from many diverse environments upfront
- Computationally expensive training
- Overkill for binary presence detection

---

## Recommended Approach for CSI-HD

**Best choice: Approach #2 (Self-Supervised Anomaly Detection)**

**Why:**
1. Aligns with your existing autoencoder plan
2. Minimal deployment overhead (just wait 30-60 min for calibration)
3. No manual labeling required
4. Automatically adapts to each unique environment
5. Simple and practical for real-world deployment

**Deployment Workflow:**
1. Install ESP32 transmitter and receiver at target location
2. Ensure room is empty
3. Run auto-calibration script (30-60 minutes)
4. System trains autoencoder on empty room baseline
5. System is ready for presence detection
6. Optional: periodic recalibration (e.g., weekly) to account for environmental changes

**Hybrid Enhancement:**
Combine Approach #2 with Approach #3:
- Use self-supervised learning with normalized features
- Makes the system even more robust and faster to calibrate