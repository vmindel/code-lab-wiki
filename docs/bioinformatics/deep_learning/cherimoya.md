# Cherimoya training of ChEC-seq data on the WEXAC cluster — a tutorial

A practical, start-to-finish guide for running [Cherimoya](https://cherimoya.readthedocs.io)
(a sequence-to-function deep learning model for TF binding / accessibility /
initiation profiles) on our LSF ("WEXAC") cluster. Written after getting the
first pilot model (SP2, full-length) working end-to-end. For the detailed
post-mortem of every failure we hit along the way, see
`cherimoya_trial/RUNBOOK.md` — this file is the
"how to do it right the first time" version.

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
`cherimoya_trial/spacing_experiment.py`. To get real motif sequences to plug
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

## See also

- `cherimoya_trial/RUNBOOK.md` — the detailed failure-by-failure log this
  tutorial was distilled from.
- `cherimoya_trial/SP2FULL_pilot.pipeline.json` — a fully worked, correct
  example config for this cluster/genome combo.
- The `cherimoya` skill's own reference docs (`concepts.md`,
  `cli-training-pipeline.md`, `interpreting-outputs.md`, `troubleshooting.md`,
  `using-tangermeme.md`) for anything not specific to this cluster.
