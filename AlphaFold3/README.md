# AlphaFold 3, with an MMseqs2 front end

Two images in one folder, meant to be run one after the other:

| Image | Definition | Job |
|---|---|---|
| `mmseqs2.sif` | [mmseqs2.def](mmseqs2.def) | Search a sequence database, write one A3M per chain, write the AlphaFold 3 input JSON |
| `alphafold3.sif` | [alphafold3.def](alphafold3.def) | Fold it |

They are separate images on purpose. The MSA search is CPU-bound, long, and wants a lot of
RAM and local disk; inference is GPU-bound and short. Splitting them lets you queue each
with the resources it actually needs, and re-fold without re-searching.

**Model weights are not included, and cannot be.** AlphaFold 3's code is Apache-2.0, but
the parameters are released under separate terms that require you to obtain them from
Google directly and forbid redistribution. Request them at
<https://forms.gle/svvpY4u2jsHEwWYS6>, then bind the directory in as `--model_dir`. For the
same reason, a built `.sif` with weights baked in cannot be passed on to anyone — which is
why this definition does not bake them in.

## Building

```
apptainer build --fakeroot mmseqs2.sif    mmseqs2.def
apptainer build --fakeroot alphafold3.sif alphafold3.def
```

Both take build arguments:

| Argument | Values | Default | Effect |
|---|---|---|---|
| `AF3_HMMER` | `no`, `yes` | `no` | `yes` builds HMMER 3.4 with upstream's jackhmmer patch, so AlphaFold's own data pipeline runs in-container. Only useful if you also have the ~630 GB of genetic databases. |
| `MMSEQS_BUILD` | `avx2`, `sse41`, `sse2`, `gpu` | `avx2` | Which upstream release binary to install. MMseqs2 ships one per instruction set and does not detect at run time. |

There is **one** accelerator build of AlphaFold 3, not two. Upstream pins
`jax[cuda12]==0.10.2` with a `uv.lock` to match, and the point of installing `--frozen` is
that this is the tree DeepMind tested. A CUDA 12 image runs on a CUDA 13 driver — a newer
driver runs older runtimes, it is the other direction that fails — so this image is fine on
580-series nodes. Blackwell (sm_100/sm_120) is a different question: jax 0.10's CUDA 12
wheels are built against a recent enough CUDA to cover it, but that has not been verified
on hardware here, so treat an sm_100 card as untested rather than supported.

## The whole run, end to end

### 1. Get a database (once)

```
apptainer exec -B /scratch/db:/db mmseqs2.sif \
    mmseqs databases UniRef90 /db/uniref90 /db/tmp --threads 16
```

`mmseqs databases` lists what it can fetch (UniRef100/90/50, NR, PDB, PDB70, Pfam, and
more). UniRef90 is around 60 GB formatted and is a reasonable default for proteins. Put it
on a filesystem you can read quickly from many nodes, and keep it out of the image.

### 2. Search, and write the JSON

One FASTA record per chain:

```
cat > query.fasta <<'EOF'
>heavy
QVQLVQSGAEVKKPGASVKVSCKAS...
>light
DIQMTQSPSSLSASVGDRVTITCRAS...
EOF

apptainer exec -B /scratch/db:/db:ro -B "$PWD":/data mmseqs2.sif \
    af3-msa-search --threads 32 /data/query.fasta /db/uniref90 /data/msas

apptainer exec -B "$PWD":/data mmseqs2.sif \
    af3-msa-json --fasta /data/query.fasta --msa-dir /data/msas \
        --name my_complex --out /data/input.json
```

`af3-msa-search` writes `msas/<record>.a3m`, one per record, named after the record's first
token. `af3-msa-json` looks them up by that same name, so **keep the FASTA identical
between the two commands**.

Both commands validate the alignment before it reaches AlphaFold, because AlphaFold's
requirements on a supplied MSA are strict and its failure mode is unhelpful. An A3M must
have the query as its first sequence, and with all insertions (lowercase) removed every row
must be exactly as long as the query. A single ragged row makes the whole alignment
unusable, and the error surfaces inside featurisation, long after you queued the job, naming
neither the file nor the row. So ragged rows are dropped here and counted in the log; pass
`--strict` to fail instead.

Watch for `WARNING: <chain> has no homologues at all`. That chain will fold effectively
MSA-free, which for most targets means a much worse prediction, and nothing downstream
will say so again.

### 3. Fold

