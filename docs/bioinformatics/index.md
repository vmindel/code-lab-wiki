# Bioinformatics
I will collect here usefull scripts and ways I analyze the data for myself and others. his page is also my refernce page for the markdown formatting.



Pipelines, analyses, and one-off scripts.

## Markdown Reference
---

## Example page — delete or replace

This page demonstrates the formatting features available. Copy the patterns when adding new entries.

### Code blocks

```bash
# STAR alignment, paired-end
STAR --runThreadN 8 \
     --genomeDir /path/to/index \
     --readFilesIn R1.fastq.gz R2.fastq.gz \
     --readFilesCommand zcat \
     --outSAMtype BAM SortedByCoordinate \
     --outFileNamePrefix sample_
```

```python
import pandas as pd
counts = pd.read_csv("counts.tsv", sep="\t", index_col=0)
counts = counts[counts.sum(axis=1) >= 10]
```

### Callouts

!!! note
    Use `note`, `tip`, `warning`, `danger`, `info`, `example` for colored callouts.

!!! warning "Gotcha"
    STAR needs the genome index built with the exact `--sjdbOverhang` matching read length minus 1 if you want junctions to behave.

### Tabs

=== "bash"
    ```bash
    samtools view -b -q 30 in.bam > filtered.bam
    ```

=== "snakemake rule"
    ```python
    rule filter_bam:
        input: "aligned/{sample}.bam"
        output: "filtered/{sample}.bam"
        shell: "samtools view -b -q 30 {input} > {output}"
    ```

### Tables

| Tool      | Use                      | Notes                        |
| --------- | ------------------------ | ---------------------------- |
| `STAR`    | Spliced RNA-seq aligner  | Memory-hungry; needs index   |
| `salmon`  | Transcript quantification| Fast; uses pseudo-alignment  |
| `DESeq2`  | Differential expression  | R; needs raw counts          |
