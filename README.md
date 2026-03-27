# Protein Design Containers - copy paste along

## Step 1

Ssh onto your HPC

EPFL
```
ssh user@kuma.hpc.epfl.ch
```

ETHZ
```
ssh user@euler.ethz.ch
```
## Step 2 Reqeust an interactive session

EPFL
```
Sinteract -p l40s -g l40s:1 -m 32GB -c 12 -t 2:00:00 TODO
```

ETHZ
```
srun –pty –gpus=1 bash
```

## Step 3 Download the apptainer in a directory
```
wget TODO

```


## Step 4 Download an example file
```
wget https://files.rcsb.org/view/6VF2.pdb 

```


## Step 5 Run the Carbonara apptainer 

```
apptainer run --nv\
--bind <path to data folder>:/data <path to the apptainer image>/carbonara.sif\
--num_sequences 5\
--imprint_ratio 0.5 /data/<name of input file>.pdb /data/output

```

