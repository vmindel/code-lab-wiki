# Consensus Peaks + Signal Over Regions (R)

This recipe does two things:

1. Reads a set of peak files (BED-like, tab-separated) and merges them into a
   single non-redundant set of **consensus regions**.
2. For any bigWig track, computes a summary statistic (mean, sum, …) of the
   signal that falls inside each region.

It is written to be generic: peaks and tracks are not hard-coded, regions on
chromosomes that are missing from a bigWig are handled gracefully, and the
output is aligned 1:1 with the input regions so you can `cbind` results from
several tracks into one matrix.

## Dependencies

```r
library(GenomicRanges)   # GRanges containers + reduce()
library(rtracklayer)     # import() for bigWig / BED
library(GenomeInfoDb)    # seqlevel manipulation
```

## 1. Build consensus regions from a folder of peak files

```r
# --- configure -------------------------------------------------------------
peaks_dir <- "~/path/to/peak_files"   # folder of BED-like, tab-separated files
# columns are assumed to be: 1 = seqnames, 2 = start, 3 = end
# ---------------------------------------------------------------------------

peak_files <- list.files(peaks_dir, full.names = TRUE)

# Read every file into a data.frame (no header, tab-separated).
loaded_peaks <- lapply(peak_files, read.csv, sep = "\t", header = FALSE)

# Convert each data.frame to a GRanges, concatenate them, then reduce() to
# collapse overlapping/adjacent intervals into a single consensus set.
consensus_peaks <- reduce(
    do.call(c, lapply(
        loaded_peaks,
        makeGRangesFromDataFrame,
        seqnames.field = "V1",
        start.field    = "V2",
        end.field      = "V3"
    ))
)

consensus_peaks
```

!!! tip "Adjust the column mapping"
    The `*.field` arguments assume the first three columns are
    `seqnames / start / end` (the standard BED layout). If your files have a
    header or a different column order, change those arguments or pass the
    column names directly.

## 2. Summarise bigWig signal over regions

```r
#' Summarise bigWig signal over a set of regions.
#'
#' @param bw_path    Path to a bigWig file.
#' @param regions    A GRanges of regions to score (e.g. `consensus_peaks`).
#' @param summary_fn View-summary function: viewMeans, viewSums,
#'                   viewMaxs, ... (default: viewMeans).
#' @return A numeric vector, one value per region, in the same order as
#'         `regions`. Regions on chromosomes absent from the bigWig get 0.
load_bigwig_over_regions <- function(bw_path, regions, summary_fn = viewMeans) {

    # Import only the signal that overlaps our regions, as an RleList.
    bw_sig <- import(bw_path, which = regions, as = "RleList")

    # Pre-fill the result with 0 so missing chromosomes default to no signal.
    result <- numeric(length(regions))

    # Only keep chromosomes present in BOTH the regions and the bigWig.
    common_seqs <- intersect(seqlevels(regions), names(bw_sig))
    if (length(common_seqs) == 0) return(result)

    # Subset the regions to those on shared chromosomes and drop unused levels.
    region_mask <- as.logical(seqnames(regions) %in% common_seqs)
    sub_regions <- keepSeqlevels(regions[region_mask], common_seqs,
                                 pruning.mode = "coarse")

    # Build per-chromosome views: signal track sliced by region coordinates.
    views <- RleViewsList(
        rleList    = bw_sig[seqlevels(sub_regions)],
        rangesList = split(ranges(sub_regions), seqnames(sub_regions))
    )

    # Apply the summary function, then flatten back to per-region order.
    region_scores <- summary_fn(views)
    region_scores <- unsplit(as.list(region_scores),
                             as.factor(seqnames(sub_regions)))

    # Place scores back into the full-length result at the right positions.
    result[region_mask] <- unlist(region_scores)
    result
}
```

## Example usage

```r
# Single track -> a numeric vector aligned with consensus_peaks
sig <- load_bigwig_over_regions("~/tracks/sampleA.bw", consensus_peaks)

# Many tracks -> a regions x samples matrix
bigwigs <- list.files("~/tracks", pattern = "\\.bw$", full.names = TRUE)
mat <- sapply(bigwigs, load_bigwig_over_regions, regions = consensus_peaks)
rownames(mat) <- as.character(consensus_peaks)

# Use a different summary statistic
total <- load_bigwig_over_regions("~/tracks/sampleA.bw",
                                  consensus_peaks, summary_fn = viewSums)
```

!!! note "Mean vs. sum"
    `viewMeans` gives the average signal per base (length-normalised, good for
    comparing regions of different sizes). `viewSums` gives the total signal
    (proportional to region length). Pick the one that matches your downstream
    analysis.
