# Apptainer container for RFD4-Proteina

RFD4-Proteina is a joint sequence-and-structure generative design model from the Institute
for Protein Design and NVIDIA, released 2026-10-21. One model covers monomers, protein and
DNA binders, antibodies and nanobodies, enzymes, and small-molecule pockets. It is the
successor to the RFdiffusion line, combined with NVIDIA's Proteina.

- Upstream code: <https://github.com/RosettaCommons/RFD4-Proteina>
- Weights: <https://huggingface.co/nvidia/RFD4-Proteina> (public, no token needed)
- Docs: <https://rfd4-proteina.org/latest/installation.html>

Model weights and a full Chemical Component Dictionary mirror are baked into the image, so
it runs with no network access.

## Two images, and how to pick

One definition builds either, selected with `--build-arg`:

```
apptainer build --fakeroot --build-arg RFD4_ACCELERATOR=cu12 rfd4-proteina-cu12.sif rfd4-proteina.def
apptainer build --fakeroot --build-arg RFD4_ACCELERATOR=cu13 rfd4-proteina-cu13.sif rfd4-proteina.def
```

Which one you need depends on the **driver** as much as the GPU. There are three independent
gates, and they fail differently:

| Gate | Fails with | Caught by |
|---|---|---|
| Driver major version | `The NVIDIA driver on your system is too old (found version 12060)` | any real job |
| Compiled arch list | `CUDA error: no kernel image is available for execution on the device` | model init |
| Attention kernels | `RuntimeError: Invalid backend` at `scaled_dot_product_attention` | **only a forward pass** |

1. **Driver.** CUDA *minor* versions are forward compatible, major versions are not. A cu13
   image needs a 580-series driver; on a 560-series driver (CUDA 12.6) it aborts at CUDA
   init, before anything looks at the GPU model.
2. **Architecture.** Measured arch lists: cu12 gives `sm_50 60 70 75 80 86 90`, cu13 gives
   `sm_75 80 86 90 100 120`. So Blackwell (sm_100/sm_120) requires cu13, and since CUDA 13
   dropped Volta, sm_70 requires cu12. Note cu13 lists no `sm_89` yet runs fine on an sm_89
   card, because a cubin for `sm_X.y` runs on `sm_X.z` when `z >= y`. Never test for literal
   membership in the arch list.
3. **Attention.** Neither flash nor memory-efficient attention exists below sm_80, so
   **V100-class hardware cannot run this model at all**, even though the cu12 image really
   does ship sm_70 kernels and passes the first two gates.

So, in short: Ada and Hopper take cu12 or cu13 depending on the driver; Blackwell takes
cu13; Ampere takes either; Volta takes neither.

A container self-test cannot see gates 1 or 3, because it neither touches the driver nor
runs a forward pass. **A passing `apptainer test` does not mean the image runs here.**

## Building

Build in a batch job on node-local scratch, then move the finished image to shared storage.
Each build takes about 5.5 minutes and produces a ~13 GiB image: 9.1 GiB of weights plus a
0.5 GiB CCD mirror.

Three things that bite when building as an unprivileged cluster user:

- **`--fakeroot` can work without a subuid mapping.** Even when `grep $USER /etc/subuid` is
  empty, Apptainer 1.4 falls back to a single-uid user-namespace mapping, which is enough
  for `apt-get` and `pip`.
- **Do not add `--ignore-fakeroot-command`.** With it, apt's http method cannot `setgroups`
  and every download fails with
  `Method gave invalid 400 URI Failure message: Failed to setgroups`. A plain
  `apptainer build` with no `--fakeroot` fails the same way.
- **Do not bootstrap from Alpine or any musl base.** Apptainer's injected `fakeroot` helper
  is glibc-linked and dies with
  `exec /.singularity.d/libs/fakeroot failed: a shared library is likely missing in the image`.

## Running

