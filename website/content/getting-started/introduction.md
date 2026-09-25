---
title: Introduction
description: What TRUST4 does, what goes in, what comes out, and where to go next.
---

TRUST4 is a computational tool to analyze T-cell receptor (TCR) and B-cell receptor (BCR) sequences using unselected RNA sequencing data, profiled from fluid and solid tissues, including tumors. It supports both single-end and paired-end, bulk or single-cell sequencing data with any read length.

## How it works

TRUST4 runs as a short pipeline driven by the `run-trust4` script:

1. **Candidate read extraction.** Reads likely to come from V, J or C genes are pulled out of the input — from a BAM alignment file with `bam-extractor`, or from FASTQ/FASTA files with `fastq-extractor`.
2. **De novo assembly.** The `trust4` program assembles the candidate reads into consensus contigs covering the V, J, C genes including the hypervariable complementarity-determining region 3 (CDR3).
3. **Annotation.** The `annotator` realigns the contigs to IMGT reference gene sequences to identify the V, D, J, C genes and the CDR1, CDR2 and CDR3 of each contig.
4. **Reporting.** Perl scripts summarise the annotation into a CDR3-focused report table and an [AIRR-format](https://docs.airr-community.org/en/latest/datarep/rearrangements.html) file, plus a per-cell report when cell barcodes are given.

Each step can be resumed individually with `--stage`; see [pipeline stages and files](/reference/pipeline/).

## Inputs

TRUST4 needs sequencing reads and two reference files:

| Input | Option | What it is |
|-------|--------|------------|
| Reads | `-b` | Alignment of RNA-seq reads in BAM format. |
| Reads | `-1`/`-2` or `-u` | Alternatively, raw paired-end or single-end reads in FASTA/FASTQ format. |
| Gene sequences | `-f` | The genomic sequence and coordinate of V, J, C genes. |
| Gene annotation | `--ref` | The reference database sequence containing annotation information, such as IMGT. Optional, but recommended. |

The repository ships these reference files for human (`hg38_bcrtcr.fa`, `hg19_bcrtcr.fa`, `human_IMGT+C.fa`) and mouse (`mouse/GRCm38_bcrtcr.fa`, `mouse/GRCm39_bcrtcr.fa`, `mouse/mouse_IMGT+C.fa`). For other species, or to rebuild them, see [reference gene files](/guides/reference-files/).

## Outputs

The main results, each named with the output prefix (`TRUST_<input name>` by default):

- `_report.tsv` — a report focusing on CDR3, compatible with other repertoire analysis tools such as VDJTools.
- `_airr.tsv` — the same results in the AIRR rearrangement format.
- `_cdr3.out` — the CDR1, 2, 3 and gene information for each consensus assembly.
- `_annot.fa` — the annotated consensus assemblies in FASTA format.
- `_barcode_report.tsv` and `_barcode_airr.tsv` — per-barcode (per-cell) results, when barcodes are given.

All columns are described in [output formats](/reference/output-formats/).

## Choosing a guide

| Your data | Start here |
|-----------|------------|
| Bulk RNA-seq, aligned (BAM) or raw (FASTQ) | [Bulk RNA-seq](/guides/bulk-rna-seq/) |
| 10x Genomics or other barcode-based single-cell data | [10x Genomics and single-cell data](/guides/single-cell/) |
| Data with UMIs | [UMIs and molecule barcodes](/guides/umi/) |
| SMART-seq or other one-file-per-cell platforms | [SMART-seq data](/guides/smart-seq/) |
| Bulk, non-UMI-based TCR-seq or BCR-seq | [Bulk RNA-seq](/guides/bulk-rna-seq/#targeted-tcr-seq-and-bcr-seq) |
| Sequences you already have, to be annotated | [Annotating sequences](/guides/annotation-only/) |
| A species other than human or mouse | [Reference gene files](/guides/reference-files/) |
| UMI-based TCR-seq or BCR-seq | [UMIs and molecule barcodes](/guides/umi/#umis-as-molecule-barcodes) |
| Long reads (PacBio, Nanopore) | [Annotating sequences](/guides/annotation-only/#long-reads) |

TRUST4 needs reads that cover the V(D)J region at the 5′ end of the receptor transcript. It works best on bulk RNA-seq and 5′ single-cell data; 3′ single-cell data gives far fewer results. See [what to expect from your data type](/guides/interpreting-results/#what-to-expect-from-your-data-type).

## Next steps

- [Installation](/getting-started/installation/) — install from Bioconda or build from source.
- [Quick start](/getting-started/quick-start/) — run the bundled example and check your installation.
