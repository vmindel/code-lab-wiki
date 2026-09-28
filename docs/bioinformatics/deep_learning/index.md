# Deep Learning

Deep-learning models and workflows applied to genomics data (sequence-to-function
models, TF binding predictors, in-silico perturbation experiments, etc.),
including cluster-specific setup and gotchas.

## Pages

- **[Cherimoya (WEXAC)](cherimoya.md)** — the main tutorial: environment and
  CUDA setup, pipeline config, `bsub` flags, the training recipe that actually
  works, interpretation at scale, cross-species fine-tuning. Start here.
- **[Cherimoya from Raw BAMs](cherimoya_raw_bams.md)** — the extra steps when
  you start from replicate BAMs rather than a finished bigwig: pilot subsets,
  calling peaks yourself, capping training loci, surviving `short-gpu`
  preemption.
- **[Fi-NeMo](finemo.md)** — calling individual motif instances from a model's
  attributions, and how to read its QC tables (use `cwm_similarity`, not
  `seqlet_recall`).
- **[KLF/SP Models](klf_sp_models.md)** — the KLF-paper result set: 26
  mammalian models, 38 yeast fine-tunes, 52 interpretation runs, their
  numbers, and which file to open.
