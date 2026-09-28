# Fi-NeMo: motif hit-calling from Cherimoya attributions

[Fi-NeMo](https://github.com/kundajelab/Fi-NeMo) (`finemo`) is a
GPU-accelerated tool from the Kundaje lab (same lineage as BPNet/ChromBPNet/
tfmodisco-lite) for finding **instances** of already-discovered motifs across
a model's attribution scores. It slots in right after
[Cherimoya](cherimoya.md)'s own attribution → seqlets → TF-MoDISco steps: TF-
MoDISco *discovers* motif patterns (as CWMs, contribution weight matrices);
Fi-NeMo takes those and systematically finds every occurrence of each one.

## What it actually does (it isn't just FIMO on a different track)

The natural comparison is FIMO — but the mechanism is meaningfully
different, not just "FIMO applied to contribution scores instead of raw
sequence":

- **FIMO**: scans raw DNA with a PWM, computes an independent log-odds match
  score at every position for every motif, thresholds. Each candidate hit is
  scored in isolation.
- **Fi-NeMo**: "solves motif instance calling as an optimization problem
  that reconstructs contribution score tracks as sparse linear combinations
  of motif CWMs, formulated as an L1-regularized linear model" (its own
  description). It treats the whole attribution track over a region as a
  signal to *explain*, and asks what sparse combination of motif placements
  best reconstructs it — a lasso-style deconvolution, not a per-position
  scan. Candidate placements **compete** for credit: if two motifs could
  nominally both explain the same bump, the weaker one's coefficient gets
  pushed toward zero rather than both being reported as independent hits.

So: same job (locate motif instances), different substrate (what the model's
attributions actually explain, not raw sequence-to-consensus similarity).

## Installing

