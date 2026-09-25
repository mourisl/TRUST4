---
title: Output formats
description: Column-by-column description of the report, CDR3, annotation, barcode and AIRR files that TRUST4 writes.
---

TRUST4 writes all its files with the output prefix, which defaults to `TRUST_` followed by the input file name up to its first `.`. Below, `trust` stands for that prefix — `trust_report.tsv` is `TRUST_example_report.tsv` for the bundled example.

## Overview

| File | Contents |
|------|----------|
| `trust_report.tsv` | Report focusing on CDR3, compatible with other repertoire analysis tools such as VDJTools. |
| `trust_airr.tsv` | The results in [AIRR format](https://docs.airr-community.org/en/latest/datarep/rearrangements.html). |
| `trust_cdr3.out` | CDR1, 2, 3 and gene information for each consensus assembly. |
| `trust_annot.fa` | Annotation of the consensus assemblies, in FASTA format. |
| `trust_raw.out`, `trust_final.out` | The contigs and corresponding nucleotide weight. |
| `trust_barcode_report.tsv` | Representative chains per barcode (cell). Only with `--barcode`. |
| `trust_barcode_airr.tsv` | The barcode report in AIRR format. Only with `--barcode`. |
| `trust_assign.out` | Read assignment. Only with `--outputReadAssignment`. |

Intermediate files are listed on [pipeline stages and files](/reference/pipeline/). How the files relate to each other, what the counts mean in each one, and which file to use for which analysis are explained in [interpreting and filtering results](/guides/interpreting-results/).

## trust_report.tsv

A tab-separated file with one row per CDR3:

```text
read_count  frequency(proportion of read_count)  CDR3_dna  CDR3_amino_acids  V  D  J  C  consensus_id  consensus_id_complete_vdj
```

The file's header line is `#count frequency CDR3nt CDR3aa V D J C cid cid_full_length`.

| Column | Meaning |
|--------|---------|
| `count` | Read count of reads covering the CDR3; raw, not normalized. With barcodes, the number of barcodes (cells) with the CDR3 instead. |
| `frequency` | Proportion of `count` within the chain group. The groups IGH, IGK+IGL, TRA, TRB, TRG and TRD are each normalized separately, so the column sums to up to 6. |
| `CDR3nt` | CDR3 nucleotide sequence. |
| `CDR3aa` | CDR3 amino acid sequence. `_` represents a stop codon, and `?` represents the ambiguous nucleotide `N` in a codon. `out_of_frame` when the CDR3 nucleotide length is not a multiple of three, and `partial` for a partial CDR3 (included only with `trust-simplerep.pl --reportPartial`). |
| `V`, `D`, `J`, `C` | Gene assignments. |
| `cid` | Consensus ID. Rows with the same CDR3, V, J and C are merged, and `cid` is the most abundant contig among them. |
| `cid_full_length` | `1` if that contig is a [complete VDJ assembly](#complete-vdj-assemblies), otherwise `0`. |

Example rows from the bundled example:

```text
8   8.247423e-02  TGTGCGAGAGGGCAGGACGG...  out_of_frame  IGHV1-3*01   IGHD1-26*01  IGHJ6*03  .     assemble1   0
5   5.154639e-02  TGTGCGAGAGATGGTACCCC...  out_of_frame  IGHV4-59*01  IGHD2-2*01   IGHJ4*02  IGHM  assemble22  0
```

Partial CDR3s are excluded from this file by default. Only one row is written per CDR3, V, J and C combination, so the same CDR3 with a different J gene is a separate row.

:::note Using the report with VDJtools
Tools that parse `CDR3aa` as an amino-acid sequence can fail on the `out_of_frame` value (VDJtools reports `Unknown symbol "o"`). Remove those rows first, e.g. `grep -v out_of_frame trust_report.tsv`, or also drop CDR3s with stop codons and ambiguous residues: `awk -F'\t' 'index($4,"_")==0 && index($4,"?")==0' trust_report.tsv` (this also drops the `out_of_frame` rows).
:::

## trust_cdr3.out

A tab-separated file with one row per CDR3 per consensus assembly. Only assemblies that contain a CDR3 are written, including partial CDR3s. A consensus can encode several similar CDR3s, such as hypermutation variants, which are numbered by `index_within_consensus`. The fields are:

| # | Field | Meaning |
|---|-------|---------|
| 1 | `consensus_id` | ID of the consensus assembly. |
| 2 | `index_within_consensus` | Index of this CDR3 within the consensus. |
| 3–6 | `V_gene`, `D_gene`, `J_gene`, `C_gene` | Gene assignments; `*` when absent. |
| 7–9 | `CDR1`, `CDR2`, `CDR3` | CDR nucleotide sequences. |
| 10 | `CDR3_score` | CDR3 score divided by 100; see [CDR3 score](#cdr3-score). |
| 11 | `read_fragment_count` | Number of read fragments assigned to this CDR3. Can be a decimal: reads compatible with several CDR3s are split between them. |
| 12 | `CDR3_germline_similarity` | Similarity of the CDR3 to the germline V and J genes that overlap it: the percentage of matches in their alignment. Low values are normal for BCRs; the alignment within the CDR3 is not very reliable, and `0` can happen. |
| 13 | `complete_vdj_assembly` | `1` if the consensus is a [complete VDJ assembly](#complete-vdj-assemblies), otherwise `0`. |

Please note that `CDR3_score` in `trust_cdr3.out` has been divided by 100, so 1.00 is the maximum score and 0.01 means an imputed CDR3.

### CDR3 score

The same score appears in `trust_annot.fa` (0–100) and `trust_cdr3.out` (0–1):

| `_annot.fa` | `_cdr3.out` | Meaning |
|-------------|-------------|---------|
| `0.00` | `0.00` | Partial CDR3. |
| `1.00` | `0.01` | CDR3 with imputed nucleotides. |
| `50.00` | `0.50` | Rescued CDR3: found with only a very short anchor on the V and J genes. |
| other values | other values | Motif strength: multiples of 100/6 for the conserved residues found around the CDR3 — `YYC` at the 5′ end and `F/WGxG` at the 3′ end. `100.00` means all six. |

Some V and J genes do not follow the conserved motif, so a lower score does not mean a CDR3 is wrong. Do not filter on it.

## trust_annot.fa

A FASTA file of the consensus assemblies. Each header is split into fields:

```text
consensus_id consensus_length average_coverage annotations
```

`annotations` itself has several fields, corresponding to the annotation of V, D, J, C, CDR1, CDR2 and CDR3 respectively.

### Gene annotations

```text
gene_name(reference_gene_length):(consensus_start-consensus_end):(reference_start-reference_length):similarity
```

Each type of gene has at most three gene candidates, ranked by their similarity.

### CDR annotations

```text
CDRx(consensus_start-consensus_end):score=sequence
```

- For CDR1 and CDR2, the score is the similarity.
- For CDR3, score `0.00` means a partial CDR3, score `1.00` means a CDR3 with imputed nucleotides, and other numbers mean the motif signal strength, with `100.00` as the strongest.

The coordinates are 0-based and closed at both ends: `(0-284)` covers 285 bases. Because the D gene is located within the CDR3, its coordinates can overlap the V or J gene by a base.

`average_coverage` is the total length of the reads aligned to the contig divided by 500 — roughly the size of the variable region — rather than by the contig length, to reduce the overestimation for short contigs. The number of reads is therefore roughly `average_coverage × 500 / read_length` (halve it for read pairs).

### Example

```text
>assemble1 389 5.52 IGHV1-3*01(296):(0-277):(17-294):100.00 IGHD1-26*01(20):(308-320):(8-19):96.00 IGHJ6*03(62):(314-371):(4-61):100.00 * CDR1(58-81):100.00=GGATACACCTTCACTAGCTATGCT CDR2(133-156):100.00=ATCAACGCTGGCAATGGTAACACA CDR3(268-341):100.00=TGTGCGAGAGGG...
```

Here `assemble1` is 389 bp long with average coverage 5.52; it has V, D and J genes, no C gene (`*`), and a CDR3 with motif strength 100.00 at positions 268–341.

## trust_barcode_report.tsv

Written when barcodes are given. TRUST4 picks the most abundant pair of chains as the representative for each barcode (cell). The format is:

```text
barcode  cell_type  IGH/TRB/TRD_information  IGK/IGL/TRA/TRG_information  secondary_chain1_information  secondary_chain2_information
```

The chain information is in CSV format:

```text
V_gene,D_gene,J_gene,C_gene,cdr3_nt,cdr3_aa,read_cnt,consensus_id,CDR3_germline_similarity,consensus_complete_vdj
```

The header line names the columns `#barcode cell_type chain1 chain2 secondary_chain1 secondary_chain2`. `cell_type` is `B`, `abT` or `gdT`. Missing chains are written as `*`; an empty chain2 means only one chain was detected. The heavy (or β/δ) chain is always in chain1 and the light (or α/γ) chain in chain2.

`read_cnt` is the number of reads for that chain in that cell, or UMIs when `--UMI` is given. The secondary columns hold the other chains found in the barcode, separated by `;`.

With `--barcodeLevel molecule`, the represented chain information is in the chain1 column.

## trust_airr.tsv and trust_barcode_airr.tsv

These follow [the AIRR rearrangement format](https://docs.airr-community.org/en/latest/datarep/rearrangements.html). `trust_barcode_airr.tsv` is the barcode report converted to AIRR. The columns written by TRUST4 v1.1.11 on the bundled example are:

```text
sequence_id  sequence  rev_comp  productive  locus  v_call  d_call  j_call  c_call
sequence_alignment  germline_alignment  cdr1  cdr2  junction  junction_aa
v_cigar  d_cigar  j_cigar  c_cigar  v_identity  j_identity  cell_id  complete_vdj  consensus_count
```

Points specific to TRUST4's AIRR output:

- `trust_airr.tsv` is built from `trust_report.tsv` and has the same rows in the same order. Each row is one clonotype, and the sequence columns are filled from one representative contig.
- `sequence_id` is the contig ID plus the index of the CDR3 within the contig (`assemble1_0`). The CDR3 of that row is substituted into the contig sequence.
- `consensus_count` is the read count for bulk data. In `trust_airr.tsv` for single-cell data it is the number of cells; in `trust_barcode_airr.tsv` it is the reads, or UMIs with `--UMI`, of that chain in that cell. Tools that expect `umi_count`, such as Dandelion, need it copied from `consensus_count`.
- There is no `cdr3` column. `junction` is the CDR3 as TRUST4 reports it, including the conserved flanking codons, and `junction_aa` equals `CDR3aa` in the report.
- `productive` is based solely on the CDR3 sequence: in frame and without a stop codon. Cell Ranger's definition checks the whole sequence, so the two can differ.
- `complete_vdj` is the [complete VDJ assembly](#complete-vdj-assemblies) flag.
- For the full amino-acid sequence of the variable region, translate `sequence_alignment`, not `sequence`. Only about the first 200 bp of the C gene are assembled.
- For somatic hypermutation, use `v_identity`, or the mismatches between `sequence_alignment` and `germline_alignment`, on complete VDJ assemblies. The CDR3 is copied identically into both alignments.

To add IMGT gaps to `sequence_alignment` and `germline_alignment`, use [`airr-imgtgap.py`](/guides/utility-scripts/#airr-imgtgap-py). Runs made with versions from before mid-2022 have empty alignment columns; rerun from `--stage 2` to fill them.

## trust_raw.out and trust_final.out

The contigs from the assembly and the corresponding nucleotide weight. `trust_raw.out` has the contigs; `trust_final.out` has them after scaffolding with mate-pair information, and is the input to the annotation stage. For barcoded data, and with `--skipMateExtension`, scaffolding is skipped and the two files are the same. Each contig record is followed by four lines giving the A, C, G and T read counts at each position. These are intermediate files, removed by `--clean 1`.

## Complete VDJ assemblies

`cid_full_length` in the report, `complete_vdj_assembly` in `_cdr3.out`, `consensus_complete_vdj` in the barcode report and `complete_vdj` in the AIRR files all use the same check. A contig is a complete VDJ assembly when:

- it has a V gene, a J gene and a CDR3;
- the V gene alignment starts at the first base of the V gene and reaches the CDR3;
- the J gene alignment starts within the CDR3 and runs to the last base of the J gene;
- the V gene ends no later than 3 bases into the J gene; and
- there is no `N` between the start of V and the end of J.

A C gene is not required (it was before v1.1.2). The CDR3 is always complete in the report and AIRR files, so the flag matters only when you need the full V and J sequence, for example for hypermutation analysis. `--repseq` trims read ends and gives fewer complete assemblies. The script `scripts/GetFullLengthAssembly.pl` predates this flag, still requires a C gene, and is no longer maintained.
