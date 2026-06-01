# Everything you need to demultiplex your data (bcl2fastq)

## 1. Prepare the sample sheet

Essentially the format for the spreadsheet is the output of the
[Get Index Sequences](get_barcodes.md) step — running that script in your run
folder produces `nova13_ss.csv`, which is exactly what `bcl2fastq` expects as
`--sample-sheet` below.

## 2. Load the needed module
``` bash
module load bcl2fastq2
```

## 3. Transfer the sample sheet to the folder
You can transfer your prepared sample sheet via SFTP or other means to the folder with the downloaded sequencing data.

## 4. Run the demultiplexing

Submit the job to the LSF cluster:

```bash
bsub -n 48 -q short -R "span[hosts=1]" -R "rusage[mem=1000]" "bcl2fastq --use-bases-mask Y*,I8,I8,Y* -p 48 --no-lane-splitting --barcode-mismatches 1,1 --mask-short-adapter-reads 1 --runfolder-dir ./ --sample-sheet ./nova13_ss.csv --output-dir ./outm0/"
```

### Command breakdown

**LSF scheduler (`bsub`):**

| Flag | Value | Meaning |
|------|-------|---------|
| `-n` | `48` | Request 48 CPU cores |
| `-q` | `short` | Submit to the `short` queue |
| `-R "span[hosts=1]"` | | All cores must be on a single node |
| `-R "rusage[mem=1000]"` | | 1 GB RAM per core (~48 GB total) |

**Demultiplexer (`bcl2fastq`):**

| Flag | Value | Meaning |
|------|-------|---------|
| `--use-bases-mask` | `Y*,I8,I8,Y*` | Read layout: Read1 (all cycles), Index1 (8 bp), Index2 (8 bp), Read2 (all cycles) — dual-indexed paired-end |
| `-p` | `48` | Use 48 processing threads |
| `--no-lane-splitting` | | Merge all lanes into one FASTQ per sample |
| `--barcode-mismatches` | `1,1` | Allow 1 mismatch in each index |
| `--mask-short-adapter-reads` | `1` | Mask reads shorter than 1 bp |
| `--runfolder-dir` | `./` | BCL input from the current directory |
| `--sample-sheet` | `./nova13_ss.csv` | Sample sheet with index–sample mapping you created|
| `--output-dir` | `./outm0/` | Output directory for FASTQ files |


!!! warning "Barcode distance error"
    With `--barcode-mismatches 1,1`, bcl2fastq requires that any two indexes differ by **at least 3 bases** (edit distance ≥ 2×mismatch + 1). If two indexes are too similar the run will fail with:

    ```
    ERROR: Barcode clash: index ... and index ... are too similar
    ```

    **How to fix:**
        Set `--barcode-mismatches 0,0` to disable mismatch tolerance. Safe when sequencing quality is high.
