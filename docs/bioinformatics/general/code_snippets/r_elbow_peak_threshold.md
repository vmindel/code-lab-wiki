# Elbow-Point Threshold for Peak Subsetting (R)

After calling peaks jointly on all replicates you usually end up with a long
tail of low-signal regions that are safer to drop. Sorting the peaks by signal
and plotting rank vs. signal gives the classic "knee" curve — the elbow is a
natural, data-driven cutoff.

This snippet finds that elbow using the **maximum-distance-to-chord** method:

1. Normalise the (rank, signal) coordinates to `[0, 1]`.
2. Draw the straight line from the first point to the last point.
3. The elbow is the point that sits furthest from that line.

No smoothing, no derivatives, no extra packages — works well when the curve is
monotonic (as it is for signal-sorted peaks).

## The function

```r
#' Find the elbow point of a monotonic curve.
#'
#' @param x Numeric vector of x-coordinates (e.g. rank of the peak).
#' @param y Numeric vector of y-coordinates (e.g. peak signal), same length as x.
#' @return A list with `index` (position in the input vectors), `rank` (the x
#'         value at the elbow) and `signal` (the y value at the elbow).
find_elbow_distance <- function(x, y) {
    # Normalize coordinates to [0, 1] so x and y contribute on the same scale.
    x_norm <- (x - min(x)) / (max(x) - min(x))
    y_norm <- (y - min(y)) / (max(y) - min(y))

    # Line from first to last point: a*x + b*y + c = 0
    x1 <- x_norm[1]
    y1 <- y_norm[1]
    x2 <- x_norm[length(x_norm)]
    y2 <- y_norm[length(y_norm)]

    a <- y2 - y1
    b <- x1 - x2
    c <- x2 * y1 - x1 * y2

    # Perpendicular distance from each point to the chord.
    distances <- abs(a * x_norm + b * y_norm + c) / sqrt(a^2 + b^2)

    elbow_idx <- which.max(distances)
    list(
        index  = elbow_idx,
        rank   = x[elbow_idx],
        signal = y[elbow_idx]
    )
}
```

## Example: subset peaks called from all replicates

Assume `peaks` is a `GRanges` (or data.frame) with a `signal` column produced
by calling peaks jointly on all replicates.

```r
# 1. Sort peaks by signal, descending.
ord    <- order(peaks$signal, decreasing = TRUE)
peaks  <- peaks[ord]

# 2. Find the elbow of the rank-vs-signal curve.
elbow  <- find_elbow_distance(seq_along(peaks), peaks$signal)

# 3. Keep only the peaks above the elbow.
peaks_kept <- peaks[seq_len(elbow$index)]

message(sprintf(
    "Elbow at rank %d (signal = %.3g); kept %d / %d peaks.",
    elbow$rank, elbow$signal, length(peaks_kept), length(peaks)
))
```

## Sanity-check plot

A quick look at the curve with the elbow marked is the fastest way to confirm
the cutoff is sensible:

```r
plot(seq_along(peaks), peaks$signal,
     type = "l", xlab = "Rank", ylab = "Signal",
     main = "Peak signal vs. rank")
abline(v = elbow$index, col = "red", lty = 2)
points(elbow$index, elbow$signal, col = "red", pch = 19)
```

!!! tip "When the elbow method misbehaves"
    The chord-distance elbow assumes a roughly monotonic curve with a single
    knee. If your signal distribution is bimodal or very flat, the "elbow"
    can drift — always eyeball the plot before committing to the cutoff.
