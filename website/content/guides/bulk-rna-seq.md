---
title: Bulk RNA-seq
description: Run TRUST4 on bulk RNA-seq from BAM alignments or raw FASTQ files, and on targeted TCR-seq or BCR-seq.
---

Bulk RNA-seq is TRUST4's primary use case: unselected RNA sequencing data, profiled from fluid and solid tissues, including tumors. This page covers the two input routes and the options that matter most for bulk data.

## From a BAM file

The primary input to TRUST4 is the alignment of RNA-seq reads in BAM format (`-b`), the file containing the genomic sequence and coordinate of V, J, C genes (`-f`), and the reference database sequence containing annotation information, such as IMGT (`--ref`):

```bash
run-trust4 -b sample.bam -f hg38_bcrtcr.fa --ref human_IMGT+C.fa -t 8
```

TRUST4 uses the gene coordinates in the `-f` file to extract candidate reads from the alignment, including unmapped reads. That is why `-f` must match the genome the BAM was aligned to — see [reference gene files](/guides/reference-files/#which-f-file-to-use).

### BAM-specific options

| Option | Use it when |
|--------|-------------|
| `--abnormalUnmapFlag` | The flag in BAM for the unmapped read-pair is nonconcordant. |
| `--mateIdSuffixLen INT` | Read IDs carry a mate suffix of this length, e.g. `2` for `.1`/`.2`. |

`--abnormalUnmapFlag` is safe to use on any BAM file, only slower. It is the fix for the error `Two reads from the unaligned fragment are not showing up together`, which also happens with BAMs made by merging mapped and unmapped reads.

### BAM input problems

**Keep the unmapped reads.** Many of the reads covering the CDR3 cannot be mapped to the reference genome, because the rearranged sequence is not in the genome. TRUST4 uses those unmapped reads as well. A BAM without unmapped reads, or a BAM subset to the immune loci (for example chr7 and chr14), loses much of the CDR3 information. Use the FASTQ files, or regenerate the BAM with unmapped reads.

**One BAM per run.** `run-trust4` does not support multiple BAM files and only uses the first `-b`. Merge the BAMs first, or run each separately.

**Chromosome names must match.** The second field of each `-f` header is the chromosome name, e.g. `chr1` in `>IGKV1OR1-1 chr1 144085628 144086098 +`. It must match the BAM's reference names; otherwise the run stops with `Unknown genome name: X`. This happens with `chr7` vs `7` naming, NCBI accession names, combined human+mouse Cell Ranger references (`GRCh38_` prefixes), and BAMs whose header lacks `@SQ` lines. Rename the `-f` entries to match, or use FASTQ input.

**Genome alignments only.** BAMs aligned to a transcriptome (such as some GDC BAMs), Cell Ranger VDJ contig BAMs, and BAMs made with SRA's `sam-dump` do not work reliably. Use the FASTQ files instead. Cell Ranger's `consensus.bam` can be converted to FASTQ first.

BAM and FASTQ input give almost identical results; BAM is faster because reads outside the V, D, J and C genes are skipped.

## From FASTQ or FASTA files

An alternative input to TRUST4 is the raw RNA-seq files in FASTA/FASTQ format: `-1`/`-2` for paired-end, `-u` for single-end. You still need `-f` and `--ref`, but you can directly use IMGT's sequence file for `-f`:

```bash
# paired-end
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -1 sample_1.fq.gz -2 sample_2.fq.gz -t 8 -o sample

# single-end
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -u sample.fq.gz -t 8 -o sample
```

### Several files per sample

`-1`, `-2` and `-u` take every following argument up to the next option, and also expand wildcards themselves. Both of these therefore pass all lanes of a sample:

```bash
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -1 sample_L001_R1.fq.gz sample_L002_R1.fq.gz \
  -2 sample_L001_R2.fq.gz sample_L002_R2.fq.gz

run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -1 "fastqs/sample_*_R1.fq.gz" -2 "fastqs/sample_*_R2.fq.gz"
```

When you list several files for `-1` and `-2`, keep them in the same order so that mates stay paired.

### Compressed and streamed input

Gzipped FASTQ files are read directly. For other compression, such as bzip2, process substitution works:

```bash
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -1 <(bzcat sample_1.fq.bz2) -2 <(bzcat sample_2.fq.bz2)
```

Streams cannot be rewound, and TRUST4 uses the first reads of the input to collect statistics before processing it, so the maintainer's advice for streamed input was to make sure the first reads are representative. With `--barcodeWhitelist`, the barcode file is also scanned in advance to collect barcode frequencies, so keep barcode files on disk rather than streaming them.

### Skipping extraction

By default TRUST4 first extracts candidate reads from the FASTQ files. `--noExtraction` directly uses the files provided with `-1 -2`/`-u` to assemble. It can only be set when using `-1 -2`/`-u` as input.

## Targeted TCR-seq and BCR-seq

For bulk, non-UMI-based TCR-seq or BCR-seq data, add `--repseq`:

```bash
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -1 tcrseq_1.fq.gz -2 tcrseq_2.fq.gz --repseq -t 8
```

`--repseq` makes the assembler trim unmatched read portions and skip extension of assemblies with mate information. The trade-offs:

- It "invokes aggressive read trimming for the portion that does not align well to V, J, C", and only considers overlaps with compatible V and J genes. It is faster, but "may lose some real read signals".
- Because the 5′ end of V and the 3′ end of J are trimmed, there are fewer complete VDJ assemblies, while the number of CDR3s is similar.
- It is for bulk TCR-seq/BCR-seq **without** UMIs. For UMI-based kits see [UMIs and molecule barcodes](/guides/umi/). For 10x VDJ-kit data, the maintainer advised against it from v1.1.0 because it lowers the number of complete assemblies.
- Primer or other non-receptor sequence in amplicon data creates spurious contigs and slows the run; `--repseq` helps here too.

TRUST4 is designed for unamplified data, so deeply amplified libraries run slowly. For very deep amplicon data, add `--repseq`, downsample, or [split and parallelize](/reference/troubleshooting/#speed).

## Output location and naming

| Option | Effect |
|--------|--------|
| `-o STRING` | Prefix of output files. Default: `TRUST_` followed by the first input file name up to its first `.`, e.g. `TRUST_sample` for `sample.bam`. |
| `--od STRING` | The directory for output files, created if it does not exist. Default: `./`. |

## Useful options for bulk data

| Option | Default | Description |
|--------|---------|-------------|
| `-t INT` | `1` | Number of threads. |
| `--contigMinCov INT` | `0` | Ignore contigs that have bases covered by fewer than INT reads. |
| `--clean INT` | `0` | Clean up files. `0`: no clean. `1`: clean intermediate files. `2`: only keep AIRR files. |
| `--outputReadAssignment` | no output | Output read assignment results to the `prefix_assign.out` file. |
| `--stage INT` | `0` | Start TRUST4 at a later stage, reusing earlier results. |

The complete list is on the [command-line interface](/reference/cli/#run-trust4) page, and the files each stage reads and writes are on [pipeline stages and files](/reference/pipeline/).

## Merging runs of the same sample

To combine two sequencing runs of one sample after extraction, concatenate their `PREFIX_toassemble*.fq` files into one prefix and restart from assembly with `--stage 1`. See [pipeline stages and files](/reference/pipeline/).

## Looking at a subset of chains

The simple report is generated from `_cdr3.out` by `trust-simplerep.pl`. If you are interested in a subset of chains, you can `grep` those from the `_cdr3.out` file and run `trust-simplerep.pl` on the subset:

```bash
grep "TRB" TRUST_sample_cdr3.out > TRUST_sample_TRB_cdr3.out
perl trust-simplerep.pl TRUST_sample_TRB_cdr3.out > TRUST_sample_TRB_report.tsv
```

To split TCRs from BCRs, filter on the gene columns, e.g. `awk -F'\t' '$5~"IG" || $7~"IG" || $8~"IG"' TRUST_sample_report.tsv` for BCRs.

How to filter and interpret the results — singletons, out-of-frame CDR3s, very high counts — is covered in [interpreting and filtering results](/guides/interpreting-results/#filtering-bulk-results).
