---
title: "TWAS Workshop: From GWAS Summary Statistics to Candidate Genes with S-PrediXcan"
excerpt: "A hands-on, no-background-needed workshop for running a transcriptome-wide association study on Mac or Windows"
last_modified_at: 2026-10-29T12:00:00-0700
categories:
  - Genomics
  - bioinformatics
tags:
  - TWAS
  - GWAS
  - eQTL
  - S-PrediXcan
  - workshop
---

GitHub: `https://github.com/BJ-Chen-Eric/twas_workshop`

This post introduces a small workshop I put together for running a **transcriptome-wide association study (TWAS)** end to end with **S-PrediXcan**. It is written for people with no bioinformatics background: one setup script, one analysis script, and reference output to check your answers against. Below is the idea behind the method, how to run it, and how to read what comes out.

## The question in plain terms

Small differences in your DNA can change how *active* a gene is, meaning how much of its product your cells actually make. A TWAS asks a simple question: **is a gene's activity level linked to a disease or trait?** It is one of the first steps researchers take to narrow ~20,000 human genes down to the handful worth studying.

The technical framing: TWAS tests whether a gene's genetically **predicted** expression is associated with a trait, using GWAS summary statistics. No lab measurements or individual-level genotypes are required.

## How the pieces fit together

```
   [GTEx reference panel]
   (real genotypes + real measured expression, same people)
             │
             ▼  (Elastic Net training, done once, ahead of time)
   [weight model: en_<tissue>.db + its covariance file]
             │
             │            [your GWAS summary statistics]
             │                       │
             └───────────┬───────────┘
                         ▼
                  [S-PrediXcan]
                         │
                         ▼
       [one row per gene: zscore, pvalue, ...]
```

### Why "predicted" expression?

Measuring expression requires tissue samples, which most GWAS cohorts don't have. Instead, a **weight model** is trained on a reference panel like GTEx, where genotypes and expression were measured in the same people. The model learns which nearby SNPs shift a gene's expression (its **eQTLs**) and by how much. For one gene, predicted expression is just a weighted sum:

```
predicted expression = beta_1 × SNP_1 + beta_2 × SNP_2 + ... + beta_n × SNP_n
```

- `SNP_i` is the genotype at SNP *i* (0, 1, or 2 copies of the alternate allele, or a 0–2 dosage for imputed data).
- `beta_i` is that SNP's eQTL effect size, stored as `weight` in the model's `.db` file.

S-PrediXcan never computes this sum per person. It folds the same betas into a gene-level test directly from GWAS summary statistics, using a SNP covariance matrix to account for correlation (linkage disequilibrium) between nearby SNPs. That reuse of a pre-trained model on summary data is what makes TWAS work on public data.

### Where the GWAS file comes from

The input `chr10_imputed_gwas.pheno.glm.logistic.hybrid` is the end product of a standard pipeline:

```
raw genotyped samples → QC → population-structure & relatedness correction
  → imputation against a reference panel → per-SNP association test → summary statistics
```

The file name decodes the whole pipeline: chromosome 10, imputed, and PLINK2's `--glm` logistic regression output for a case-control trait (the "hybrid" algorithm falls back to Firth regression when plain logistic regression doesn't converge).

## Running the workshop

Download the repo (green **`<> Code`** button → **Download ZIP**, or `git clone`), open a terminal in the folder, then:

### Step 1: Setup (run once)

```bash
# Mac
bash script/mac/step1_setup.sh
```

```powershell
# Windows (the leading .\ is required in PowerShell)
.\script\windows\step1_setup.bat
```

This checks Python, creates a virtual environment, installs the packages (`numpy<2.0`, `pandas<2.0`, `scipy`, `patsy`, `h5py`, `sqlalchemy<2.0`, `pyliftover`, `statsmodels`), downloads the toy weight models (~130 MB) and GWAS file (~31 MB) from Zenodo, and verifies everything is ready. The MetaXcan toolkit itself ships inside the repo.

### Step 2: Tissue-level TWAS

```bash
# Mac
bash script/step2_run_native_tissue.sh
```

```powershell
# Windows
.\venv\Scripts\python.exe script\step2_run_native_tissue.py
```

This runs S-PrediXcan for two tissues, **Spleen** and **Whole Blood**, each with its own weight model and covariance file, against the chromosome 10 GWAS. Compare your output in `out/` against `demo_out/`; tiny differences in the last decimal place across machines are normal.

### Step 3 (optional): Cell-type track

```bash
bash script/step3_run_native_singlecell.sh
```

