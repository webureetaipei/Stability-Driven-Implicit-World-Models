# SD-IWM: Stability-Driven Implicit World Models

**Target:** NeurIPS 2026 | **Status:** Active Development (Phase 1)

## Why This Matters

Standard implicit world models (like V-JEPA) suffer from **representation drift** and "hallucinations" over long prediction horizons ($T > 50$), making them unreliable for complex robotic planning.

**SD-IWM** addresses this gap by combining **Drift-Aware Latent Dynamics**, **Multimodal Fusion**, and an **Embedded-First** inference engine. It bridges the gap between high-level theoretical stability and deployable, memory-constrained edge systems.

---

## Reviewer Guide (10-Minute Read)

Select the Design Document that matches your expertise:

### 1. The Master Blueprint (Start Here)
👉 **[docs/design/system_design_overview.md](./docs/design/system_design_overview.md)**
* **Content:** High-Level Architecture, Cloud-to-Edge Topology, Trade-offs, and Project Goals.

### 2. For Embedded Systems
👉 **[docs/design/embedded_system_design.md](./docs/design/embedded_system_design.md)** *(Planned)*
* **Content:** TinyEngine (C++) Implementation, Patch-based im2col, Int8 Quantization strategy.

### 3. For Distributed Systems / MLOps
👉 **[docs/design/distributed_system_design.md](./docs/design/distributed_system_design.md)** *(Planned)*
* **Content:** FSDP Scaling Strategy, **Data Engine Architecture (Ray + Parquet)**, and IO Bottleneck Analysis.

### 4. For AI Research (Stability-Driven)
👉 **[docs/design/stability_driven_multimodal_system_design.md](./docs/design/stability_driven_multimodal_system_design.md)** *(Planned)*
* **Content:** Drift-Aware Consistency Loss ($L_{drift}$), Cross-Attention VLA Fusion, and Manifold Regularization.

### 5. Evidence & Benchmarks (The Proof)
👉 **[docs/reports/evaluation_results.md](./docs/reports/evaluation_results.md)** *(Planned)*
* **Content:**
    * **System:** Memory Analysis (TinyEngine vs. TFLite).
    * **Stability:** Horizon Reliability Index (HRI) & Long-Horizon Drift Analysis.
    * **Scalability:** Cloud Training Throughput & GPU Utilization.

---

## Key Features

### 1. Stability-Driven Core
Solves "Representation Drift" by introducing a **Drift-Aware Consistency Objective**. This mathematically bounds the error accumulation in the latent space, allowing the model to remain geometrically stable for $T=100+$ steps.

### 2. TinyEngine (C++ Inference)
A custom bare-metal runtime designed for **ARM Cortex-M** (STM32).
* **Patch-based Inference:** Dynamic patch computation reduces Peak RAM by 10x.
* **Int8 PTQ:** Custom SIMD kernels optimized for microcontroller arithmetic.
* **Zero-Dependency:** No Python/OS overhead.

### 3. Cloud-Scale Training
Designed to train on **2TB+ Video Datasets** (Kinetics-400).
* **FSDP Sharding:** Enables ViT-Huge training on consumer GPUs.
* **Ray Pipelines:** Asynchronous producer-consumer data loading to maximize Tensor Core usage.

### 4. Multimodal Fusion (VLA)
Integrates **Vision, Language, and Action** modalities. The system uses Cross-Attention to fuse text instructions (T5) into the world model's transition dynamics, enabling instruction-following prediction.

---

## Roadmap & Progress

We follow a strict **100-Task Engineering Plan** to ensure reproducibility and steady progress.
👉 **[View Full 100-Task Roadmap](./docs/ROADMAP.md)**

| Phase | Focus | Key Tech | Status |
| :--- | :--- | :--- | :--- |
| **Phase 1** | **Infrastructure Setup** | WSL2, Docker, Conda | 🟡 In Progress |
| **Phase 2** | **Research Core** | Drift-Aware Loss, VLA Fusion | ⚪ Planned |
| **Phase 3** | **Cloud Distributed** | PyTorch FSDP, Ray Data | ⚪ Planned |
| **Phase 4** | **Edge Optimization** | TinyEngine, C++17, Int8 | ⚪ Planned |
| **Phase 5** | **Evaluation & Paper** | Benchmarking, Ablation | ⚪ Planned |

---

## System Architecture & Docs

This project follows **Tier-1 Research Engineering** practices.

* **Blueprint:** [System Design Overview](./docs/design/system_design_overview.md)
* **Specs:** [Model Card](./docs/model_card.md)
* **Setup:** [Environment Setup Guide](./docs/setup_guide.md)

---

### Tech Stack
* **Data:** Kinetics-400, Ray Data, Parquet
* **Model:** PyTorch, ViT (Vision Transformer), FSDP
* **Embedded:** C++17, CMSIS-NN, ARM GCC (TinyEngine)
* **Infra:** Docker, WSL2, Hydra, WandB

---
*Author: Tsung Lung Yang*