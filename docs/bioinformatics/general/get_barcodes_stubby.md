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
