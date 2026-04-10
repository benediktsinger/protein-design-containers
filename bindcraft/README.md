# Apptainer container for BindCraft

This is an Apptainer container for BindCraft, a one-shot protein binder design tool based on AlphaFold 2 and ProteinMPNN. It generates de novo protein binders against a target structure and filters them by predicted folding quality. The container includes all necessary dependencies and AlphaFold 2 weights.

## Building the Container

Requires a linux system with Apptainer installed. You can build the container using the following command:

```
apptainer build --force ./bindcraft-v1.5.2.sif ./bindcraft-v1.5.2.def
```

## Running the Container

```
apptainer exec --nv \
  --bind "$(pwd)":/data \
  bindcraft-v1.5.2.sif \
  python /usr/local/apps/BindCraft/bindcraft.py \
    --settings /data/settings.json \
    --filters /usr/local/apps/BindCraft/settings/filters/default_filters.json \
    --advanced /usr/local/apps/BindCraft/settings/advanced_settings.json
```

The `settings.json` file specifies the target PDB, binder length, and output directory. Example settings files are available in the BindCraft repository.

## Acknowledgements

```bibtex
@article{pacesa2024bindcraft,
  title={BindCraft: one-shot design of functional protein binders},
  author={Pacesa, Martin and Nickel, Lennart and Schmidt, Patrick and Pyber, Melanie and Spratt, Alexandra and Cha, Hana and Liebschner, Dorothee and Balle, Laila and Hinze, Georg and Ovchinnikov, Sergey and others},
  journal={bioRxiv},
  year={2024},
  publisher={Cold Spring Harbor Laboratory}
}
```

## Legal Notice

BindCraft MIT License: See LICENSE.md. Original:
https://github.com/martinpacesa/BindCraft
