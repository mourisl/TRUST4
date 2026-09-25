---
title: Utility scripts
description: Scripts for preparing barcode input, computing diversity statistics, clustering CDR3s, filtering and converting barcode reports, adding IMGT gaps, and epitope annotation.
---

The `scripts/` folder of the repository contains utility scripts that work alongside the main pipeline — some prepare input for `run-trust4`, most process its results. Detailed information for each Python script is printed by `python3 the_script.py -h`, and for each Perl script by `perl the_script.pl` with no arguments.

| Script | Stage | Purpose |
|--------|-------|---------|
| `combinatorial-barcode-whitelistgen.py` | Before a run | Generate whitelists or translation tables for combinatorial barcoding. |
| `trust-stats.py` | After a run | Compute the clonotype diversity in a sample. |
| `trust-cluster.py` | After a run | Cluster similar CDR3s. |
| `barcoderep-filter.py` | After a run | Filter clonotypes likely to come from diffused mRNA. |
| `barcoderep-expand.py` | After a run | Give secondary chains their own barcode lines. |
| `trust-barcoderep-to-10X.pl` | After a run | Convert the barcode report to 10x Cell Ranger VDJ format. |
| `airr-imgtgap.py` | After a run | Add IMGT gaps to the AIRR alignment fields. |
| `GetFullLengthAssembly.pl` | After a run | Keep the assemblies in an `_annot.fa` file that have V, J and C genes annotated. |
| `AddSequenceToCDR3File.pl` | After a run | Add assembly sequences from `_annot.fa` to a `_cdr3.out` file. |

The scripts that build reference files, `BuildImgtAnnot.pl` and `BuildDatabaseFa.pl`, are in the repository root and described in [reference gene files](/guides/reference-files/).

## Preparing input

### combinatorial-barcode-whitelistgen.py

Generates the whitelist or barcode translation table for combinatorial barcode platforms, such as Split-seq and Parse Biosciences. It enumerates combinatorial barcodes from the input barcode files, one per barcoding round.

| Option | Description |
|--------|-------------|
| `barcode_files` | Input barcode files. |
| `--input-translate` | Input files are for barcode translation, with the barcode in the second column. |
| `--input-translate-delimiter` | Delimiter for input translator files (default: tab, space). |
| `--translate-output FILE` | Output file for barcode translation (default: no translation). |
| `--translate-delimiter` | Delimiter for the translation output file. |

