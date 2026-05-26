# Motif Finding with Homer

[Homer](http://homer.ucsd.edu/homer/)'s `findMotifsGenome.pl` scans a set of
genomic regions (e.g. peaks) for enriched DNA motifs — both known TF motifs and
*de novo* motifs — by comparing your regions against a matched set of background
sequences sampled from the genome.

This page shows how to launch it as an LSF batch job on the cluster (WEXAC) and
explains every flag.

!!! warning "On WEXAC: load Homer first"
    If `findMotifsGenome.pl` isn't on your `PATH`, load the module before
    submitting:

    ```bash
    module load homer
    ```

    This puts Homer's executables on your `PATH` so you can call them by name
    (no full path needed).

## The job

```bash
bsub -q short -J findMotifs -n 8 -R "span[hosts=1]" \
     -o findMotifs_%J.out -e findMotifs_%J.err \
     findMotifsGenome.pl \
         input_peaks.bed \
         mm10 \
         output_motifs_dir/ \
         -size 200 \
         -p 8 \
         -cache 2000 \
         -preparsedDir /path/to/preparsed/mm10/
```

## The LSF submission (`bsub`)

Everything before `findMotifsGenome.pl` tells the LSF scheduler *how* to run the
job; it is not part of Homer.

| Flag | Meaning |
| --- | --- |
| `-q short` | Submit to the **short** queue (for jobs that finish quickly). Use a longer queue for large peak sets. |
| `-J findMotifs` | **Job name** — how the job shows up in `bjobs`. |
| `-n 8` | Reserve **8 CPU cores** for the job. |
| `-R "span[hosts=1]"` | Force all 8 cores onto a **single host**, so Homer's multithreading actually sees them. |
| `-o findMotifs_%J.out` | Write **stdout** to this file; `%J` is replaced by the job ID. |
| `-e findMotifs_%J.err` | Write **stderr** to this file (errors / warnings). |

!!! tip "Match `-n` and `-p`"
    The cores you reserve from LSF (`-n 8`) should match the threads you give
    Homer (`-p 8`). Asking Homer for more threads than you reserved won't speed
    things up and annoys the scheduler.

## The Homer command (`findMotifsGenome.pl`)

```bash
findMotifsGenome.pl <input> <genome> <output_dir> [options]
```

### Positional arguments

| Argument | Meaning |
| --- | --- |
| `input_peaks.bed` | Your regions of interest. Accepts BED or Homer peak format. Homer reads the coordinates, extracts the underlying genomic sequence, and searches it for motifs. |
| `mm10` | The **genome build**. Homer uses it both to fetch sequence and to generate matched background regions. Must be installed in Homer (see below). |
| `output_motifs_dir/` | Output **directory** (created if missing). Results land here — most importantly `knownResults.html` and `homerResults.html`. |

### Options used here

| Option | Meaning |
| --- | --- |
| `-size 200` | Use a **200 bp window centered on each peak**, rather than the full peak width. 200 bp is the standard for TF ChIP/ChEC-style peaks; use `-size given` to keep each peak's own width. |
| `-p 8` | Run with **8 parallel threads** (matches the 8 cores reserved above). |
| `-cache 2000` | Cache size in **MB** for the enrichment statistics (default 500). A larger cache keeps more of the calculation in memory, speeding up the run at the cost of RAM. |
| `-preparsedDir /path/.../mm10/` | Directory for Homer's **pre-parsed background sequences**. Generating these is slow; pointing every run at a shared, writable folder lets Homer reuse them instead of regenerating each time. |

## Before you run

- **Install the genome once** (if not already present):

    ```bash
    configureHomer.pl -install mm10
    ```

- **Pre-parsed directory must be writable.** On first use Homer fills
  `-preparsedDir` with background files; subsequent runs reuse them. A shared lab
  path avoids everyone regenerating the same data.

## Reading the output

Open the HTML reports in `output_motifs_dir/`:

- **`knownResults.html`** — enrichment of *known* motifs from Homer's database,
  with p-values and target/background percentages.
- **`homerResults.html`** — *de novo* motifs discovered in your sequences, each
  matched to its closest known motif.

!!! note "Pick the right background"
    By default Homer builds a background of random genomic regions matched for
    GC content. If you have a more appropriate control (e.g. a set of
    non-bound regions), pass it with `-bg control.bed` for cleaner enrichment
    estimates.
