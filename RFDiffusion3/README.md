# Apptainer container for RFDiffusion3

This is an Apptainer container for RFDiffusion3, a software package for molecular modeling and drug discovery. The container includes all the necessary dependencies including the weights and can be easily deployed on various platforms.

## Building the Container

Requires a linux system with Apptainer installed. You can build the container using the following command:

```
apptainer build --force ./rfdiffusion3.sif ./rfdiffusion3.def
```

## Running the Container

You can run the container using the following command:

```
apptainer exec --nv \
  -B <path your input folder>:/inputs \
  -B <path your output folder>:/outputs \
  rfdiffusion3.sif \
  rfd3 design out_dir=/outputs/<specify output folder> inputs=/inputs/<input filename>.json skip_existing=False dump_trajectories=True prevalidate_inputs=True ++ckpt_path=/root/.foundry/checkpoints/rfd3_latest.ckpt
```

## Acknowledgements

```bibtex
@article{corley2025accelerating,
  title={Accelerating biomolecular modeling with atomworks and rf3},
  author={Corley, Nathaniel and Mathis, Simon and Krishna, Rohith and Bauer, Magnus S and Thompson, Tuscan R and Ahern, Woody and Kazman, Maxwell W and Brent, Rafael I and Didi, Kieran and Kubaney, Andrew and others},
  journal={bioRxiv},
  year={2025}
}

@article {butcher2025_rfdiffusion3,
    author = {Butcher, Jasper and Krishna, Rohith and Mitra, Raktim and Brent, Rafael Isaac and Li, Yanjing and Corley, Nathaniel and Kim, Paul T and Funk, Jonathan and Mathis, Simon Valentin and Salike, Saman and Muraishi, Aiko and Eisenach, Helen and Thompson, Tuscan Rock and Chen, Jie and Politanska, Yuliya and Sehgal, Enisha and Coventry, Brian and Zhang, Odin and Qiang, Bo and Didi, Kieran and Kazman, Maxwell and DiMaio, Frank and Baker, David},
    title = {De novo Design of All-atom Biomolecular Interactions with RFdiffusion3},
    elocation-id = {2025.09.18.676967},
    year = {2025},
    doi = {10.1101/2025.09.18.676967},
    publisher = {Cold Spring Harbor Laboratory},
    URL = {https://www.biorxiv.org/content/early/2025/11/19/2025.09.18.676967},
    eprint = {https://www.biorxiv.org/content/early/2025/11/19/2025.09.18.676967.full.pdf},
    journal = {bioRxiv}
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

@article{dauparas2025atomic,
  title={Atomic context-conditioned protein sequence design using LigandMPNN},
  author={Dauparas, Justas and Lee, Gyu Rie and Pecoraro, Robert and An, Linna and Anishchenko, Ivan and Glasscock, Cameron and Baker, David},
  journal={Nature Methods},
  pages={1--7},
  year={2025},
  publisher={Nature Publishing Group US New York}
}
```

## Legal Notice

RFdiffusion BSD License: See LICENSE. Original:
https://github.com/RosettaCommons/RFdiffusion

