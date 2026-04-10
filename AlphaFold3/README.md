# Apptainer container for AlphaFold 3

This is an Apptainer container for AlphaFold 3, Google DeepMind's biomolecular structure prediction system. It predicts 3D structures of protein complexes, nucleic acids, small molecules, and other biomolecules. This build runs inference only — no MSA/template pipeline and no HMMER required.

**Important:** Model weights are NOT included in the container. Each user must request and download them individually from Google (see below).

## Obtaining Model Weights

1. Request access at https://forms.gle/svvpY4u2jsHEwWYS6
2. Once approved, download the model parameters from the provided link

## Building the Container

The def file uses `%files` to copy the AlphaFold3 source into the image, so the AlphaFold 3 source repository must be present in the build directory. Clone it first:

```
git clone https://github.com/google-deepmind/alphafold3.git .
apptainer build --force ./alphafold3.sif ./alphafold3.def
```

## Running the Container

```
apptainer run --nv \
  --bind <path to weights>:/models \
  --bind <path to inputs>:/inputs \
  --bind <path to outputs>:/outputs \
  alphafold3.sif \
    --json_path /inputs/input.json \
    --model_dir /models \
    --output_dir /outputs \
    --jax_compilation_cache_dir /outputs/jax_cache
```

Reuse the same `--jax_compilation_cache_dir` across runs — JAX recompiles on the first run and caches the result, significantly speeding up subsequent runs.

### Input format

AlphaFold 3 takes a JSON file describing the sequences and molecules to predict:

```json
{
  "name": "my_prediction",
  "dialect": "alphafold3",
  "version": 2,
  "modelSeeds": [1],
  "sequences": [
    {
      "protein": {
        "id": ["A"],
        "sequence": "MNIFEMLRIDE...",
        "unpairedMsa": "",
        "pairedMsa": "",
        "templates": []
      }
    }
  ]
}
```

## Acknowledgements

```bibtex
@article{abramson2024accurate,
  title={Accurate structure prediction of biomolecular interactions with AlphaFold 3},
  author={Abramson, Josh and Adler, Jonas and Dunger, Jack and Evans, Richard and Green, Tim and Pritzel, Alexander and Ronneberger, Olaf and Willmore, Lindsay and Ballard, Andrew J and Bambrick, Joshua and others},
  journal={Nature},
  volume={630},
  pages={493--500},
  year={2024},
  publisher={Nature Publishing Group}
}
```

## Legal Notice

AlphaFold 3 CC BY-NC-SA 4.0 License: See LICENSE.md. Original:
https://github.com/google-deepmind/alphafold3
