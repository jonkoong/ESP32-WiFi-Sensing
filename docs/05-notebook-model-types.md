# Notebook Overview and Model Types

## Summary
This document summarizes the 8 Jupyter notebooks in the repository, their purposes, model architectures, and key findings relevant to the CSI-HD presence detection project.

---

## Notebooks Analysis

### 1. **CSI_full_training.ipynb** - Complete End-to-End Pipeline
**Purpose**: Full workflow from raw data to trained CNN model

**Pipeline**:
1. Load raw CSI data from session 2 (sess2)
2. Extract amplitude: √(real² + imag²)
3. Apply noise filtering (Hampel + Savitzky-Golay)
4. Drop guard bands: [2,3,4,5,32,59,60,61,62,63]
5. Segment into 200-packet windows (200×53 matrices)
6. Train 2D CNN

**Model Architecture**:
```python
Conv2D(32, 7×7) → MaxPool(2)
Conv2D(96, 5×5) → MaxPool(2)
Flatten → Dense(128) → Dropout(0.5)
Dense(64) → Dropout(0.25)
Dense(7, softmax)
```

**Results**:
- Training: 75.77%
- Validation: 46% (overfitting issue)
- Activities: All 7 (LL, LA, JJ, RL, RA, NA, SO)

---

### 2. **SVM_LR_RF_3act.ipynb** - Classical ML Comparison
**Purpose**: Compare traditional ML algorithms on flattened CSI segments

**Data Format**: 200×54 segments flattened to 11,000-dimensional vectors

**Models Tested**:
| Model | Accuracy | Notes |
|-------|----------|-------|
| Logistic Regression | 82.3% | L2 penalty, C=1.0 |
| SVM (linear) | 82.9% | Linear kernel, C=1 |
| Random Forest | **93.0%** | 100 estimators, max_depth=20 |
| XGBoost | 90.5% | Bonus model |

**Key Insight**: Random Forest achieved best performance without deep learning

**Activities**: 3 only (SO, LL, RA)

---

### 3. **CSI_noise_filter.ipynb** - Filter Development
**Purpose**: Prototype and visualize noise filtering techniques

**What it does**:
- Loads single raw CSV (tvat-SO-3.csv)
- Tests Hampel filter (outlier removal)
  - window_size=10, n_sigma=5.0
- Tests Savitzky-Golay filter (smoothing)
  - window_length=5, polyorder=3
- Visualizes before/after filtering

**Key Finding**: Raw CSI has significant noise spikes that filtering removes

**Note**: Development notebook only - not used in production pipeline

---

### 4. **3_act_CSI_CNN_train.ipynb** - 3-Class CNN
**Purpose**: Train CNN on subset of activities with better performance

**Model Architecture**:
```python
Conv2D(32, 7×7) → MaxPool(2)
Conv2D(64, 5×5) → MaxPool(2)
Flatten → Dense(256) → Dropout(0.5)
Dense(128) → Dropout(0.25)
Dense(64) → Dropout(0.25)
Dense(3, softmax)
```

**Data Split**: 70% train / 15% validation / 15% test

**Results**: ~94% accuracy

**Activities**: 3 classes (LL=0, RA=1, SO=2)

**Saved Model**: ✅ `3act_94_model.h5`

---

### 5. **CSI_CNN_train.ipynb** - 7-Class CNN
**Purpose**: Train CNN on all 7 activities

**Label Mapping**:
- JJ=0, LA=1, LL=2, NA=3, RA=4, RL=5, SO=6

**Enhanced Architecture**:
- Larger Dense layers (256→128→64)
- Higher dropout rates (0.5, 0.25, 0.25)
- 100 epochs, batch_size=40

**Data Split**: 80% train / 10% validation / 10% test

**Saved Model**: ✅ Implied (not explicitly shown)

---

### 6. **CSI_NoSegment_RF_MLP.ipynb** - Non-Segmented Approach
**Purpose**: Test models on continuous (non-windowed) filtered CSI

**Key Difference**: No 200-packet windowing - each packet processed independently

**Models Trained**:

