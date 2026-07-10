# Tiling Peaks into Fixed-Width Windows (R)

Splits a `GRanges` of peaks into non-overlapping tiles of a fixed width.
Peaks already narrower than the window are kept as-is; wider peaks are split
into consecutive tiles (the last tile may be shorter). All metadata columns are
propagated and the result is returned sorted.

## Dependencies

```r
library(GenomicRanges)   # GRanges, width, start, end, seqnames, strand, mcols
library(IRanges)         # IRanges, sequence
```

## Function

```r
tile_strict_fast <- function(gr, window = 200L) {

  small <- gr[width(gr) <= window]
  large <- gr[width(gr) >  window]

  if (length(large) == 0) return(sort(small))

  # For each large peak, compute how many tiles it needs
  n_tiles <- ceiling(width(large) / window)

  # Repeat peak index once per tile — vectorized expansion
  peak_idx <- rep(seq_along(large), times = n_tiles)

  # Tile-within-peak index (0-based offset)
  tile_idx <- sequence(n_tiles) - 1L

  # Compute starts and ends vectorized
  starts <- start(large)[peak_idx] + tile_idx * window
  ends   <- pmin(starts + window - 1L, end(large)[peak_idx])

  tiled_gr <- GRanges(
    seqnames = seqnames(large)[peak_idx],
    ranges   = IRanges(starts, ends),
    strand   = strand(large)[peak_idx]
  )

  # Propagate mcols if any
  if (ncol(mcols(large)) > 0) {
    mcols(tiled_gr) <- mcols(large)[peak_idx, , drop = FALSE]
  }

  combined <- c(small, tiled_gr)
  sort(combined)
}
```

## Usage

```r
# Tile peaks into 200 bp windows (default)
tiled <- tile_strict_fast(peaks_gr)

# Custom window size
tiled <- tile_strict_fast(peaks_gr, window = 500L)
```
