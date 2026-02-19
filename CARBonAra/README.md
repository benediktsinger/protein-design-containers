# Apptainer container for CARBonAra

This is an Apptainer container for CARBonAra, a context-aware geometric deep learning framework for protein sequence design. The container includes all the necessary dependencies including the weights and can be easily deployed on various platforms.

## Building the Container

Requires a linux system with Apptainer installed. You can build the container using the following command:

```
apptainer build --force ./carbonara.sif ./carbonara.def
```

## Running the Container

You can run the container using the following command:

```
apptainer run --nv \                                                                                                                Py base singer@SV-55L-004 18:01:35
  --bind "$(pwd)":/data \
  carbonara.sif \
  --num_sequences 10 \
  --imprint_ratio 0.5 \
 <path to input>.pdb <desired output path>
```

## Acknowledgements

```bibtex
@article{Krapp2024,
  author = {Krapp, Lucien F. and Meireles, Fernando A. and Abriata, Luciano A. and others},
  title = {Context-aware geometric deep learning for protein sequence design},
  journal = {Nature Communications},
  volume = {15},
  pages = {6273},
  year = {2024},
  doi = {10.1038/s41467-024-50571-y}
}
```

## Legal Notice

CARBonAra CC BY-NC-SA 4.0 License: See LICENSE. Original:
https://github.com/LBM-EPFL/CARBonAra