1. **Random Forest**:
   - Input: Filtered CSI packets (47,370→82,400 rows)
   - Result: **69.2% accuracy**

2. **LSTM-based MLP**:
   ```python
   LSTM(1024) → Dense(128) → Dense(7, softmax)
   ```
   - Input shape: (1, 64) - single timestep
   - 200 epochs training
   - Saved as: ✅ `mlp_model.h5`

**Key Finding**: Non-segmented approach underperforms (69% vs 93%)

---

### 7. **4act_86_CSI_CNN_train.ipynb** - 4-Class Variant
**Purpose**: CNN training on 4 activities (inferred from filename)

**Note**: Not fully analyzed in detail - appears to be experimental variant

---

### 8. **Model_CSI_sample.ipynb** - Example/Demo Notebook
**Purpose**: Likely contains model usage examples or sampling techniques

**Note**: File exists in directory but details not available

---

## Performance Comparison Table

| Approach | Input Format | Model | Activities | Accuracy | Saved? |
|----------|-------------|-------|------------|----------|--------|
| Segmented | 200×54 windows | **Random Forest** | 3 | **93.0%** | ❌ |
| Segmented | 200×53 windows | CNN (2D) | 3 | ~94% | ✅ |
| Segmented | 200×53 windows | CNN (2D) | 7 | 75.77% | ❌ |
| Segmented | Flattened 11k | Logistic Reg | 3 | 82.3% | ❌ |
| Segmented | Flattened 11k | SVM | 3 | 82.9% | ❌ |
| **Non-segmented** | Single packets | Random Forest | 7 | **69.2%** | ❌ |
| Non-segmented | (1,64) sequence | LSTM-MLP | 7 | Unknown | ✅ |

---

## Key Insights for CSI-HD Project

### 1. **Segmentation is Critical**
- Segmented data: 93-94% accuracy
- Non-segmented: 69% accuracy
- **Why**: Temporal patterns + noise filtering

### 2. **Random Forest is Strong Baseline**
- Achieved 93% on flattened segments
- No need for deep learning complexity
- Good candidate for initial presence detection

### 3. **CNNs Need Proper Regularization**
- CSI_full_training showed overfitting (75%→46%)
- Dropout and smaller batch sizes help
- 3-class models performed better than 7-class

### 4. **Noise Filtering is Essential**
- Hampel filter removes outliers
- Savitzky-Golay smooths amplitude curves
- Both require multi-packet windows (10+ packets)

### 5. **Architecture Recommendations for Binary Classification**

**Option A - Random Forest** (Simplest):
```
200×54 segments → Flatten(10,800) → Random Forest → Binary output
```

**Option B - Autoencoder** (Your proposal):
```
200×54 segments → Flatten(10,800)
→ Dense(512) → Dense(128) → Dense(32) [bottleneck]
→ Dense(128) → Dense(512) → Dense(10,800) → Reshape(200,54)
→ Reconstruction error → Threshold → Binary output
```

**Training Strategy**:
- Train on "empty room" data only
- High reconstruction error = Occupied (anomaly)
- Low reconstruction error = Empty (normal)

---

## Saved Models Summary

| Model File | Architecture | Input Shape | Classes | Purpose |
|-----------|--------------|-------------|---------|---------|
| `3act_cnn.onnx` | 2D CNN | (200,53,1) | 7 | Deployment to Jetson Nano |
| `3act_94_model.h5` | 2D CNN | (200,55,1) | 3 | 3-class classifier |
| `mlp_model.h5` | LSTM-MLP | (1,64) | 7 | Non-segmented approach |
| `rf.pkl` | Random Forest | (64,) | 7 | Non-segmented RF |

---

## Next Steps for Your Project

1. **Decide on approach**: Segmented (recommended) vs non-segmented
2. **Choose model**:
   - Autoencoder (anomaly detection) - aligns with your proposal
   - Random Forest (strong baseline) - simpler fallback
3. **Adapt pipeline**:
   - 7 classes → 2 classes (occupied/empty)
   - Activity recognition → Presence detection
4. **Consider**: Sliding windows for near-real-time detection (~1 sec updates)
