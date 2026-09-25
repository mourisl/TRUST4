---
title: Interpreting and filtering results
description: Which output file to use, what the counts mean in each one, what TRUST4 filters for you, and how to filter the rest.
---

The questions that come up most often on the issue tracker are about reading the results rather than running TRUST4. This page collects the maintainer's answers to them. The exact column layout of each file is on [output formats](/reference/output-formats/).

## Which file to use

| Data | Use | Why |
|------|-----|-----|
| Bulk RNA-seq, TCR-seq or BCR-seq | `_report.tsv` or `_airr.tsv` | One row per CDR3 (clonotype), with read counts. |
| Single-cell data with barcodes | `_barcode_report.tsv` or `_barcode_airr.tsv` | One row per barcode (cell) in the report, or per chain per cell in the AIRR file; the barcode is in `cell_id`. |
| Any data, when you need every contig | `_cdr3.out` | Contig-driven: includes partial CDR3s and minor CDR3s. |

For single-cell data, `_report.tsv` and `_airr.tsv` are still written, but they treat the data as **pseudo-bulk**: each row is a clonotype across cells, and the count is the number of cells carrying it.

## How the files relate

```text
_final.out  → annotator          → _annot.fa, _cdr3.out
_cdr3.out   → trust-barcoderep.pl → _barcode_report.tsv   (with barcodes only)
_cdr3.out   → trust-simplerep.pl  → _report.tsv           (with barcodes, limited to the barcode report's primary chains)
_report.tsv + _annot.fa         → trust-airr.pl → _airr.tsv
_barcode_report.tsv + _annot.fa → trust-airr.pl → _barcode_airr.tsv
```

What happens at each step explains most "why don't the numbers match?" questions:

- **`_annot.fa` → `_cdr3.out`.** Assemblies without a CDR3 are not written to `_cdr3.out`. A consensus contig can encode several similar CDR3s — for example somatic hypermutation variants of one B-cell clone — so one contig can have several rows, numbered by `index_within_consensus`. Contig IDs have gaps because contigs are merged or filtered during assembly.
- **`_cdr3.out` → `_report.tsv`.** The report is CDR3-driven. It drops partial CDR3s and coalesces the entries with the same CDR3, V, J and C genes into one row, keeping the most abundant contig's ID in `cid`. That is why a contig ID from `_annot.fa` may not appear in the report.
- **`_report.tsv` → `_airr.tsv`.** The AIRR file is built from the report plus `_annot.fa`, so it has the same rows in the same order. The contig sequence in the AIRR record has the row's CDR3 substituted in.
- **`_cdr3.out` → `_barcode_report.tsv`.** For each barcode, the most abundant chain pair becomes the representative; other chains go to the secondary columns.

To get the total reads of one contig, including its minor CDR3s, sum column 11 (`read_fragment_count`) of `_cdr3.out` over that contig ID.

## What the counts mean

| File | Count column | Bulk data | With barcodes | With `--UMI` |
|------|--------------|-----------|---------------|--------------|
| `_report.tsv` | `count` | Reads covering the CDR3 | Number of barcodes (cells) with the CDR3 | Number of barcodes |
| `_airr.tsv` | `consensus_count` | Reads covering the CDR3 | Number of barcodes (cells) | Number of barcodes |
| `_barcode_report.tsv` | `read_cnt` in each chain field | — | Reads for that chain in that cell | UMIs for that chain in that cell |
| `_barcode_airr.tsv` | `consensus_count` | — | Reads for that chain in that cell | UMIs for that chain in that cell |
| `_cdr3.out` | `read_fragment_count` | Read fragments assigned to the CDR3 | Same | Same |

Details worth knowing:

