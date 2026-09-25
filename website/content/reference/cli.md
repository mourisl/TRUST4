---
title: Command-line interface
description: Complete option reference for run-trust4, annotator, trust-smartseq.pl, the reporting scripts and the reference-building scripts.
---

Every TRUST4 program prints its usage message when run without arguments. This page collects those options in one place, for TRUST4 v1.1.11.

## run-trust4

The driver for the whole pipeline: candidate read extraction, assembly, annotation and reporting. See [bulk RNA-seq](/guides/bulk-rna-seq/) and [single-cell data](/guides/single-cell/) for worked examples.

```text
Usage: ./run-trust4 [OPTIONS]
```

### Required

| Option | Description |
|--------|-------------|
| `-b STRING` | Path to BAM file. |
| `-1 STRING -2 STRING` | Path to paired-end read files. |
| `-u STRING` | Path to single-end read file. |
| `-f STRING` | Path to the FASTA file with the coordinate and sequence of V/D/J/C genes. |

`-f` is always required, together with one of the read-input forms: `-b`, `-1`/`-2` or `-u`. `-1`, `-2` and `-u` accept several files and wildcards.

### Optional: general

| Option | Default | Description |
|--------|---------|-------------|
| `--ref STRING` | not used but recommended | Path to detailed V/D/J/C gene reference file from the IMGT database. |
| `-o STRING` | inferred from file prefix | Prefix of output files. |
| `--od STRING` | `./` | The directory for output files. |
| `-t INT` | `1` | Number of threads. |
| `-k INT` | `9` | The starting k-mer size for indexing contigs. |
| `--stage INT` | `0` | Start TRUST4 on the specified stage. `0`: start from beginning (candidate read extraction). `1`: start from assembly. `2`: start from annotation. `3`: start from generating the report table. |
| `--clean INT` | `0` | Clean up files. `0`: no clean. `1`: clean intermediate files. `2`: only keep AIRR files. |
| `--outputReadAssignment` | no output | Output read assignment results to the `prefix_assign.out` file. |

### Optional: barcodes and UMIs

| Option | Default | Description |
|--------|---------|-------------|
| `--barcode STRING` | not used | If `-b`, the BAM field for the barcode; if `-1 -2`/`-u`, the file containing barcodes. |
| `--barcodeLevel STRING` | `cell` | Barcode is for `cell` or `molecule`. |
| `--barcodeWhitelist STRING` | not used | Path to the barcode whitelist. |
| `--barcodeTranslate STRING` | not used | Path to the barcode translate file. |
| `--UMI STRING` | not used | If `-b`, the BAM field for 10x Genomics-like UMI; if `-1 -2`/`-u`, the file containing 10x Genomics-like UMIs. |
| `--readFormat STRING` | — | Format for read, barcode and UMI files, e.g. `r1:0:-1,r2:0:-1,bc:0:15,um:16:-1` for paired-end files with barcode and UMI. |

