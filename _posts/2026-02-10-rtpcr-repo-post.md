---
title: "RTPCR: A Unified RT-qPCR Analysis Pipeline in R"
excerpt: "How to run the RTPCR repository for two-time-point and multi-time-point RT-qPCR workflows"
last_modified_at: 2026-02-11T10:20:00-0800
categories:
  - bioinformatics
  - data
tags:
  - RT-qPCR
  - R
  - pipeline
  - reproducibility
  - github
---

Repository: <https://github.com/BJ-Chen-Eric/RTPCR>

I built **RTPCR** because my RT-qPCR analysis kept drifting into repetitive manual work.

This post is a practical guide to the repository and how I use it for both:

- two-time-point experiments
- multi-time-point experiments

The goal is simple: fewer manual steps, fewer inconsistent spreadsheets, and more reproducible outputs.

## Why I built this

When I handled RT-qPCR runs manually, I kept repeating the same loop: clean sample names, check control genes, compute fold changes, export tables, then rebuild plots when one new file arrived.

So I turned that loop into a single pipeline. The result is:

- consistent across experiments
- easier to rerun when data updates
- easier to share with collaborators

## The calculation concept (Livak 2^-ΔΔCt)

This pipeline uses standard relative quantification with the **Livak method (2^-ΔΔCt)**.

In RT-qPCR, the Ct (cycle threshold) is the cycle number where fluorescence crosses a threshold. Lower Ct means higher starting template abundance.

Expression is estimated through these steps:

```text
ΔCt = Ct(target) - Ct(reference_gene)
ΔΔCt = ΔCt(sample) - ΔCt(calibrator)
Fold change = 2^(-ΔΔCt)
```

Interpretation:

- fold change `> 1`: up-regulated vs calibrator
- fold change `= 1`: no change
- fold change `< 1`: down-regulated vs calibrator

In this repository, the reference gene is typically auto-detected as `5.8S` first, then `TUB` (or can be set explicitly with `--control-gene`), and the calibrator is usually a control sample/time (for example `0h`, configurable with `--control-time`).

### Calculation assumptions

The 2^-ΔΔCt method is most reliable when:

- target and reference assays have similar amplification efficiencies
- reference gene expression is stable across conditions
- Ct values are from quality-controlled technical/biological replicates

### How this is applied in the pipeline

For each run, the pipeline:

1. Parses and standardizes raw qPCR export files.
2. Resolves sample names and time labels.
3. Aggregates replicate Ct values per sample and gene.
4. Computes ΔCt using the selected control gene.
5. Computes ΔΔCt relative to the selected control condition/time.
6. Exports fold-change tables and figures with consistent formatting.

So every run follows the same logic, which is exactly what I wanted for cross-batch consistency.

## What you get

The repo exposes the same workflow in two ways:

- script mode: `work_pp/rt_pct_strain_source_unified.R`
- package mode: `pipeline_Rpack`

Main package entrypoints:

- `pipelineRpack::run_rt_pct_strain_source_unified()`
- `pipelineRpack::rt_pct_strain_source_unified_cli()`

## Installation

Clone the repository:

```bash
git clone git@github.com:BJ-Chen-Eric/RTPCR.git
cd RTPCR
```

Install required R packages:

```r
install.packages(c(
  "dplyr", "purrr", "fs", "seqinr", "stringr", "data.table", "ggplot2",
  "tidyr", "tibble", "openxlsx", "cowplot", "scales"
))
```

Optional: install as local R package:

```bash
cd RTPCR/pipeline_Rpack
R CMD INSTALL .
```

Or install package from GitHub:

```r
install.packages("remotes")
remotes::install_github("BJ-Chen-Eric/RTPCR", subdir = "pipeline_Rpack")
```

## Basic usage

From repository root:

```bash
Rscript work_pp/rt_pct_strain_source_unified.R --help
```

Or from inside `work_pp/`:

```bash
cd RTPCR/work_pp
Rscript rt_pct_strain_source_unified.R --help
```

Single file run:

```bash
Rscript work_pp/rt_pct_strain_source_unified.R \
  --files "RTpcr/raw/example.csv" \
  --out-name run_demo
```

Multiple files run:

```bash
Rscript work_pp/rt_pct_strain_source_unified.R \
  --files "RTpcr/raw/file1.csv,RTpcr/raw/file2.csv" \
  --out-name run_merge
```

## Multi-time-point example

```bash
Rscript work_pp/rt_pct_strain_source_unified.R \
  --files "RTpcr/raw/file1.csv,RTpcr/raw/file2.csv" \
  --analysis-mode multi \
  --multi-compare-style all_time \
  --control-time 0h \
  --out-name run_multi
```

## Expected inputs and outputs

Input expectation:

- qPCR-export CSV format
- header row begins with `Well`
- includes sample name, sample type, target gene, and Ct columns

Default locations:

- raw input: `RTpcr/raw/`
- output folder: `RTpcr/output/<out-name>/`

Typical outputs:

- `all_raw_results.xlsx`
- `all_raw_time_results.xlsx`
- `all_time_results.xlsx` (when generated)
- `figure/*.png`

## Common pitfalls

- Missing control gene rows in a sample/time group can stop analysis.
- If time labels are not detectable (e.g., missing `0h`, `1h`), parsing will fail.
- If package dependencies are missing, install them first before re-running.

## Reproducibility

For stable reruns, I recommend pairing this repository with a project-scoped Conda R environment and locking package versions when possible.

That keeps reports reproducible across machines and over time.

## Links

- GitHub repo: <https://github.com/BJ-Chen-Eric/RTPCR>
- Package path: `pipeline_Rpack`
- Script path: `work_pp/rt_pct_strain_source_unified.R`
