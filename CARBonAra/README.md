# Apptainer container for CARBonAra

CARBonAra does **context-aware protein sequence design**. Given a backbone, it predicts
which amino acids fit each position, conditioned on everything else in the structure —
other chains, nucleic acids, ligands, ions, even waters if you leave them in. Unlike most
inverse-folding models it reads no sequence information at all, only geometry and element
identity, and that is what lets it design *around* a non-protein partner.

- Upstream: <https://github.com/LBM-EPFL/CARBonAra>
- Paper: Krapp, Meireles, Abriata *et al.*, *Nat. Commun.* **15**, 6273 (2024)

The weights (6 MB) ship inside the installed package, so the image needs no network and
nothing to stage.

## Three builds

```
apptainer build --fakeroot --build-arg CARBONARA_ACCELERATOR=cu12 carbonara-cu12.sif carbonara.def
apptainer build --fakeroot --build-arg CARBONARA_ACCELERATOR=cu13 carbonara-cu13.sif carbonara.def
apptainer build --fakeroot --build-arg CARBONARA_ACCELERATOR=cpu  carbonara-cpu.sif  carbonara.def
```

`cu12` is the default. Pick a GPU build by the **driver** of the machine you will run on,
not by its GPU: the driver caps the CUDA major version, minor versions are forward
compatible and major versions are not. See the [top-level README](../README.md) for all
three gates.

`cpu` is a real option, not a fallback — the model is 6 MB and a single design is seconds.
It is worth it for a handful of structures, or anywhere you would rather not negotiate a
driver. Sampling many sequences per scaffold is where a GPU starts to pay (the `cpu` image
is also a tenth of the size, 580 MB against 4 GB, since it carries no CUDA libraries).

## Running

```
apptainer run --nv --bind "$PWD":/data carbonara-cu12.sif \
    --device cuda --num_sequences 10 --imprint_ratio 0.5 \
    /data/scaffold.pdb /data/out
```

The positional arguments are the input PDB and an output **directory** (not a file). Each
run writes:

| File | Contents |
|---|---|
| `<name>_scaffold.pdb` | the structure as the model saw it, after cleaning |
| `<name>_pssm.csv` | per-position amino-acid probabilities |
| `<name>_<k>.fasta` | one per sampled sequence, with its score in the header |

Chains are joined by `:` inside a single FASTA record.

### Options worth knowing

| Option | Default | Meaning |
|---|---|---|
| `--num_sequences N` | 1 | sequences to sample |
| `--imprint_ratio R` | 0.5 | how much of the sampled sequence is fed back during iterative sampling, 0–1. Lower is more diverse, higher more self-consistent. |
| `--sampling_method` | `sampled` | How the first, imprint-free prediction becomes a sequence: `sampled` draws from the per-position confidence, `max` takes the argmax. Neither is reproducible — see below. |
| `--known_chains A,B` | — | hold these chains' sequences fixed, design the rest |
| `--known_positions ...` | — | residue **indices** (0-based, over the whole structure) to hold fixed |
| `--unknown_positions ...` | — | the inverse: design only these indices |
| `--ignored_amino_acids C,M` | — | exclude residue types — cysteine-free designs, methionine-free for expression |
| `--ignore_hetatm` | off | drop HETATM records |
| `--ignore_water` | off | drop waters |
| `--device cuda\|cpu` | `cuda` | see the warning below |

**`--ignore_hetatm` is off by default, and that is the point of this model.** Ligands,
ions and nucleic acids are context it designs against; dropping them turns CARBonAra into
an ordinary inverse-folding model.

**`--device cuda` falls back to CPU silently.** With no usable GPU, torch picks CPU without
complaint, so a job that is mysteriously slow is worth checking here first.

## There is no seed, and `max` is not deterministic

This is worth knowing before you build a pipeline on it. Every sampling iteration masks the
imprint at random:

```python
mm = (np.random.rand(yt.shape[0]) <= (1.0 - imprint_ratio))
yt[mm] = 0.0
```

and nothing seeds numpy. `--sampling_method` chooses only how the *first*, imprint-free
prediction is turned into a sequence. So:

- `--sampling_method max --num_sequences 10` gives ten **different** sequences, not ten
  copies of one.
- Re-running the identical command gives different sequences again, on the same hardware.
- Measured on `examples/2oob.pdb` at the default `imprint_ratio=0.5`: three independent
  pairs of runs agreed at **104/117 (88.9%)**, **107/117 (91.5%)** and **109/117 (93.2%)**
  — eight to thirteen residues move between runs.

Two settings are deterministic, because they make the random mask irrelevant:

| Setting | Why |
|---|---|
| `--imprint_ratio 1.0` | nothing is masked, so the imprint is exactly the sampled sequence — the most self-consistent end |
| `--imprint_ratio 0.0` | everything is masked, so every iteration reduces to the imprint-free prediction, and all `N` sequences come out identical |

`imprint_ratio=1.0` was checked rather than assumed: two GPU runs produced byte-identical
sequences, and the CPU image produced the same sequence again — **117/117 across devices**.

Anywhere in between, treat a sequence as a sample rather than a result you can quote and
expect to regenerate. Record the FASTA files you actually got.

## What changed in this definition

The previous version built on `nvidia/cuda:11.8.0-runtime-ubuntu22.04` with an unpinned
`git clone` and no self-test. It now:

- starts from `python:3.11-slim` and lets `uv --torch-backend` keep the CUDA libraries
  inside the torch wheels, which is what makes the `cpu` build small and the whole thing
  base-image-independent;
- pins the upstream commit, so a rebuild is a rebuild rather than a new version;
- offers `cu12`/`cu13`/`cpu` instead of one fixed CUDA;
- runs a real design in `%test` — on `examples/2oob.pdb`, on the CPU, checking all three
  output files appear.

Two notes for anyone editing it:

- **The old "pre-download model weights" line downloaded nothing.** The weights are
  `package_data`, resolved relative to the installed package's `__file__`, so that line was
  only ever a smoke test. The build now asserts the weights, the CDF table and the variant's
  `config.py`/`model.py` all travelled with the package, and that `model.pt` is not a
  truncated file — `package_data` is silent when it misses a pattern, and a broken install
  looks identical to a good one until the first run.
- **numpy is deliberately not pinned.** Sibling models from this group need `numpy<2`
  because they predate the numpy 2 removals; CARBonAra does not. All of `carbonara.py` and
  `src/` was checked for the removed aliases (`np.float`, `np.bool`, `np.int`, `in1d`,
  `row_stack`, `alltrue`) and uses none. If a future rebuild fails inside preprocessing
  with an `AttributeError` on a numpy attribute, repeat that check before adding a pin.

The image installs the `s_v6_4` variant, which is what upstream ships and what the paper
reports. The repository also contains `s_v6_5`, but `setup.py` does not package it, so it is
not in site-packages here.

## Acknowledgements

```bibtex
@article{Krapp2024,
  title   = {Context-aware geometric deep learning for protein sequence design},
  author  = {Krapp, Lucien F. and Meireles, Fernando A. and Abriata, Luciano A. and
             Devillard, Jean and Vacle, Sarah and Marcaida, Maria J. and
             Dal Peraro, Matteo},
  journal = {Nature Communications},
  volume  = {15},
  pages   = {6273},
  year    = {2024},
  doi     = {10.1038/s41467-024-50571-y}
}
```

## Legal Notice

CARBonAra is CC BY-NC-SA 4.0 — **non-commercial use only**, and derivative works inherit
the terms. See [LICENSE.md](LICENSE.md). Original:
<https://github.com/LBM-EPFL/CARBonAra>
