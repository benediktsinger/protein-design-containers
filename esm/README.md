# ESM2 - Docker & Apptainer/Singularity Build Guide

## 1. Create the Dockerfile

Save the following as `Dockerfile` in your working directory:

```dockerfile
FROM nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04

# Set environment variables
ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1

# Install system dependencies and Python 3.10
RUN apt-get update && apt-get install -y \
    python3.10 \
    python3-pip \
    python3.10-dev \
    git \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Upgrade pip
RUN python3 -m pip install --upgrade pip

# Install PyTorch with CUDA 11.8 support
RUN pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118

# Install required packages
RUN pip3 install \
    huggingface-hub \
    datasets \
    fair-esm \
    transformers \
    accelerate \
    pytorch-lightning \
    hydra-core \
    wandb \
    pyarrow \
    tables

# Default command
CMD ["/bin/bash"]
```

## 2. Build Docker Image

```bash
docker build -t esm2:latest .
```

## 3. Convert to Apptainer/Singularity (via Docker Daemon)

```bash
singularity build esm2.sif docker-daemon://esm2:latest
```

Or using the `apptainer` command (identical syntax):

```bash
apptainer build esm2.sif docker-daemon://esm2:latest
```

## 4. Run ESM2

### Interactive Shell

```bash
singularity shell --nv esm2.sif
```

### Run a Python Script

```bash
singularity exec --nv esm2.sif python3 my_esm2_script.py
```

### Bind-Mount a Local Data Directory

```bash
singularity exec --nv -B /path/to/data:/data esm2.sif python3 /data/run_esm2.py
```


`--nv` enables NVIDIA GPU passthrough inside the container.

## Notes

- The Dockerfile uses `nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04` as the base image with PyTorch compiled for CUDA 11.8.
- Both `fair-esm` (Meta's original library) and `transformers` (Hugging Face) are installed, so you can use either API.
- Model weights are downloaded automatically on first use. To cache them inside the container's bound directory, set `HF_HOME`:
  ```bash
  singularity exec --nv -B /path/to/data:/data --env HF_HOME=/data/hf_cache esm2.sif python3 your_script.py
  ```
- For the full list of ESM2 models and usage, see the [ESM GitHub repo](https://github.com/facebookresearch/esm).
