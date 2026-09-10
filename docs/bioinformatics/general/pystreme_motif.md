# Motif Discovery with pystreme (GPU)

pystreme is the lab's de novo motif finder. It works the way
[STREME](https://meme-suite.org/meme/doc/streme.html) does (seed words,
PWM refinement, a significance test on held-out sequences, erase the sites,
one motif per round), but it runs batched on a GPU and is called from
Python.

In the [mammalian ChEC-seq pipeline](../mammalian/chec_pipeline.md) it is the
**fast QC** step: a few minutes after peak calling, you see which motifs drive
each sample. This page explains what that step does, how to read its output,
and what the numbers mean.

!!! info "Status"
    pystreme is under active development. The pipeline uses a fixed,
    pinned version (commit `ec2950d`, installed in
    `/home/labs/barkailab/LAB/envs/pystreme`), so results don't change
    under you when the code does. Every result folder records the commit
    that produced it in `{id}.provenance.tsv`.

## Running it

From a pipeline run folder, with `snakemake_runner` active:

```bash
snakemake motifs_all --profile profiles/lsf
```

It runs as one LSF job per sample on `short-gpu`, up to 10 at a time. See
[the pipeline page](../mammalian/chec_pipeline.md#4-motif-discovery-fast-qc-gpu)
for the output files.

## What the step does

1. **Pick the peaks.** It takes `cleaned_peaks/{id}.clean.narrowPeak` (MACS3
   peaks minus the blacklist) and keeps the **1000 strongest by signalValue**
   (narrowPeak column 7).
2. **Center them.** Each peak becomes a **200 bp window centered on its MACS
   summit**. Positional statistics need centered sequences. Where
   blacklist removal truncated a peak so the summit falls outside what's
   left, the summit is moved back inside the interval.
3. **Discover.** pystreme compares these sequences against a
   dinucleotide-shuffled control and runs **up to 10 rounds**, finding one
   motif per round and erasing its sites before the next. Motifs that fail
   the hold-out test are kept, so the report can show 10 motifs when there
   are 10.
4. **Name them.** Each motif is compared to HOCOMOCO v12 core (human +
   mouse), and the best 3 matches are recorded.
5. **Report.** The motifs are sorted by hold-out p-value and plotted. Only
   significant motifs are written to the `.meme` and `.sites.bed` files.

| Setting (config) | Value | Why |
| --- | --- | --- |
| `motif_top_peaks` | 1000 | Enough sites for strong motifs, quick enough for QC |
| `motif_width` | 200 | Standard TF window; matches the [Homer](homer_motif.md) `-size 200` convention |
| `n_motifs` | 10 | Report length |
| `patience` | 10 | Keeps searching through weak rounds so the report can reach 10 |

## Reading the numbers

Each motif has three p-values, all **natural logs** (more negative = stronger):

| Column | Meaning | Use it for |
| --- | --- | --- |
| `holdout_logp` | Enrichment re-tested on **held-out peaks** that weren't used to learn the motif | **The one to quote.** Significant at <= -3.0 (p <= 0.05) |
| `train_logp` | Enrichment on the training peaks the motif was learned from | Nothing: it is optimistic by construction |
| `central_logp` | How concentrated the sites are **at the summit** | Separating the bound factor (sharply central) from co-factors (spread out) |

!!! warning "Don't cherry-pick"
    Re-running discovery over overlapping peak subsets and keeping the best
    result undoes the hold-out guarantee. To test robustness, change the seed
    and keep a set of peaks that you never look at.

In the PDF and summary:

- Motifs with `holdout_logp` above -3.0 are shown, labelled
  **[BELOW hold-out threshold, log p > -3.0]** in the logo title, with
  `passes_holdout = False` in `summary.tsv`. Treat them as noise.
- The HOCOMOCO name in each title is the closest known motif, not proof of
  which protein binds. Related families (ETS, AP-1, SP/KLF, SOX) match many
  members almost equally well.

### What a good sample looks like

From the 2026-09 mm10 ChEC run (42 samples, `20260831_WajdAileen`):

- **Clear hit.** ELK1: the ETS motif `ACTTCCGG` is rank 1 in all three
  replicates (`holdout_logp` -42, -42, -48), and SP/KLF, NF-Y and NRF1
  follow as co-factors.
- **Hit behind a co-factor.** In ETV5, Sox13 and Sox17, AP-1 (`TGAGTCA`)
  is rank 1 and the factor's own ETS or SOX motif is rank 2. AP-1 is strong
  across this cell system, so check rank 2 before calling a sample a
  failure.
- **Worth a second look.** The best motif is only about -4 to -6, the
  top motifs are poly-A or simple repeats, or the replicates disagree
  (e.g. one replicate is poly-A while the other two find the expected motif).

## Validation

Numbers from pystreme's own benchmark record, run on WEXAC against MEME
Suite 5.5.0 using mouse SP1 ChIP-seq peaks (mm10, 200 bp windows):

| Test | Result |
| --- | --- |
| Scanner vs FIMO, 2000 peaks, JASPAR SP1 and KLF4 | Pearson r = 1.00000 on the best score per peak; 99.95–100% same site (or tied within FIMO's resolution) |
| Same motifs as STREME, 2000 peaks, 3 motifs | All three (SP1 GC-box, NF-Y CCAAT box, AP-1), in the same order. Still the same after the September 2026 correctness fixes, which changed only the numbers attached to them |
| Same motifs as STREME, all 7232 peaks, 10 motifs | 8 of 10 shared, including the same primary motif. The differences were weak motifs at the end of both lists. *Measured before the September fixes; not yet re-measured on the pinned version* |
| Speed, 7232 peaks, 10 motifs | **61.9 s** on one NVIDIA A40 (pinned version), vs 191.3 s for STREME on the same sequences, about 3x faster |
| ChEC-seq QC (this pipeline), 1000 peaks, 10 motifs | ~1–1.5 min per sample job including startup, <1 GB RAM |

The central-enrichment test separates the bound factor from co-factors:
on SP1 peaks the GC-box is far more central (central log p ≈ -370) than
the NF-Y (≈ -130) or AP-1 (≈ -26) sites.

!!! note "Versus Homer"
    [Homer](homer_motif.md) is still the tool for **known-motif enrichment**
    against a curated database with a GC-matched genomic background. For
    de novo discovery inside the pipeline, pystreme is faster and gives
    hold-out and central-enrichment statistics per motif.