- A read counts towards a CDR3 when it maps to the assembly and overlaps the CDR3 region.
- `read_fragment_count` can be a decimal: reads compatible with several CDR3s are split between them by an EM step. `trust-simplerep.pl` truncates the count to an integer unless `--decimalCnt` is given.
- With `--UMI`, the per-cell counts are UMI counts; TRUST4 does not report reads and UMIs side by side. To get read counts as well, run without `--UMI` under another prefix and join on the contig ID.
- The counts are raw. TRUST4 does not normalize for library size, so divide by the total reads of each sample when you compare immune infiltration between samples.
- The `average_coverage` in `_annot.fa` is the total aligned read length divided by 500 — roughly the size of the variable region — not by the contig length, to reduce the bias for short contigs. Short contigs therefore show low coverage.
- The "Found N reads" line in the log counts each end of a pair separately and includes C-gene reads and extraction false positives; it is not a count of receptor molecules.

### Frequency

`frequency` in `_report.tsv` is the fraction of the count within its **chain group**. The groups are IGH, IGK+IGL, TRA, TRB, TRG and TRD, each normalized on its own. The frequency column therefore sums to up to 6, not 1. It is not recomputed when you extract a subset of rows, and does not need to be.

## What TRUST4 filters for you

- **Partial CDR3s** (CDR3 score 0) are left out of `_report.tsv`, `_airr.tsv` and the barcode report by default. Many of them come from V or J genes before recombination. `trust-simplerep.pl --reportPartial` and `trust-barcoderep.pl --reportPartial` put them back.
- **Low-fraction TCR CDR3s within a contig**: `trust-simplerep.pl` drops TCR CDR3s below 5% of the contig's representative CDR3 (`--filterTcrError`, default 0.05). BCR CDR3s are not filtered this way (`--filterBcrError`, default 0), because hypermutation produces real minor variants.
- **Out-of-frame CDR3s as a cell's main chains**: in the barcode report (cell level), an out-of-frame or partial CDR3 is not chosen as a cell's representative chain, because it "might create many false positive reports in single-cell mode". In molecule mode (`--barcodeLevel molecule`) out-of-frame CDR3s are kept.
- **γδ calls in αβ or B cells**: in the barcode report, a γδ T-cell call is dropped when good αβ T or B contigs exist in the same barcode.

Nothing else is filtered. In particular, the barcode report is not filtered by read support and ambient cells are not removed.

## Filtering bulk results

The maintainer's usual practice:

- **Partial CDR3s** — excluded by default; use them with caution if you put them back.
- **Singletons** (count 1) have lower precision. Keep them for diversity analysis; filter them when you care about particular CDR3s.
- **Out-of-frame CDR3s and stop codons** (`out_of_frame`, `_` in `CDR3aa`) are real rearrangements, just not productive. Keep them for nucleotide-level clonotypes; remove them for protein-level analysis.
- **Very high counts** usually reflect clonal expansion or, for BCR, plasma cells, which express far more immunoglobulin than other cells. Do not filter them; compare frequencies instead.

Low TCR/BCR read counts, many more BCR than TCR reads, and many more light-chain than heavy-chain reads are all normal for bulk tumour RNA-seq.

## Filtering single-cell results

### Keep only real cells

TRUST4 assembles contigs from every barcode it sees, including empty droplets and low-quality cells. Ambient BCR mRNA from plasma cells in particular can get into empty droplets and create many BCR calls in "null" cells; this is much less obvious for TCR. So:

1. **Intersect with the barcodes that passed gene-expression QC** (for example in Seurat or Scanpy). This is the most important step, and it applies to TCR/BCR-kit data too. For FASTQ input, remember that barcodes have no `-1` suffix, unlike Cell Ranger's `CB` tag.
2. **Filter on read or UMI support.** "The read support probably is the best way to QC." The right cutoff depends on the data: keep single-read calls for unamplified GEX data; for an amplified BCR library, a cutoff of 4 reads was called "good (or even a bit conservative)". To regenerate the pseudo-bulk report with such a cutoff:

```bash
perl trust-simplerep.pl TRUST_sample_cdr3.out --barcodeCnt \
  --filterBarcoderep TRUST_sample_barcode_report.tsv --filterBarcoderepReadCnt 4 \
  > TRUST_sample_report_filtered.tsv
```

