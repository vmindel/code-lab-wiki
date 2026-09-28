# Cherimoya from raw BAMs on the WEXAC cluster

A follow-up to [Cherimoya (WEXAC)](cherimoya.md) for the case where you're
starting from **raw aligned-read BAMs** (no pre-called peaks, no bigWig)
rather than an already-processed signal track. This changes almost nothing
about the model itself, but changes the scale you're operating at, and that
scale breaks a few assumptions the bigwig-based tutorial's defaults make.
Written after two from-scratch ChEC-seq runs (HAND1/TCF3/HOXA11 and
KLF15DBD) starting from raw replicate BAMs.

## 1. Pilot first: subset to ~1e3 reads per replicate

Before committing GPU time to a real run, subsample each replicate BAM down
to roughly 1000 reads and run the *exact same* pipeline config on that. This
won't produce a usable model (MACS3 genuinely can't call peaks from that few
reads — expect zero), but it validates every other piece of plumbing first:
environment, BAM handling, `bam2bw` merging multiple replicates into one
pooled bigWig, mm10 chromosome config, negative sampling. Cheap (~1 minute
turnaround) and catches config mistakes before a ~30-60 min real run does.

```bash
module load bzip2   # samtools needs it on this cluster
samtools view -b -s 42.0000197 -o rep1.subset1e3.bam full_rep1.bam  # seed.fraction
samtools sort -o rep1.subset1e3.bam rep1.subset1e3.bam
samtools index rep1.subset1e3.bam
```
`samtools view -s`'s subsampling hashes on read name, so paired-end mates are
kept or dropped together — safe for paired BAMs.

## 2. Call peaks yourself, outside `cherimoya pipeline`

