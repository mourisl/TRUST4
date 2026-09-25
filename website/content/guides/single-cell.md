---
title: 10x Genomics and single-cell data
description: Assemble per cell with --barcode, extract barcodes with --readFormat, and use whitelists, barcode translation and combinatorial barcoding.
---

When given barcodes, TRUST4 only assembles the reads with the same barcode together, and then picks a representative pair of chains for each barcode (cell). This page covers 10x Genomics data and other barcode-based single-cell protocols. What to expect from each platform, and how to filter the results, is on [interpreting and filtering results](/guides/interpreting-results/).

## From a Cell Ranger BAM file

For 10x Genomics data, usually the input is the BAM file from Cell Ranger. Use `--barcode` to name the BAM field that holds the barcode, e.g. `--barcode CB`:

```bash
run-trust4 -b possorted_genome_bam.bam -f hg38_bcrtcr.fa --ref human_IMGT+C.fa \
  --barcode CB -t 8 -o sample
```

With BAM input, `--barcode` and `--UMI` are **field names** in the BAM file — `CB` and `UB` for Cell Ranger — not file paths and not the full tag such as `CB:Z:ACGT`. Use `CB` (corrected barcode) rather than `CR` (raw barcode). Barcode values that are not plain ACGT, such as the `-1` suffix or spot names, are fine for BAM input.

When you have the BAM, do not give the FASTQ files as well.

## From raw FASTQ files

If your input is raw FASTQ files, use `--barcode` to specify the barcode file and `--readFormat` to tell TRUST4 how to extract the barcode information. TRUST4 supports wildcards in the `-1 -2`/`-u` options, so a typical way to run 10x Genomics single-end data is:

```bash
run-trust4 -f hg38_bcrtcr.fa --ref human_IMGT+C.fa \
  -u path_to_10X_fastqs/*_R2_*.fastq.gz \
  --barcode path_to_10X_fastqs/*_R1_*.fastq.gz \
  --readFormat bc:0:15 \
  --barcodeWhitelist cellranger_folder/cellranger-cs/VERSION/lib/python/cellranger/barcodes/737K-august-2016.txt \
  [other options]
```

Here read 2 carries the cDNA and read 1 starts with the 16 bp cell barcode. The exact options depend on your 10x Genomics kit — in particular the barcode and UMI lengths and the whitelist file. The I1/I2 files are sample indexes and are not needed.

### Single-end or paired-end?

This is the most common source of trouble. Decide by what read 1 contains:

| Read 1 contains | Run as | Example |
|-----------------|--------|---------|
| Only the barcode and UMI (e.g. 26 or 28 bp) | **Single-end**: read 2 to `-u`, read 1 to `--barcode` (and `--UMI`) | `-u R2.fq.gz --barcode R1.fq.gz --readFormat bc:0:15` |
| Barcode and UMI, then cDNA | **Paired-end**, with `r1:` set to skip the barcode and UMI | `-1 R1.fq.gz -2 R2.fq.gz --barcode R1.fq.gz --readFormat bc:0:15,um:16:27,r1:28:-1` |

Passing a barcode-only read 1 as `-1` makes TRUST4 treat the short barcode reads as cDNA, which "may introduce unexpected assembly artifacts". Leaving out `r1:` when read 1 does carry cDNA makes TRUST4 assemble the barcode and UMI bases too, which also inflates memory. `r2:0:-1` is the default and can be omitted.

Other rules:

- `-u` is ignored when `-1`/`-2` are given; do not mix them.
- `-1` needs a matching `-2`. Giving only `-1` stops the run with `The two mate-pair read files have different number of reads.`
- Data downloaded from SRA or ENA sometimes contains only read 2. Use `fasterq-dump` (with its split options) to get read 1 and read 2 as separate files.

### Common 10x layouts

The maintainer's examples for common 10x read 1 layouts:

| Read 1 layout | `--readFormat` |
|---------------|----------------|
| 16 bp barcode, then UMI (single-end run) | `bc:0:15` for the barcode, adding `um:16:25` (10 bp UMI, v2 chemistry) or `um:16:27` (12 bp UMI, v3) when `--UMI` is given |
| 16 bp barcode + 12 bp UMI + cDNA | `bc:0:15,um:16:27,r1:28:-1` |
| 25 bp barcode + 10 bp UMI + cDNA | `bc:0:24,um:25:34,r1:35:-1` |

Check the layout of your kit before copying these.

