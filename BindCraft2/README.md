# Apptainer container for BindCraft2

De novo binder design: miniproteins, scaffolded binders, VHHs/nanobodies, antibodies,
cyclic peptides and multistate design, in one workflow. A binder is optimised through
AlphaFold2, its sequence redesigned with ProteinMPNN, then re-predicted and filtered with
models **held back from the optimisation** — so a design is never scored by a model that
shaped it.

- Upstream: <https://github.com/PacesaLab/BindCraft2> (v1.0.4)

**This is not the same program as [`bindcraft/`](../bindcraft/) in this repository.** That
folder packages the original BindCraft (v1.5.2, MIT, `martinpacesa/BindCraft`). BindCraft2
is a new repository under a new organisation, with a different workflow and a different
licence. Both are kept; pick deliberately.

## Building

```
apptainer build --fakeroot --build-arg BINDCRAFT_ACCELERATOR=cu12 bindcraft2-cu12.sif bindcraft2.def
apptainer build --fakeroot --build-arg BINDCRAFT_ACCELERATOR=cu13 bindcraft2-cu13.sif bindcraft2.def
```

**There is no default.** Pick by the driver of the cluster you will run on, not by its GPU:
the driver caps the CUDA major version, and minor versions are forward compatible while
major versions are not. Upstream's installer makes the same distinction (`install.sh cuda12`
/ `cuda13`) but chooses by reading the driver of whatever machine runs it — which on a build
node is the wrong machine. Hence a required build argument: an unattended build must not
silently pick CUDA 13 because the build node has no GPU at all.

The image is about 12.5 GB (5.3 GB of AlphaFold parameters plus the accelerator wheels), and
the build wants roughly **20 GB of scratch** — the parameter archive and its unpacked
contents coexist before the archive is deleted. Build on node-local disk, not on a shared
filesystem.

**BindCraft2 cannot run on CPU.** Upstream refuses `install.sh cpu` outright, because a
single trajectory folds an AlphaFold ensemble hundreds of times.

## Running

The entry point is `bindcraft`, so arguments start at the subcommand:

```
apptainer run --nv --bind "$PWD":/data bindcraft2-cu12.sif design /data/design.json
```

A minimal `design.json`, with `target.pdb` beside it:

```json
{
  "targets": [{"name": "my_target", "target_path": "/data/target.pdb",
               "chains": "A", "hotspots": "54,56,66-70"}],
  "modality": "binder",
  "binder_lengths": [80, 80],
  "number_of_final_designs": 10,
  "project_folder": "/data/results/my_target"
}
```

Other modalities: `VHH` (drop `binder_lengths` — the scaffold sets the length), `peptide`,
`antibody`. List the shipped example targets with `design --list-targets`.

## Before you start a long campaign

**Two design workers on one GPU crash each other.** This is the first thing to set, and it
cost five of seven campaigns here before it was found.

`workers_per_gpu` defaults to `auto`, which packs as many workers onto a card as its free
memory allows — two on a 48 GB L40S. `attention_backend` also defaults to `auto`, which
probes cuDNN with a tiny attention, finds it works, and selects it. Put those two defaults
together and both worker processes capture cuDNN attention graphs on the same device, where
they fail each other with:

```
jax.errors.JaxRuntimeError: INTERNAL: detail::cuda_graph_end_capture(fake_stream, &graph_obj)
failed with capturing stream has unjoined work
  ... cuda_dnn.cc: 'graph.deserialize(...)'  [executable_name='jit_sequence_design_loss']
```

The campaign does **not** stop. The surviving worker carries on and the campaign can still
finish and accept designs — it just does so at half throughput, and exits non-zero (134,
SIGABRT) at the end. The dead worker is reported in one line of the campaign log and the
traceback goes to `results/workers/<pid>_worker_NN_gpu_0.log`, so a job that looks like it
merely "failed at the end" has in fact been running at half speed for hours.

Measured on an L40S over seven campaigns, same target and settings:

| Configuration | Workers on the card | Campaigns that lost a worker |
|---|---|---|
| `workers_per_gpu=auto`, `attention_backend=auto` (**the defaults**) | 2 | **5 of 7** |
| `workers_per_gpu=1` | 1 | 0 of 2 |
| `attention_backend=chunked`, workers still `auto` | 2 | 0 of 2 |

The third row is the one that identifies the cause: two workers on one GPU are **fine** as
long as they are not both using cuDNN attention. So pick by what you are short of:

```
bindcraft design campaign.json --set 'workers_per_gpu=1'       # safe, half the throughput
bindcraft design campaign.json --set 'attention_backend=chunked'  # keeps both workers, slower kernels
```

`BINDCRAFT_WORKERS_PER_GPU=1` in the environment does the same as the first. The failure is
not deterministic — two of seven runs survived the defaults untouched — so a campaign that
once worked is not evidence that the defaults are safe.

**Confirm JAX actually sees the card.** JAX warns once and then runs on the CPU, which looks
exactly like a working job that will never finish:

```
apptainer exec --nv bindcraft2-cu12.sif \
    python -c "import jax; print(jax.devices()); assert jax.default_backend()=='gpu'"
```

One `CudaDevice` per card is the answer. `[CpuDevice(id=0)]` means stop and fix it.