The `--readFormat` syntax is described in full on the [single-cell page](/guides/single-cell/#the-readformat-specification). `--barcodeWhitelist`, `--barcodeTranslate` and `--readFormat` apply to FASTQ/FASTA input only.

### Optional: assembly and annotation

| Option | Default | Description |
|--------|---------|-------------|
| `--repseq` | not set | The data is from bulk, non-UMI-based TCR-seq or BCR-seq. |
| `--contigMinCov INT` | `0` | Ignore contigs that have bases covered by fewer than INT reads. |
| `--minHitLen INT` | auto | The minimal hit length for a valid overlap. |
| `--mateIdSuffixLen INT` | not used | The suffix length in read id for mate. |
| `--skipMateExtension` | not used | Do not extend assemblies with mate information, useful for SMART-seq. |
| `--skipReadRealign` | not used | Do not realign reads in annotator, useful for reducing computation cost of barcode/UMI-based repseq. |
| `--cgeneEnd INT` | `200` | Skip reads aligned after the first INT bp of the C gene. |
| `--abnormalUnmapFlag` | not set | The flag in BAM for the unmapped read-pair is nonconcordant. |
| `--noExtraction` | extraction first | Directly use the files provided with `-1 -2`/`-u` to assemble. |
| `--imgtAdditionalGap STRING` | no | Description for additional gap codon position in IMGT (0-based), e.g. `"TRAV:7,83"` for mouse. |
| `--assembleWithRef` | use `-f` file | Conduct the assembly with the `--ref` file. |

### Legacy options

Answers on older issues often use options from TRUST4 versions before 1.0.10. `run-trust4` still parses them but they are hidden from the usage text, outdated and buggy. Use `--readFormat` instead:

| Old option | `--readFormat` equivalent |
|------------|---------------------------|
| `--barcodeRange 0 15 +` | `bc:0:15` |
| `--barcodeRange 0 15 -` | `bc:0:15:-` |
| `--umiRange 16 27 +` | `um:16:27` |
| `--read1Range 16 -1` | `r1:16:-1` |
| `--read2Range 0 -1` | `r2:0:-1` |

Combine the fields with commas, e.g. `--readFormat bc:0:15,um:16:27,r1:28:-1`.

### Checks run-trust4 makes

`run-trust4` stops before doing any work when:

- no reads are given — `Need to use -b/{-1,-2}/-u to specify input reads.`
- `-f` is missing — `Need to use -f to specify the sequence of annotated V/D/J/C genes' sequence.`
- `--noExtraction` is combined with `-b` — it can only be set when using `-1 -2`/`-u` as input.
- `--assembleWithRef` is used without `--ref`.
- an input or reference file does not exist — `Could not find file ...`
- an option is not recognised — `Unknown parameter ...`

## annotator

Annotates V, D, J, C genes and CDRs. `run-trust4` calls it on the assemblies; it can also be run on its own — see [annotating sequences](/guides/annotation-only/).

```text
Usage: ./annotator [OPTIONS]
```

### Required

| Option | Description |
|--------|-------------|
| `-f STRING` | FASTA file containing the receptor genome sequence. |
| `-a STRING` | Path to the assembly file. |

### Optional

| Option | Default | Description |
|--------|---------|-------------|
| `-r STRING` | — | Path to the reads used in the assembly. |
| `--fasta` | false | The assembly file is in FASTA format. |
| `--fastq` | false | The assembly file is in FASTQ format. |
| `-t INT` | `1` | Number of threads. |
| `-o STRING` | `trust` | The prefix of the file containing CDR3 information. |
| `--barcode` | not set | There is barcode information in the `-a` and `-r` files. |
| `--UMI` | not set | There is UMI information in the `-r` file. |
| `--geneAlignment` | not set | Output the gene alignment. |
| `--airrAlignment` | not set | Output the aligned sequences to `prefix_airr_align.tsv`. |
| `--noImpute` | impute | Do not impute CDR3 sequence for TCR. |
| `--notIMGT` | IMGT format | The receptor genome sequence is not in IMGT format. |
| `--imgtAdditionalGap STRING` | no | Description for additional gap codon position in IMGT (0-based), e.g. `"TRAV:7,83"` for mouse. |
| `--outputCDR3File` | no output | Output the CDR3 file when not using the `-r` option. |
| `--needReverseComplement` | no | Reverse complement sequences on another strand. |
| `--outputFormat INT` | `0` | `0`: FASTA, `1`: AIRR. |
| `--readAssignment STRING` | no output | Output the read assignment to the file. |

The annotated result is written to standard output.

## trust-smartseq.pl

Runs TRUST4 on each cell of a plate-based experiment. See [SMART-seq data](/guides/smart-seq/).

```text
Usage: perl trust-smartseq.pl [OPTIONS]
```

| Option | Default | Description |
|--------|---------|-------------|
| `-1 STRING` | — | File containing the list of read 1 (or single-end) files. |
| `-2 STRING` | — | File containing the list of read 2 files. |
| `-f STRING` | — | Path to the FASTA file with the coordinate and sequence of V/D/J/C genes. |
| `--ref STRING` | not used but recommended | Path to detailed V/D/J/C gene reference file from the IMGT database. |
| `-o STRING` | `TRUST` | Prefix of final output files. |
| `-t INT` | `1` | Number of threads. |
| `--representative INT` | `1` | Number of representatives for each detected chain. |
| `--cgeneEnd INT` | `200` | Skip reads aligned after the first 200 bp of the C gene. Currently rejected; see the [caution](/guides/smart-seq/#options). |
| `--trust-path STRING` | same as this script | TRUST4 executable files. |

## Reporting scripts

`run-trust4` calls these at [stage 3](/reference/pipeline/#reporting). They can be rerun on their own.

### trust-simplerep.pl

```text
Usage: ./trust-simplerep.pl xxx_cdr3.out [OPTIONS] > trust_report.out
```

| Option | Default | Description |
|--------|---------|-------------|
| `--decimalCnt` | not used | The count column uses decimal instead of truncated integer. |
| `--barcodeCnt` | not used | The count column is the number of barcodes instead of read support. |
| `--junction trust_annot.fa` | not used | Output junction information for the CDR3. |
| `--reportPartial` | no partial | Include partial CDR3 in the report. |
| `--filterBarcoderep barcode_report.tsv` | not used | Only summarize primary CDR3s in the barcode report file. |
| `--filterBarcoderepReadCnt FLOAT` | `0` | Filter primary CDR3s in the barcode report with less than this read count. |
| `--filterTcrError FLOAT` | `0.05` | Filter TCR CDR3s less than this fraction of the representative CDR3 in the consensus. |
| `--filterBcrError FLOAT` | `0` | Filter BCR CDR3s less than this fraction of the representative CDR3 in the consensus. |

### trust-barcoderep.pl

```text
Usage: ./trust-barcoderep.pl xxx_cdr3.out [OPTIONS] > trust_barcode_report.tsv
```

| Option | Default | Description |
|--------|---------|-------------|
| `-a xxx_annot.fa` | not used | TRUST4's annotation file. |
| `--noImputation` | impute | Do not perform imputation for partial CDR3. |
| `--imputeBCR` | no | Perform imputation for BCR partial CDR3. |
| `--reportPartial` | no partial | Include partial CDR3 in the report. |
| `--chainsInBarcode INT` | `2` | Number of chains in a barcode. |

### trust-airr.pl

```text
Usage: trust-airr.pl trust_report.tsv trust_annot.fa [OPTIONS] > trust_airr.tsv
```

| Option | Description |
|--------|-------------|
| `--format STRING` | `simplerep`, `cdr3` or `barcoderep`: the format of the report file. |
| `--airr-align STRING` | The `trust_airr_align.tsv` file. |

## Reference-building scripts

See [reference gene files](/guides/reference-files/).

### BuildImgtAnnot.pl

```text
Usage: perl BuildImgtAnnot.pl species_name(Homo_sapiens or others) > output.fa
```

Downloads the IMGT reference sequences with `wget` and keeps those of the given species.

### BuildDatabaseFa.pl

```text
Usage: perl BuildDatabaseFa.pl reference.fa annotation.gtf interested_gene_name_list > output.fa
```

Builds the `-f` file, with genomic coordinates, from a reference genome, its GTF annotation and a list of gene names such as `human_vdjc.list`.

## Streams

- `run-trust4` logs each step, with a timestamp and the exact `SYSTEM CALL` it runs, to standard error. Results go to files.
- `annotator` and the reporting scripts write their results to standard output.
