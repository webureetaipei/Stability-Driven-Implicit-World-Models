# SD-IWM System Design Overview

**Status:** Draft (Phase 1)
**Author:** SD-IWM Research Team
**Target:** NeurIPS 2026

This document outlines the architectural decisions, trade-offs, and optimization strategies for the **Stability-Driven Implicit World Model (SD-IWM)** project. It serves as the master blueprint (High-Level Design) for our implementation. Detailed Low-Level Designs (LLD) for each subsystem will follow in separate documents.

---

## 1. Embedded Optimization: The "TinyEngine" Architecture
**Goal:** Deploy long-horizon world models on **ARM Cortex-M** devices (e.g., STM32H7) with **<512KB SRAM** and **<20ms Latency**.

### 1.1 Core Implementation Strategy
We reject standard runtimes (TFLite Micro) in favor of a specialized bare-metal C++ approach.

* **Patch-based `im2col` Inference:**
    * *Mechanism:* Instead of materializing full feature maps (which causes OOM on MCU), we compute convolutions on small overlapping patches dynamically.
    * *Benefit:* Reduces Peak RAM usage by **10x** (from ~5MB to ~380KB), enabling execution on devices with only 512KB SRAM.
* **Drift-Aware Int8 Quantization:**
    * *Mechanism:* We utilize **Post-Training Quantization (PTQ)** specifically calibrated on the "stable manifold" of our Drift-Aware model, rather than standard min-max calibration.
    * *Benefit:* Achieves **4x Model Compression** with <2% drop in Horizon Reliability Index (HRI).
* **Source-to-Source Compilation:**
    * *Mechanism:* Python model definitions are transpiled into dependency-free C++17 source code.
    * *Optimization:* Hard-coded memory offsets eliminate the need for a runtime interpreter or dynamic memory allocation (`malloc/free`), preventing memory fragmentation.
* **SIMD Optimization (CMSIS-NN):**
    * *Mechanism:* Custom C++ Kernels utilizing **ARM Helium (M-Profile Vector Extension)** instructions.
    * *Benefit:* Maximizes arithmetic throughput on the Cortex-M7 core.

### 1.2 Bottlenecks & Trade-offs
* **Bottleneck Solved:** **The Memory Wall**. Standard Transformers require megabytes of buffer; ours fits in kilobytes.
* **Trade-off:** **Latency vs. Feasibility**.
    * *Decision:* We accept a slightly higher latency (due to patch re-computation overhead) to achieve **feasibility** on hardware that otherwise couldn't run the model at all.
    * *Reference:* See `docs/design/embedded_system_design.md` for memory mapping details.

---

## 2. Multimodal AI System: Stability-Driven Latent Dynamics
**Goal:** Solve "Representation Drift" in Implicit World Models for long horizons ($T > 50$).

### 2.1 Model Architecture
We propose a stability-first architecture that prioritizes geometric consistency over short-term reconstruction accuracy.

* **Drift-Aware Consistency Objective:**
    * *Mechanism:* A novel loss function $L_{drift} = ||z_{t+k} - Enc(x_{t+k})||^2$ that bounds the accumulation of error in the latent space over time.
    * *Reasoning:* Mathematically forces the latent trajectory to stay on the data manifold, preventing "causal hallucinations."
* **Vision-Language-Action (VLA) Fusion:**
    * *Mechanism:* **Cross-Attention** layers fuse Text Embeddings (T5) and Action Vectors into the World Model's transition dynamics.
    * *Benefit:* Enables the robot to "understand" the consequences of actions based on language instructions (e.g., "If I drop the cup, it breaks").
* **Manifold Regularization:**
    * *Mechanism:* Constrains the Jacobian of the latent transition to ensure the Lipschitz constant $< 1$.
    * *Benefit:* Guarantees theoretical stability for infinite-horizon prediction.