3. **Remove contamination from diffused mRNA** with [`barcoderep-filter.py`](/guides/utility-scripts/#barcoderep-filter-py). It is designed for plasma-cell BCR mRNA leaking into other droplets of a GEX library, and is usually not needed for TCR. It writes a new barcode report; rerun `trust-airr.pl` on it to get a filtered AIRR file.

TRUST4 does not detect doublets. Extra chains in a barcode go to the secondary columns; use a doublet detector on the expression data if you need one. One light-chain CDR3 shared by many cells is a sign of ambient RNA.

### Do not filter on these

- **CDR3 score.** It measures how many conserved motif residues flank the CDR3, and some V and J genes do not follow the motif. A full CDR3 is already required for the reports.
- **`complete_vdj` / `cid_full_length`.** The CDR3 is always complete in the report and AIRR files, so there is no need to filter on full length for clonotype analysis; the flag matters only when you need the full V and J sequence, for example to study hypermutation. Filtering on it is too aggressive for diversity analysis.
- **`productive` alone, when comparing with Cell Ranger.** TRUST4's `productive` is based solely on the CDR3 sequence (no stop codon, in frame). Cell Ranger checks the whole sequence, so the two definitions differ.

Germline similarity should also be used with care. A low value is suspicious for TCR but normal for BCR because of hypermutation, the alignment within the CDR3 is not very reliable, and a value of 0 can happen.

## Gene calls

- Genes are ranked by alignment similarity, the last number of each gene field in `_annot.fa`. Up to three candidates are listed.
- On a tie, TRUST4 prefers the gene used more often in the sample.
- D-gene calls are unreliable because D genes are short.
- A contig that covers only a short part of V, or a heavily hypermutated V, can be assigned the wrong V gene. Use the CDR3 as the anchor rather than the V call in that case.
- Differences from IgBLAST calls come from the different alignment algorithms. Re-annotating with IgBLAST is not needed unless a downstream tool requires IgBLAST's format.
- A TRDV gene paired with a TRAJ gene is not an error: some V genes are shared between TRA and TRD.
- Contigs with only IGKC or another C gene and no V/J are germline transcripts without recombination.

## What to expect from your data type

TRUST4 needs reads that cover the V(D)J region at the 5′ end of the receptor transcript. How many cells or clonotypes it finds depends mostly on that coverage.

| Data | What to expect |
|------|----------------|
| Bulk RNA-seq | The primary use case. About 50M 150 bp paired-end reads gives a decent number of CDR3s. |
| 10x 5′ gene expression | Works well without the VDJ kit; roughly half the sensitivity of the TCR/BCR kit, and it can find γδ T cells the kit misses. Only part of the T and B cells get a call; that is normal. |
| 10x 3′ gene expression, BD Rhapsody WTA, other 3′-biased data | Low sensitivity, "around 5%" of cells for 10x 3′, because most reads fall in the long C gene. Calls that are found are reliable. Use the CDR3 and isotype rather than full V(D)J. |
| Visium and other multi-cell spots | TRUST4 picks one representative pair and one cell type per barcode. Look at the secondary chains, or at `_cdr3.out`, for the full picture. |
| 10x VDJ kit, other amplified libraries | Works, but TRUST4 is designed for unamplified data and is slower than Cell Ranger here. Expect many more barcodes than Cell Ranger reports; filter as above. |
| Bulk TCR-seq/BCR-seq | Use `--repseq` for non-UMI data; see [UMIs](/guides/umi/) for UMI-based kits. |
| WES, WGS, other DNA | Works as it is, but unrecombined V or J segments produce false contigs, such as CDR3s with only V or only J called. |
| scATAC-seq | Only chance hits. |
| Long reads (PacBio HiFi, corrected Nanopore) | Works but not optimized; indel errors are not modeled. See [annotating long reads](/guides/annotation-only/#long-reads). |

On low-coverage data, γδ chains can become the primary call in αβ T cells whose α/β chains were missed; check TRDC/TRGC expression before interpreting γδ calls.
