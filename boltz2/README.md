# Boltz - Docker & Singularity Build Guide

## Build Docker Image

```bash
docker build -t boltz:latest .
```

## Convert to Singularity (via Docker Daemon)

```bash
singularity build boltz.sif docker-daemon://boltz:latest
```

## Run Inference

```bash
singularity run --nv boltz.sif  --env NUMBA_CACHE_DIR=/tmp/numba_cache boltz predict input.yaml --use_msa_server
```

To bind-mount a local data directory:

```bash
singularity run --nv -B /path/to/data:/data boltz.sif boltz predict /data/input.yaml --use_msa_server
```

`--nv` enables NVIDIA GPU passthrough inside the container.

## Notes

- The Dockerfile uses `nvidia/cuda:12.4.0-runtime-ubuntu22.04` as the base image and installs `boltz[cuda]` via pip.
- For all available prediction options: `boltz predict --help`
- Input should be a YAML file (or directory of YAMLs) describing the biomolecules to model. See the [Boltz docs](https://github.com/jwohlwend/boltz) for input format details.
