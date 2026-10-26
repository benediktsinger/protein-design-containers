# Apptainer container for PeSTo

PeSTo (Protein Structure Transformer) predicts protein **binding interfaces** from a
structure alone. No MSA, no evolutionary input, and — the unusual part — no parameters
describing the atoms beyond their chemical element. That makes it fast enough to run over
every frame of an MD trajectory, which is what it was built for.

- Upstream: <https://github.com/LBM-EPFL/PeSTo>
- Paper: Krapp, Abriata, Cortés Rodriguez & Dal Peraro, *Nat. Commun.* **14**, 2175 (2023)

Model weights ship inside the repository (a few MB), so the image needs no network and
nothing to stage.

## Three builds, and which to pick

```
apptainer build --fakeroot --build-arg PESTO_ACCELERATOR=cpu  pesto-cpu.sif  pesto.def
apptainer build --fakeroot --build-arg PESTO_ACCELERATOR=cu12 pesto-cu12.sif pesto.def
apptainer build --fakeroot --build-arg PESTO_ACCELERATOR=cu13 pesto-cu13.sif pesto.def
```

**Start with `cpu`.** A single structure is seconds on a laptop core, so a GPU only pays
for a large batch or a trajectory — and the CPU image is the only one that runs anywhere
without first checking a driver. `cpu` is the default if you pass no build argument.

For the GPU builds, pick by the **driver** of the machine you will run on, not by its GPU.
The driver caps the CUDA major version; minor versions are forward compatible, major
versions are not. A `cu13` image on a 560-series driver (CUDA 12.6) aborts at CUDA init
with `The NVIDIA driver on your system is too old`, before anything looks at the card. See
the [top-level README](../README.md) for all three gates.

## Running

```
# every PDB in a directory, on CPU
apptainer run --bind "$PWD":/data pesto-cpu.sif /data/structures --output_dir /data/out

# one file, on a GPU
apptainer run --nv --bind "$PWD":/data pesto-cu12.sif /data/7a3n.pdb \
    --output_dir /data/out --device cuda
```

Output is one PDB per predicted interface type, named `<input>_i<N>.pdb`, with the
per-residue interface probability (0–1) in the **b-factor column**. Visualise it in PyMOL
with:

```
spectrum b, blue_white_red, all, 0, 1
```

How many files you get depends on the model variant. The default `i_v4_1` — the variant
used in the paper — distinguishes protein, nucleic acid, lipid, ligand and ion interfaces;
`i_v3_1` predicts protein–protein interfaces only. Pass `--variant` to choose.

| Option | Default | Meaning |
|---|---|---|
| `--output_dir DIR` | beside each input | where to write results |
| `--variant` | `i_v4_1` | `i_v4_1`, `i_v4_0`, `i_v3_1`, `i_v3_0` |
| `--checkpoint` | `model_ckpt.pt` | or `model.pt`, inside the variant directory |
| `--device` | `cpu` | `cpu`, `cuda`, `cuda:0`, … |
| `--nn_topk` | `64` | neighbours kept when extracting topology (upstream's value) |

Inputs that are directories are searched recursively for `*.pdb`, and files matching this
tool's own output pattern `_i<N>.pdb` are skipped, so re-running over an output directory
does not predict on predictions. A file named **explicitly** on the command line is never
skipped.

(Upstream's notebook filters on the bare substring `_i`, which also discards innocent
names — `my_input.pdb`, `protein_ion.pdb`. That is the one place this driver deliberately
does not copy the notebook.)

## The one thing to know before trusting this image

**Upstream ships no command-line interface.** Inference lives in `apply_model.ipynb`, a
notebook, which is miserable to run inside a batch job. So this image installs
`pesto-predict`, a script written to follow that notebook step for step: same model, same
preprocessing, same topology cut-off of 64 neighbours, same b-factor encoding, same
`_i<N>.pdb` naming.

It is a transcription, not a reimplementation — but it is the one piece of code in this
folder that is *ours* rather than upstream's. Two consequences:

1. If upstream changes the notebook, this script is what has to be revisited.
2. Before you rely on a result, compare one structure against the notebook. The container
   self-test runs a real forward pass, which proves the plumbing works; it does not prove
   the numbers match a notebook run.

A second detail, for anyone editing the definition: each `model/save/<variant>` directory is
its own Python package, carrying its own `config.py`, `data_handler.py` and `model.py`
beside the checkpoint. The notebook puts that directory on `sys.path` before importing
them, so choosing a variant changes **which module is imported**, not merely which weights
are loaded. Importing a top-level `model` instead would load the training code.

## Verified

On an L40S (sm_89, 560-series driver / CUDA 12.6), both images predicting the same two
structures from `/opt/PeSTo/pdbs_test`:

- the `i_v4_1` variant writes **5 interface channels** per input, with sensible
  probabilities — the protein–protein channel peaked at 0.96 on the test structure;
- `pesto-cu12` on the GPU and `pesto-cpu` on the CPU gave **identical b-factors**, largest
  difference 0.0000 over ten output files.

That last point is worth having: it means you can develop on the CPU image and scale up on
the GPU one without the numbers moving.

## Acknowledgements

```bibtex
@article{krapp2023pesto,
  title   = {PeSTo: parameter-free geometric deep learning for accurate prediction of
             protein binding interfaces},
  author  = {Krapp, Lucien F. and Abriata, Luciano A. and Cort{\'e}s Rodriguez, Fabio and
             Dal Peraro, Matteo},
  journal = {Nature Communications},
  volume  = {14},
  pages   = {2175},
  year    = {2023},
  doi     = {10.1038/s41467-023-37701-8}
}
```

## Legal Notice

PeSTo is CC BY-NC-SA 4.0 — **non-commercial use only**, and derivative works inherit the
terms. See [LICENSE.md](LICENSE.md). Original:
<https://github.com/LBM-EPFL/PeSTo>
