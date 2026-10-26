# Apptainer container for AutoDock Vina

Molecular docking: score and optimise the pose of a small-molecule ligand against a rigid
(or partly flexible) receptor. Packaged with **Meeko**, which is what turns an ordinary
structure file into the PDBQT format Vina actually reads, and **RDKit**, which is Meeko's
chemistry backend.

- Upstream: <https://github.com/ccsb-scripps/AutoDock-Vina> (v1.2.7)
- Meeko: <https://github.com/forlilab/Meeko> (0.8.0)

## This container does not use a GPU

Vina is a CPU docking engine. It scales with cores, not cards.

```
apptainer build --fakeroot vina.sif vina.def
```

Run it **without `--nv`**, and ask your scheduler for CPUs rather than a GPU.
`--exhaustiveness` and `--cpu` are the knobs that matter. (AutoDock-GPU is a different
program and is not packaged here.)

It is also the only image in this repository that is **architecture-portable**: upstream
publishes static Linux binaries for x86_64 *and* aarch64, and the definition picks by
`uname -m`. So this one builds and runs on an ARM node such as a GH200. Nothing
cross-builds — build on the architecture you will run on.

| Build argument | Values | Default | Effect |
|---|---|---|---|
| `VINA_PYTHON` | `yes`, `no` | `yes` | The Python API. On x86_64 it installs from a wheel; on ARM there is no wheel, so it compiles and keeps Boost and SWIG. `no` gives a smaller image with the command-line tools intact. |

## The workflow

Vina reads PDBQT, which nothing else writes, so preparation is a real step rather than a
formality.

```
# 1. ligand: SDF / MOL2 / SMILES -> PDBQT
apptainer exec --bind "$PWD":/data vina.sif \
    mk_prepare_ligand.py -i /data/ligand.sdf -o /data/ligand.pdbqt

# 2. receptor: PDB / mmCIF -> PDBQT. Strip waters and unwanted heteroatoms FIRST.
apptainer exec --bind "$PWD":/data vina.sif \
    mk_prepare_receptor.py -i /data/receptor.pdb -o /data/receptor -p \
    --box_center 11.0 22.0 33.0 --box_size 20 20 20

# 3. dock
apptainer exec --bind "$PWD":/data vina.sif \
    vina --receptor /data/receptor.pdbqt --ligand /data/ligand.pdbqt \
         --center_x 11.0 --center_y 22.0 --center_z 33.0 \
         --size_x 20 --size_y 20 --size_z 20 \
         --exhaustiveness 32 --cpu 16 --seed 42 \
         --out /data/docked.pdbqt

# 4. split the multi-model output into one file per pose, and convert back to SDF
apptainer exec --bind "$PWD":/data vina.sif vina_split --input /data/docked.pdbqt
apptainer exec --bind "$PWD":/data vina.sif \
    mk_export.py /data/docked.pdbqt -s /data/docked.sdf
```

Scores are in the `REMARK VINA RESULT` lines of the output PDBQT, best first, in kcal/mol.

## Three things that decide whether the answer means anything

**The search box is the result.** Vina reports the best pose *inside the box you gave it*,
with a confident-looking number, whether or not the box contains the real site. A box
centred on the wrong pocket does not fail — it returns a plausible affinity for the wrong
place. Get the centre from a known ligand, a pocket-detection tool, or an interface
prediction ([PeSTo](../PeSTo/) is in this repository and is quick), and keep the box as
small as the site allows: a larger box is both slower and less accurate, because the search
has more irrelevant volume to cover.

**Match `--cpu` to your allocation.** Vina defaults to every core it can see, which inside
a container is every core on the *node*, not the cores your job was given. On a shared node
that means oversubscribing and slowing everyone down, your own job included. Pass `--cpu`
explicitly, equal to `--cpus-per-task`.

**`--exhaustiveness` trades time for reproducibility, not for accuracy.** It is the number
of independent Monte Carlo runs. The default of 8 is low for anything but a smoke test; 32
is a reasonable working value. Fix `--seed` if you want a run you can repeat — without it
the same input gives slightly different scores each time, and that variation is easily
mistaken for a real difference between ligands.

## Packaging notes

Two things that bite on a rebuild, both discovered the hard way here:

- **Meeko declares no dependencies at all.** Its wheel metadata has an empty
  `requires_dist`, so `pip install meeko` installs nothing else and then `import meeko`
  fails on whichever package is missing. The definition lists them explicitly: `rdkit`,
  `numpy`, `scipy`, `gemmi`, `pandas`, `prody`. `gemmi` is the first one you hit on import;
  `prody` is needed only by `mk_prepare_receptor.py`, so a build without it looks fine right
  up to the first receptor. Not installed: `torch`, `openff`/`espaloma` and
  `AutoDockTools`, which Meeko imports only on optional paths.
- **`vina` 1.2.7 does publish manylinux wheels** for cp38–cp312 on x86_64, with Boost
  vendored in. An earlier version of this definition forced a source build with
  `--no-binary :all:`, which needed Boost and SWIG, left a broken import after the dev
  packages were purged, and intermittently produced corrupt object files under Apptainer's
  `--fakeroot` (`libfakeroot internal error: payload not recognized!`, then
  `ld: file format not recognized` on a `.o`). None of that is necessary on x86_64.

The container self-test runs a genuine docking: RDKit builds aspirin from SMILES, Meeko
writes both PDBQT files, Vina docks and the result is checked for a `REMARK VINA RESULT`
line. A passing `vina --version` would prove none of that.

## Verified

The build itself docks: RDKit builds aspirin from SMILES, Meeko writes the ligand and
receptor PDBQT, and Vina returns nine ranked poses with `REMARK VINA RESULT` scores. The
Python bindings (`import vina`) and Meeko both import in the finished image.

Not checked here: docking accuracy on a benchmark set. This image is verified as working
plumbing, not as a validated docking protocol.

## Acknowledgements

```bibtex
@article{eberhardt2021vina,
  title   = {AutoDock Vina 1.2.0: New Docking Methods, Expanded Force Field, and Python
             Bindings},
  author  = {Eberhardt, Jerome and Santos-Martins, Diogo and Tillack, Andreas F. and
             Forli, Stefano},
  journal = {Journal of Chemical Information and Modeling},
  volume  = {61},
  pages   = {3891--3898},
  year    = {2021},
  doi     = {10.1021/acs.jcim.1c00203}
}

@article{trott2010vina,
  title   = {AutoDock Vina: improving the speed and accuracy of docking with a new scoring
             function, efficient optimization, and multithreading},
  author  = {Trott, Oleg and Olson, Arthur J.},
  journal = {Journal of Computational Chemistry},
  volume  = {31},
  pages   = {455--461},
  year    = {2010},
  doi     = {10.1002/jcc.21334}
}
```

## Legal Notice

AutoDock Vina is Apache-2.0; see [LICENSE.md](LICENSE.md). Meeko is LGPL-2.1 and RDKit is
BSD-3-Clause, each under its own terms. Original:
<https://github.com/ccsb-scripps/AutoDock-Vina>
