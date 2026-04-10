# Docker container for Chai-1

This is a Docker container for Chai-1, a multi-modal biomolecular structure prediction model from Chai Discovery. It predicts 3D structures of proteins, small molecules, DNA, RNA, and glycosylations. Model weights are downloaded automatically on first run.

## Building the Container

```bash
docker build -t chai:latest .
```

## Convert to Singularity/Apptainer (via Docker Daemon)

```bash
singularity build chai.sif docker-daemon://chai:latest
```

## Running the Container

### Docker

```bash
docker run --rm --gpus all \
  -v "$(pwd)":/data \
  chai:latest \
  python3 -c "
from pathlib import Path
from chai_lab.chai1 import run_inference
run_inference(
    fasta_file=Path('/data/input.fasta'),
    output_dir=Path('/data/output'),
    num_trunk_recycles=3,
    num_diffn_timesteps=200,
    seed=42,
)
"
```

### Singularity/Apptainer

```bash
apptainer exec --nv \
  --bind "$(pwd)":/data \
  chai.sif \
  python3 -c "
from pathlib import Path
from chai_lab.chai1 import run_inference
run_inference(
    fasta_file=Path('/data/input.fasta'),
    output_dir=Path('/data/output'),
    num_trunk_recycles=3,
    num_diffn_timesteps=200,
    seed=42,
)
"
```

### Input format

Chai-1 takes a FASTA file. Use prefixes to specify molecule type:

```
>protein|name=my_protein
MNIFEMLRIDEGLRLKIYKDTEGYYTIGIGHLLTKSPSLNAAKSELDKAIGRNTGVITKDEAEKLFNQDVTAAAEELGLTQWPMFVVIIAKSATSAAHEEVKPSLL
>ligand|name=my_ligand
N[C@@H](Cc1ccc(O)cc1)C(=O)O
```

Supported prefixes: `protein`, `ligand` (SMILES), `dna`, `rna`.

Model weights are downloaded automatically to `$CHAI_DOWNLOADS_DIR` (default `/tmp/cache`). To persist them across runs, bind-mount a local directory and set the variable:

```bash
--bind /path/to/weights:/weights --env CHAI_DOWNLOADS_DIR=/weights
```

## Acknowledgements

```bibtex
@techreport{chai2024chai1,
  title={Chai-1: Decoding the molecular interactions of life},
  author={{Chai Discovery}},
  year={2024},
  url={https://www.chaidiscovery.com/blog/introducing-chai-1}
}
```

## Legal Notice

Chai-1 Apache 2.0 License: See LICENSE.md. Original:
https://github.com/chaidiscovery/chai-lab