This runs **individual-level PrediXcan** for `CD14-positive_monocyte`, which computes the weighted sum above for each person and regresses it against a phenotype. The genotypes here are **entirely synthetic**: simulated per SNP from real allele frequencies, without real linkage between SNPs. Since multi-SNP prediction relies on that linkage, the results are close to noise by design. The track only exists to show the individual-level pipeline running end to end. (A "Skipping plasmablast" message is expected.)

Prefer double-clicking? Every step has a `.command` (Mac) or `.bat` (Windows) equivalent in `script/mac/` and `script/windows/`.

## Reading the output

Each tissue produces a CSV with one row per gene:

| Column | Meaning |
|---|---|
| `gene`, `gene_name` | Ensembl ID and gene symbol |
| `zscore`, `pvalue` | The association test. **Use these.** |
| `effect_size` | Always `NA` here: the GWAS supplies Z-scores, not beta + SE |
| `var_g` | Variance of predicted expression; an internal normalizing term |
| `pred_perf_r2`, `pred_perf_pval` | How well the model predicted *measured* expression during training (fixed property of the model, unrelated to this GWAS) |
| `pred_perf_qval` | FDR version of the above; `NULL` throughout `en_Whole_Blood.db` |
| `n_snps_used` ≤ `n_snps_in_cov` ≤ `n_snps_in_model` | How much of the model's SNP panel actually overlapped the GWAS |

It helps to keep two groups apart: the `pred_perf_*` columns describe the **model**, while `zscore`/`pvalue`/`n_snps_*` describe what happened when the model met **this GWAS**.

### Picking out a meaningful gene

Three checks separate "worth a closer look" from "probably noise":

1. **Is the model trustworthy for this gene?** `pred_perf_pval < 0.05` and `pred_perf_r2 > 0.01`.
2. **Is the association strong enough?** Use a Bonferroni cutoff of `0.05 / (genes actually tested)`. The often-quoted `2.5e-6` assumes ~20,000 genes; compute your own from your row count.
3. **Does it rest on enough SNPs?** A hit with `n_snps_used = 1` is just one SNP's GWAS signal wearing a gene's name.

### What the demo run shows

The toy run tests 240 genes in Spleen and 269 in Whole Blood, so the Bonferroni cutoffs are roughly `2.1e-4` and `1.9e-4`. The top genes are:

| Tissue | Gene | zscore | pvalue | pred_perf_r2 | n_snps_used |
|---|---|---|---|---|---|
| Spleen | C10orf128 | 2.12 | 0.034 | 0.33 | 48 / 85 |
| Spleen | ACTA2 | -2.11 | 0.035 | 0.06 | 34 / 38 |
| Whole Blood | SLIT1 | 2.71 | 0.0068 | 0.30 | 36 / 56 |
| Whole Blood | MSRB2 | 2.46 | 0.014 | 0.03 | 18 / 24 |
| Whole Blood | FAS | -2.42 | 0.015 | 0.08 | 8 / 8 |

These genes pass checks 1 and 3: the models predict expression well and use plenty of SNPs. But none comes close to the Bonferroni threshold. That is a useful lesson in itself: a nominal p < 0.05 among a few hundred tests is what you would expect by chance, not a discovery.

## The big caveat: association is not causation

A significant TWAS gene means its *predicted* expression is associated with the trait. It does **not** prove the gene causes it. Neighbouring genes share SNPs through linkage disequilibrium, so a TWAS can point at a bystander sitting next to the real causal gene.

Follow-up methods address this:

| Method | Question it answers |
|---|---|
| **Mendelian Randomization** (IVW + MR-Egger) | Does the genetic evidence support a causal effect of expression on the trait, and is there pleiotropy (variants acting through another path)? MR-Egger needs ≥3 independent instruments, which a single-eQTL gene often lacks. |
| **COLOC** | Are the GWAS and eQTL signals driven by the *same* causal variant? Look at **PP.H4**. |
| **SuSiE** (+ `coloc.susie`) | At loci with multiple independent signals, which credible sets of variants are likely causal on each side? |
| **SMR + HEIDI** | A fast single-SNP MR using the top eQTL; a non-significant HEIDI is consistent with one shared causal variant. |

LD score regression (LDSC) is related but answers a genome-wide question (SNP-heritability and genetic correlation between traits), not a per-hit one. It is a good sanity check that a GWAS has enough heritable signal to be worth running a TWAS on.

## Summary

- TWAS links genes to traits by combining pre-trained eQTL weight models with GWAS summary statistics.
- The workshop runs S-PrediXcan on Spleen and Whole Blood in three commands, on Mac or Windows, with reference output to check against.
- Read results with three filters: model quality, multiple-testing-corrected significance, and SNP coverage.
- Treat any hit as a lead, then test it with colocalization or MR before calling it causal.

Toolkit: [hakyimlab/MetaXcan](https://github.com/hakyimlab/MetaXcan)
