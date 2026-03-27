###  Link collection for available Methods
#### Pipelines (combining two or more methods)
- **ColabDesign**: https://colab.research.google.com/github/sokrypton/ColabDesign/blob/main/rf/examples/diffusion_ori.ipynb
- **ProteinMPNN + ColabFold**: https://huggingface.co/spaces/simonduerr/ProteinMPNN
- **BindCraft** (AF2-backpropagation): https://colab.research.google.com/github/martinpacesa/BindCraft/blob/main/notebooks/BindCraft.ipynb

#### Structural Design
- **AFDesign**: https://colab.research.google.com/github/sokrypton/ColabDesign/
- **RFDiffusion** (conditional diffusion possible): https://colab.research.google.com/github/sokrypton/ColabDesign/blob/main/rf/examples/diffusion.ipynb *(requires login with a non-EPFL Google account)*
- **RFDiffusion All-Atom** (small molecules, cofactors): https://github.com/baker-laboratory/rf_diffusion_all_atom *(GitHub only)*
- **ProteinGenerator** (only unconditional, but you can write in the sequence): https://huggingface.co/spaces/merle/PROTEIN_GENERATOR
- **EvoDiff**: https://huggingface.co/spaces/colbyford/evodiff
- **Protpardelle**: https://huggingface.co/spaces/ProteinDesignLab/protpardelle
- **Chroma** (Generate Bio, supports conditioning): https://colab.research.google.com/github/generatebio/chroma/blob/main/notebooks/chroma_demo.ipynb
- **Genie2** (unconditional & motif scaffolding): https://github.com/aqlaboratory/genie2 *(GitHub only)*
- **FrameDiff** (SE(3) diffusion): https://github.com/jasonkyuyim/se3_diffusion *(GitHub only)*
- **Latent Labs** (web-based de novo protein design): https://www.latentlabs.bio/

#### Inverse Folding
- **CARBonAra** (made by me - feel free to give feedback): https://huggingface.co/spaces/LBM-EPFL/CARBonAra
- **ProteinMPNN**: https://huggingface.co/spaces/simonduerr/ProteinMPNN
- **LigandMPNN** (ligand- and small molecule-aware): https://github.com/dauparas/LigandMPNN *(GitHub only)*
- **ESM-IF**: https://colab.research.google.com/github/facebookresearch/esm/blob/main/examples/inverse_folding/notebook_multichain.ipynb
- **ProstT5** (3Di token-based, sequence↔structure): https://huggingface.co/Rostlab/ProstT5

#### In Silico Validation
- **ColabFold** (enhanced AlphaFold2): https://colab.research.google.com/github/sokrypton/ColabFold/blob/main/AlphaFold2.ipynb
- **AlphaFold3** (requires access): https://golgi.sandbox.google.com/
- **AlphaFold-Multimer**: https://colab.research.google.com/github/sokrypton/ColabFold/blob/main/beta/AlphaFold2_complexes.ipynb
- **OpenFold** (open reimplementation of AF2): https://colab.research.google.com/github/aqlaboratory/openfold/blob/main/notebooks/OpenFold.ipynb
- **RoseTTAFold2** (improved multimer support): https://github.com/uw-ipd/RoseTTAFold2 *(GitHub only)*
- **RoseTTAFold All-Atom** (proteins + small molecules + nucleic acids): https://github.com/baker-laboratory/RoseTTAFold-All-Atom *(GitHub only)*
- **Chai-1**: https://lab.chaidiscovery.com/ or https://huggingface.co/chaidiscovery/chai-1
- **Boltz-1**: https://huggingface.co/spaces/simonduerr/boltz-1
- **Boltz-2** (with affinity prediction): https://huggingface.co/spaces/simonduerr/boltz-2
- **ESMFold**: https://colab.research.google.com/github/sokrypton/ColabFold/blob/main/ESMFold.ipynb
- **OmegaFold**: https://colab.research.google.com/github/sokrypton/ColabFold/blob/main/beta/omegafold.ipynb

#### NVIDIA BioNeMo / NIM (containerised, API-accessible)
- **BioNeMo Framework** (ESM2, ProteinMPNN, DiffDock, MolMIM and more): https://github.com/NVIDIA/BioNeMo
- **NVIDIA NIM Biology** (hosted API endpoints, no local GPU needed): https://build.nvidia.com/explore/biology
- **ESMFold via NIM**: https://build.nvidia.com/meta/esmfold
- **AlphaFold2 via NIM**: https://build.nvidia.com/ipd/alphafold2
- **DiffDock-L via NIM** (molecular docking): https://build.nvidia.com/mit/diffdock
- **Proteina** (large-scale flow-based backbone generator, fold-class conditioned): https://github.com/NVIDIA-Digital-Bio/proteina *(GitHub only)*
- **La-Proteina** (fully atomistic sequence+structure co-design, up to 800 residues): https://github.com/NVIDIA-Digital-Bio/la-proteina *(GitHub only)* — [paper](https://arxiv.org/abs/2507.09466)
- **Proteina-Complexa** (atomistic binder design for protein & small molecule targets): https://github.com/NVIDIA-Digital-Bio/Proteina-Complexa *(GitHub only)*
