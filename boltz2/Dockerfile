# Use NVIDIA CUDA base image compatible with Docker 12.4
FROM nvidia/cuda:12.4.0-runtime-ubuntu22.04

# Set environment variables to avoid interactive installation
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=UTC

# Install Python and other dependencies
RUN apt-get update && apt-get install -y \
    python3.10 \
    python3-pip \
    build-essential \
    cmake \
    libboost-all-dev \
    wget \
    git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Create a symbolic link for python
RUN ln -sf /usr/bin/python3.10 /usr/bin/python

# Configure pip to use proxy

# Now install boltz
RUN pip install boltz[cuda] -U

# Add Docker 12.4 specific configurations
ENV NVIDIA_DRIVER_CAPABILITIES=compute,utility
ENV NVIDIA_VISIBLE_DEVICES=all

