# 🧬 Barkai Lab TF Grammar Pipeline
> **Cluster Guide: Running Sequence Grammar Engineering Directly from `/home/labs/barkailab/aileenc/TF_Grammar_Pipeline`**
>
> 💡 **No setup, cloning, or downloads needed!** Everything (Python environment, libraries, and script dependencies) is already installed and pre-configured on the cluster. Anyone in the lab can run it immediately from any directory.

---

## 📌 What Does This Tool Do?

This pipeline redesigns transcription factor (TF) sequence syntax while automatically **protecting DNA-Binding Domains (DBDs) and structural folds**:

1. **Hydrophobic Shifts (`shift`):** Disperses hydrophobic residues (`L, I, V, F, W, Y`) by 1 amino acid left/right to break transactivation motifs while **preserving 100% of the amino acid composition**.
2. **Substitutions (`substitution`):** Systematically replaces chosen amino acids with a target residue (e.g. all `Q` &rarr; `N`, `{Q, N}` &rarr; `G`, or all `LIVFWY` &rarr; `N`).

Every run automatically queries **UniProt** and **EMBL-EBI InterPro** to lock structured domains, edits the disordered regions (IDRs), and outputs:
- **Codon-optimized cDNA coding sequences (CDS)** ready to order from Twist / IDT / GenScript.
- **Engineered protein FASTAs**.
- **Multi-page visual alignment diagrams (PDF & high-res PNGs)** with colored residues and domain overlays.
- **Audit spreadsheet (CSV)** recording motifs broken, clusters created, and domain coordinates.

---

## 🚀 Quick Usage: 4 Ways to Run the Pipeline

---

### 🌟 Method 1: In ANY Jupyter Notebook (Run Directly from Your Own Folder!)
You don't need to navigate to this folder. In **your own notebook**, in your own directory:

#### Option A: The 1-Line Python Runner (100% Reliable, Zero Widget Issues!)
*No browser extensions needed! Automatically executes and displays the high-resolution sequence drawings right inside your notebook:*

```python
import sys
sys.path.append("/home/labs/barkailab/aileenc/TF_Grammar_Pipeline")
from grammar_pipeline import run_grammar

# Run default hydrophobic shifts on Sox2:
run_grammar("Sox2")

# Run on multiple proteins:
run_grammar(["Sox2", "Nanog", "Foxa3"])

# Run on Yeast or Mouse:
run_grammar("Gal4", organism="yeast")

# Run Substitution:
run_grammar("Sox2", op="sub", source="Q", target="N")
```

#### Option B: Interactive Dropdown Dashboard (`show_gui()`)
If you want clickable dropdown buttons in your notebook:
```python
import sys
sys.path.append("/home/labs/barkailab/aileenc/TF_Grammar_Pipeline")
from grammar_pipeline import show_gui

show_gui()
```
> ⚠️ **Note on `VBox(children=(...`:** If your notebook prints `VBox(children=(...` instead of rendering buttons, it means your browser or JupyterLab session doesn't have the `ipywidgets` extension enabled. Simply use **Option A (`run_grammar('Sox2')`)** above—it never requires extensions and always displays the visual results!

#### Option C: 1-Click Shortcut in your Jupyter Sidebar (Symlink)
If you want the notebook dashboard to appear in your JupyterLab file browser so you can double-click it:
Run this 1-line command in your terminal:
```bash
ln -s /home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar_gui.ipynb ~/TF_Grammar_GUI.ipynb
```

---

### ⚡ Method 2: The 1-Line Terminal Shortcut (Run from Any Directory!)
In your own directory, you can run:

#### A. Single Protein (Hydrophobic Shifts)
```bash
# Run default hydrophobic shifts on human Sox2:
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar Sox2

# Run on Nanog:
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar Nanog
```

#### B. Multiple Proteins at Once
```bash
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar Sox2 Nanog Foxa3 Cebpa
```

#### C. Yeast or Mouse Proteins
```bash
# Yeast transcription factors (automatically uses yeast codon optimization):
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar yeast Gal4
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar yeast Gal4 Gcn4 Ste12

# Mouse transcription factors:
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar mouse Foxp3 Sox2
```

