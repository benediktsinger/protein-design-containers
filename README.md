# Protein Design Containers


## Step 1



## Example command on how to run CARBonAra 

```
apptainer run --nv\
--bind <path to data folder>:/data <path to the apptainer image>/carbonara.sif\
--num_sequences 5\
--imprint_ratio 0.5 /data/<name of input file>.pdb /data/output

```
