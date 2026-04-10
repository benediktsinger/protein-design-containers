# Apptainer container for BoltzGen

This is an Apptainer container for BoltzGen, a generative protein design tool built on top of Boltz. It supports design of proteins, peptides, nanobodies, and small-molecule binders from YAML design specifications. The container includes all necessary dependencies; model weights (~6 GB) are downloaded automatically at runtime to the HuggingFace cache.

## Building the Container

Requires a linux system with Apptainer installed. You can build the container using the following command:

```
apptainer build --force ./boltzgen.sif ./boltzgen.def
```

## Running the Container

Model weights are downloaded to `$HF_HOME` on first run. Bind a local directory as `/workspace` to persist the cache and outputs across runs:

```
apptainer run --nv \
  --bind "$(pwd)":/workspace \
  --env HF_HOME=/workspace/.cache/huggingface \
  boltzgen.sif \
  run /workspace/spec.yaml \
  --output /workspace/output \
  --protocol protein-anything \
  --num_designs 10 \
  --budget 2
```

`--num_designs` controls the number of intermediate designs generated; in practice use 10,000–60,000 for production runs. `--budget` sets the size of the final diversity-optimised set. Add `--reuse` to resume an interrupted run without losing progress.

## Legal Notice

BoltzGen MIT License: See LICENSE.md. Original:
https://github.com/hstark/boltzgen