:::tip Adding the UMI
Pass read 1 to `--UMI` as well as `--barcode`, and add an `um` field, e.g. `--barcode R1.fq.gz --UMI R1.fq.gz --readFormat bc:0:15,um:16:27`. An `um` field without `--UMI`, or a `bc` field without `--barcode`, is silently ignored: the run then becomes bulk, or the UMI is not used. See [UMIs and molecule barcodes](/guides/umi/).
:::

## The --readFormat specification

The value is a comma-separated string. Each field describes one segment:

```text
[r1|r2|bc|um]:start:end:strand
```

| Part | Meaning |
|------|---------|
| `r1`, `r2`, `bc`, `um` | Which sequence the segment belongs to: read 1, read 2, barcode or UMI. |
| `start`, `end` | 0-based positions, both inclusive. `-1` means the end of the read. A 12 bp UMI at the start of a read is `um:0:11`. |
| `strand` | `+` or `-`. If `-`, the barcode is reverse-complemented after extraction. May be omitted when it is `+`, and is ignored on `r1` and `r2`. |

You may use multiple fields to specify non-consecutive segments, e.g. `bc:0:15,bc:32:-1`. The segments are concatenated in the order they are written in `--readFormat`, not in position order.

For example, when the barcode is in the first 16 bp of read 1 and the rest of read 1 is sequence:

```bash
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -1 read1.fq.gz -2 read2.fq.gz --barcode read1.fq.gz \
  --readFormat bc:0:15,r1:16:-1
```

The usage text's example for paired-end files with barcode and UMI is `r1:0:-1,r2:0:-1,bc:0:15,um:16:-1`.

:::caution The option is --readFormat
`run-trust4` accepts `--readFormat` (camel case). Any other spelling, such as `--read-format`, stops the run with `Unknown parameter`.
:::

### Barcodes and UMIs in the FASTQ header

The `bc` and `um` fields can also parse the barcode and UMI from the FASTQ header comment:

```text
[bc|um]:hd:field:start:end:strand
```

- `hd` is a keyword, so the search will be in the header comment.
- `field` can be a number (0-based), which specifies which field in the comment (the read ID is excluded) contains the barcode/UMI.
- `field` can also be a string, in which case TRUST4 searches for the pattern starting with `field` and extracts the barcode/UMI from there.

For example, if the header looks like

```text
@r1 CR:Z:NNNN CB:Z:ACGT UR:Z:NNNN
```

then `bc:hd:1:5:-1` or `bc:hd:CB:5:-1` will extract the barcode `ACGT` from the header.

The comment is split into fields on spaces. Headers that separate fields with other characters, such as `;`, need to be reformatted first. The header still has to be given through `--barcode`: pass the read file itself, e.g. `-u reads.fq.gz --barcode reads.fq.gz --readFormat bc:hd:CB:5:-1`.

### Already demultiplexed reads

If the barcode is not in the reads at all — for example reads already split by cell — write a FASTA or FASTQ file with one record per read, in the same order as the reads, whose sequence is the barcode (optionally followed by the UMI). Then pass it as `--barcode` (and `--UMI`):

```bash
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa -u reads.fq.gz \
  --barcode bc_umi.fa --UMI bc_umi.fa --readFormat bc:0:7,um:8:-1
```

## Barcode whitelists

`--barcodeWhitelist FILE` gives the path to the barcode whitelist. For 10x Genomics data, the whitelist ships with Cell Ranger (the `737K-august-2016.txt` file in the example above is one of them); use the one for your kit.

- The file is plain text, **one full barcode per line**, ACGT only. Gzipped files are accepted. For combinatorial barcodes, each line is the concatenated barcode.
- Do not use the two-column files in Cell Ranger's `barcodes/translation/` folder as a whitelist.
- Use the **complete** whitelist, not a list of the cells you are interested in. Correction needs the full list; with a partial list an erroneous barcode "might be corrected to another 'wrong' barcode". Restrict to your cells afterwards.

How correction works: a barcode that is in the whitelist is kept. Otherwise TRUST4 tries every single-base substitution. If more than one whitelist barcode is one mismatch away, it picks the one seen more often in the data, then the one whose corrected position has the lower base quality. A barcode that cannot be corrected is written as `missing_barcode`, and its reads are not used. Without a whitelist, no correction is done.

:::note FASTQ input only
`run-trust4` passes `--barcodeWhitelist`, `--barcodeTranslate` and `--readFormat` to the FASTQ extraction step only. With `-b`, the barcodes are taken from the BAM field as they are — for a Cell Ranger BAM, the `CB` field already holds corrected barcodes.
:::

## Barcode translation and combinatorial barcoding