```
apptainer exec --nv \
  -B /path/to/inputs:/inputs:ro -B /path/to/outputs:/outputs \
  rfd4-proteina-cu13.sif \
  rfd4-generate inputs=/inputs/binder.cif num_replicates=2 out_dir=/outputs '~metrics'
```

Designs are written to `<out_dir>/inference_samples/`. Bind `/tmp` through as well if your
site's `/tmp` is node-local, since the image points every writable cache there.

Useful Hydra overrides:

| Override | Effect |
|---|---|
| `'~metrics'` | Skip scoring. **Keep this** unless you staged the refolding oracles. |
| `inference_experiment=quickstart` | Skip scoring *and* the `torch.compile` warm-up. |
| `compile_lengths=null` | Skip only the `torch.compile` warm-up; good for a smoke test. |
| `inference_backend=bioir` | Use the BioIR runtime (cu13 builds only). |
| `num_replicates=N` | Designs per input. |

**Do not combine `'~metrics'` with `inference_experiment=quickstart`.** The quickstart
experiment has no `metrics` entry in its defaults list, so Hydra aborts before the model
loads with `Could not delete 'metrics'. No match in the defaults list`. Use one or the
other: `quickstart` for a smoke test, `'~metrics'` for a normal run.

## Performance notes

Measured on one design from the bundled `protein_binder` example, 256 padded tokens:

| GPU | Generation |
|---|---|
| L40S (Ada, cu12) | 41 s |
| RTX PRO 6000 Blackwell (cu13) | 33 s |

**Prefer Blackwell for large jobs, by more than the clock suggests.** On Ada the log reports

```
cuEquivariance has no fused compiled inference triangle multiplication for
batch=1, n=256, c_z=384, c_hidden=256, bf16=True; using the torch backend
```

so triangle multiplication falls back to plain torch, whereas on Blackwell the
`trimul_sm100_inference` and `trimul_sm120_projection` kernels are selected. Expect the gap
to widen with design size, since triangle multiplication is what dominates there.

The first run at each padded length pays a `torch.compile` warm-up of a few minutes. Setting
`TORCH_COMPILE_CACHE_BLOB` to a file on persistent storage reuses it across jobs. Be careful
with it: torch reads the blob at start and rewrites it at exit, so two concurrent jobs
pointed at one path can clobber each other. Give concurrent jobs separate paths.

Blackwell also emits a wall of `UserWarning: Dynamo detected a call to a functools.lru_cache`
messages from the cuEquivariance kernels. That is upstream noise, not a problem.

## What is and is not in the image

**In:** the `rfd4-proteina` package with the `all-cuda13` or `all-cuda12` extra (TMol plus
the matching `tmol-ops`, fused cuEquivariance and CUTLASS DSL kernels, PoseBusters, and on
cu13 also BioIR); the model weights (9.1 GiB of unique blobs) in `HF_HOME` at
`/opt/rfd4/cache/huggingface`; and a full CCD mirror (51,494 components, 496 MiB) at
`/opt/ccd_mirror`.

**Out:** the refolding oracles. The weights repo is 96.8 GiB, of which 89.8 GiB is prebuilt
runtime images: `alphafold2/` (49.2 GiB) and `openfold3/` (23.5 GiB) are the optional
scoring oracles, and `runtime/` (17.1 GiB) is NVIDIA's own prebuilt RFD4 `.sif`/`.sqsh`.
None belongs inside another container. This is why `'~metrics'` is the default above:
without a refolding oracle, scoring has nothing to run.

Three details about the baked data that are easy to get wrong when rebuilding:

- **Both Hub revisions are cached, deliberately.** `inference.yaml` pins `hf_ckpt_revision`
  to a commit that is *not* current `main`. Caching only `main` gives an image that imports
  fine and then fails on first generation under `HF_HUB_OFFLINE=1` with
  `OfflineModeIsEnabled`. The definition reads the pin out of the downloaded config rather
  than hardcoding it, so it keeps working when upstream bumps it.
