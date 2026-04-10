FROM nvidia/cuda:12.4.1-runtime-ubuntu22.04

#####################################
# Environment
#####################################
ENV LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PYTHONFAULTHANDLER=1 \
    PYTHONPYCACHEPREFIX=/tmp/.chai_lab_pycache \
    PATH="/opt/venv/bin:${PATH}" \
    CHAI_DOWNLOADS_DIR=/mnt/shared-ro/weights/chai/chai-weights \
    UV_HTTP_TIMEOUT=1000 \
    XDG_CACHE_HOME=/tmp/cache

#####################################
# System packages + Python 3.10
#####################################
RUN set -eux; \
    umask 0000; \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        wget \
        git \
        curl \
        build-essential \
        libstdc++6 \
        python3.10 \
        python3.10-dev \
        python3.10-venv \
        kalign && \
    rm -rf /var/lib/apt/lists/*

#####################################
# Install uv + create venv
#####################################
RUN curl -LsSf https://astral.sh/uv/0.5.4/install.sh | sh && \
    . /root/.local/bin/env && \
    uv venv --python=python3.10 /opt/venv && \
    . /opt/venv/bin/activate && \
    uv pip install pip

#####################################
# Install chai-lab
#####################################
RUN . /root/.local/bin/env && \
    . /opt/venv/bin/activate && \
    uv pip install chai_lab

#####################################
# Cache and working directories
# World-writable for multi-user access
#####################################
RUN mkdir -p /opt/chai-weights /tmp/cache /tmp/.chai_lab_pycache && \
    chmod -R 777 /opt /tmp /root

#####################################
# Default user / workdir
#####################################
WORKDIR /root
CMD ["/bin/bash"]