#### D. Amino Acid Substitutions
Syntax: `run_grammar sub <source_AAs> <target_AA> <proteins>`
```bash
# Replace all Q with N in Sox2:
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar sub Q N Sox2

# Replace both Q and N with G in yeast Gal4:
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar sub QN G Gal4

# Replace all hydrophobic residues (L,I,V,F,W,Y) with N in Nanog:
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar sub LIVFWY N Nanog
```

#### E. Interactive Step-by-Step Terminal Wizard
If you run without arguments, it prompts you interactively:
```bash
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar
```

> **Tip:** You can add an alias to your `~/.bashrc` to make it even faster:
> ```bash
> alias run_grammar="/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar"
> ```
> Then you can just type `run_grammar Sox2` from anywhere!

---

### 🐍 Method 3: In Your Own Python Script or Jupyter Notebook
You can import the Grammar Pipeline directly into your Python scripts without copying or installing anything:

```python
import sys
# Add the Grammar Pipeline directory to your python path
sys.path.append("/home/labs/barkailab/aileenc/TF_Grammar_Pipeline")
from grammar_pipeline import run_grammar_pipeline

# -------------------------------------------------------------
# Example 1: Hydrophobic Shifts on Mammalian Factors
# -------------------------------------------------------------
results = run_grammar_pipeline(
    operation="shift",                  # "shift" or "substitution"
    organism="human",                   # "human", "mouse", or "yeast"
    proteins=["Sox2", "Nanog", "FoxM1"], # gene symbols or UniProt IDs
    mode="structured",                  # "structured" (default), "dbd", or "all"
    output_dir="./my_tf_shifts"         # where to save outputs
)

print("Generated Files:")
print(" - Protein FASTA: ", results["prot_fasta"])
print(" - cDNA CDS FASTA:", results["cdna_fasta"])
print(" - Summary CSV:   ", results["summary_csv"])
print(" - Drawings PDF:  ", results["pdf_drawings"])

# -------------------------------------------------------------
# Example 2: Systematic Substitutions in Yeast Factors
# -------------------------------------------------------------
sub_results = run_grammar_pipeline(
    operation="substitution",
    source_aas="Q,N",                   # amino acids to replace
    target_aa="G",                      # replacement amino acid
    organism="yeast",
    proteins=["GAL4", "MSN2"],
    mode="structured",                  # protects DBDs; substitutes IDRs only
    output_dir="./my_yeast_substitutions"
)
```

---

### 💻 Method 4: Slurm Batch Job (`sbatch`) on the Cluster
For large batches of proteins, submit a job script like this:

```bash
#!/usr/bin/env bash
#SBATCH --job-name=tf_grammar
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G
#SBATCH --time=01:00:00
#SBATCH --output=grammar_%j.log

# Run the Grammar Pipeline on a batch of human TFs:
/home/labs/barkailab/aileenc/TF_Grammar_Pipeline/run_grammar \
  --op shift \
  --organism human \
  --proteins Sox2 Foxa3 Zic2 Pax7 Gata3 Gata6 FoxP3 Gata2 Sox4 Hnf1a Nanog Cebpa FoxA1 FoxM1 Tcf4 \
  --mode structured \
  --output-dir ./batch_shift_results
```

---

## 🎛️ Summary of Options & Parameters

| Parameter | Options | Description | Default |
| :--- | :--- | :--- | :--- |
| **`--op`** | `shift`, `substitution` | Operation to perform. | `shift` |
| **`--source`** | e.g. `Q`, `Q,N`, `LIVFWY` | Source amino acid(s) to replace (substitution only). | `Q` |
| **`--target`** | e.g. `N`, `G`, `A` | Target replacement amino acid (substitution only). | `N` |
| **`--organism`** | `human`, `mouse`, `yeast` | Organism for UniProt search and codon optimization. | `human` |
| **`--proteins`** | Gene symbols / Accessions | Space-separated list (e.g. `Sox2 Nanog Gal4`). If omitted, runs all benchmark TFs. | Benchmark list |
| **`--mode`** | `structured`, `dbd`, `all` | **`structured`**: Locks DBDs + structural folds (InterPro). Edits IDRs only.<br>**`dbd`**: Locks DBDs only.<br>**`all`**: Edits the whole protein (except start Met-0). | `structured` |
| **`--output-dir`**| Path | Directory where FASTAs, CSV, and drawings are saved. | `./grammar_pipeline_output` |

