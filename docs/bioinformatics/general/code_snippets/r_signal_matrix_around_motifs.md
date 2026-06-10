# Strand-Aware Signal Matrix Around Motif Sites (R)

Given a motif and a set of peaks, this recipe:

1. Scans the peaks for motif occurrences with **motifmatchr**.
2. Builds a fixed-width window centered on every motif hit.
3. Extracts a **per-site signal matrix** from a coverage track, one row per
   motif site, with **minus-strand rows reversed** so every row is oriented
   5′ → 3′ relative to the motif.

The output is a `length(sites) × (2 * window_size)` matrix in the same row order
as the input sites — ready for `colMeans()`, heatmaps, k-means clustering, etc.

## Dependencies

```r
library(motifmatchr)     # matchMotifs
library(GenomicRanges)   # GRanges, resize
library(IRanges)         # Views
library(S4Vectors)       # split() on GRanges
```

## 1. Find motif occurrences in peaks

```r
# klf.motif    : a PWMatrix / PWMatrixList (e.g. from TFBSTools / JASPAR2024)
# n.peaks      : GRanges of peaks to scan
# genome       : a BSgenome object (e.g. BSgenome.Mmusculus.UCSC.mm10)
matched_motifs <- matchMotifs(
    klf.motif, n.peaks, genome,
    out      = "positions",   # return GRanges of hits, not a binary matrix
    p.cutoff = 5e-04          # match stringency
)
```

`matched_motifs` is a `GRangesList` — one element per input PWM. Each element
is a `GRanges` of motif hits with a `strand` set to `+` or `-`.

## 2. Extract a strand-aware signal matrix

```r
#' Extract a per-site signal matrix centered on motif occurrences.
#'
#' @param sites        GRanges of motif hits (must have strand info).
#' @param cov.obj      An RleList coverage track (e.g. coverage(reads) or
#'                     import(bw, as = "RleList")).
#' @param window_size  Half-window in bp; output is 2 * window_size wide.
#' @return A numeric matrix (n_sites x 2*window_size). Minus-strand sites are
#'         reversed so every row reads 5' -> 3' relative to the motif.
#'         Row order matches the input `sites` order.
extract_signal <- function(sites, cov.obj, window_size) {

    # Resize every site to a fixed-width window centered on the motif.
    motif_windows <- resize(sites, width = window_size * 2, fix = "center")

    # Tag each window with its position in the input so we can restore the
    # original order after the per-chromosome split/rbind round-trip.
    motif_windows$.orig_idx <- seq_along(motif_windows)

    # Split by chromosome so we can extract from the matching Rle.
    motif_windows_by_chr <- split(motif_windows, seqnames(motif_windows))

    sig_list <- lapply(names(motif_windows_by_chr), function(chr) {
        windows_chr <- motif_windows_by_chr[[chr]]
        if (length(windows_chr) == 0) return(NULL)

        # Slice the chromosome's coverage Rle by the window coordinates.
        views <- Views(cov.obj[[chr]],
                       start = start(windows_chr),
                       end   = end(windows_chr))
        m <- as.matrix(views)

        # Flip minus-strand rows left<->right so every row is 5' -> 3'
        # relative to the motif (not the reference genome).
        is_minus <- as.character(strand(windows_chr)) == "-"
        if (any(is_minus)) {
            m[is_minus, ] <- t(apply(m[is_minus, , drop = FALSE], 1, rev))
        }

        # Carry the original index along as the row name so we can reorder.
        rownames(m) <- as.character(windows_chr$.orig_idx)
        m
    })

    fin_mat <- do.call(rbind, sig_list)

    # Restore the input row order (the split scrambled it by chromosome).
    fin_mat[order(as.integer(rownames(fin_mat))), , drop = FALSE]
}
```

## Example usage

```r
# Pick the first PWM's hits, 200 bp on each side -> 400-wide matrix
mat <- extract_signal(matched_motifs[[1]], needed_cov, 200)

# Average profile across all sites (the classic "metaplot" line)
avg_profile <- colMeans(mat)
plot(seq(-200, 199), avg_profile, type = "l",
     xlab = "Distance from motif center (bp)", ylab = "Mean signal")
```

!!! tip "Why reverse minus-strand rows?"
    On the minus strand, a motif's 5′→3′ direction runs *right to left* in
    genome coordinates. Reversing those rows lines every site up in motif
    orientation, so column *i* always means "*i* bp downstream of the motif
    start" regardless of strand. Skip this step and asymmetric features (e.g.
    nucleosome shifts on one side of the motif) will average out to nothing.

!!! note "Coverage object format"
    `cov.obj` must be an `RleList`-like object indexable by chromosome name.
    Build one from aligned reads with `coverage(reads)` or load a bigWig with
    `rtracklayer::import(bw_path, as = "RleList")`.

!!! warning "Windows must stay inside the chromosome"
    `Views()` will error if any window runs off the chromosome end. If your
    sites sit near telomeres, trim first:

    ```r
    sites <- trim(resize(sites, width = window_size * 2, fix = "center"))
    sites <- sites[width(sites) == window_size * 2]
    ```