### 2.2 Bottlenecks & Trade-offs
* **Bottleneck Solved:** **Long-Horizon Divergence**. Standard V-JEPA diverges after 50 steps; SD-IWM remains stable up to 100 steps.
* **Trade-off:** **Training Compute vs. Stability**.
    * *Decision:* Calculating $L_{drift}$ requires unrolling the model for $k$ steps during training (Back-propagation Through Time), which increases memory usage by $O(k)$ compared to single-step prediction.
    * *Reference:* See `docs/design/stability_driven_multimodal_design.md` for mathematical proofs.

---

## 3. Distributed System: Cloud Infrastructure
**Goal:** Scale training to **2TB+ (Kinetics-400)** datasets using consumer-grade GPU clusters.

### 3.1 Distributed Architecture
* **PyTorch FSDP (Fully Sharded Data Parallel):**
    * *Mechanism:* Shards model parameters, gradients, and optimizer states across 8x GPUs.
    * *Benefit:* Allows training ViT-Huge (600M+) models on GPUs with limited VRAM (24GB).
* **Ray Data Pipeline:**
    * *Mechanism:* A producer-consumer pipeline that streams sharded Parquet files from disk to GPU memory asynchronously.
    * *Benefit:* Solves the IO bottleneck where GPUs starve while waiting for data loading (volitile utilization).

### 3.2 Bottlenecks & Trade-offs
* **Bottleneck Solved:** **IO Starvation**. Random access on video files kills performance.
    * *Solution:* We preprocess videos into sequentially sharded Parquet files.
* **Trade-off:** **Preprocessing Time vs. Training Speed**.
    * *Decision:* We invest significant time in offline data sharding (Phase 1) to maximize online GPU utilization (Phase 3).
    * *Reference:* See `docs/design/distributed_system_design.md` for the Ray pipeline topology.

---

## 4. Data Engine Strategy: High-Throughput Video Pipeline
**Goal:** Stream **2TB+ (Kinetics-400)** video data to GPUs with **zero CPU blocking**.

### 4.1 Data Pipeline
* **Ingestion (Offline):**
    * Raw videos are decoded, resized to 224x224, and repacked into **Apache Parquet** shards (columnar storage).
* **Streaming (Online):**
    * We use **Ray Data** to implement a streaming producer-consumer pipeline.
    * *Prefetching:* Data blocks are loaded into Host RAM asynchronously while the GPU computes the previous batch.
* **Decoding Policy:**
    * We store data as **raw byte tensors** (uint8) inside Parquet to avoid expensive JPEG/Video decoding during the training loop.

### 4.2 Bottlenecks & Trade-offs
* **Bottleneck Solved:** **IO Starvation (The "Small File" Problem)**. Reading millions of individual MP4 files cripples disk IOPS.
    * *Solution:* Aggregating data into large (2GB) Parquet shards enables sequential disk reads, maximizing SSD throughput.
* **Trade-off:** **Storage Size vs. CPU Load**.
    * *Decision:* Storing decoded frames (even compressed) uses 10x more disk space than raw video. We accept this storage cost to eliminate the CPU decode bottleneck during training.

---

## 5. Success Criteria (Engineering KPIs)
We define the project's success through strict quantitative metrics.

### 5.1 System Performance KPIs
| Component | Metric | Target | Baseline (Standard) |
| :--- | :--- | :--- | :--- |
| **Edge (TinyEngine)** | Inference Latency | **< 20 ms** | > 120 ms (TFLite) |
| **Edge (TinyEngine)** | Peak SRAM Usage | **< 480 KB** | > 4 MB |
| **Cloud (FSDP)** | GPU Utilization | **> 90%** | < 45% (IO Bound) |
| **Cloud (FSDP)** | Effective Batch Size | **512** | 64 |

### 5.2 Research Success Metrics (NeurIPS)
* **Horizon Reliability Index (HRI):** Achieve **> 0.85** cosine similarity at $T=100$ steps (vs. 0.38 for V-JEPA).
* **FVD (Fréchet Video Distance):** Achieve lower FVD scores on the Kinetics-400 validation set compared to DreamerV3.