---

## 🧠 How the Hydrophobic Shift Algorithm Works

The algorithm is specifically designed to **disrupt potential transactivation motifs without creating large hydrophobic clumps**:

1. **Target Residues:** Only shifts `L, I, V, F, W, Y`.
2. **Methionine Rules:**
   - **Start Met (Met-0):** Position 1 is strictly locked to protect the `ATG` start codon.
   - **Internal Met:** In IDRs, internal Methionines are not locked and can swap with an adjacent `LIVFWY`.
3. **Strict 1-AA Movement:** Residues move only **1 position left or right** ($\pm 1$), swapping with their immediate neighbor. They never jump 2+ positions.
4. **Alternating Direction ("One left, one right..."):**
   Across successive hydrophobics along the sequence, shifts alternate in direction:
   $$\text{Residue } 1 \to \text{Left } (-1), \quad \text{Residue } 2 \to \text{Right } (+1), \quad \text{Residue } 3 \to \text{Left } (-1), \quad \dots$$
   *Example:* In a motif like `A L D G F N`, `L` moves left and `F` moves right &rarr; `L A D G N F` ($D=3 \to D=5$, motif broken!).
5. **Clusters of 2 are Permitted ($\Phi-\Phi$):**
   - Creating a pair of 2 adjacent hydrophobics ($D=1$) is completely allowed.
   - If creating a cluster of 2 widens the distance on the other side to break an adjacent motif, **the algorithm earns a +350 bonus** (*"that's good for us"*).
6. **Clusters of 3 are FORBIDDEN ($\Phi-\Phi-\Phi$):**
   - 3 consecutive adjacent hydrophobics in a row is strictly prohibited with a prohibitive **$-100,000$ penalty** in the optimizer.
7. **Tri-Way Curation:**
   The algorithm automatically runs 3 distinct strategies:
   - **Way 1:** N-to-C alternating scan
   - **Way 2:** C-to-N alternating scan
   - **Way 3:** Two-stage scan (`L, I, V` first, `F, W, Y` second)
   It selects the highest-scoring winner that breaks the most motifs with zero clusters of 3.

---

## 📁 What Output Files Do You Get?

When you run a job, the output folder contains:

| File Name | Format | Content & Usage |
| :--- | :--- | :--- |
| **`grammar_<op>_<mode>_proteins.fasta`** | FASTA | Full engineered protein sequences with stats in headers. |
| **`grammar_<op>_<mode>_cDNA_CDS.fasta`** | FASTA | Full-length codon-optimized cDNA (`ATG` &rarr; `Stop`), ready for Twist/IDT ordering. |
| **`grammar_<op>_<mode>_summary.csv`** | CSV | Spreadsheet with motifs broken, clusters created, locked coordinates, and active IDRs. |
| **`grammar_<op>_<mode>_all_drawings.pdf`** | PDF | Multi-page PDF with wrapped 50-AA sequence alignments for all TFs. |
| **`XX_<TF>_<op>_<mode>.png`** | PNG | Standalone high-res diagram for each individual protein. |

### Visual Sequence Diagram Layout:
- **Top Row (Orig):** Wild-type sequence with amino acid colors.
- **Bottom Row (Eng):** Engineered sequence. Each amino acid keeps its original color; **shifted/mutated residues are highlighted with a red border (`#dc2626`)**.
- **Red Shaded Blocks:** Locked DNA-Binding Domains (DBDs).
- **Blue Shaded Blocks:** Locked Structural Folds (InterPro).

---

## 🔬 Pre-Run Benchmark Results (View Right Now!)

You can inspect pre-computed results for 15 human benchmark TFs inside the **[`example_outputs/`](example_outputs/)** directory:
- 📊 **[Summary Audit CSV](example_outputs/grammar_shift_structured_summary.csv)**
- 📑 **[All Drawings PDF](example_outputs/grammar_shift_structured_all_drawings.pdf)**
- 🖼️ **[Sox2 High-Res PNG](example_outputs/01_Sox2_shift_structured.png)**
- 🧬 **[Engineered Proteins FASTA](example_outputs/grammar_shift_structured_proteins.fasta)**
- 🧬 **[Codon-Optimized cDNA FASTA](example_outputs/grammar_shift_structured_cDNA_CDS.fasta)**

---

