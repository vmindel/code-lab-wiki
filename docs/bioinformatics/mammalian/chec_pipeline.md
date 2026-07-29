# ChEC-seq Pipeline

A Snakemake pipeline that processes CheC-seq fastqs through alignment, peak
calling, bedgraph/summary generation, and QC — for hg38 and mm10 genomes.

All dependencies (bowtie2, samtools, bedtools, cutadapt, macs3, deeptools,
and the Python packages used by the QC/analysis scripts) are pinned in a
single `environment.yaml` and built automatically by Snakemake. **No
`module load` commands and no personal package installs are required.**

## 1. One-time setup (per user, only needs to be done once)

Snakemake's `--use-conda` flag needs a reasonably modern conda (>=24.7.1) and
Snakemake 8+ with the LSF-compatible cluster-generic executor plugin. Create
a small dedicated conda env for this — it doesn't touch or replace anything
else you already have installed:

```bash
conda create -n snakemake_runner -c conda-forge -c bioconda "conda>=24.7.1" "snakemake>=8" -y
conda activate snakemake_runner
pip install snakemake-executor-plugin-cluster-generic
```

You only need to do this once. From then on, just `conda activate
snakemake_runner` whenever you want to run the pipeline.

## 2. Copy the pipeline files

The template lives here on the cluster:

```
/home/labs/barkailab/LAB/data/MammalianCompendium/Unsorted_processed/checseq_pipeline_template/
```

Copy the variant you need (`hg38/` for human celllines hg38 assembly, `mm10/` for mosue mm10) *together
with* the shared `environment.yaml` one level up, into your own working
folder. For example:

```bash
mkdir -p ~/my_run
cp -r /home/labs/barkailab/LAB/data/MammalianCompendium/Unsorted_processed/checseq_pipeline_template/environment.yaml ~/my_run/
cp -r /home/labs/barkailab/LAB/data/MammalianCompendium/Unsorted_processed/checseq_pipeline_template/hg38 ~/my_run/
```

!!! danger
    For now we run all the data in the `/home/labs/barkailab/LAB/data/MammalianCompendium/Unsorted_processed`
    directory to gather all of the data together in one place, so please do not move files that
    you process yourself from the `RAWSEQ` folder — copy them if needed.



Keep `environment.yaml` at the same relative location (one directory above
the pipeline folder) — the `Snakefile`'s `conda:` directives point to
`../environment.yaml`.

Put your fastq pairs in the `fastqs/` subfolder, and check `config_hg38.yaml`
(or `config_mm10.yaml`) for sample names/paths — these configs already point
at the shared lab genome indices and annotation files under `/shareDB` and
`/home/labs/barkailab/vovam/Mammalian/...`, so no changes are needed there
unless you're adding new samples or output locations.

## 3. Run it

```bash
conda activate snakemake_runner
cd ~/my_run/hepg2   # or nmumg

snakemake all --snakefile Snakefile \
  --executor cluster-generic \
  --rerun-incomplete \
  --cluster-generic-submit-cmd "bsub -n 8 -q short -R 'span[hosts=1]' -R 'rusage[mem=2000]'" \
  --jobs 30 --latency-wait 120 \
  --use-conda --conda-frontend conda
```

The first time you run this, Snakemake will build the conda environment
from `environment.yaml` — this takes a few minutes but only happens once
(it's cached under `.snakemake/conda/` and reused on every later run).

## Notes

- If a job fails with a strange error (e.g. exit 127, missing output right
  after a job appeared to succeed), it may be an LSF `short` queue
  preemption/requeue, not a pipeline bug — just re-run the same command;
  `--rerun-incomplete` will pick up where it left off.
- Don't edit `environment.yaml` casually — Snakemake's default
  rerun-triggers treat any change to it as invalidating every rule that
  references it, which forces a full pipeline rerun (including the
  expensive alignment step).

## Roadmap.
- Soon I will add the possibilty to run the de-novo motif enrichemnt for those who have the [Homer installed in their path](../general/homer_motif.md) right away in the pipeline.
