# How to Get UDI Barcodes for a Plate

## 1. Log in to WEXAC

Connect to the VPN if you are outside the institute, then SSH in.

## 2. Activate a Python environment

You need Python with `pandas` and `numpy`. Activate whichever conda environment has them:

```bash
conda activate <your_env>
```

Check it worked:

```bash
which python
```

You should see something like `~/miniconda/envs/<your_env>/bin/python`.

## 3. Run the script

```bash
python /home/labs/barkailab/vovam/pipelines/get_plate_indices.py <plate_number> <output.csv>
```

Replace `<plate_number>` with the plate you need (1–16) and `<output.csv>` with the path where you want the result saved.

**Example — plate 2:**

```bash
python /home/labs/barkailab/vovam/pipelines/get_plate_indices.py 2 ./p2.csv
```

The barcodes for that plate will be written to `p2.csv` in your current directory.

## What the output looks like

The CSV is already in the correct column order for the sequencing spreadsheet:
**i7 → Index 1**, **reverse i5 → Index 2**.

| Well | Index Pair | Index 1 (i7) | Index Pair | Index 2 (i5 rev) |
| ---- | ---------- | ------------ | ---------- | ---------------- |
| A01 | xGen 10nt UDI Index Pair 97 | GCGCGGTTAA | xGen 10nt UDI Index Pair 97 | AAGCTATAGC |
| B01 | xGen 10nt UDI Index Pair 109 | TAGGAAGCGG | xGen 10nt UDI Index Pair 109 | CCAATCCAGG |
| C01 | xGen 10nt UDI Index Pair 121 | ACGGTCGCAT | xGen 10nt UDI Index Pair 121 | GTACTCTCTA |
| D01 | xGen 10nt UDI Index Pair 133 | CCTATGTGTA | xGen 10nt UDI Index Pair 133 | TAGTGAGTCG |
| E01 | xGen 10nt UDI Index Pair 145 | TTGGATTGTC | xGen 10nt UDI Index Pair 145 | CGGCTGTTAG |
| F01 | xGen 10nt UDI Index Pair 157 | GCCGTGAACA | xGen 10nt UDI Index Pair 157 | GCACCACCAT |
| G01 | xGen 10nt UDI Index Pair 169 | TAAGATCGGA | xGen 10nt UDI Index Pair 169 | TTCATCCGTG |
| H01 | xGen 10nt UDI Index Pair 181 | ATCCTAGAAC | xGen 10nt UDI Index Pair 181 | TGAAGTCTGC |
| A02 | xGen 10nt UDI Index Pair 98 | TTCCTTGAGG | xGen 10nt UDI Index Pair 98 | TAAGACAGCA |
| B02 | xGen 10nt UDI Index Pair 110 | GCATCCTACT | xGen 10nt UDI Index Pair 110 | GACATCGCGA |
| C02 | xGen 10nt UDI Index Pair 122 | TTACACACGG | xGen 10nt UDI Index Pair 122 | AACTCCGAGA |
| D02 | xGen 10nt UDI Index Pair 134 | CAACGTTGAC | xGen 10nt UDI Index Pair 134 | TTGGACGGAA |
| E02 | xGen 10nt UDI Index Pair 146 | AATTGGCACA | xGen 10nt UDI Index Pair 146 | CAGTCGGAGT |
| … | … | … | … | … |