TRUST4 can translate input cell barcodes to another set of barcodes. Specify the translation file with `--barcodeTranslate FILE`. The translation file is a two-column TSV/CSV file with:

1. the translated barcode in the first column, and
2. the original barcode in the second column.

This option also supports combinatorial barcoding, such as SHARE-seq. TRUST4 translates each barcode segment given in the second column to the ID in the first column, and concatenates the IDs with `-` in the output.

Behaviour to be aware of:

- Whitelist correction runs **before** translation.
- Several original barcodes may map to the same translated ID. If the same original barcode appears twice, the last line wins.
- A barcode missing from the translation table stops the run with `Barcode ... does not exist in the translation table.` The table must cover every barcode that can occur.

For combinatorial platforms such as Split-seq and Parse Biosciences, `scripts/combinatorial-barcode-whitelistgen.py` generates the whitelist or the barcode translation table from per-round barcode files. See [utility scripts](/guides/utility-scripts/#combinatorial-barcode-whitelistgen-py). Give the barcode segments in `--readFormat` in the same order as the rounds in the generated whitelist.

For BD Rhapsody, the whole cell-label region, including the linkers between the cell-label segments, can be treated as one barcode; there is no length limit. Sequencing errors in the linkers then cost some reads.

## Checking barcode extraction

After the extraction stage, `PREFIX_toassemble_bc.fa` holds the barcode of each candidate read. Count the reads whose barcode could not be extracted or corrected:

```bash
grep -c missing_barcode PREFIX_toassemble_bc.fa
```

A large fraction means the `--readFormat` positions, the strand, or the whitelist do not match your data. Some platforms need the barcode reverse-complemented (`bc:...:-`).

## Common mistakes

- `-1 R1 -2 R2` when read 1 holds only the barcode and UMI — use `-u R2`.
- A `bc:`/`um:` field in `--readFormat` without `--barcode FILE`/`--UMI FILE`.
- A number or a BAM path given to `--barcode`: with FASTQ input it is a file, with BAM input it is a tag name.
- Off-by-one ranges: `start` and `end` are both inclusive.
- A whitelist restricted to the cells of interest.
- Options copied from old versions: `--barcodeRange`, `--read1Range`, `--read2Range` and `--umiRange` are outdated and buggy; use `--readFormat`. See the [mapping on the CLI page](/reference/cli/#legacy-options).
- Running the `trust4` binary instead of `run-trust4`.

## What changes in the output

With barcodes:

- The abundance in `_report.tsv` is the **number of barcodes** for each CDR3 instead of the read count.
- TRUST4 also generates `_barcode_report.tsv`, in which it picks the most abundant pair of chains as the representative for each barcode (cell).
- TRUST4 converts the barcode report to `_barcode_airr.tsv`, following the AIRR format.
- Contig IDs are `BARCODE_N`, where `N` is an internal contig index with gaps. Use `cell_id` in the AIRR file for the barcode.

The format of the barcode report is:

```text
barcode  cell_type  IGH/TRB/TRD_information  IGK/IGL/TRA/TRG_information  secondary_chain1_information  secondary_chain2_information
```

Each chain's information is in CSV form:

```text
V_gene,D_gene,J_gene,C_gene,cdr3_nt,cdr3_aa,read_cnt,consensus_id,CDR3_germline_similarity,consensus_complete_vdj
```

Full details are on the [output formats](/reference/output-formats/#trust-barcode-report-tsv) page. A chain field of `*` means that chain was not detected in the cell.

### CDR3 imputation

When a cell has only a partial CDR3 for a chain, `trust-barcoderep.pl` can complete it from another cell with the same partner chain, the same gene usage and a matching CDR3 substring. The consensus ID then shows the source cell. By default this is done for TCRs only; `--imputeBCR` extends it to BCRs and `--noImputation` turns it off. This is why the barcode report can contain CDR3s that are not in `_cdr3.out` for that barcode.

## Cleaning up the barcode report

The `scripts/` folder has tools for common single-cell follow-up:

- `barcoderep-filter.py` removes lowly expressed clonotypes that are identical to a highly expressed clonotype in another cell, a sign of contamination from diffused mRNA.
- `barcoderep-expand.py` puts the secondary chains in their own barcode lines, useful when a barcode contains multiple cells.
- `trust-barcoderep-to-10X.pl` converts the barcode report to the 10x Cell Ranger VDJ format.

Before any of these, keep only the barcodes that pass gene-expression QC. See [filtering single-cell results](/guides/interpreting-results/#filtering-single-cell-results) and [utility scripts](/guides/utility-scripts/).
