# Protein Design Containers

A collection of Apptainer (`.def`) and Docker (`Dockerfile`) container definitions for GPU-accelerated protein design and structure prediction tools. Each subfolder is self-contained with a build definition, run instructions, and license.

## Available Containers

| Subfolder | Tool | Task |
|-----------|------|------|
| [AlphaFold3/](AlphaFold3/) | AlphaFold 3 **+ MMseqs2** | Structure prediction (proteins, ligands, nucleic acids), with an MSA front end. Two images, run one after the other. |
| [AutoDockVina/](AutoDockVina/) | AutoDock Vina | Small-molecule docking (**CPU only**, with Meeko + RDKit) |
| [bindcraft/](bindcraft/) | BindCraft | De novo protein binder design (AF2-based) |
| [BindCraft2/](BindCraft2/) | BindCraft2 | Binders, VHHs, antibodies, cyclic peptides, multistate design |
| [boltz2/](boltz2/) | Boltz-2 | Structure prediction + binding affinity |
| [boltzgen/](boltzgen/) | BoltzGen | Generative design of proteins, peptides, and binders |
| [CARBonAra/](CARBonAra/) | CARBonAra | Inverse folding / sequence design, conditioned on non-protein context |
| [Chai/](Chai/) | Chai-1 | Structure prediction (proteins, ligands, DNA, RNA) |
| [esm/](esm/) | ESM2 | Protein language model embeddings and folding |
| [LigandMPNN/](LigandMPNN/) | LigandMPNN | Inverse folding with ligand/DNA context |
| [PeSTo/](PeSTo/) | PeSTo | Binding-interface prediction from structure alone (no MSA) |
| [RFD4-Proteina/](RFD4-Proteina/) | RFD4-Proteina | Joint sequence+structure design: binders, antibodies, enzymes, pockets |
| [RFDiffusion3/](RFDiffusion3/) | RFdiffusion3 | Diffusion-based all-atom biomolecular design |
| [RISoTTo/](RISoTTo/) | RISoTTo | RNA sequence design from a backbone, context-aware |

`bindcraft/` and `BindCraft2/` are **different programs**, not two versions of one: a new
repository, a new workflow and a different licence. Both are kept deliberately.

See [LINK_COLLECTION.md](LINK_COLLECTION.md) for online demos and further tools not yet
containerised here.

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

- Most containers require an NVIDIA GPU (`--nv` / `--gpus all`). The exceptions:
  **AutoDock Vina is CPU-only** — do not pass `--nv`, ask for cores instead — and PeSTo,
  CARBonAra and MMseqs2 all have useful CPU builds.
- **Check both the driver and the GPU architecture before running on a cluster.** These are
  separate constraints and they fail differently:
  - The **driver** caps the CUDA major version. A CUDA 13 image on a CUDA 12.x driver dies
    at CUDA init with `The NVIDIA driver on your system is too old`, before anything looks
    at the GPU. Minor versions are forward compatible; major versions are not.
  - The **architecture** must be covered by the image's compiled arch list, or model init
    fails with `CUDA error: no kernel image is available for execution on the device`.
    CUDA 13 dropped Volta, and CUDA 12.6 has no Blackwell kernels.

  A container self-test catches neither, since it does not touch the driver. This is why
  RFD4-Proteina ships as two images; see [RFD4-Proteina/README.md](RFD4-Proteina/README.md).
- **AlphaFold 3 weights must be requested separately** from Google and may not be
  redistributed, so no built `.sif` of that image can be passed on — see
  [AlphaFold3/README.md](AlphaFold3/README.md).
- BoltzGen and ESM2 download model weights automatically on first run.
- Weights ride *inside* the image for RFD4-Proteina, BindCraft2, CARBonAra, PeSTo and
  RISoTTo, so those run with no network at all.
- Several definitions take a `--build-arg` to choose the accelerator (`cu12`, `cu13`, and
  for some a `cpu` build). The right choice is set by the **driver** of the machine you
  will run on, not by its GPU; see the two bullet points above and each folder's README.

## Legal Notice

This repository contains Apptainer/Docker container definitions that package third-party software. Each tool is subject to its own license; see the `LICENSE.md` file in the respective subfolder for the full license text.

| Tool | License | Source |
|------|---------|--------|
| AlphaFold 3 | Apache 2.0 (**code**); weights under separate, non-redistributable terms | https://github.com/google-deepmind/alphafold3 |
| AutoDock Vina | Apache 2.0 | https://github.com/ccsb-scripps/AutoDock-Vina |
| BindCraft | MIT | https://github.com/martinpacesa/BindCraft |
| BindCraft2 | Source-Available (Hosting-Restricted) | https://github.com/PacesaLab/BindCraft2 |
| Boltz-2 | MIT | https://github.com/jwohlwend/boltz |
| BoltzGen | MIT | https://github.com/hstark/boltzgen |
| Chai-1 | Apache 2.0 | https://github.com/chaidiscovery/chai-lab |
| CARBonAra | CC BY-NC-SA 4.0 | https://github.com/LBM-EPFL/CARBonAra |
| ESM2 | MIT | https://github.com/facebookresearch/esm |
| LigandMPNN | MIT | https://github.com/dauparas/LigandMPNN |
| MMseqs2 | MIT | https://github.com/soedinglab/MMseqs2 |
| PeSTo | CC BY-NC-SA 4.0 | https://github.com/LBM-EPFL/PeSTo |
| RFD4-Proteina | BSD 3-Clause | https://github.com/RosettaCommons/RFD4-Proteina |
| RFdiffusion3 | BSD 3-Clause | https://github.com/RosettaCommons/RFdiffusion |
| RISoTTo | CC BY-NC-SA 4.0 | https://github.com/LBM-EPFL/RISoTTo |

The container build scripts in this repository are provided as-is. Users are responsible
for complying with the licenses of all bundled software. Three cases deserve attention:

- **CARBonAra, PeSTo and RISoTTo are CC BY-NC-SA 4.0** — non-commercial use only, and
  derivative works inherit the terms.
- **AlphaFold 3's code and its weights are licensed differently.** The code is Apache-2.0.
  The weights must be obtained from Google, may not be redistributed, and carry their own
  prohibited-use policy; model outputs carry terms of their own as well. All four documents
  are in [AlphaFold3/](AlphaFold3/).
- **BindCraft2 is source-available, not open source.** Internal and commercial use and
  redistribution are allowed; offering the software itself as a hosted service is not.
  Supplying design *outputs* to others is explicitly permitted.
