# Cherimoya training of ChEC-seq data on the WEXAC cluster — a tutorial

A practical, start-to-finish guide for running [Cherimoya](https://cherimoya.readthedocs.io)
(a sequence-to-function deep learning model for TF binding / accessibility /
initiation profiles) on our LSF ("WEXAC") cluster. Written after getting the
first pilot model (SP2, full-length) working end-to-end, and kept current
through a 26-TF KLF/SP batch and its yeast fine-tunes. For the detailed
post-mortem of every failure we hit along the way, see
`cherimoya_trial/final/RUNBOOK.md` — this file is the
"how to do it right the first time" version.

!!! danger "Read this before you train anything: don't use stock `cherimoya fit`"
    `Cherimoya.fit` validates on, and checkpoints, an **exponential moving
    average** of the weights (decay 0.999, hard-coded, no CLI switch). On
    datasets our size that average lags the live network by 2-10 epochs, and
    the scalar **count head** — unlike the softmax profile head — cannot
    survive being averaged. Symptoms: validation count Pearson craters around
    epoch 3-4, "best epoch" gets picked at a count MSE of 10-30, and headline
    count Pearson lands 0.3-0.7 instead of 0.8+. Sections 3-7 below still
    describe the pipeline correctly, but for the **training** step use
    `fit_sweep.py` with `"ema_decay": 0` — see [§9](#9-the-training-recipe-that-actually-works-fit_sweeppy-ema_decay-0).

## 1. What Cherimoya needs from you

At minimum: a **reference genome FASTA** and a **signal track** (bigWig, or
BAM/fragments Cherimoya will convert). Everything else — peaks, negatives,
motif database — is optional; the pipeline calls/samples/skips them if absent.
See the `cherimoya` skill's `input-files.md` for the full breakdown if you're
picking this up for the first time.

## 2. Setting up the environment

```bash
conda create -n seqtfun python=3.12
conda activate seqtfun
pip install cherimoya
```

**Then immediately fix the CUDA build — do this before running anything.**
This cluster's GPU nodes mostly run NVIDIA driver **560.35.05**, which only
supports up to **CUDA 12.6**. A plain `pip install cherimoya` will pull the
latest torch, which may be built for a newer CUDA (e.g. cu130) that this
driver can't run — it fails at the first `.to("cuda")` call with
`RuntimeError: The NVIDIA driver on your system is too old`.

Check what's actually installed and fix it if needed:
```bash
python -c "import torch; print(torch.__version__, torch.version.cuda)"

# check cherimoya's minimum torch version first:
python -c "import importlib.metadata as m; print(m.metadata('cherimoya').get_all('Requires-Dist'))"

# then install the newest torch version that (a) satisfies that minimum and
# (b) has a cu126 build (check with: pip index versions torch --index-url https://download.pytorch.org/whl/cu126)
pip install torch==<version> --index-url https://download.pytorch.org/whl/cu126 --force-reinstall
```
(cu124's wheel index lagged behind on torch versions when we did this — it
didn't have a build new enough to satisfy `cherimoya`'s `torch>=2.9.0`
requirement. cu126 did, at the same torch version already installed, so
nothing else about the environment had to change.)

**Before running a real job, confirm what's actually on the cluster** rather
than guessing:
```bash
lshosts -gpu                        # per-host driver version + GPU model
bhosts -gpu                         # current GPU occupancy per host
bqueues -l short-gpu | grep HOSTS    # which hosts a given queue can use
```

**TF-MoDISco's HTML report also needs the real MEME-suite `tomtom` binary**
(a pure-Python `memelite` package is installed as a dependency but isn't
actually used for this — see RUNBOOK.md §5 for why). Don't install `meme`
into the conda env; there's already a module for it:
```bash
module load MEME/5.5.0-GCC-10.3.0
```

## 3. Building the pipeline config

```bash
cherimoya pipeline-json \
  -s /shareDB/iGenomes/<Organism>/UCSC/<build>/Sequence/WholeGenomeFasta/genome.fa \
  -i signal.bw \
  -p peaks.bed \
  -m motifs.meme \
  -u \
  -n RUN_NAME \
  -o RUN_NAME.pipeline.json
```
- `-u` (unstranded) is right for a single pooled ChIP/ATAC coverage bigWig
  (one file, not a `+`/`-` pair). Confirm strandedness before assuming — see
  the skill's `assay-defaults.md`.
- Confirm the **genome build matches everything**: FASTA, peaks, and bigWig
  must all be the same build (mm10, hg38, ...) or you get silently-wrong
  results, not an error.

**Non-human genome? Fix the chromosome split before training.** The
defaults are hg38 names (`chr1-22, X, Y`). For mm10 (mouse has only
chr1-19, X, Y — and check whether your cell line even has signal on Y):
```json
"training_chroms":   ["chr2","chr4","chr5","chr7","chr9","chr10","chr11",
                       "chr12","chr13","chr14","chr15","chr16","chr17","chr19","chrX"],
"validation_chroms":  ["chr8","chr18"]
```
(A couple of chromosomes intentionally left out of both lists mirrors the
hg38 default's own held-out set — not required, just a convention we followed.)
Also set `"callpeaks_gsize": "mm"` in `preprocessing_parameters` for
consistency (only matters if MACS3 is calling peaks, i.e. you didn't pass `-p`).

## 4. Submitting the job

```bash
bsub -q short-gpu -gpu num=1:j_exclusive=no:gmem=16G \
  -R rusage[mem=40000] -R affinity[thread*10] \
  -oo RUN_NAME.out.txt -eo RUN_NAME.err.txt \
  "source \"\$(conda info --base)/etc/profile.d/conda.sh\" && conda activate seqtfun && \
   module load MEME/5.5.0-GCC-10.3.0 && export TORCHDYNAMO_DISABLE=1 && \
   cherimoya pipeline -p RUN_NAME.pipeline.json"
```

Notes on the flags, learned the hard way:

- **`j_exclusive=no`**: don't reserve a whole physical GPU unless you actually
  need it. A small model like this needs only ~8-16GB, and `j_exclusive=yes`
  can make you wait a long time on a busy shared node for no benefit.
- **Don't try to pin specific hosts with `-m`** to work around a driver
  mismatch — this cluster's `short-gpu` queue has an ESUB plugin that can
  silently ignore host pinning and dispatch elsewhere anyway. Fix the actual
  torch/CUDA build instead (§2).
- **`export TORCHDYNAMO_DISABLE=1`**: disables `torch.compile`. During
  attribution we saw it trigger repeated Triton autotuning on tiny matrix ops
  and grow memory unboundedly. Disabling it is both safer and (for ops this
  small) faster.
- **RAM (`rusage[mem=...]`), not GPU memory, is usually what kills this
  pipeline.** The `attribute` step's memory use scales with how many loci you
  feed it — see §5 below for the fix.

## 5. Keep runs small: subset peaks before attribution/seqlets/MoDISco

You don't need every training peak for interpretation. We restrict to the
**top-scoring peaks** (by the score column in the peaks BED):
```bash
sort -k5,5 -rn peaks.bed | head -n 1500 | sort -k1,1 -k2,2n > peaks_top1500.bed
```
(This is about *interpretation* loci. For **training** loci, the opposite
lesson applies — see [§9](#9-the-training-recipe-that-actually-works-fit_sweeppy-ema_decay-0):
training the count head on one TF's own peaks makes it memorise, and pooling
peaks across all TFs into one shared locus set is worth as much as every
optimiser tweak put together.)

1000-2000 peaks is a solid range for reliable *de novo* motif discovery;
~300-500 is a realistic floor for recovering one dominant motif. Some loci
will still get dropped by N-filtering (too close to assembly gaps), so expect
the usable count to come in a bit lower than what you started with.

!!! warning "Set the loci override in both places"
    If you restrict loci this way, set it in **both**
    `attribute_parameters.loci` **and** `seqlet_parameters.loci` in the pipeline
    JSON. Only setting the first causes:

    ```
    IndexError: Boolean index has wrong length: N instead of <full peak count>
    ```

    because the seqlets step reloads its own (default: full) loci list and tries
    to apply the attribute step's N-filtering mask against it.

## 6. Resuming after a step fails

`cherimoya pipeline` **always re-runs attribution, seqlets, TF-MoDISco, and
marginalization** on every invocation — only training has a real skip
condition. To skip re-training after a successful fit, set in the top-level
JSON:
```json
"model": "RUN_NAME.torch",
"negatives": "RUN_NAME.negatives.bed"
```
then resubmit the same `cherimoya pipeline -p RUN_NAME.pipeline.json` command.
It'll jump straight past training/negative-sampling into attribution.
(To resume from *further* along, e.g. skip re-running attribution too, you can
instead fix the relevant per-step JSON snapshot — `RUN_NAME.seqlets.json`,
etc. — and call e.g. `cherimoya seqlets -p RUN_NAME.seqlets.json` directly.)

## 7. Reading the outputs

- **`RUN_NAME.performance.tsv`** — the scorecard. Headline number:
  `count_pearson` (how well predicted total signal matches truth on held-out
  chromosomes). >0.5 is decent, 0.7-0.9+ is good. `profile_pearson` measures
  shape separately — a model can be good at one and mediocre at the other.
- **`RUN_NAME_modisco/report.html`** — open in a browser. The de novo
  discovered motif patterns, named against your motif database when matches
  exist. Each pattern's descriptive name is built from its top tomtom matches.
- **`RUN_NAME.motif_seqlet_count.tsv`** — quick tally of which known motifs
  the model's seqlets matched most often.
- **`RUN_NAME_marginalize/marginalization.html`** — inserts every motif in
  your database into background sequence and reports how much the model's
  prediction responds — a more direct "does the model care about this motif"
  test than the seqlet-matching tally.

## 8. Custom in-silico experiments (beyond the built-in pipeline)

The CLI's `marginalize` step only tests one motif at a time. For questions
like "how does predicted signal change as I vary the distance between two
motifs," use `tangermeme` directly against the trained model:

```python
from cherimoya import Cherimoya, ControlWrapper, LogCountWrapper
from tangermeme.io import extract_loci
from tangermeme.ersatz import multisubstitute

model = Cherimoya.load("RUN_NAME.torch", device="cuda").eval()
wrapper = LogCountWrapper(ControlWrapper(model))   # ControlWrapper is a no-op if you trained without controls

X = extract_loci("RUN_NAME.negatives.bed", genome_fasta, in_window=2114, n_loci=100)
X_pair = multisubstitute(X, [motif_a_seq, motif_b_seq], spacing=[distance_bp])
predicted = wrapper(X_pair.to("cuda"))   # mean over the batch = one point on your spacing curve
```

A worked, parametrized example (sweeping 0-200bp) is in
`cherimoya_trial/attic/SP2_bpres_cherimoya/spacing_experiment.py`. To get real motif sequences to plug
in rather than guessing consensus strings by eye:

- **From your own model's discovered patterns**: pull the PPM (`sequence`)
  and CWM (`contrib_scores`) arrays for a pattern out of `_modisco_results.h5`,
  then trim to the informative core by contribution-score magnitude
  (`score = abs(cwm).sum(axis=1)`, keep positions where
  `score >= 0.3 * score.max()`) before taking the per-position argmax as the
  consensus — the raw un-trimmed window is padded with low-information flanks.
- **From a motif database entry** (e.g. testing a motif your model didn't
  discover itself): load it with `memelite.io.read_meme(path)[motif_name]`
  and trim by per-position information content instead (`IC > 1 bit` is a
  reasonable cutoff), since MEME PWMs are probabilities, not raw contribution
  scores.

## 9. The training recipe that actually works: `fit_sweep.py` + `ema_decay: 0`

Two changes take within-peak count Pearson from ~0.3-0.7 to ~0.5-0.8 on the
same data. Neither is reachable from the stock CLI.

**(a) Train on live weights, not the EMA.** `fit_sweep.py`
(`cherimoya_trial/final/scripts/fit_sweep.py`) mirrors
`cherimoya_cli/commands/fit.py` and monkey-patches the hard-coded decay so
the JSON gets an `"ema_decay"` key; `0` means validate and checkpoint the
actual network. It also adds `"validation_loci"`, `"count_weight_ratio"`,
`"count_head_lr_mult"` and `"count_bias_init"`, and evaluates both `.torch`
and `.final.torch`. Effect on three TFs: 0.33→0.45, 0.38→0.47, 0.67→0.69,
with profile Pearson unchanged and count MSE calibrated from epoch 3.

Things that were tested and did **not** help, so don't spend a week on them:
count-loss weight (the library's learned Kendall weights already settle at
15-30×; forcing 1000× costs profile Pearson), `max_jitter`, count-head LR
multipliers, and count-head bias initialisation. Also note the pipeline
JSON's `count_loss_weight` key is **dead** — `fit.py` never reads it.

**(b) Train on a pooled locus set, validate on the TF's own peaks.** With
live weights the count head still plateaued at half the replicate ceiling,
training MSE 0.1 vs validation 0.3 — memorising one TF's peaks. Pooling all
TFs' peaks into one 64k-locus set (per-TF ≥20-read filter, GC-matched
negatives sampled once for the pooled set and reused by every TF) gave the
other half of the gain: in a 26-TF batch, e.g. 0.29→0.58, 0.37→0.55,
0.70→0.82. Keep `validation_loci` pointed at the TF's *own* held-out peaks so
the numbers stay comparable across recipes, and give it room —
`max_epochs: 80`, because the curve was still rising at 45.

```bash
# per TF, from an existing <TF>.fit.json; batch driver: scripts/batch_newpeaks.py
bsub -q short-gpu -gpu num=1:j_exclusive=no:gmem=16G -R rusage[mem=40000] \
  "... conda activate seqtfun && export TORCHDYNAMO_DISABLE=1 && \
   /path/to/envs/seqtfun/bin/python fit_sweep.py -p <TF>.fit.json"
```

!!! tip "Which number is the model's score?"
    `cherimoya evaluate` reports **peaks-only** count Pearson. The
    ENCODE/BPNet numbers people quote are **peaks + negatives**, which is a
    much easier correlation — compute both before deciding a model is bad.
    And establish the **replicate ceiling** first (correlate replicate bigwigs
    against each other over the same loci): a model cannot beat the
    reproducibility of its own data, and several TFs here were already at
    80-90% of it.

## 10. Interpretation at scale: use ISM, and run each head separately

**DeepLIFT/SHAP does not converge on this architecture.** The convergence
delta — attribution sum vs. the actual output difference — came out roughly
*equal to* the output difference itself, regardless of `compile`/`eval`/`train`
mode. That is a failed attribution, not a noisy one. If you use
`cherimoya attribute`, check convergence before trusting anything downstream.

We used `tangermeme`'s saturation mutagenesis instead, which is exact by
construction, over the central 400 bp of each 2114 bp window:

```python
from tangermeme.saturation_mutagenesis import saturation_mutagenesis
wrapper = (LogCountWrapper if head == "count" else ProfileWrapper)(ControlWrapper(model))
mid = X.shape[-1] // 2
attr = saturation_mutagenesis(wrapper, X, start=mid-200, end=mid+200,
                              batch_size=128, hypothetical=True, device="cuda")
```

Practicalities, from 676 such jobs (26 TFs × 2 heads × 13 chunks of 5,000
loci):

- **ISM costs 400 × 3 forward passes per locus** — ~23 min of GPU per 5,000
  loci (p10 13 min, p90 26 min). Chunk it so each job fits in `short-gpu`.
- **Request 24 GB host RAM per chunk job.** At `rusage[mem=12000]` every
  single job died with `TERM_MEMLIMIT` / exit 137; the peak is ~10.4 GB and
  12 GB leaves no headroom.
- **Write `chunk<i>.npz.tmp` then `os.replace`** so a preempted job never
  leaves a half-written file that a resume step mistakes for finished.
- **A dead chunk silently orphans its dependent job.** `-w done(...)` becomes
  "dependency never satisfied" and the MoDISco job PENDs forever without
  erroring. Poll for the *output files*, not `bjobs` text, and have a
  resubmit script that looks for missing `chunk<i>.npz`.
- **MoDISco on the top-N subset, Fi-NeMo on everything.** Rank loci by
  observed signal in that TF's own bigwig over the same window
  `extract_loci` centres on, save the row *indices* so the subset stays a
  view of the full arrays, run `modisco motifs -n 100000 -w 400` on the top
  10k (~30 min CPU), then Fi-NeMo the resulting motifs back over all 64k
  (2-4 min GPU, 200-320k hits).
- **Interpret each head against its own motifs.** The count head's CWMs on
  the count head's attributions, profile on profile. A unified motif set
  across heads is not meaningful and our first attempt at one also carried a
  +101 bp coordinate offset.
- **`modisco report` still crashes on an unmatched pattern**
  (`AttributeError: 'float' object has no attribute 'strip'` — it calls
  `.strip()` on a NaN tomtom match name). The `.h5` is complete by then, so
  wrap the report step in `|| true` and read the motifs out of the `.h5`
  directly.

## 11. Fine-tuning onto another species

There is no CLI support — training happens outside `cherimoya pipeline`, so
the fine-tune script builds the muon/adam/lw optimiser param groups exactly as
`fit.py` does and calls `fit` itself. What actually matters:

- **The base checkpoint dominates.** Redoing 38 mammalian→yeast fine-tunes
  from §9's retrained bases moved them far more than any LR or epoch tuning
  had: final-epoch count Pearson median **0.72** (range 0.54-0.88), profile
  Pearson median 0.64.
- **Nothing about the genome carries over.** Chromosome names
  (`NC_001133.9`, … for sacCer, not `chr1`), genome size for MACS3, and the
  motif database (a species-appropriate one — note that a yeast motif DB has
  no KLF entry, so tomtom will confidently mislabel a real KLF motif as
  something else; check the CWM by eye).
- **Window shrinking is arithmetic, not a flag.** The architecture's
  `trimming = 46 + sum(2**i for i in range(n_layers))` = **557** for the
  standard 9-layer config, independent of window size, so
  `in_window = out_window + 1114` whenever you shrink the output window on an
  already-trained checkpoint.
- **Loci choice dominates the rest.** Promoters at `out_window` 1000 beat
  every peak-based variant we tried for yeast ChEC.

## See also

- `cherimoya_trial/final/RUNBOOK.md` — the detailed failure-by-failure log
  this tutorial was distilled from (§19 the EMA finding, §20 the yeast
  fine-tunes, §21 interpretation at scale, §22 where everything lives).
- `cherimoya_trial/final/README.md` — layout of the current result set:
  models, bigwigs, loci, attributions, motifs and hits, plus an old→new path
  translation table for anything quoted from the RUNBOOK or a job log.
- `cherimoya_trial/attic/SP2_bpres_cherimoya/SP2bp_pilot_long.pipeline.json` —
  a fully worked, correct example config for this cluster/genome combo.
- The `cherimoya` skill's own reference docs (`concepts.md`,
  `cli-training-pipeline.md`, `interpreting-outputs.md`, `troubleshooting.md`,
  `using-tangermeme.md`) for anything not specific to this cluster.