The output is used with `run-trust4 --barcodeWhitelist` or `--barcodeTranslate`; see [barcode translation](/guides/single-cell/#barcode-translation-and-combinatorial-barcoding).

## Repertoire statistics

### trust-stats.py

Computes the clonotype diversity in a sample.

```bash
python3 scripts/trust-stats.py -r TRUST_sample_report.tsv
```

| Option | Description |
|--------|-------------|
| `-r REPFILE` | Repertoire file. Required. |
| `-f FORMAT` | Repertoire file format (`TRUST4_report`, `TRUST4_barcode_report`, ...). Default: `TRUST4_report`. |
| `--ntaa NTAA` | Use nucleotide (`nt`) or amino acids (`aa`). In the current version this option is parsed but not used: clonotypes are always defined by the CDR3 amino-acid sequence. |

The output has one row for IGH overall, one per IGH isotype, and one for each of IGK, IGL, TRA, TRB, TRG and TRD:

| Column | Meaning |
|--------|---------|
| `Abundance` | Total count of the chain's CDR3s: reads for a report, cells for a barcode report. |
| `Richness` | Number of distinct CDR3s (clonotypes). |
| `CPK` | Clonotypes per kilo-reads: `Richness / Abundance × 1000`. Higher clonality means lower CPK. |
| `Entropy` | Shannon entropy of the clonotype frequencies. |
| `Clonality` | `1 − Entropy / log(Richness)`. Already normalized for depth. |

Out-of-frame, partial and stop-codon CDR3s are left out. The maintainer's recommendations:

- Analyse each chain separately; "I would recommend using TRB's CPK and clonality" for T cells.
- Only compute diversity for samples with at least about 10 distinct clonotypes.
- Batch effects between bulk samples are weak; use CPK or clonality, which account for depth, rather than raw richness.
- The script is meant for bulk reports. For single-cell data, give it the barcode report with `-f TRUST4_barcode_report`.

:::caution Human isotypes only
The isotype lists in `trust-stats.py` (`isotypeRanks` and `isotypeOrder`) are hard-coded for human IGH constant genes. On mouse or other species, an unknown isotype such as `IGHA` or `IGHG2B` stops the script with a `KeyError`. Edit the two lists for your species before running it.
:::

### trust-cluster.py

Clusters similar CDR3s based on the `_cdr3.out` or `_report.tsv` file. Output goes to standard output.

```bash
python3 scripts/trust-cluster.py TRUST_sample_cdr3.out > TRUST_sample_cluster.out
```

| Option | Default | Description |
|--------|---------|-------------|
| `-s FLOAT` | `0.8` | Similarity of two CDR3s. |
| `--prefix STRING` | `cluster` | Prefix to the new cluster name. |
| `--center` | no | Use the center of the cluster for similarity comparison. |
| `--representative` | no | Use the representative CDR3 from each contig for clustering. |
| `--format [cdr3, simplerep]` | `cdr3` | The input format type. Use `simplerep` for a `_report.tsv` file. |

From `_cdr3.out`, partial CDR3s and contigs without both a V and a J gene are skipped. The default mode is aggressive (single-linkage); `-s 0.9 --center` gives clusters whose members are at least about 80% similar to each other. To cluster across samples, concatenate their files first.

## Single-cell barcode reports

### barcoderep-filter.py

Filters the lowly expressed clonotype if it is identical to another highly expressed clonotype in another cell. This strategy is inspired by the [10x VDJ pipeline](https://support.10xgenomics.com/single-cell-vdj/software/pipelines/latest/algorithms/cell-calling) to remove contaminations from diffused mRNAs. Output goes to standard output.

```bash
python3 scripts/barcoderep-filter.py -b TRUST_sample_barcode_report.tsv > TRUST_sample_barcode_report_filtered.tsv
```

| Option | Description |
|--------|-------------|
| `-b BARCODE_REPORT` | The barcode report file. Required. |
| `-a ANNOT` | The annotation file. |
| `--highAbund HIGHABUND` | The minimum abundance to be regarded as a potential source of diffusion. |
| `--diffuseFrac DIFFUSEFRAC` | The maximum fraction of the diffusion source abundance to be regarded as noise. Default: `0.02`. |

The `--highAbund` default is `50`. Roughly: a cell A is removed when its chains have the same CDR3s as another cell B, B's chain has an abundance of at least `--highAbund`, and A's abundance is less than `--diffuseFrac` × B's. Both of A's chains must fit this; a chain missing from A counts as fitting. With `-a TRUST_sample_annot.fa`, the comparison uses the contig sequences instead of only the CDR3s: A's contig must be contained in B's. If contamination remains, lower `--highAbund` or raise `--diffuseFrac`.

- It needs the barcode report, not `_report.tsv`.
- It is designed for plasma-cell BCR mRNA diffusing into other droplets in gene-expression libraries, and is usually not needed for TCRs.
- It writes a new barcode report. To get a filtered AIRR file, run `trust-airr.pl` on it: `perl trust-airr.pl filtered_barcode_report.tsv TRUST_sample_annot.fa --format barcoderep > filtered_barcode_airr.tsv`.
- Keep only the barcodes that pass gene-expression QC as well; see [filtering single-cell results](/guides/interpreting-results/#filtering-single-cell-results).

### barcoderep-expand.py

Puts the secondary chains in the barcode report in their own barcode lines, renaming the barcode. This is useful if a barcode contains multiple cells. `trust-airr.pl` can then be used to create AIRR entries for the secondary chains.

| Option | Description |
|--------|-------------|
| `-b BARCODE_REPORT` | The barcode report file. Required. |
| `--chain CHAIN` | Expand chain1 or chain2. |
| `--frac FRAC` | The abundance needs to be more than this fraction of the primary chain. |

### trust-barcoderep-to-10X.pl

Converts the barcode report to the 10x Cell Ranger VDJ format (the format used at least in Cell Ranger 3).

```bash
perl scripts/trust-barcoderep-to-10X.pl TRUST_sample_barcode_report.tsv 10X_report_prefix
```

## AIRR output

### airr-imgtgap.py

Adds the gaps defined in the IMGT file to the `sequence_alignment` and `germline_alignment` fields in the AIRR output. Output goes to standard output.

```bash
python3 scripts/airr-imgtgap.py -i human_IMGT+C.fa -a TRUST_sample_airr.tsv > TRUST_sample_airr_imgtgap.tsv
```

| Option | Description |
|--------|-------------|
| `-i IMGTFILE` | IMGT file. |
| `-a AIRRFILE` | AIRR file. |

## Assemblies and CDR3 files

### GetFullLengthAssembly.pl

Reads an `_annot.fa` file and writes out only the assemblies whose V, J and C genes are all annotated.

:::note Superseded
This script is no longer maintained. It still requires a C gene, unlike the current [complete VDJ flag](/reference/output-formats/#complete-vdj-assemblies) (`cid_full_length`, `complete_vdj`), which is the better way to select full-length assemblies.
:::

```bash
perl scripts/GetFullLengthAssembly.pl TRUST_sample_annot.fa > full_length_annot.fa
```

### AddSequenceToCDR3File.pl

Appends a column to each row of a `_cdr3.out` file holding the assembly sequence from `_annot.fa`, with the CDR3 region replaced by that row's CDR3 sequence.

```bash
perl scripts/AddSequenceToCDR3File.pl TRUST_sample_cdr3.out TRUST_sample_annot.fa > TRUST_sample_with_seq_cdr3.out
```

## Downstream tools

| Tool | Input from TRUST4 | Notes |
|------|-------------------|-------|
| VDJtools | `_report.tsv` | Remove `out_of_frame` rows first; see [output formats](/reference/output-formats/#trust-report-tsv). |
| immunarch | `_report.tsv` | Bulk data only. |
| scRepertoire, Platypus | `_barcode_report.tsv` or `_barcode_airr.tsv` | Single-cell data. |
| scirpy, Dandelion | `_barcode_airr.tsv` | Dandelion needs `umi_count`; copy it from `consensus_count`. |
| Immcantation (Change-O, IgPhyML) | `_airr.tsv`, `_barcode_airr.tsv` | Re-annotating with IgBLAST is only needed if a tool requires IgBLAST's format. `AddSequenceToCDR3File.pl` was written for this workflow. |
| Cell Ranger-style tools | `_barcode_report.tsv` | Convert with `trust-barcoderep-to-10X.pl`, which writes `filtered_contig_annotations`-style `_t.csv` and `_b.csv` files. |

Prefer the AIRR files when a tool accepts them.

## Epitope annotation

TRUST4's report file and AIRR output are compatible with epitope prediction methods, such as [TCRMatch](https://github.com/IEDB/TCRMatch). You can use commands like:

```bash
./tcrmatch -i trust_report.tsv -d CEDAR_data.tsv -t 8 -r > trust_with_epitope.txt
```

or, with the AIRR file:

```bash
./tcrmatch -i trust_airr.tsv -d CEDAR_data.tsv -t 8 -a > trust_with_epitope.txt
```

## Reporting scripts used by the pipeline

`trust-simplerep.pl`, `trust-barcoderep.pl` and `trust-airr.pl` in the repository root are the scripts `run-trust4` itself calls to produce the reports. They can be rerun by hand, for instance on a subset of `_cdr3.out`; their options are on the [command-line interface](/reference/cli/#reporting-scripts) page.
