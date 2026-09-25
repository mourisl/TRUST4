---
title: Quick start
description: Run TRUST4 on the bundled example data from BAM and from FASTQ input, and check the results.
---

This page runs TRUST4 on the small example data set that comes with the source distribution. The whole run takes seconds and confirms that the installation works.

:::note Before you begin
Build TRUST4 as described in [installation](/getting-started/installation/), and run the commands below from the TRUST4 source folder, where `example/`, `hg38_bcrtcr.fa` and `human_IMGT+C.fa` live.
:::

## Run on a BAM file

The directory `./example` contains one BAM file as input for TRUST4. Run TRUST4 with:

```bash
./run-trust4 -b example/example.bam -f hg38_bcrtcr.fa --ref human_IMGT+C.fa
```

- `-b` is the alignment of RNA-seq reads.
- `-f` is the file with the genomic sequence **and coordinates** of the V, J, C genes. The coordinates are what lets TRUST4 extract candidate reads from a BAM file.
- `--ref` is the IMGT reference used to annotate genes and CDRs.

The run will generate the files `TRUST_example_raw.out`, `TRUST_example_final.out`, `TRUST_example_annot.fa`, `TRUST_example_cdr3.out`, `TRUST_example_report.tsv` and several fq/fa files in seconds. The output prefix `TRUST_example` was inferred from the input file name; set it yourself with `-o`.

The results should be the same as the files in the `example` folder.

### Check the result automatically

```bash
bash trust-example-test.sh
```

This script runs the command above with the prefix `example_test`, compares the report with `example/TRUST_example_report.tsv`, deletes its own output and prints `TRUST4 is ready to use.` when they match.

## Run on FASTQ files

The directory also contains two FASTQ files, and you can run TRUST4 with:

```bash
./run-trust4 -f hg38_bcrtcr.fa --ref human_IMGT+C.fa \
  -1 example/example_1.fq -2 example/example_2.fq -o TRUST_example
```

The coordinate information in `hg38_bcrtcr.fa` is not needed for FASTQ input, so you can use the IMGT reference file for the `-f` option too:

```bash
./run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -1 example/example_1.fq -2 example/example_2.fq -o TRUST_example
```

The run will generate the same files as from BAM input. Being able to use the IMGT file for `-f` is useful when analyzing species without reference genomes or genome annotations.

## Look at the results

The file most people want first is the report table:

```bash
head TRUST_example_report.tsv
```

```text
#count  frequency     CDR3nt          CDR3aa        V            D            J         C     cid         cid_full_length
8       8.247423e-02  TGTGCGAGAGGG... out_of_frame  IGHV1-3*01   IGHD1-26*01  IGHJ6*03  .     assemble1   0
6       6.185567e-02  TGTGCGAGGGGG... out_of_frame  IGHV4-61*01  IGHD2-2*01   IGHJ3*02  .     assemble2   0
```

(The CDR3 sequences are truncated here.) Each row is a CDR3 with its read count, its frequency, the nucleotide and amino-acid CDR3 sequence, and the V, D, J and C gene assignments. The same results are in AIRR format in `TRUST_example_airr.tsv`. The [output formats](/reference/output-formats/) page explains every column.

## Run on your own data

A typical run on a sample adds threads and an output directory:

```bash
run-trust4 -b sample.bam -f hg38_bcrtcr.fa --ref human_IMGT+C.fa \
  -t 8 -o sample --od trust4_out
```

:::caution Match -f to your reference genome
For BAM input, the coordinates in the `-f` file must match the genome the reads were aligned to: `hg38_bcrtcr.fa` for hg38, `hg19_bcrtcr.fa` for hg19, and the files in `mouse/` for GRCm38 or GRCm39.
:::

Where to go next depends on your data:

- [Bulk RNA-seq](/guides/bulk-rna-seq/) — BAM and FASTQ input, targeted TCR-seq/BCR-seq.
- [10x Genomics and single-cell data](/guides/single-cell/) — cell barcodes, whitelists and read formats.
- [SMART-seq data](/guides/smart-seq/) — one file pair per cell.
