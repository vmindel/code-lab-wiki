# KLF/SP ChEC-seq models — the runs, and where everything lives

The current result set for the KLF paper: **26 mammalian (NMuMG) Cherimoya
models**, **38 yeast fine-tunes** derived from them, and a full
ISM → TF-MoDISco → Fi-NeMo interpretation of **26 TFs × 2 heads = 52 runs**
over 64,404 loci.

This page is the index: what was run, what the numbers are, and which file to
open. For *how* to run it, see [Cherimoya (WEXAC)](cherimoya.md); for the
incident-by-incident log, see
`/home/labs/barkailab/vovam/KLF_paper_analysis/cherimoya_trial/final/RUNBOOK.md`.

!!! info "Where it lives"
    `~/KLF_paper_analysis/cherimoya_trial/final/` (37 G) holds everything
    below. `../attic/` (50 G) holds superseded work and is deletable — but
    **not blindly**: a few kept scripts still read the old per-TF run dirs
    under `attic/runs/`. `final/README.md` is the directory-level map and
    carries an old→new path translation table, because `.md`/`.log`/`.txt`
    files in the tree still quote pre-reorganisation paths on purpose.

## The mammalian models

26 TFs, trained with `fit_sweep.py` at `ema_decay: 0` on a **pooled 64k-locus
set** shared by every TF, validated on each TF's *own* held-out chr8/chr18
peaks so the numbers stay comparable across recipes. 80 epochs. See
[§9 of the tutorial](cherimoya.md#9-the-training-recipe-that-actually-works-fit_sweeppy-ema_decay-0)
for why both of those choices matter — together they are worth more than every
optimiser knob we tried.

```
final/models_mammal/<TF>_raw/<TF>_raw_ema0_newpeaks.torch      # the checkpoint
                             <TF>_raw_ema0_newpeaks.final.torch
                             ...fit.json .detailed.log .performance.tsv
final/models_mammal/newpeaks_batch_summary.csv                 # the table below
```

Final-epoch validation, 26 TFs (`.torch` best-epoch numbers run ~0.04 higher
but are selected on the same validation set — treat them as optimistic):

| metric | min | median | max |
|---|---|---|---|
| count Pearson (within peaks) | 0.43 (SP7DBD) | **0.59** | 0.85 (SP3FULL) |
| profile Pearson | 0.31 | **0.82** | 0.92 |
| count Pearson ÷ replicate ceiling | 0.51 | **0.72** | — |

**24 of 26 improved** over the previous (EMA, own-peaks) training recipe;
median count Pearson went 0.53 → 0.59, with individual TFs moving much
further (KLF10DBD 0.29 → 0.58, SP2FULL 0.70 → 0.82).

!!! tip "Always read a model against its ceiling, not against 1.0"
    `final/tables/count_ceiling.csv` holds the **replicate ceiling** — the
    count correlation between independent replicate bigwigs of the same TF
    over the same loci. A model cannot beat the reproducibility of its own
    data. The median ceiling here is 0.82, so a median model at 0.59 is
    sitting at ~72% of what is achievable, not at "59% correct". The two TFs
    with the lowest raw scores (SP7DBD, SP7FULL) also have the noisiest data.

    Also note `cherimoya evaluate` reports **peaks-only** count Pearson, while
    published BPNet-style numbers are **peaks + negatives**, which is a much
    easier correlation. `scripts/eval_peaksnegs.py` computes the comparable
    one.

## The yeast fine-tunes

38 runs, each starting from a mammalian checkpoint above and fine-tuned on
yeast (sacCer) ChEC-seq over promoters at `out_window` 1000, `ema_decay: 0`,
100 epochs.

```
final/models_yeast/<TF>_yeast_finetune_ema0/
    <TF>_yeast_finetune_ema0.torch / .final.torch / .performance.tsv
    <TF>_raw_ema0_newpeaks.torch      -> symlink to the mammalian base
    <TF>_yeast_raw.bw                 -> symlink to final/signal/yeast/
    yeast_all_proms.bed / .negatives.bed -> symlinks to final/loci/
```

Final-epoch validation on the two held-out chromosomes (`NC_001140.6`,
`NC_001142.9`): **count Pearson median 0.72** (0.54 SP7FULL – 0.88
KLF16FULL), **profile Pearson median 0.64**.

**The base checkpoint matters, and you can see it in the numbers.** 24 of the
38 yeast TFs have a mammalian model of the same name to start from; the other
14 fall back to a domain-matched KLF15 base (`KLF15DBD` for DBD constructs,
`KLF15FULL` for full-length — this covers KLF2/6/13/14, SP2DBD, SP4, SP5,
LZY1 and the two KLF3 mutants):

| base | n | median count Pearson | range |
|---|---|---|---|
| matched (same TF) | 24 | **0.73** | 0.54 – 0.88 |
| KLF15 fallback | 14 | **0.67** | 0.57 – 0.82 |

So a matched base is worth ~0.06 Pearson on average — real, but a fallback
base is far from useless. If a yeast TF matters and has no mammalian model,
training one first is the highest-yield thing to do.

!!! warning "The run dirs are symlink farms, not self-contained"
    Each fine-tune dir links to its inputs rather than copying them. Moving
    the targets breaks all of them silently. They are currently **relative**
    links inside `final/`, so `final/` can be moved or copied as a unit; the
    absolute paths inside the `.py`/`.json`/`.sh` files cannot, so reading the
    data survives a move but rerunning anything does not.

## The interpretation

Every one of the 26 mammalian models, both output heads, over all **64,404**
pooled loci — 52 independent interpretation runs.

```
final/interpret/<TF>/<head>/          # head = count | profile
    full/attr.npz  full/ohe.npz       # ISM over ALL 64,404 loci, 64404 x 4 x 400
    attr.npz  ohe.npz                 # the top-10k subset MoDISco ran on
    modisco_results.h5                # modisco motifs -n 100000 -w 400
    report/                           # modisco report vs H12CORE
    finemo/hits/  finemo/report/      # this head's motifs vs this head's 64k attributions
final/interpret/<TF>/top10k_idx.npy   # row indices into the full arrays
final/interpret/modisco_summary.tsv   # all 1,104 patterns of all 52 runs
```

Key facts for anyone reading these files:

- **Attributions are ISM, not DeepLIFT/SHAP.** DeepLIFT does not converge on
  this architecture — the attribution sum came out equal to the output
  difference itself. `tangermeme`'s
  `saturation_mutagenesis(..., hypothetical=True)` is exact by construction
  and was used instead. ([§10 of the tutorial](cherimoya.md#10-interpretation-at-scale-use-ism-and-run-each-head-separately).)
- **Row order is the locus order of `final/loci/newpeaks_clean.parquet`**, and
  the 400 bp window is `[mid-200, mid+200)` where `mid = start + 99`. The
  top-10k subset is a *view* of the full array via `top10k_idx.npy`, not a
  separate computation — index, don't re-derive.
- **Each head was interpreted against its own motifs.** Count-head CWMs on
  count-head attributions, profile on profile. Do not mix them; an earlier
  unified-motif run also carried a +101 bp coordinate offset and lives in the
  attic.
- **Two HTML reports are missing by design**: SP1FULL and SP2FULL `profile`
  hit modiscolite's crash on a pattern with no tomtom match
  (`'float' object has no attribute 'strip'`). Their `.h5` files are complete
  and both are represented in `modisco_summary.tsv` — only the rendered HTML
  is absent.
- **Fi-NeMo QC: use `cwm_similarity`, not `seqlet_recall`.** Recall is
  measured against the specific seqlets MoDISco found in the top-10k discovery
  set, so it collapses toward zero whenever you scan a larger region set —
  which is exactly what these runs do. See [Fi-NeMo](finemo.md).

!!! danger "Motif names from tomtom are suggestions, not identifications"
    The KLF/SP family motifs are close enough that tomtom's top hit routinely
    names a sibling. In the **yeast** runs it is worse: the yeast motif
    database contains no KLF motif at all, so a genuine, strong KLF motif gets
    confidently labelled as whatever GC-rich yeast factor is nearest (RAP1, in
    the KLF15 case). Check the CWM by eye before believing a name.

## Inputs, kept alongside

```
final/loci/     newpeaks_clean.parquet   # the 64,404 pooled loci (199 bp each)
                newpeaks_clean.narrowPeak# same, row order + summit (end-start)//2, for Fi-NeMo
                newpeaks.negatives.bed   # GC-matched negatives, shared by all 26 fits
                peaks/                   # per-TF MACS peak tables
                yeast_all_proms.bed      # + .negatives.bed, shared by all 38 fine-tunes
final/signal/   <TF>_raw.bw              # the pooled mammalian track each model trained on
                <TF>_raw_peaks.bed       # that TF's own peaks = its validation loci
                replicates/              # per-replicate tracks behind count_ceiling.csv
                yeast/                   # the 38 yeast ChEC tracks
final/scripts/  fit_sweep.py batch_newpeaks.py finetune_yeast_ema0.py
                eval_peaksnegs.py count_ceiling.py scatter_counts*.py
final/tables/   count_ceiling.csv  klf15_diff.csv  observed_counts.parquet ...
```

External and unchanged:

- mm10: `/shareDB/iGenomes/Mus_musculus/UCSC/mm10/Sequence/WholeGenomeFasta/genome.fa`
- sacCer: `/home/labs/barkailab/felixj/RefData/yeast/cer.fa`
- motifs: `/home/labs/barkailab/vovam/H12CORE_meme_format.meme`
- source BAMs: `/home/labs/barkailab/LAB/data/MammalianCompendium/nmumg/mapped/`

## Rerunning any of it

Everything is driven by scripts kept beside the outputs. All of them submit to
LSF — **nothing here runs on the login node**.

```bash
PY=/home/labs/barkailab/vovam/miniconda/envs/seqtfun/bin/python  # never a bare `python`

cd final/scripts
python batch_newpeaks.py prep                 # CPU: pooled loci + shared negatives
python batch_newpeaks.py submit <prep_jobid>  # 26 × short-gpu fits

# one yeast fine-tune: run it FROM the run dir — its .finetune.json names
# loci/negatives/signals as bare filenames, resolved through that dir's symlinks
cd ../models_yeast/<TF>_yeast_finetune_ema0
bsub -q short-gpu -gpu num=1:j_exclusive=no:gmem=16G -R rusage[mem=32000] \
  "... conda activate seqtfun && $PY finetune_yeast_ema0.py \
   -p <TF>_yeast_finetune_ema0.finetune.json"

cd ../../interpret
python submit.py --dry-run                    # inspect the 676-job ISM plan first
python submit.py                              # prep -> ISM chunks -> combine -> modisco
python resubmit_missing.py                    # only the chunks that died, re-chaining modisco
python finemo_submit.py                       # hit-calling, per TF/head
```

Two operational notes that will cost you a day if you rediscover them:
ISM chunk jobs need `rusage[mem=24000]` (12 GB kills every one of them), and a
dead chunk leaves its dependent MoDISco job PENDing forever on an unsatisfied
`-w done(...)` rather than failing — poll for output *files*, never `bjobs`
text. Both are covered in
[§10 of the tutorial](cherimoya.md#10-interpretation-at-scale-use-ism-and-run-each-head-separately).

## See also

- [Cherimoya (WEXAC)](cherimoya.md) — how to run any of this from scratch.
- [Cherimoya from raw BAMs](cherimoya_raw_bams.md) — going from replicate BAMs
  to a trained model, which is how these 26 started.
- [Fi-NeMo](finemo.md) — the hit-calling step and how to read its QC tables.