```
apptainer run --nv \
    -B /path/to/weights:/models:ro -B "$PWD":/data \
    alphafold3.sif \
      --json_path=/data/input.json \
      --model_dir=/models \
      --output_dir=/data/out \
      --norun_data_pipeline \
      --resolve_msa_overlaps=false \
      --jax_compilation_cache_dir=/data/jax_cache
```

### Homomers, and other chain arrangements

Give a repeated sequence once in the FASTA and pass `--copies`:

```
af3-msa-json --fasta query.fasta --msa-dir msas --copies 2,1 \
    --name dimer_plus_partner --out input.json
```

That folds record 1 as two chains (ids `A`, `B`) sharing one alignment, and record 2 as one
chain (`C`). Use `--ids` to choose the letters yourself.

Ligands, ions, nucleic acids and modifications are **not** written by `af3-msa-json`. Add
those entries to the JSON by hand afterwards; see upstream's
[input documentation](https://github.com/google-deepmind/alphafold3/blob/main/docs/input.md).
The protein chains are the part that needs an alignment, which is all this helper is for.

## Four flags that are easy to get wrong

| Flag | Why |
|---|---|
| `--norun_data_pipeline` | How you negate a boolean in absl: prefix `no` to the name. `--run_data_pipeline=false` also works; `--run_data_pipeline false` does not. Required for the default build, which has no HMMER. |
| `--resolve_msa_overlaps=false` | **Required whenever you supply your own `unpairedMsa`.** By default AlphaFold deduplicates the unpaired MSA against the paired one; your `pairedMsa` is `""` but still gets the query added to it, and the deduplication can then reorder or drop rows you placed deliberately. |
| `--jax_compilation_cache_dir` | The first run at each token bucket compiles for minutes. Point this at persistent storage and reuse it — but give **concurrent** jobs separate directories. |
| `--flash_attention_implementation=xla` | Needed on hardware without the fused attention kernels. The default (`triton`) is right for Ampere and later. |

## Input JSON shape

```json
{
  "name": "my_complex",
  "dialect": "alphafold3",
  "version": 2,
  "modelSeeds": [1],
  "sequences": [
    {"protein": {"id": ["A"], "sequence": "MNIFEML...",
                 "unpairedMsaPath": "msas/heavy.a3m",
                 "pairedMsa": "",
                 "templates": []}}
  ]
}
```

`unpairedMsa` and `pairedMsa` must **both** be set or **both** be `null` — there is no
middle state, and supplying one while omitting the other is rejected. Paths in
`unpairedMsaPath` / `pairedMsaPath` resolve relative to the JSON file's own directory, or
absolute.

## Memory settings, and a change from the previous image

The image ships upstream's defaults: `XLA_PYTHON_CLIENT_PREALLOCATE=true` and
`XLA_CLIENT_MEM_FRACTION=0.95`, which fold up to 5,120 tokens on an 80 GB card. If you run
out of VRAM, the escape hatch is unified memory:

```
apptainer run --nv \
  --env XLA_PYTHON_CLIENT_PREALLOCATE=false \
  --env TF_FORCE_UNIFIED_MEMORY=1 \
  --env XLA_CLIENT_MEM_FRACTION=1.5 \
  ...
```

which lets allocations spill into host RAM. It works, and it is slow — tens of minutes
where a fitting model takes one.

**The previous version of this image defaulted to unified memory.** If an old job suddenly
OOMs where it used to merely crawl, that is the change, and the three variables above
restore the old behaviour.

On a compute-capability 7 card (V100) the `XLA_FLAGS` baked into the image is the wrong
one; use `XLA_FLAGS="--xla_disable_hlo_passes=custom-kernel-fusion-rewriter"` instead,
since triton gemm is not supported there at all.

## How good are MMseqs2 alignments here?

Stated plainly: **these are not the alignments AlphaFold 3 was evaluated with.** The
official pipeline is jackhmmer over roughly 630 GB of genetic databases, which is hours of
CPU per target. MMseqs2 does a comparable job in minutes.

Expect well-covered, well-conserved targets to come out much the same. Expect the
difference to show on shallow families, on orphan sequences, and on anything where the MSA
was carrying the prediction rather than decorating it. If a result has to be defensible,
build with `--build-arg AF3_HMMER=yes`, stage the databases with upstream's
`fetch_databases.sh`, and run the official pipeline.

The search defaults here are deliberately sensitive: `--num-iterations 3` (an iterative
profile search, which is what makes MMseqs2 competitive on remote homologues),
`--sensitivity 8.0`, `--max-seqs 10000`, `--evalue 0.1`. An MSA tolerates a few false
positives better than it tolerates being shallow. Anything after the three positional
arguments goes straight to `mmseqs search`:

```
af3-msa-search q.fasta /db/uniref90 msas -- --db-load-mode 2 --split-memory-limit 100G
```

### The GPU MMseqs2 build

`--build-arg MMSEQS_BUILD=gpu` is not a drop-in. The target database has to be rebuilt in
padded form with `mmseqs makepaddedseqdb`, searched with `--gpu 1 --db-load-mode 2`, and it
needs an sm_80 or newer card. It is **untested here**. For one-off jobs the CPU build is
simpler; for a sweep of hundreds of targets it is where the time goes back.

## What `apptainer test` does and does not prove

`mmseqs2.sif` runs a genuine end-to-end search on a toy database built inside the test:
`createdb`, `search`, `result2msa`, `unpackdb`, the rename from the `.lookup` file, the A3M
validator and the JSON writer. If that passes, the pipeline works.

`alphafold3.sif` cannot do the equivalent, because there are no weights in it: the test
imports the compiled extension and the model code, checks that `build_data` produced both
CCD pickles and that a CCD lookup resolves, and confirms the version came from the git tag
rather than setuptools_scm's fallback. It never touches the driver and never runs a forward
pass. **A passing test does not mean the image folds anything on your cluster.**

## Verified end to end

Both images, on one L40S (48 GB, sm_89, 560-series driver / CUDA 12.6), folding ubiquitin
(76 residues) from sequence alone:

| Step | Time | Result |
|---|---|---|
| `mmseqs createdb` on `pdb_seqres` (233 MB FASTA) | 1.5 s | — |
| `af3-msa-search`, 2 iterations, 8 threads | 11 s | **2,301 sequences** kept, 0 dropped as ragged |
| `af3-msa-json` | instant | valid JSON, `unpairedMsaPath` + `pairedMsa: ""` |
| AlphaFold 3 inference, 1 diffusion sample, cold XLA cache | 48 s | `ranking_score` **0.84**, pTM **0.84**, `fraction_disordered` 0.0, 76 CA records |

So the two-step pipeline works on real data, and the whole thing is a minute of GPU time
for a small single chain. Note this was with weights released in November 2024 against code
at v3.0.4 — they loaded and folded without complaint.

## Acknowledgements

```bibtex
@article{abramson2024accurate,
  title   = {Accurate structure prediction of biomolecular interactions with AlphaFold 3},
  author  = {Abramson, Josh and Adler, Jonas and Dunger, Jack and Evans, Richard and
             Green, Tim and Pritzel, Alexander and Ronneberger, Olaf and Willmore, Lindsay
             and Ballard, Andrew J and Bambrick, Joshua and others},
  journal = {Nature},
  volume  = {630},
  pages   = {493--500},
  year    = {2024}
}

@article{steinegger2017mmseqs2,
  title   = {MMseqs2 enables sensitive protein sequence searching for the analysis of
             massive data sets},
  author  = {Steinegger, Martin and S{\"o}ding, Johannes},
  journal = {Nature Biotechnology},
  volume  = {35},
  pages   = {1026--1028},
  year    = {2017},
  doi     = {10.1038/nbt.3988}
}
```

## Legal Notice

AlphaFold 3 **code** is Apache-2.0; see [LICENSE.md](LICENSE.md). The **weights** are not.
They may only be obtained from Google, they may not be redistributed, and their use is
restricted — see [WEIGHTS_TERMS_OF_USE.md](WEIGHTS_TERMS_OF_USE.md) and
[WEIGHTS_PROHIBITED_USE_POLICY.md](WEIGHTS_PROHIBITED_USE_POLICY.md). Model outputs carry
their own terms, [OUTPUT_TERMS_OF_USE.md](OUTPUT_TERMS_OF_USE.md), which among other things
restrict using them to train competing models.

Note that this folder's `LICENSE.md` previously held the CC BY-NC-SA 4.0 text. That was
correct for the original 2024 release and is wrong for v3: the code is now Apache-2.0,
while the restrictions that used to sit in one licence now live in the three weights and
output documents above. If you relied on the old file, re-read those.

MMseqs2 is MIT. The helper scripts in `mmseqs2.def` are part of this repository.
