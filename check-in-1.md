# Research
- Reading how to read and parse CSI data (guard band, DC null)
- How to visualise CSI data
- CSI resolution determined by WiFi standard, bandwidth, and chipset capabilities
- Explored 3 CSI data transformation methods:
  - Amplitude-only (most stable, common practice)
  - Phase/amplitude differences between antennas (cancels environmental noise)
  - Doppler representation (domain-invariant, requires synced receivers)
- Subcarrier features: low frequencies detect large movements, high frequencies capture fine movements
- Multiple antennas improve spatial diversity and signal quality, not subcarrier count
- Domain adaptation needed for transfer learning due to environment dependency
- Few-shot/one-shot learning as alternative to supervised learning with massive datasets


# Datasets
- ESP32 wifi CSI sensing has been done before, but publicly available datasets are limited
- Low sample sizes in existing datasets
- Identified potential dataset: jasminkarki/wifi-sensing-har for initial model testing


# ML
- Data processing - using raw data, filtered/denoised, and segmented data
- Autoencoder that detects presence of a human by measuring reconstruction loss
- Model architectures explored:
  - CNN (<20 layers) for spatial feature extraction across subcarriers
  - 2D kernels can extract both spatial and temporal features simultaneously
- Learning approaches: supervised (data-intensive), transfer learning (needs domain adaptation), unsupervised (AutoFi), few-shot
- Transformers: CSI tokens can represent spatial-temporal regions with positional encoding

# Future Possibilities
- Course localisation
- Few shot learning
- UDP instead of MQTT for handling 3 synced clients (MQTT may be too slow)


# Next steps
- Try out model using dataset from jasminkarki/wifi-sensing-har
- Test different data transformation methods (amplitude vs differential vs Doppler)
- Experiment with CNN architectures for feature extraction