**Keep the compiled-graph cache for a sweep.** Each prediction shape costs about 60 s to
compile and 1.5 s to read back from cache. The image caches to node-local `/tmp`, which is
lost when the job ends. For a series of campaigns on nodes carrying the same GPU model,
point it at shared storage:

```
apptainer run --nv --env JAX_COMPILATION_CACHE_DIR=/shared/path/bc2_xla_cache ...
```

Use **one cache directory per GPU model**. A compiled executable is not portable across
cards, and mixing models silently defeats the cache rather than failing.

This matters more here than in most images because the container has no `nvidia-smi`:
upstream files its cache by GPU model read from `nvidia-smi`, and falls back to
`${TMPDIR}/bindcraft_xla_cache` when it cannot be reached. So
`JAX_COMPILATION_CACHE_DIR` is set explicitly in the definition rather than left to that
fallback.

**Mount every tree your paths point into**, including anything a symlink crosses. A
parameter or target directory that is a link into an unmounted filesystem exists to `ls`
and not to the campaign, and upstream's preflight reports it as simply missing.

## Weights

Around 5.3 GB of AlphaFold2 parameters are **baked into the image**. `BINDCRAFT_WEIGHTS`
is set to `/opt/bindcraft-weights`, and the parameters themselves — 15 `params_*.npz`
files, the monomer, `_ptm` and `_multimer_v3` sets — sit in
`/opt/bindcraft-weights/alphafold/`. ProteinMPNN's checkpoints already ship inside the
package. So the image runs with no network and nothing to stage.

To use a different copy, set `BINDCRAFT_AF2_PARAMS` to a directory holding
`params_<model>.npz` (i.e. the `alphafold/` level, not its parent); it overrides
everything. `BINDCRAFT_WEIGHTS` is the cache root, one level up.

The build asserts at least seven `params_*.npz` files are present and that none is under
100 MB — a stopped transfer leaves a file with the right name and the wrong size, and that
is the failure worth catching at build time rather than two hours into a campaign.

## Packaging notes

- **The clone is the installation.** `install.sh` performs an editable install, so moving
  or deleting `/opt/BindCraft2` breaks the `bindcraft` command. It stays where it is put,
  and the venv built beside it is added to `PATH`.
- `BINDCRAFT_WEIGHTS` is exported *before* `install.sh` runs, so the parameters land inside
  the image rather than in root's home directory.

## Verified

On an L40S (sm_89, 560-series driver / CUDA 12.6), the `cu12` image:

- `jax.devices()` returns `[CudaDevice(id=0)]` and `jax.default_backend()` is `gpu`;
- `bindcraft design --list-targets` lists the six shipped targets (`dynorphin_a`, `hIL2R`,
  `hIL7RA`, `hPD1`, `hPDL1`, `mPDL1`);
- all 15 AlphaFold parameter files are present and full-size.

**A full campaign has now been run**, against the shipped `hPDL1` target under `--core
benchmark`, and it accepted a design on the third trajectory:

| | |
|---|---|
| Binder | 163 residues, de novo |
| `i_pTM` / `pLDDT` / `pTM` | 0.82 / 0.90 / 0.86 |
| `i_pAE` / `i_pDAE` | 0.21 / 0.31 |
| Interface | 14 binder residues against 16 target residues, 866 Å² buried |
| Hotspot contact fraction | **1.0**, with 0.0 off-epitope contact |
| Clashes | 0 backbone, 7 all-atom |

So the whole workflow runs: AF2 hallucination, ProteinMPNN redesign, re-prediction and
filtering with held-back models, ranking, and the mmCIF and CSV outputs. Ten redesign
candidates were scored to produce that one accepted design.

Three trajectories is a container check, **not** a production campaign. The accepted design
has not been tested against anything, and nine of the ten candidates were filtered out.

**Budget trajectories, not designs.** Across twelve short campaigns against this target
under `--core benchmark`, **4 designs were accepted from 36 trajectories** — roughly one in
nine, and seven of the twelve campaigns accepted nothing at all. A short run that ends with
`0 accepted design(s)` is therefore the *likely* outcome, not a fault: upstream's own
`pdl1.json` asks for 10 designs and the shipped IL-2 example allows 60 trajectories for 5.
Set `max_trajectories` to something like ten times the designs you want, and treat a
campaign that accepts nothing within a handful of trajectories as uninformative rather than
as evidence about the image or the target.

The campaigns behind the cuDNN table above were run at the same time; between them they are
what found that defect.

The `cu13` image has been verified only up to the GPU, on an RTX PRO 6000 Blackwell
(sm_120, driver 580): `jax.devices()` returns `[CudaDevice(id=0)]`, a matmul and softmax
compute on the card, and `--list-targets` runs. **No campaign has been run on Blackwell.**

## Legal Notice

BindCraft2 is under the **BindCraft2 Source-Available Licence (Hosting-Restricted)** — not
MIT, unlike the original BindCraft. In short: free for any internal use including
commercial, and redistribution is allowed; what is restricted is offering the Software
itself to third parties as a hosted or managed service. Providing design **outputs**
(sequences, structures) to others is explicitly permitted. Read
[LICENSE.md](LICENSE.md) before building a product on it.

Original: <https://github.com/PacesaLab/BindCraft2>
