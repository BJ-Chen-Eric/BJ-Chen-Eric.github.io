---
title: "qp_primer_design.py"
layout: post
---

# qp_primer_design.py

This post summarizes how I design RT‑qPCR primers with `qp_primer_design.py` and explains the core primer‑design concepts in plain language.

GitHub: `https://github.com/BJ-Chen-Eric/qpcr-primer-disign`

## Primer life cycle in RT‑qPCR (RNA to Ct)
Before we talk about hairpins, GC%, or junctions, it helps to understand what actually happens to a primer inside an RT‑qPCR reaction.

Act I — RNA becomes DNA
Everything starts with RNA. In RT‑qPCR, your original molecule is mRNA. Reverse transcriptase converts this RNA into single‑stranded cDNA. At this point, there is only one strand. No forward primer yet. No exponential amplification. Just a lone cDNA molecule floating in solution.

Act II — the reverse primer moves first
When PCR cycling begins, the mixture is heated and any secondary structure melts away. Now the single‑stranded cDNA is exposed. Only one primer can bind at this moment: the reverse primer. It anneals to the sense cDNA and polymerase extends from its 3′ end, creating the complementary antisense strand. This first extension quietly builds the missing half of the molecule. No amplification yet.

Act III — the forward primer joins
Next cycle. The double strand is denatured again. Now two templates exist: sense and antisense. Reverse primer binds sense. Forward primer binds antisense. Both primers extend. This is the first true amplification cycle.

Act IV — exponential growth
From here on, every round doubles the number of molecules. Fluorescence rises slowly at first, then crosses the threshold. That crossing point is your Ct. Everything before that—primer binding, 3′ stability, secondary structures, dimer formation—determines when this moment happens.

Why primer quality controls Ct
A primer must find its target, bind stably, present a clean 3′ end, avoid folding onto itself, avoid pairing with its partner, and allow polymerase to initiate extension. If any step is inefficient, amplification slows. Slower amplification means higher Ct, noisier curves, and poorer reproducibility. Primer design is not cosmetic—it directly controls amplification physics.

Transition
With that lifecycle in mind, the design rules start to make sense: hairpins matter because folded primers cannot bind; Tm matters because both primers must work in the same thermal window; GC% balances stability; 3′ ends are where polymerase fires; dimers create false signal; junctions protect against genomic DNA.

## Primer design concepts (plain language)

### Hairpin (self‑folding)
Some primers fold back and hug their own sequence, forming a hairpin. When that happens, part of the primer becomes unavailable. Polymerase can’t use it properly, leading to weak amplification and unstable Ct values. I reject primers with strong self‑folding, especially near the 3′ end.

### Tm (melting temperature)
Tm is the temperature where about half of a primer is bound and half is unbound. If one primer binds strongly and the other weakly, the reaction becomes asymmetric. Forward and reverse primers should have similar Tm values so they work in the same annealing window.

### GC%
G/C pairs are stronger than A/T pairs. Too much GC makes primers bind too tightly and form secondary structures; too little GC makes binding weak. This pipeline uses a hard filter between 45–60% GC for both primers.

### 3′‑end stability
Polymerase extends from the 3′ end, so that end must be clean and stable. I prefer a small GC anchor at the 3′ end and avoid long runs (e.g., CCCC or GGGG) and strong self‑complementarity that can trigger primer‑dimers.

### Primer‑dimer (cross‑dimer)
If the left and right primers bind each other—especially at their 3′ ends—polymerase can extend them and create false signal. I check for strong 3′ complementarity and filter out problematic pairs.

### Amplicon size
Shorter products amplify faster and more consistently. The pipeline ranks candidates around an optimal amplicon size and avoids overly long products.

### Junction‑spanning primers (mRNA)
For multi‑exon genes, junction‑spanning primers help avoid genomic DNA amplification. If a primer overlaps the exon–exon junction, it won’t match unspliced genomic DNA. That’s why junction mode is preferred when possible.

### Putting it all together
Each candidate goes through GC%, Tm balance, hairpin checks, 3′ stability, self‑dimer and cross‑dimer checks, amplicon size ranking, and transcript structure constraints. Optional steps add BLAST‑to‑Ensembl mapping and seqkit QC. Only primers that survive all of this are reported.

## 1. Install

### 1.1 Dependencies
- Python 3
- `primer3_core` (compiled and executable)
- `seqkit` (only if `--qc` is used)
- Primer3 manual: see
  `https://primer3.org/manual.html`

### 1.2 Python packages

```sh
pip install requests matplotlib
```

## 2. Quick Start

### 2.1 Gene mode

```sh
python qp_primer_design.py \
  --gene TP53 \
  --species homo_sapiens \
  --primer3 /abs/path/to/primer3_core \
  --out /abs/path/output.txt
```

