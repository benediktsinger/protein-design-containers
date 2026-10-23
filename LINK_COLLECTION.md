###  Link collection for available Methods

Feel free to submit  a PR in case you think somehting important is missing! This list is non-exhaustive and to the best of my own knowledge

#### Pipelines (combining two or more methods)
- **ColabDesign**: https://colab.research.google.com/github/sokrypton/ColabDesign/blob/main/rf/examples/diffusion_ori.ipynb
- **ProteinMPNN + ColabFold**: https://huggingface.co/spaces/simonduerr/ProteinMPNN
- **BindCraft** (AF2-backpropagation): https://colab.research.google.com/github/martinpacesa/BindCraft/blob/main/notebooks/BindCraft.ipynb

#### Structural Design
- **RFD4-Proteina** (Oct 2026, IPD + NVIDIA; joint sequence+structure, one model for binders, antibodies, enzymes and small-molecule pockets): https://github.com/RosettaCommons/RFD4-Proteina | weights https://huggingface.co/nvidia/RFD4-Proteina | hosted https://build.nvidia.com/ipd/rfd4-proteina
- **RFDiffusion3**: https://huggingface.co/spaces/hugging-science/RFdiffusion3
- **AFDesign**: https://colab.research.google.com/github/sokrypton/ColabDesign/
- **RFDiffusion** (conditional diffusion possible): https://colab.research.google.com/github/sokrypton/ColabDesign/blob/main/rf/examples/diffusion.ipynb
- **ProteinGenerator** (only unconditional, but you can write in the sequence): https://huggingface.co/spaces/merle/PROTEIN_GENERATOR
- **EvoDiff**: https://huggingface.co/spaces/colbyford/evodiff
- **Protpardelle**: https://huggingface.co/spaces/ProteinDesignLab/protpardelle
- **Chroma** (Generate Bio, supports conditioning): https://colab.research.google.com/github/generatebio/chroma/blob/main/notebooks/chroma_demo.ipynb
- **RFDiffusion2 via NIM**: https://build.nvidia.com/ipd/rfdiffusion

#### Inverse Folding
- **CARBonAra**: https://huggingface.co/spaces/LBM-EPFL/CARBonAra
- **ProteinMPNN**: https://huggingface.co/spaces/simonduerr/ProteinMPNN
- **ProstT5** (3Di token-based, sequence↔structure): https://huggingface.co/Rostlab/ProstT5

#### In Silico Validation
- **ColabFold** (enhanced AlphaFold2): https://colab.research.google.com/github/sokrypton/ColabFold/blob/main/AlphaFold2.ipynb
- **AlphaFold3** (requires access): https://golgi.sandbox.google.com/
- **AlphaFold-Multimer**: https://colab.research.google.com/github/sokrypton/ColabFold/blob/main/beta/AlphaFold2_complexes.ipynb
- **OpenFold** (open reimplementation of AF2): https://colab.research.google.com/github/aqlaboratory/openfold/blob/main/notebooks/OpenFold.ipynb
- **Chai-1**: https://lab.chaidiscovery.com/ or https://huggingface.co/chaidiscovery/chai-1
- **Boltz-1**: https://huggingface.co/spaces/simonduerr/boltz-1
- **Boltz-2** (with affinity prediction): https://huggingface.co/spaces/simonduerr/boltz-2
- **ESMFold2** (Arc/Biohub, 2026; ESMC representations → all-atom complexes, used for binder design): https://biohub.org/news/world-model-of-protein-biology/
- **ESMFold**: https://colab.research.google.com/github/sokrypton/ColabFold/blob/main/ESMFold.ipynb
- **OmegaFold**: https://colab.research.google.com/github/sokrypton/ColabFold/blob/main/beta/omegafold.ipynb
- **ESMFold via NIM**: https://build.nvidia.com/meta/esmfold
- **AlphaFold2 via NIM**: https://build.nvidia.com/ipd/alphafold2
- **Boltz2 via NIM**: https://build.nvidia.com/mit/boltz2

#### Propietary models (Fun to play around with but useless)
- **Latent Labs** (fully integrated pipeline - binders, mabs): https://www.latentlabs.bio/
- **Tamarind Bio** (pay-per-use API system): https://www.tamarind.bio/

#### Notes on this list
Dates in brackets are release dates, not paper dates. Anything marked here as new has not
necessarily been benchmarked by us. For what is containerised here, see
[README.md](README.md).
