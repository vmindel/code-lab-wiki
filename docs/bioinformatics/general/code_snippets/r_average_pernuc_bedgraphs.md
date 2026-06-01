# Averaging Per Nucleotide Cut Bedgraphs (R)

For replicate samples of the same factor (e.g. ChEC-seq / per-nucleotide
cut tracks), this recipe averages the **normalised read counts at every
genomic position** across all replicates and exports the mean track as a
bigWig.

It assumes the per-sample tracks are stored as **Parquet files** with columns
`Chromosome`, `Start`, `End`, `Normalized_Reads` (one row per cut site).
Parquet is used instead of raw bedGraph because it is much smaller and
faster to load — the columns are otherwise identical to a standard bedGraph.

## Dependencies

```r
library(dplyr)           # filter / group_by / summarise
library(arrow)           # read_parquet
library(GenomicRanges)   # makeGRangesFromDataFrame, coverage
library(rtracklayer)     # export() to bigWig
library(GenomeInfoDb)    # Seqinfo
```

## Input layout

A `df` of file paths and the factor each one belongs to, e.g.

| names | bed_files |
| --- | --- |
| Reb1 | /tracks/Reb1_rep1.parquet |
| Reb1 | /tracks/Reb1_rep2.parquet |
| Abf1 | /tracks/Abf1_rep1.parquet |

You also need a `Seqinfo` for the genome the cuts come from (chromosome
lengths and assembly name) — used to anchor the GRanges and so the
exported bigWig has correct chromosome sizes.

```r
# Example Seqinfo for mm10
mm10_seqinfo <- Seqinfo(genome = "mm10")
```

## The function

```r
#' Average per-nucleotide bedGraphs (stored as Parquet) for one factor and
#' export the mean coverage as a bigWig.
#'
#' @param tfname        Factor name to pull replicates for (matches df$names).
#' @param df            data.frame with columns: names, bed_files (paths).
#' @param seqinfo       A Seqinfo object for the genome (e.g. mm10).
#' @param out_dir       Directory to write the bigWig into.
#' @return (invisibly) the path to the written bigWig.
average_bgs_and_save <- function(tfname, df, seqinfo, out_dir = ".") {

    # 1. Find every replicate file belonging to this factor.
    bedgraph_paths <- df %>%
        filter(names == tfname) %>%
        pull(bed_files)

    # 2. Load all replicates and stack them into one long data.frame.
    loaded_beds      <- lapply(bedgraph_paths, read_parquet)
    n_samples        <- length(bedgraph_paths)
    concatenated_dfs <- do.call(rbind, loaded_beds)

    # 3. Average across replicates: group by genomic position, sum the
    #    normalised reads, then divide by the number of samples.
    #    (Dividing by n_samples — not by group size — treats positions
    #    missing from a replicate as zero, which is what you want for
    #    sparse per-cut tracks.)
    avg_df <- concatenated_dfs %>%
        group_by(Chromosome, Start, End) %>%
        summarise(
            total   = sum(Normalized_Reads),
            mean    = total / n_samples,
            .groups = "drop"
        )

    # 4. Turn the averaged table into a GRanges anchored to the genome.
    gr <- makeGRangesFromDataFrame(
        avg_df,
        seqinfo            = seqinfo,
        keep.extra.columns = TRUE
    )

    # 5. Build an Rle coverage track weighted by the mean signal.
    cov_raw <- coverage(gr, weight = "mean")

    # 6. Export as bigWig.
    out_path <- file.path(out_dir, paste0(tfname, "_mean.bw"))
    export(cov_raw, out_path, format = "bigWig")

    invisible(out_path)
}
```

## Example usage

```r
# One factor
average_bgs_and_save("Reb1", df, mm10_seqinfo, out_dir = "~/tracks/mean")

# All factors in df, in one go
tfs <- unique(df$names)
lapply(tfs, average_bgs_and_save,
       df = df, seqinfo = mm10_seqinfo, out_dir = "~/tracks/mean")
```

!!! tip "Sum vs. mean denominator"
    The mean here divides the per-position sum by the **number of replicates**,
    not by how many replicates actually had a read at that position. For sparse
    per-nucleotide cut tracks this is the right choice: a position seen in only
    1 of 3 replicates should *not* score the same as one seen in all 3.
    If you instead want a presence-conditional average, group on the unstacked
    data and use `mean(Normalized_Reads)`.

!!! warning "Memory"
    `do.call(rbind, loaded_beds)` materialises every replicate's full table in
    memory at once. For many large samples, consider streaming chromosome by
    chromosome, or aggregating with `data.table::rbindlist` + keyed grouping.
