# How to get the barcodes of the enrichment you used?


## 1. Login via ssh into the WEXAC 
(Connect to the VPN) if you are outside the insititue

## 2. Ensure you have python loaded in your PATH
Type in:

```bash
which python
```

If you see the output like:
```
~/miniconda/envs/env_name/bin/python
```

Or something else similar to it you are good.

## 3. Get you barcodes

Run the following command:

```bash
python ../LAB/scripts/felix_bcl2fastq/demult.py . dna1
```

Where the ```dna1``` is the number of enrichemnt primer you used (Enr1 here) 
You will see in the directory you are a new file named :
```
nova13_ss.csv
```
These are the index sequences!!