FROM nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04

# Set environment variables
ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1
# Set environment variables

# Install system dependencies and Python 3.9+
RUN apt-get update && apt-get install -y \
    python3.10 \
    python3-pip \
    python3.10-dev \
    git \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Upgrade pip
RUN python3 -m pip install --upgrade pip

# Install PyTorch with CUDA support
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

