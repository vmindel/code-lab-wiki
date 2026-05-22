# Bioinformatics File Formats

## Sequence

| Format | Encoding | Description |
|--------|----------|-------------|
| FASTA | Text | Sequence only (DNA, RNA, protein). Header line starts with `>` |
| FASTQ | Text | Sequence + per-base quality scores (Phred). Header starts with `@` |

## Alignment

| Format | Encoding | Description |
|--------|----------|-------------|
| SAM | Text | Sequence Alignment Map. Human-readable; coordinates are **1-based** |
| BAM | Binary | Compressed SAM. Requires index (`.bai`); coordinates stored as **0-based** internally |
| CRAM | Binary | Reference-anchored, highly compressed BAM alternative |

## Annotation & Intervals

!!! info "Coordinate systems"
    - **0-based, half-open** `[start, end)` — start is 0-indexed, end is excluded. BED-derived formats.
    - **1-based, closed** `[start, end]` — both start and end are included. GTF/GFF convention.

| Format | Coordinates | Encoding | Description |
|--------|------------|----------|-------------|
| BED | **0-based**, half-open | Text | Minimal: chrom, start, end. Up to 12 standard fields |
| GTF | **1-based**, closed | Text | Gene/transcript annotation. Used by Ensembl and most aligners |
| GFF3 | **1-based**, closed | Text | More structured than GTF; supports hierarchical features via `Parent=` |
| BEDGraph | **0-based**, half-open | Text | Continuous signal values per interval (e.g. coverage tracks) |
| bigWig | **0-based**, half-open | Binary | Indexed binary BEDGraph. Efficient for genome browsers (IGV, UCSC) |
| bigBed | **0-based**, half-open | Binary | Indexed binary BED. Fast random access for large annotation sets |

## Peak Files

Peak files follow the BED coordinate convention (**0-based, half-open**).

| Format | Fields | Typical Use |
|--------|--------|-------------|
| narrowPeak | BED6 + 4 extra | Point-source peaks — TF ChIP-seq, ATAC-seq, CUT&RUN |
| broadPeak | BED6 + 3 extra | Broad chromatin domains — histone marks (H3K27me3, H3K9me3) |

Extra fields shared by both:

| Field | Description |
|-------|-------------|
| `signalValue` | Overall enrichment signal |
| `pValue` | -log10 p-value |
| `qValue` | -log10 FDR-corrected q-value |
| `peak` | narrowPeak only — offset from start to summit |