### 2.2 Sequence mode (U→T auto-converted)

```sh
python qp_primer_design.py \
  --sequence ACTG... \
  --primer3 /abs/path/to/primer3_core \
  --out /abs/path/output.txt
```

### 2.3 BLAST-assisted sequence mode

```sh
python qp_primer_design.py \
  --sequence ACTG... \
  --blast \
  --primer3 /abs/path/to/primer3_core \
  --out /abs/path/output.txt
```

### 2.4 Region-restricted design (cDNA 1-based inclusive)

```sh
python qp_primer_design.py \
  --gene TP53 \
  --species homo_sapiens \
  --region 200-400 \
  --primer3 /abs/path/to/primer3_core \
  --out /abs/path/output.txt
```

### 2.5 Last-half (default on)

By default, primers are designed within the last 50% of the transcript. Disable with `--last-half false`.

```sh
python qp_primer_design.py \
  --gene TP53 \
  --species homo_sapiens \
  --last-half false \
  --primer3 /abs/path/to/primer3_core \
  --out /abs/path/output.txt
```

## 3. Modes

- Junction mode (default): If transcript has >= 2 exons, primers are designed on exon–exon junctions.
- Single-exon mode: If transcript has only 1 exon, it switches automatically; junction constraints are ignored.
- Sequence-only mode: Provide `--sequence`; transcript/gene fields are not used.
- Last-half constraint (default on): If `--region` is not provided, design is restricted to the last 50% of the transcript.

## 4. QC (Canonical Unique)

Enable QC with `--qc`. This runs `seqkit amplicon` on a transcriptome FASTA and requires:
- Exactly 1 amplicon
- The amplicon is on the canonical transcript

### 4.1 QC reference
- If `--qc-ref-fasta` is provided, that file is used.
- Otherwise, the script auto-downloads Ensembl `cdna.all.fa.gz` for the given species and caches it in the output directory.

### 4.2 QC output
- Main report: `--out`
- QC report (if `--qc`): `<out>.qc.tsv`

## 5. 3' End Filtering (Hard Filter)

Stage 1 (last 3 bp):
- 3' terminal base must be G/C.
- GC count in last 3 bp >= 1.
- Last 3 bp cannot be GGG or CCC.
- No 3' homopolymer run >= 4.

Stage 2 (last 5 bp):
- GC count in last 5 bp between 1 and 3.
- Max run (any nt) < 4.
- Max run (G/C) < 3.
- No strong 3' self-complementarity (simple check).
- No strong 3' cross-dimer between primers (simple check).

Additional hard filters:
- GC% must be between 45–60% for both primers.

## 6. Common Options

- `--gene <symbol>`: Ensembl gene symbol (used with `--species`).
- `--species <ensembl_species>`: Ensembl species name, e.g. `homo_sapiens`, `zea_mays`.
- `--sequence <seq>`: User-provided sequence (RNA or DNA). RNA will be auto-converted U→T.
- `--blast`: BLAST the sequence against NCBI `nt` and try to map to Ensembl.
- `--primer3 <path>`: Path to `primer3_core`.
- `--region <start-end>`: cDNA region (1-based, inclusive) to design within.
- `--last-half <true|false>`: Force design within last 50% of transcript (default `true`). Ignored if `--region` is set.
- `--allow-single-exon`: Allow single-exon mode (no junction).
- `--qc`: Enable QC with `seqkit amplicon`.
- `--qc-ref-fasta <path>`: Transcriptome FASTA for QC (cdna.all.fa.gz).
- `--ensembl-release <int or current>`: Ensembl release for auto-download (default `current`).
- `--out <path>`: Output report path.
- `--top <int>`: Number of top candidates to report.

## 7. Notes

- Region coordinates use transcript cDNA, 1-based inclusive (e.g., `--region 200-400`).
- `--last-half true` is the default; it sets the region to the last 50% unless `--region` is explicitly provided.
- BLAST uses NCBI `nt` and tries top N hits (default 5) to map to Ensembl.
- If BLAST→Ensembl mapping fails, it falls back to sequence-only mode.
- Ensembl query rule: try `/lookup/symbol/<species>/<gene>` first, then `/lookup/id/<gene>`; transcript selection prefers `canonical_transcript`, otherwise first `is_canonical == 1`.
- The canonical transcript is based on Ensembl database.



## Optimization ideas (work in progress)

High‑GC regions are still tricky. Possible strategy:
1. Run 2,500 pairs without region restriction.
2. Take the first forward primer, round its position to 50, then add 200 bp to define a focused region.
3. Re‑run within that region and return 50 primer sets.
4. (Optional) Cluster primer sites into hotspot regions and loop with k=20–25, w=1 to build a candidate space, then re‑score using `primer3_core` to pick the final set.