At real read depth, MACS3 can call **far** more peaks than any pre-made-bigwig
run in the original tutorial ever saw — two raw-BAM ChEC-seq runs here both
called **~190,000 peaks genome-wide** (~150-160k on usable chroms), vs.
~7,600 for the original bigwig pilot. `cherimoya pipeline` calling MACS3
*internally* means you don't get to see or subset that peak count before the
expensive steps run. Call it yourself first, on the CPU `short` queue (MACS3
doesn't need a GPU — ~10-25 min for 3-4 replicates at full depth):

```bash
bsub -q short -n 4 -R rusage[mem=16000] -R affinity[thread*4] \
  -oo macs3.out.txt -eo macs3.err.txt \
  "source \"\$(conda info --base)/etc/profile.d/conda.sh\" && conda activate seqtfun && \
   macs3 callpeak -f BAMPE -g mm -n RUN_NAME -q 0.05 -t rep1.bam rep2.bam rep3.bam"
```
Then pass the result to `pipeline-json` with `-p RUN_NAME_peaks.narrowPeak` —
this makes `cherimoya pipeline` skip its own MACS3 call entirely.

## 3. Cap *training* loci too, not just attribute/seqlet

The original tutorial's §5 (restrict to top-scoring peaks) only mentioned
attribution and seqlets. At raw-BAM scale it matters for **training** as
well: `fit`'s `PeakGenerator` loads every training-chrom locus's one-hot
sequence + signal into memory in one shot before the first batch — much
lighter per-locus than attribution, so it won't crash at 150k+ loci, but it
massively inflates wall-clock time per epoch relative to a few-thousand-locus
run.

Build **two** top-N subsets by score (narrowPeak column 7, `signalValue`),
restricted to usable chroms:
- ~1500 for `attribute_parameters.loci` / `seqlet_parameters.loci` (unchanged
  from the original tutorial)
- ~25,000 for the **top-level `loci`** — this is what `fit` actually trains
  on, and what negative-sampling GC-matches against. Leaving top-level `loci`
  as the full MACS3 output is what makes training slow, not the model.

## 4. `short-gpu` preemption — protect the checkpoint the moment training ends

`short-gpu` has a hard **360-minute** run limit (`bqueues -l short-gpu`), but
the far more common failure is **preemption by a higher-priority job**, well
before that limit, followed by an automatic LSF requeue onto a different
host (`Re-runnable` is the default). `bhist -l <jobid>` shows this plainly:
`Suspended: Job was preempted...` → `Job is being requeued` → dispatched
elsewhere. The killed process's dying stack trace is what shows up as a
stray `Traceback` in the log at the preemption timestamp — it isn't a code
bug.

The requeue re-runs the **entire** `cherimoya pipeline` command from scratch,
because the JSON on disk still has `"model": null`. One run made it all the
way through training → attribution → seqlets → TF-MoDISco → into
marginalization (the very last step) before a preemption threw all of it
away and forced a full retrain.

!!! warning "Protect the checkpoint as soon as training finishes"
    The instant a run reaches `Step 2: Calculating attributions` in the log
    (training is done, `RUN_NAME.torch` is stable), immediately:

    ```bash
    cp RUN_NAME.torch RUN_NAME.torch.bak
    cp RUN_NAME.negatives.bed RUN_NAME.negatives.bed.bak
    python3 -c "
    import json
    cfg = json.load(open('RUN_NAME.pipeline.json'))
    cfg['model'] = 'RUN_NAME.torch'
    cfg['negatives'] = ['RUN_NAME.negatives.bed']
    json.dump(cfg, open('RUN_NAME.pipeline.json', 'w'), indent=4)
    "
    ```
    Patching the JSON on disk is safe — the running process already parsed
    its own copy and won't re-read it. If a *second* preemption hits after
    this, the requeued run reads the patched JSON and skips straight back to
    attribution instead of retraining from zero. Don't do this *before*
    training finishes, though — we once lost a good checkpoint by letting a
    requeued run get one epoch into a worse retrain before killing it; that
    epoch's checkpoint overwrote the original best with no backup.

## 5. The epoch ~3-4 count-Pearson crash — it's not the warmup, it's patience

!!! warning "Superseded (2026-09-09) — the real cause is the weight EMA"
    Generous `early_stopping` treats the symptom. The crash is `Cherimoya.fit`
    validating and checkpointing an **exponential moving average** of the
    weights (decay 0.999, hard-coded), which lags 2-10 epochs on datasets this
    small. Train with `fit_sweep.py` and `"ema_decay": 0` instead of
    `cherimoya fit` and the count head is calibrated from epoch 3 with no
    crash at all. See [§9 of the base tutorial](cherimoya.md#9-the-training-recipe-that-actually-works-fit_sweeppy-ema_decay-0), or RUNBOOK §19. Everything below still describes what the
    curve looks like if you are stuck on stock `cherimoya fit`.


Across three independent from-scratch training runs on real ChEC-seq data,
**validation count Pearson** (not profile Pearson, not count MSE — both
improved monotonically every run) showed the same shape: climbs for the
first 2-3 epochs, **craters sharply** (sometimes negative) around epoch 3-4,
then climbs steadily back up over the next 10-20 epochs, eventually
**exceeding** the pre-crash peak if given room to.

First guess was `n_warmup_epochs` (default 2) — the LR schedule hits its
absolute peak right at the warmup→cosine-decay handoff, which lines up with
where the crash happens. **Tested and falsified**: raising
`n_warmup_epochs` 2→5 shifted the LR peak later, but the count-Pearson crash
still hit at the same epoch, just now mid-warmup. It isn't the LR schedule.

**What actually works: raise `early_stopping` (default 5), not
`n_warmup_epochs`.** `early_stopping` counts epochs *since the best-so-far
checkpoint* — if the crash happens at epoch 3 and default patience is 5,
training stops at epoch ~8, right in the middle of the recovery climb, and
never finds the real optimum. Concretely, on one run: patience 5 stopped at
count Pearson 0.42; patience 10 stopped at 0.51 (still climbing when it cut
off); **patience 20 ran the full 45 epochs and hit 0.6695** (best epoch 38),
a real plateau. `cherimoya` has no "ignore the first N epochs" burn-in
option — the checkpoint logic always saves whatever's actually best-so-far
regardless of patience, so generous `early_stopping` is the correct, cheap
fix (each epoch here was only ~12-13s, so even a full 45-epoch run is
~10 minutes).

## 6. `OSError`/`PermissionError` on `RUN_NAME.log` or `RUN_NAME.detailed.log`

If a resubmitted run crashes immediately (within the first epoch) with
`Text file busy` or `Permission denied` on a training log file, before
assuming NFS flakiness: **check whether the file is open in Excel (or
anything else) on a machine that has this directory mounted over SMB/AFP.**
A file opened locally for viewing can hold a lock that a Linux process on
the cluster fails to open for writing — close it, delete the stale
`RUN_NAME.log`/`RUN_NAME.detailed.log`, and resubmit.

## 7. Splitting attribution across many chunks — and a cgroup gotcha

If you want to run attribution/`finemo` hit-calling on a peak set too large
to attribute in one call safely (see §3), split it into chunks and run
`cherimoya attribute` once per chunk — the per-locus computation is fully
independent (no interaction between loci), so this is exact, not an
approximation. Concatenate the resulting `.ohe.npz`/`.attr.npz`/`.idxs.npy`
arrays afterward with `numpy.concatenate(..., axis=0)`.

!!! warning "Run each chunk as its own job if you have many of them"
    Running all chunks sequentially *within one LSF job* isn't safe just
    because each `cherimoya attribute` call is its own subprocess. Even
    though each subprocess's own memory is released on exit, **LSF's
    cgroup-level memory accounting can still accumulate page cache from
    repeatedly reading the multi-GB genome FASTA across many chunks in the
    same job** — one run got through 8 of 10 chunks fine, each individually
    modest, then hit `TERM_MEMLIMIT` on chunk 8 from nothing more than
    accumulated cache. Splitting into a few separate job submissions (or at
    least resubmitting the remainder as a fresh job) resets that accounting.

## See also

- [Cherimoya (WEXAC)](cherimoya.md) — the base tutorial (bigwig/pre-called-peaks
  workflow, environment setup, reading outputs).
- `cherimoya_trial/final/RUNBOOK.md` §§7-10 — the detailed log this page was
  distilled from.
- [Fi-NeMo](finemo.md) — motif hit-calling directly from a Cherimoya model's
  attribution output.
- [KLF/SP ChEC-seq models](klf_sp_models.md) — the 26 mammalian models these
  lessons produced, their yeast fine-tunes, and where everything lives.