- **The CCD mirror must be the exploded RCSB layout**, `<root>/<code[0]>/<code>/<code>.cif`,
  not a single `components.cif`. atomworks resolves codes by that exact path and its mirror
  scan ignores anything else, so a concatenated dictionary reads as an *empty* mirror:
  generation completes diffusion and then dies in structure export with
  `Cannot load CCD component 'SER'`. Pointing `CCD_MIRROR_PATH` at the file instead of the
  directory is a separate trap that raises `NotADirectoryError`. A `.ccd_codes_cache` file
  is written so jobs skip a 51k-entry directory walk at startup.
- **Do not fetch the weights with `hf download --exclude "a/*" "b/*" "c/*"`.** The CLI's
  `--exclude` takes one value and treats the rest as positional filenames, i.e. an allow
  list, so that form downloads exactly the directories you meant to skip and none of the
  weights. It warns and exits 0, so it looks like success. The definition uses the Python
  API, where the argument is named.

No PDB mirror is included, since a full one is far too large for a container. atomworks
notes its absence at startup; the notice is harmless, and it is irrelevant when you pass
your own CIF or YAML. If you need to resolve inputs by PDB ID, bind a mirror in and set
`PDB_MIRROR_PATH`.

If you ever need to run without the CCD mirror, atomworks falls back to biotite's bundled
dictionary when `CCD_MIRROR_PATH` is unset **and** `ALLOW_BIOTITE_CCD=1`. That is an escape
hatch, not the supported path.

### Using NVIDIA's prebuilt image instead

NVIDIA publishes `runtime/rfd4-proteina-cu13-amd64.sif` (3.8 GiB) in the weights repo. It is
a legitimate shortcut if you do not want to build, but it ships **without** weights or a CCD
mirror, so you trade a 5-minute build for staging ~7 GiB of checkpoints, building the CCD
mirror yourself, and managing `HF_HOME`. The definition here exists so the image is
self-contained and reproducible from source.

### ARM / Grace-Hopper

Upstream qualifies only Linux x86_64, but every dependency does publish an `aarch64` wheel:
`tmol` and `tmol-ops-cu13` (manylinux_2_28_aarch64), `cuequivariance-ops-cu13`, `bionemo-ir`
(manylinux_2_34_aarch64), and Torch `+cu130`. A GH200 build therefore looks plausible. It is
untested here, and note that an x86 `.sif` will not run on ARM, so it needs its own build.

## Acknowledgements

```bibtex
@article{rfd4proteina2026,
  title   = {RFD4-Proteina: joint sequence and structure generative design of biomolecular interactions},
  author  = {Institute for Protein Design and NVIDIA GenAIR},
  year    = {2026},
  note    = {Preprint, 116 pp., not yet peer reviewed},
  url     = {https://github.com/RosettaCommons/RFD4-Proteina}
}

@article{geffner2025proteina,
  title   = {Proteina: Scaling flow-based protein structure generative models},
  author  = {Geffner, Tomas and Didi, Kieran and Zhang, Zuobai and Reidenbach, Danny and
             Cao, Zhonglin and Yim, Jason and Geiger, Mario and Dallago, Christian and
             Kucukbenli, Emine and Vahdat, Arash and Kreis, Karsten},
  journal = {ICLR},
  year    = {2025}
}

@article{butcher2025_rfdiffusion3,
  title   = {De novo Design of All-atom Biomolecular Interactions with RFdiffusion3},
  author  = {Butcher, Jasper and Krishna, Rohith and Mitra, Raktim and Brent, Rafael Isaac and
             Li, Yanjing and Corley, Nathaniel and Baker, David},
  journal = {bioRxiv},
  doi     = {10.1101/2025.09.18.676967},
  year    = {2025}
}
```

## Legal Notice

RFD4-Proteina is BSD 3-Clause; see [LICENSE.md](LICENSE.md). The model weights carry their
own terms on the Hugging Face repository — check them before any commercial use.