```bash
pip install finemo
```
No pinned torch version in its dependencies, so this won't disturb an
existing cu126 torch install fixed per the [Cherimoya
tutorial](cherimoya.md#2-setting-up-the-environment) — `pip` sees torch is
already satisfied and leaves it alone.

## The bridge: Cherimoya's attribution output is already the right format

!!! note "On this project we fed Fi-NeMo ISM attributions, not `cherimoya attribute` output"
    `cherimoya attribute` uses DeepLIFT/SHAP, which **did not converge** on
    this architecture (the attribution sum came out equal to the output
    difference itself). The 26-model batch used `tangermeme`'s
    `saturation_mutagenesis(..., hypothetical=True)` over the central 400 bp
    instead, saved as the same `ohe`/`attr` `.npz` pair, so everything below
    applies unchanged. One further change: **one MoDISco + Fi-NeMo run per
    head** (count and profile), each head scanned with its own motifs — never
    a unified motif set across heads. See
    [§10 of the base tutorial](cherimoya.md#10-interpretation-at-scale-use-ism-and-run-each-head-separately).


`cherimoya attribute` writes `RUN_NAME.attributions.ohe.npz` (one-hot
sequences) and `RUN_NAME.attributions.attr.npz` (hypothetical contribution
scores) — this is exactly what `finemo extract-regions-modisco-fmt` expects
(it's built for tfmodisco-lite's own input format, which Cherimoya's
attribution step already matches):

```bash
finemo extract-regions-modisco-fmt \
  -s RUN_NAME.attributions.ohe.npz \
  -a RUN_NAME.attributions.attr.npz \
  -p RUN_NAME_peaks.filtered.narrowPeak \
  -w 400 \
  -o regions.npz
```

!!! warning "Two things that will silently misalign your regions"
    - **`-p/--peaks` needs 10-column narrowPeak, not the plain BED6 the
      Cherimoya tutorial's top-N subsetting produces.** Reformat first —
      column 10 (summit offset) should be `(end - start) // 2`: Cherimoya's
      `attribute` command crops its output window around the *midpoint* of
      each locus (`extract_loci` centers the input window there, then
      `attribute.py` crops to `mid-200:mid+200`), so the peak-region
      midpoint is the correct "summit" for Fi-NeMo's purposes even without a
      true MACS3 summit call.
    - **The peaks file must be filtered by `RUN_NAME.attributions.idxs.npy`
      first.** That's the boolean N-filter mask `cherimoya attribute`
      applies internally (loci too close to assembly gaps get dropped) — the
      `ohe.npz`/`attr.npz` arrays only contain the *surviving* loci, so an
      unfiltered peaks file will be longer than the arrays and everything
      after the first dropped locus will be off by one (or more).
    - **`-w/--region-width` must match Cherimoya's actual attribution
      window**, not `finemo`'s own default of 1000. Cherimoya's default crop
      is 400bp (`mid-200:mid+200`) — pass `-w 400`, or you'll get a shape
      mismatch.

```python
# Building the filtered narrowPeak from a Cherimoya top-N BED + idxs mask:
import numpy as np
idxs = np.load("RUN_NAME.attributions.idxs.npy")
rows = [l.split() for l in open("RUN_NAME_topN.bed")]
kept = [r for r, keep in zip(rows, idxs) if keep]
with open("RUN_NAME_peaks.filtered.narrowPeak", "w") as fh:
    for chrom, start, end, name, score, strand in kept:
        start, end = int(start), int(end)
        summit = (end - start) // 2
        fh.write(f"{chrom}\t{start}\t{end}\t{name}\t0\t.\t{score}\t-1\t-1\t{summit}\n")
```

## Calling hits and generating the report

```bash
finemo call-hits \
  -r regions.npz \
  -m RUN_NAME_modisco_results.h5 \
  -o finemo_hits/

finemo report \
  -r regions.npz \
  -H finemo_hits/ \
  -m RUN_NAME_modisco_results.h5 \
  -o finemo_report/
```
No GPU needed for a few thousand regions — CPU is fine and fast. Submit to
`short-gpu` anyway if you want, but it's not the bottleneck at this scale.

## Reading the outputs

- **`finemo_hits/hits.tsv`** — every called instance: coordinates, strand,
  `motif_name`, and the fitted `hit_coefficient` (its weight in the sparse
  reconstruction — roughly, how much this instance was needed to explain the
  observed signal).
- **`finemo_report/report.html`** — the full visual report: per-motif logos,
  hit-distribution plots, motif co-occurrence, seqlet-vs-hit confusion
  matrix.
- **`finemo_report/motif_report.tsv`** — the key per-motif QC table.
  `seqlet_recall` (fraction of TF-MoDISco's original seqlets for that motif
  that Fi-NeMo also called as a hit) and `cwm_similarity` are the two
  numbers to check before trusting a motif: both close to 1.0 means the hit
  calls line up well with what TF-MoDISco already found; low values (we saw
  some motifs as low as 0.13-0.35 recall) mean that pattern had few
  supporting seqlets to begin with and shouldn't be over-interpreted.

## Running on a different/larger peak set than what TF-MoDISco used

You don't need to retrain the model or rediscover motifs — TF-MoDISco's job
(discovering the CWMs) is already done and independent of which peaks you
now scan. You *do* need fresh attributions, though: contribution scores are
locus-specific outputs of the trained model, not something you can slice out
of an existing `attr.npz` for different genomic regions. Re-run just
`cherimoya attribute` (not the whole pipeline — no need to redo
seqlets/TF-MoDISco/marginalization) with `model` pointed at your existing
`.torch` checkpoint and `loci` pointed at the new peak set, then feed the
new `ohe.npz`/`attr.npz` into `extract-regions-modisco-fmt` as above.

If that new peak set is large, see [Cherimoya from raw BAMs
§7](cherimoya_raw_bams.md#7-splitting-attribution-across-many-chunks-and-a-cgroup-gotcha)
for chunking attribution safely — including a cgroup memory-accounting
gotcha specific to running many chunks sequentially in one job.

## Worked example: same motifs, different scan sizes

From the KLF15DBD raw-BAM run ([Cherimoya from raw BAMs](cherimoya_raw_bams.md)):
same trained model, same TF-MoDISco motifs (`KLF15DBD_raw_modisco_results.h5`,
discovered from the 1,500-peak set), scanned twice — once over the 1,482
loci that survived N-filtering from the original top-1500 set, once over the
24,962 that survived from the top-25,000 training set (chunked per
[§7](cherimoya_raw_bams.md#7-splitting-attribution-across-many-chunks-and-a-cgroup-gotcha)).
Everything else about the `finemo` invocation was identical.

| | 1,482 peaks | 24,962 peaks |
|---|---|---|
| total hits | 6,341 | 78,732 |
| `pos_patterns.pattern_0` hits | 4,741 | 60,037 |
| `pos_patterns.pattern_0` seqlet_recall | 0.947 | **0.004** |
| `pos_patterns.pattern_0` cwm_similarity | 0.996 | 0.996 |

Hit counts scaled up roughly in proportion to region count (~17x more
regions → ~12x more hits) — no surprise there, and the dominant motif
(`pos_patterns.pattern_0`) stayed dominant by a wide margin in both runs.

The one number that looks alarming but isn't: **`seqlet_recall` collapsed
from 0.947 to 0.004** for every motif when scanning the larger set. This is
purely an artifact of what the metric measures — it's the overlap between
`finemo`'s hits and the *specific* TF-MoDISco seqlets that were discovered
from the original 1,500-peak set (`num_seqlets` in the report table is
identical across both runs, since it's a property of the discovery set, not
the scan). Once you're scanning a mostly-different, much larger region set,
almost none of the new hits happen to land on those original seqlet
coordinates by construction — low recall here doesn't mean the new hits are
untrustworthy. **`cwm_similarity` is the metric that still means what it
says regardless of scan size** (0.996 in both runs above) — use it, not
`seqlet_recall`, to QC hit quality once you've scanned beyond the original
discovery peaks.

## See also

- [Cherimoya (WEXAC)](cherimoya.md) / [Cherimoya from raw BAMs](cherimoya_raw_bams.md)
- [Fi-NeMo GitHub](https://github.com/kundajelab/Fi-NeMo) /
  [API docs](https://kundajelab.github.io/Fi-NeMo/finemo.html)
