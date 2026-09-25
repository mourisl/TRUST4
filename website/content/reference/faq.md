---
title: FAQ
description: Choosing reference files, reading the reports, and what to do when a run does not behave.
---

## Setup and reference files

### Which -f file should I use?

For BAM input, a file with genomic coordinates that match the genome your reads were aligned to: `hg38_bcrtcr.fa` or `hg19_bcrtcr.fa` for human, `mouse/GRCm38_bcrtcr.fa` or `mouse/GRCm39_bcrtcr.fa` for mouse. For FASTQ input, the coordinates are not needed, and the IMGT file (`human_IMGT+C.fa`, `mouse/mouse_IMGT+C.fa`) can be used for `-f` as well. See [reference gene files](/guides/reference-files/#which-f-file-to-use).

### Do I need --ref?

It is optional but recommended. `--ref` is the detailed IMGT reference that the annotation uses to identify genes and CDRs. If you leave it out and `-f` is an IMGT-format file, `run-trust4` uses the `-f` file as the reference automatically.

### Can I use TRUST4 for a species other than human or mouse?

Yes. Build the IMGT reference with `BuildImgtAnnot.pl` for your species. With a reference genome and GTF annotation you can also build a coordinate file for BAM input with `BuildDatabaseFa.pl`; without them, run on FASTQ files with the IMGT file as both `-f` and `--ref`. See [a complete recipe for a new species](/guides/reference-files/#a-complete-recipe-for-a-new-species).

### I installed TRUST4 with Conda. Where are hg38_bcrtcr.fa and human_IMGT+C.fa?

The reference and example files are in the [GitHub repository](https://github.com/liulab-dfci/TRUST4). Clone or download the repository to get them.

## Running TRUST4

### How do I check that my installation works?

Run `bash trust-example-test.sh` in the TRUST4 folder. It prints `TRUST4 is ready to use.` when the example results match the pre-generated ones. See the [quick start](/getting-started/quick-start/).

### The run stops with "Unknown parameter"

`run-trust4` rejects any option it does not know. Option names are case-sensitive and use camel case — `--readFormat`, `--barcodeWhitelist`, `--barcodeTranslate`, `--contigMinCov`, `--skipMateExtension`. A hyphenated spelling such as `--read-format` is not accepted. The [command-line interface](/reference/cli/) lists every option.

### The run stops with "Could not find file"

`run-trust4` checks that the `-f`, `--ref` and read files exist before starting, and, for FASTQ input, the barcode files too. Check the path; with wildcards, check that the pattern matches at least one file.

### Where did my output go, and what is it called?

Into the current directory, or the directory given by `--od`. The prefix defaults to `TRUST_` followed by the first input file name up to its first `.` — `sample.R1.fq.gz` gives `TRUST_sample`. Set it explicitly with `-o`.

### Can I re-run only part of the pipeline?

Yes. `--stage 1` starts from assembly, `--stage 2` from annotation and `--stage 3` from the report tables, reusing the files of the earlier stages. Keep the same `-o` and `--od`, and do not use `--clean` on the run you want to resume. See [pipeline stages and files](/reference/pipeline/#restarting-from-a-stage).

### How do I make TRUST4 use less disk space?

`--clean 1` removes intermediate files at the end of the run, and `--clean 2` only keeps the AIRR files.

## Single-cell data

### My 10x FASTQ run finds few or no cells

Check the barcode options against your kit: which read carries the barcode, its length in `--readFormat`, and the whitelist given to `--barcodeWhitelist`. With the 10x layout in the [single-cell guide](/guides/single-cell/#from-raw-fastq-files), the cDNA read goes to `-u` and the barcode read to `--barcode`. Count `missing_barcode` in `PREFIX_toassemble_bc.fa` to see whether barcodes are being extracted; see [checking barcode extraction](/guides/single-cell/#checking-barcode-extraction).

### Should read 1 go to -1 or only to --barcode?

If read 1 contains only the barcode and UMI, run single-end: read 2 to `-u`, read 1 to `--barcode` (and `--UMI`). If read 1 continues into cDNA, run paired-end and skip the barcode and UMI bases with an `r1:` field. See [single-end or paired-end?](/guides/single-cell/#single-end-or-paired-end)

### Should I use --UMI for my UMI-based TCR-seq kit?

No. `--UMI` is for 10x-like single-cell data. For UMI-based bulk repertoire sequencing, give the UMI as a molecule barcode: `--barcode UMIFILE --barcodeLevel molecule`. See [UMIs and molecule barcodes](/guides/umi/).

### Does --barcodeWhitelist work with BAM input?

No. `--barcodeWhitelist`, `--barcodeTranslate` and `--readFormat` are applied when extracting reads from FASTQ files. With `-b`, the barcode comes straight from the BAM field named by `--barcode`.

### Why are the counts in the report so small for single-cell data?

With barcodes, the abundance in `_report.tsv` is the number of barcodes (cells) carrying the CDR3, not the read count.

### A barcode seems to contain more than one cell

The secondary chains of each barcode are kept in the `secondary_chain1` and `secondary_chain2` columns of `_barcode_report.tsv`. `scripts/barcoderep-expand.py` can move them to their own barcode lines. See [utility scripts](/guides/utility-scripts/#barcoderep-expand-py).

## Reading the results

### What does "out_of_frame" mean in the CDR3aa column?

The length of the CDR3 nucleotide sequence is not a multiple of three, so it is not translated into amino acids. Such a CDR3 is non-productive.

### What do "_" and "?" mean in an amino-acid sequence?

`_` represents a stop codon, and `?` represents the ambiguous nucleotide `N` in a codon.

### What is the CDR3 score?

In `_annot.fa`, `0.00` means a partial CDR3, `1.00` means a CDR3 with imputed nucleotides, and other numbers are the motif signal strength with `100.00` the strongest. In `_cdr3.out` the same score is divided by 100. See [output formats](/reference/output-formats/#cdr-annotations).

### Why does the frequency column sum to more than 1?

In `_report.tsv`, the frequency is normalized separately within each chain group: IGH, IGK+IGL, TRA, TRB, TRG and TRD. The column therefore sums to up to 6. It is not normalized for library size; divide the counts by the sample's total reads to compare immune infiltration between samples.

### What do the counts mean?

It depends on the file and on whether barcodes and UMIs were given: reads for bulk data, cells in the pseudo-bulk `_report.tsv` of single-cell data, and reads or UMIs per cell in the barcode files. See [what the counts mean](/guides/interpreting-results/#what-the-counts-mean).

### Why is a contig in _annot.fa or _cdr3.out missing from the report?

The report leaves out partial CDR3s and merges rows with the same CDR3, V, J and C genes, keeping one contig ID. Assemblies without a CDR3 are not in `_cdr3.out` at all. See [how the files relate](/guides/interpreting-results/#how-the-files-relate).

### Should I filter on cid_full_length, CDR3 score or germline similarity?

Usually not. The CDR3 in the report is always complete, the CDR3 score reflects conserved motifs that some genes lack, and germline similarity is normally low for hypermutated BCRs. See [what not to filter on](/guides/interpreting-results/#do-not-filter-on-these).

### Why are there many more cells than Cell Ranger reports?

TRUST4 assembles contigs from every barcode, including empty droplets and low-quality cells, and ambient BCR mRNA from plasma cells creates calls in "null" cells. Keep only the barcodes that pass gene-expression QC, and filter on read support. See [filtering single-cell results](/guides/interpreting-results/#filtering-single-cell-results).

### Why does my 10x 3′ data give so few results?

The V(D)J region is at the 5′ end of the receptor transcript and the C gene is long, so few 3′ reads reach the CDR3. Sensitivity is around 5% of cells for 10x 3′ data, and the maintainer does not think it can be improved computationally. The calls that are found are reliable. See [what to expect from your data type](/guides/interpreting-results/#what-to-expect-from-your-data-type).

### Why do V gene calls differ from IgBLAST or Cell Ranger?

The alignment algorithms differ. With short V coverage or strong hypermutation, the V call is less certain; use the CDR3 as the anchor. See [gene calls](/guides/interpreting-results/#gene-calls).

### How do I get CDR1 and CDR2 for mouse TRAV genes?

IMGT has additional gaps on the TRAV genes. Add `--imgtAdditionalGap TRAV:7,83`. See [reference gene files](/guides/reference-files/#mouse-trav-genes-and-imgt-gaps).

## Performance

### How much memory and time does TRUST4 need?

Memory depends on the number of candidate reads, not the input size; about 16–20 GB for 8 million candidate reads was the maintainer's estimate. Exit code 9 means the job was killed, usually for memory. Threads speed up extraction and annotation but not assembly. See [memory](/reference/troubleshooting/#memory) and [speed](/reference/troubleshooting/#speed).

## Getting help

Error messages and exit codes are explained on [troubleshooting](/reference/troubleshooting/). If none of that helps, create a [GitHub issue](https://github.com/liulab-dfci/TRUST4/issues). See [citation and support](/about/citation/#support) for what to include.
