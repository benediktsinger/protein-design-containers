# Apptainer container for LigandMPNN

This is an Apptainer container for LigandMPNN, a machine learning-based tool for protein sequence design that considers ligands, DNA, and other molecular contexts. The container includes all the necessary dependencies including the weights and can be easily deployed on various platforms.

## Building the Container

Requires a linux system with Apptainer installed. You can build the container using the following command:

```
apptainer build --force ./ligandmpnn.sif ./ligandmpnn.def
```

## Running the Container

You can run the container using the following command:

```
apptainer exec --nv \
  --bind "$(pwd)":/data \
  ligandmpnn.sif \
  python3 /app/ligandmpnn/run.py \
    --model_type "ligand_mpnn" \
    --pdb_path "/data/input.pdb" \
    --out_folder "/data/output"
```

## Acknowledgements

```bibtex
@article{dauparas2025atomic,
  title={Atomic context-conditioned protein sequence design using LigandMPNN},
  author={Dauparas, Justas and Lee, Gyu Rie and Pecoraro, Robert and An, Linna and Anishchenko, Ivan and Glasscock, Cameron and Baker, David},
  journal={Nature Methods},
  pages={1--7},
  year={2025},
  publisher={Nature Publishing Group US New York}
}

@article{dauparas2022robust,
  title={Robust deep learning--based protein sequence design using ProteinMPNN},
  author={Dauparas, Justas and Anishchenko, Ivan and Bennett, Nathaniel and Bai, Hua and Ragotte, Robert J and Milles, Lukas F and Wicky, Basile IM and Courbet, Alexis and de Haas, Rob J and Bethel, Neville and others},
  journal={Science},
  volume={378},
  number={6615},
  pages={49--56},
  year={2022},
  publisher={American Association for the Advancement of Science}
}
```

## Legal Notice

LigandMPNN MIT License: See LICENSE. Original:
https://github.com/dauparas/LigandMPNN
