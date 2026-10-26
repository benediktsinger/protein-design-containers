# Apptainer container for RISoTTo

RISoTTo (RIbonucleic acid Sequence design from TerTiary structure) designs **RNA sequences**
predicted to fold into a given backbone, conditioned on the surrounding molecular context —
proteins, other nucleic acids, ligands, ions. It is the RNA counterpart to
[CARBonAra](../CARBonAra/), from the same group, and the architecture follows it closely.

- Upstream: <https://github.com/LBM-EPFL/RISoTTo>

Weights ship inside the repository (~15 MB), so there is nothing to download at run time.

## Read this before picking a build

```
apptainer build --fakeroot --build-arg RISOTTO_ACCELERATOR=cu12 risotto-cu12.sif risotto.def
apptainer build --fakeroot --build-arg RISOTTO_ACCELERATOR=cu13 risotto-cu13.sif risotto.def
```

The two are **not** equivalent, and the difference is not only CUDA.

Upstream pins `torch==2.3.1` and tests on CUDA 12.1. That build compiles `sm_50` through
`sm_90` — Volta, Turing, Ampere, Ada, Hopper — but it has **no Blackwell kernels**, and no
CUDA 13 build of torch 2.3.1 exists. On an sm_100/sm_120 card it dies at model init with
`CUDA error: no kernel image is available for execution on the device`.

| Build | What it is |
|---|---|
| `cu12` | Exactly upstream's pinned environment, torch 2.3.1. **The faithful build**, and the one to use for anything you intend to publish. |
| `cu13` | Upstream's requirements with the torch pin removed and a current CUDA 13 torch installed instead. The only way to use Blackwell, and **not a configuration upstream has tested.** |

Why that caveat is more than box-ticking: torch minor releases have changed reduction order
and default matmul precision before now, and this model *samples* from a probability
distribution. A changed numeric path can move the sequences it produces — not break, move.

To check one image against the other, diff them **on the CPU**:

```
for sif in risotto-cu12.sif risotto-cu13.sif; do
  apptainer run $sif --pdb_filepath /opt/RISoTTo/test_pdb/1csl.pdb \
      --output_dir /tmp/$sif.out --num_samples 0 --device cpu
done
diff <(grep -A1 '^>seq_0' /tmp/risotto-cu12.sif.out/1csl_designs.fasta) \
     <(grep -A1 '^>seq_0' /tmp/risotto-cu13.sif.out/1csl_designs.fasta)
```

Identical output means the newer torch did not move this model's numerics **on that
structure**. One structure is not a proof; a difference is a clear warning.

The CPU is not a detail here — see the next section for why comparing on the GPU gives you
a diff with no information in it.

## How reproducible is the output, really?

Measured on the shipped `test_pdb/1csl.pdb` (28 nt), against upstream's own
`1csl_designs.fasta`:

| Record | Reproducibility |
|---|---|
| `seq_wt` | **Exact**, every run, both devices. Matches upstream byte for byte, which also means the structure is being parsed the same way. |
| `seq_0` | Nothing random touches it — it comes from the imprint-free prediction, before the sampling loop. But a near-tied position can still flip on a GPU: across **five** `cuda` runs, four reproduced upstream's sequence and one flipped the **last** nucleotide (A→U). Two CPU runs were stable and both matched upstream. |
| `seq_1` … | Sampled, and the imprint mask is drawn randomly per iteration. Different every run by design. |

So treat a one-position difference in `seq_0` as float noise, not as a finding.

**And one hazard in `--num_samples`:** the sampling loop collects *distinct* sequences
(`while len(seqs_set) < num_samples + 1`), discarding repeats, with no iteration cap. For a
short or heavily constrained RNA the model may not produce that many distinct answers, and
then the loop does not fail — it hangs. If a job stops making progress, look there first.

## Running

```
apptainer run --nv --bind "$PWD":/data risotto-cu12.sif \
    --pdb_filepath /data/structure.pdb \
    --output_dir /data/out \
    --num_samples 5 \
    --imprint_ratio 0.5 \
    --sampling max_confidence \
    --device cuda:0
```

Output is a FASTA in `--output_dir`. `seq_wt` is the native sequence read off the
structure and `seq_0` the maximum-confidence design, taken from the imprint-free prediction
before any sampling. `seq_1` onwards are sampled and differ between runs. Each header
carries `recovery` (identity to native) and `confidence` (mean predicted confidence). On
how reproducible each of those is, see below — the short answer is `seq_wt` exactly,
`seq_0` very nearly.

**Include the context you want the design conditioned on** in the input PDB. Proteins,
other nucleic acids, ligands and ions all count, and context is the entire point of the
method — stripping a structure down to the RNA throws away what distinguishes this model
from a sequence-only one.

## Memory

Large complexes are memory-hungry: upstream measured around 8.8 GB peak for a 9,800-atom
structure, which only fits a 12 GB card with `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`.
That is set in the image by default, since it costs nothing on a larger card.

## What is not in the image

**EternaFold is deliberately not built.** It is a git submodule needed only to recompute the
2D self-consistency numbers in the paper's benchmark, not to design sequences, and it needs
a C++ toolchain. The clone in the definition is therefore not recursive.

One packaging note if you rebuild: `parmed`, which upstream pins for its AF3 output
converter, is published on PyPI as an **sdist only** and compiles a C++ extension. The
definition installs `gcc`/`g++` for it and removes them again afterwards, with an import
check placed after the removal — because `apt-get autoremove` is quite capable of taking
something torch needs along with the compiler.

## Acknowledgements

```bibtex
@misc{risotto,
  title  = {RISoTTo: context-aware geometric deep learning for RNA sequence design},
  author = {Laboratory for Biomolecular Modeling, EPFL},
  url    = {https://github.com/LBM-EPFL/RISoTTo}
}
```

## Legal Notice

RISoTTo is CC BY-NC-SA 4.0 — **non-commercial use only**, and derivative works inherit the
terms. See [LICENSE.md](LICENSE.md). Original:
<https://github.com/LBM-EPFL/RISoTTo>
