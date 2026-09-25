---
title: UMIs and molecule barcodes
description: UMI-based abundance estimation for 10x Genomics-like data, and UMI-based TCR-seq/BCR-seq kits with --barcodeLevel molecule.
---

TRUST4 can use unique molecular identifiers (UMIs) in two different ways. Choosing the right one matters.

| Data | Use | What the UMI does |
|------|-----|-------------------|
| 10x Genomics-like single-cell data: cell barcode + UMI | `--barcode ... --UMI ...` | Counts UMIs instead of reads for each chain in each cell. |
| UMI-based bulk TCR-seq/BCR-seq (5′RACE and similar kits) | `--barcode UMIFILE --barcodeLevel molecule` | Each UMI is treated as a barcode, and the reads of each molecule are assembled separately. |

`--UMI` is "for abundance estimation only for data like 10x". For UMI-based repertoire sequencing, the molecule-barcode mode is much more robust.

## 10x Genomics-like UMIs

For 10x Genomics data, TRUST4 supports UMI-based abundance estimation. Use `--UMI` to specify the UMI sequence file (FASTQ input) or the field in the BAM file (BAM input). If the sequence contains non-UMI information, use `--readFormat` with the keyword `um` to specify the UMI sequence range.

From a Cell Ranger BAM file:

```bash
run-trust4 -b possorted_genome_bam.bam -f hg38_bcrtcr.fa --ref human_IMGT+C.fa \
  --barcode CB --UMI UB -t 8
```

From FASTQ files, where read 1 holds a 16 bp barcode followed by a 12 bp UMI:

```bash
run-trust4 -f hg38_bcrtcr.fa --ref human_IMGT+C.fa \
  -u sample_R2.fastq.gz \
  --barcode sample_R1.fastq.gz --UMI sample_R1.fastq.gz \
  --readFormat bc:0:15,um:16:27
```

Note that in 10x Genomics data, the UMI **plus the cell barcode** is the real unique molecular identifier, so `--UMI` is used together with `--barcode`. An `um:` field in `--readFormat` without `--UMI` is ignored.

The [`--readFormat` specification](/guides/single-cell/#the-readformat-specification), including extraction from the FASTQ header, is described on the single-cell page.

### What --UMI changes

- UMIs are used **only for quantification**, after assembly, and for choosing each cell's representative chains. They do not affect the assembly: reads are not deduplicated before assembling, and each UMI counts once at quantification.
- In `_barcode_report.tsv` and `_barcode_airr.tsv`, the count becomes the number of UMIs for that chain in that cell. TRUST4 does not report both reads and UMIs; to get read counts as well, rerun without `--UMI` under another `-o` and join on the contig ID.
- For 10x gene-expression data, using UMIs or read counts "yields very similar results"; the effect is tiny. Add the UMI if you are concerned about PCR artifacts. It matters much more for amplified libraries.

## UMIs as molecule barcodes

In other platforms, the UMI can be really unique and be regarded as a molecule barcode. Run TRUST4 with the UMI file as the barcode and `--barcodeLevel molecule`:

```bash
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -1 sample_1.fq.gz -2 sample_2.fq.gz \
  --barcode UMIfile --barcodeLevel molecule
```

TRUST4 then assembles the reads of each molecule together. In this output, the represented chain information is in the chain1 column of the `_barcode_report.tsv` file, and the deduplicated, molecule-level abundance of each CDR3 is in `_report.tsv` and `_airr.tsv`.

`--barcodeLevel` accepts `cell` (the default) or `molecule`.

### Example: a UMI at the start of read 2

For a kit whose UMI is in the first 12 bp of read 2, followed by a fixed sequence before the cDNA (the SMARTer Human TCR v2 layout discussed on the issue tracker), the maintainer's suggestion was:

```bash
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -1 sample_R1.fq.gz -2 sample_R2.fq.gz \
  --barcode sample_R2.fq.gz --barcodeLevel molecule \
  --readFormat bc:0:11,r2:19:-1
```

If read 1 starts with primer sequence, skip it with an `r1:` field as well (for example `r1:70:-1`). The positions depend on the exact kit and read length, so check them against your reads. When the UMI is split between read 1 and read 2, concatenate the two halves into one file and pass that as `--barcode`.

You do not need to trim adapters first: TRUST4 trims adapters internally by detecting read-through.

`--repseq` is for bulk TCR-seq/BCR-seq **without** UMIs; do not combine it with the molecule-barcode mode.

### Speeding up UMI-based repseq

- `--skipReadRealign` skips realigning reads in the annotator, which reduces the computation cost of barcode/UMI-based repseq.
- `--contigMinCov INT` ignores contigs with bases covered by fewer than INT reads. In molecule mode it therefore filters out UMIs with fewer than INT reads; `--contigMinCov 3` was the maintainer's suggestion.

### Filtering low-support UMIs

UMI counts are much lower than read counts, and TRUST4 has no default minimum number of reads per UMI. To drop UMIs supported by few reads, regenerate the report from the barcode report with a read-count cutoff:

```bash
perl trust-simplerep.pl PREFIX_cdr3.out --barcodeCnt \
  --filterBarcoderep PREFIX_barcode_report.tsv --filterBarcoderepReadCnt 2 \
  > PREFIX_report_filtered.tsv
```

Out-of-frame CDR3s are kept in the barcode report in molecule mode.

To get the reads that were not assembled, remove the read IDs in `PREFIX_assembled_reads.fa` from the input.
