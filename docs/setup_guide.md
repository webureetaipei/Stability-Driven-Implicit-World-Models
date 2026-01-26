# Environment Setup Guide for SD-IWM (NeurIPS 2026)

## 1. Prerequisite: System & Driver (Task 01)
* **OS:** Windows 11 with WSL2 (Ubuntu 22.04 LTS) OR Native Linux.
* **GPU:** NVIDIA RTX GPU (Driver version **535.xx** or later).

## 2. Python & Deep Learning Environment (Task 02)
We use Miniconda to isolate dependencies.
```bash
conda create -n sdiwm python=3.10 -y
conda activate sdiwm
```
## 3. Core Dependencies (Task 03)
```bash
pip install torch torchvision torchaudio --index-url [https://download.pytorch.org/whl/cu121]
pip install "ray[data,train,default]" accelerate deepspeed wandb hydra-core
```
## 4. Verification (Task 04)
Run python check_env.py to verify GPU visibility and Ray Cluster.

## 5. Project Structure Initialization (Task 05)
Run the following commands to create the standard Google-style research directory structure and initialize Python packages.

```bash
# 1. Create Data and Source Directories
mkdir -p src/{models,training,data,distributed,tinyengine_port}
mkdir -p configs

# 2. Initialize Python Packages (Make directories importable)
touch src/__init__.py
touch src/models/__init__.py
touch src/training/__init__.py
touch src/data/__init__.py
touch src/distributed/__init__.py
touch src/tinyengine_port/__init__.py
# 3. Create Placeholder for Design Docs (If not cloning from repo)
# Note: In a cloned repo, these files will already exist.
touch docs/design/{stability_driven_multimodal_system_design.md,distributed_system_design.md,embedded_system_design.md,model_card.md}
touch docs/reports/{evaluation_results.md}
echo "✅ Project structure created successfully."