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

snakemake all --profile profiles/lsf
```

All the cluster settings live in `profiles/lsf/config.yaml`, shipped with the
template: the LSF executor, the `bsub` line, `--jobs 30`, `--latency-wait 120`,
`--rerun-incomplete` and `--use-conda`. Every pipeline rule is submitted with
the same `bsub -n 8 -q short -R 'span[hosts=1]' -R 'rusage[mem=2000]'` line as
before, and per-job LSF logs go to `logs/lsf/`.

??? note "The old long command (pre-September 2026 template copies)"
    Copies of the template made before 2026-09-10 have no `profiles/` folder.
    They still run with the original command:

    ```bash
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

!!! warning "Activate `snakemake_runner` first"
    With the profile, Snakemake checks the conda version before doing
    anything. Without the env active it stops with *"Conda must be version
    24.7.1 or later"*.

## 4. Motif discovery (fast QC, GPU)

Once `cleaned_peaks/` exists, you can run de novo motif discovery on the 1000
strongest peaks of every sample:

```bash
snakemake motifs_all --profile profiles/lsf
```

- It is **opt-in** and not part of `all`, so it never holds up or fails a data
  run. Run it with the pipeline or at any time afterwards.
- Each sample takes ~1–4 min on one `short-gpu` GPU, and up to 10 samples run
  at once (change with `--resources gpu=N`). For comparison, 42 samples took
  ~20 min with the earlier cap of 2.
- Nothing to install: it uses a shared, pinned env at
  `/home/labs/barkailab/LAB/envs/pystreme`, and `environment.yaml` is unchanged.

Results land in `results/motifs/`:

| File | What it is |
| --- | --- |
| `{id}.motifs.pdf` | Up to 10 motifs, most significant first: logo, positional histogram, best HOCOMOCO v12 match in the title |
| `{id}.summary.tsv` | One row per motif: `passes_holdout`, consensus, width, sites, `holdout_logp`, `central_logp` |
| `{id}.annotation.tsv` | Top 3 HOCOMOCO v12 matches per motif |
| `{id}.meme`, `{id}.sites.bed` | **Significant motifs only**, for FIMO/Tomtom or browsing sites |
| `{id}.provenance.tsv` | Which pystreme commit and settings produced the results |

!!! tip "Reading it in 10 seconds"
    Look at motif 1 in the PDF. The tagged factor's own motif at rank 1, or
    just below a strong co-factor, with `holdout_logp` far below -3.0 means
    the sample looks right. Motifs above -3.0 are shown but labelled
    **BELOW hold-out threshold**: treat them as noise. Details, the statistics
    and the settings are on the [pystreme motif discovery](../general/pystreme_motif.md) page.

If your run folder is owned by someone else (so you can't write into it),
make your own folder with symlinks to their `fastqs/` and `cleaned_peaks/`,
copy in the template files, and run `motifs_all` there.

## Notes

- If a job fails with a strange error (e.g. exit 127, missing output right
  after a job appeared to succeed), it may be an LSF `short` queue
  preemption/requeue, not a pipeline bug — just re-run the same command;
  `--rerun-incomplete` will pick up where it left off.
- Don't edit `environment.yaml` casually — Snakemake's default
  rerun-triggers treat any change to it as invalidating every rule that
  references it, which forces a full pipeline rerun (including the
  expensive alignment step).

- Don't `pip install` into the shared pystreme env. A new pystreme version
  gets its own pinned env, so older motif results stay reproducible.

## Changelog

- **2026-09-10:** Added opt-in motif discovery (`motifs_all`, pystreme on
  GPU) and the `profiles/lsf` run command. The pre-change template is backed up at
  `Unsorted_processed/checseq_pipeline_template_backup_20260910_pre-pystreme.zip`.
  [Homer](../general/homer_motif.md) is still available for known-motif
  enrichment outside the pipeline.
