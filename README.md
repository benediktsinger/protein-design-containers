# Protein Design Containers

A collection of Apptainer (`.def`) and Docker (`Dockerfile`) container definitions for GPU-accelerated protein design and structure prediction tools. Each subfolder is self-contained with a build definition, run instructions, and license.

## Available Containers

| Subfolder | Tool | Task |
|-----------|------|------|
| [AlphaFold3/](AlphaFold3/) | AlphaFold 3 | Structure prediction (proteins, ligands, nucleic acids) |
| [bindcraft/](bindcraft/) | BindCraft | De novo protein binder design (AF2-based) |
| [boltz2/](boltz2/) | Boltz-2 | Structure prediction + binding affinity |
| [boltzgen/](boltzgen/) | BoltzGen | Generative design of proteins, peptides, and binders |
| [CARBonAra/](CARBonAra/) | CARBonAra | Inverse folding / sequence design |
| [Chai/](Chai/) | Chai-1 | Structure prediction (proteins, ligands, DNA, RNA) |
| [esm/](esm/) | ESM2 | Protein language model embeddings and folding |
| [LigandMPNN/](LigandMPNN/) | LigandMPNN | Inverse folding with ligand/DNA context |
| [RFDiffusion3/](RFDiffusion3/) | RFdiffusion3 | Diffusion-based all-atom biomolecular design |

See [LINK_COLLECTION.md](LINK_COLLECTION.md) for online demos and further tools not yet containerised here.

## How to Get the Already Built Containers

If you don't want to build the containers yourself, feel free to reach out and I can send you the pre-built `.sif` files individually.

## General Usage

### Apptainer containers

Build:
```
apptainer build --force ./<tool>.sif ./<tool>.def
```

Run (GPU passthrough, bind-mount a local data directory):
```
apptainer exec --nv --bind "$(pwd)":/data <tool>.sif <command>
```

See the `README.md` in each subfolder for the tool-specific command.

### Docker containers

Build:
```
docker build -t <tool>:latest .
```

Convert to Singularity/Apptainer if needed:
```
singularity build <tool>.sif docker-daemon://<tool>:latest
```

### Notes

- All containers require an NVIDIA GPU (`--nv` / `--gpus all`).
- AlphaFold 3 weights must be requested separately — see [AlphaFold3/README.md](AlphaFold3/README.md).
- BoltzGen and ESM2 download model weights automatically on first run.

## Legal Notice

This repository contains Apptainer/Docker container definitions that package third-party software. Each tool is subject to its own license; see the `LICENSE.md` file in the respective subfolder for the full license text.

| Tool | License | Source |
|------|---------|--------|
| AlphaFold 3 | CC BY-NC-SA 4.0 | https://github.com/google-deepmind/alphafold3 |
| BindCraft | MIT | https://github.com/martinpacesa/BindCraft |
| Boltz-2 | MIT | https://github.com/jwohlwend/boltz |
| BoltzGen | MIT | https://github.com/hstark/boltzgen |
| Chai-1 | Apache 2.0 | https://github.com/chaidiscovery/chai-lab |
| CARBonAra | CC BY-NC-SA 4.0 | https://github.com/ideaslabut/CARBonAra |
| ESM2 | MIT | https://github.com/facebookresearch/esm |
| LigandMPNN | MIT | https://github.com/dauparas/LigandMPNN |
| RFdiffusion3 | BSD 3-Clause | https://github.com/RosettaCommons/RFdiffusion |

The container build scripts in this repository are provided as-is. Users are responsible for complying with the licenses of all bundled software, including any non-commercial restrictions (e.g. AlphaFold 3 and CARBonAra are CC BY-NC-SA 4.0).
