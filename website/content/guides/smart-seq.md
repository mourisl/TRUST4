---
title: SMART-seq data
description: Process plate-based single-cell data, one file or file pair per cell, with the trust-smartseq.pl wrapper.
---

Platforms like SMART-seq produce a separate file, or pair of files, for each cell rather than one file with cell barcodes. TRUST4 provides a wrapper, `trust-smartseq.pl`, that runs TRUST4 on every cell and collects the results into one report.

## Prepare the file lists

Give the path to each file in a text file, one path per line and nothing else. For paired-end data, list the read 1 files in one text file and the read 2 files in another, in the same order. For example, `read1_list.txt`:

```text
/data/plate1/cellA.R1.fastq.gz
/data/plate1/cellB.R1.fastq.gz
```

and `read2_list.txt`:

```text
/data/plate1/cellA.R2.fastq.gz
/data/plate1/cellB.R2.fastq.gz
```

Each cell's name is inferred by the file name before the first `.` — `cellA` and `cellB` above — so make sure that part of the name is unique per cell.

## Run the wrapper

```bash
perl trust-smartseq.pl -1 read1_list.txt -2 read2_list.txt -t 8 \
  -f hg38_bcrtcr.fa --ref human_IMGT+C.fa -o TRUST
```

For single-end data, give only `-1`.

The script creates:

- `TRUST_report.tsv` for the general summary,
- `TRUST_annot.fa` for the assemblies, and
- `TRUST_airr.tsv` with the AIRR records, whose `cell_id` is the cell name.

Their formats are described on [output formats](/reference/output-formats/). Consensus IDs are prefixed with the cell name, e.g. `cellA_assemble0`, so the combined files stay unambiguous.

## How the wrapper works

For each cell in turn, `trust-smartseq.pl` calls `run-trust4` with the prefix `tmp_smartseq` in the current directory — adding `--skipMateExtension` for paired-end cells — then appends that cell's representative chains to the combined files and deletes `tmp_smartseq_*`.

Two practical consequences:

- Run each invocation from its **own working directory**. Two copies running in the same directory would overwrite each other's `tmp_smartseq_*` files.
- A cell with no assemblies is skipped with `WARNING: no assemblies from <cell>.` on standard error.

`--representative INT` controls how many representatives are kept for each detected chain of a cell. The default, `1`, keeps the most abundant chain and the most abundant partner chain (for example, one IGH and one IGK/IGL).

### Interpreting the results

- The representatives are chosen by abundance. They are not necessarily `assemble0` and `assemble1`; contig numbers carry no ranking.
- To look for dual TCRs, such as cells with two TRA chains, use `--representative 2` and then apply a ratio cutoff between the first and second chain of each type.
- The wrapper does not write a combined `_cdr3.out` by design. Use `TRUST_airr.tsv` with single-cell tools such as scRepertoire or scirpy.
- For SMART-seq, pick the most abundant clonotype per chain in each cell rather than filtering out low-count chains.

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `-1 STRING` | — | File containing the list of read 1 (or single-end) files. |
| `-2 STRING` | — | File containing the list of read 2 files. |
| `-f STRING` | — | Path to the FASTA file with the coordinate and sequence of V/D/J/C genes. |
| `--ref STRING` | not used but recommended | Path to the detailed V/D/J/C gene reference file from the IMGT database. |
| `-o STRING` | `TRUST` | Prefix of the final output files. |
| `-t INT` | `1` | Number of threads. |
| `--representative INT` | `1` | Number of representatives for each detected chain. |
| `--cgeneEnd INT` | `200` | Skip reads aligned after the first 200 bp of the C gene. |
| `--trust-path STRING` | same as this script | Directory of the TRUST4 executable files. |

:::caution --cgeneEnd is listed but not accepted
In the current version, `trust-smartseq.pl` rejects `--cgeneEnd` with `Unknown parameter`, although its usage text lists it. Leave it out; the default of 200 applies.
:::
