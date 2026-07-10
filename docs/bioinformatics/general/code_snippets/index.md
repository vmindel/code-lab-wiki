# Useful Code Snippets

A growing collection of small, reusable code blocks for common bioinformatics
chores. Each snippet lives on its own page — copy a block with the button in its
top-right corner and adapt the variables at the top to your own paths.

## Snippets

| Snippet | Language | What it does |
| --- | --- | --- |
| [Consensus peaks + signal over regions](r_consensus_peaks_signal.md) | R | Merge peak files into consensus regions and summarise bigWig signal over them |
| [Averaging per-nucleotide cut bedGraphs](r_average_pernuc_bedgraphs.md) | R | Average normalised per-nucleotide cut tracks across replicates and export a mean bigWig |
| [Strand-aware signal matrix around motif sites](r_signal_matrix_around_motifs.md) | R | Build a per-site signal matrix centered on motif hits, with minus-strand rows reversed |
| [Tiling peaks into fixed-width windows](tiling_the_peaks.md) | R | Split a GRanges of peaks into non-overlapping tiles; narrow peaks kept as-is, wider peaks split vectorized |

!!! tip "Adding a new snippet"
    Drop a new Markdown file in `docs/bioinformatics/general/code_snippets/`,
    add a row to the table above, and register it under
    `Useful Code Snippets` in `mkdocs.yml`.
