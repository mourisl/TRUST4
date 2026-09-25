---
title: Pipeline stages and files
description: What each stage of run-trust4 runs, the files it reads and writes, and how to restart from a stage or clean up.
---

`run-trust4` is a Perl driver that runs four stages in order. Each stage is an ordinary program or script, and `run-trust4` logs every command it runs to standard error as a `SYSTEM CALL` line, so you can see exactly what happened.

Below, `PREFIX` is the output prefix: the value of `-o` (default `TRUST_<input name>`), placed inside `--od` if that is given.

## Stage overview

| Stage | `--stage` value | Program | Reads | Writes |
|-------|-----------------|---------|-------|--------|
| Candidate read extraction | `0` | `bam-extractor` or `fastq-extractor` | Input reads, `-f` | `PREFIX_toassemble*.fq`, barcode/UMI files |
| Assembly | `1` | `trust4` | `PREFIX_toassemble*` | `PREFIX_raw.out`, `PREFIX_final.out`, `PREFIX_assembled_reads.fa` |
| Annotation | `2` | `annotator` | `PREFIX_final.out`, `--ref` (or `-f`) | `PREFIX_annot.fa`, `PREFIX_cdr3.out`, `PREFIX_airr_align.tsv` |
| Reporting | `3` | `trust-simplerep.pl`, `trust-barcoderep.pl`, `trust-airr.pl` | `PREFIX_cdr3.out`, `PREFIX_annot.fa` | `PREFIX_report.tsv`, `PREFIX_airr.tsv`, barcode reports |

`--stage N` starts TRUST4 on stage N and runs every later stage. The earlier stages' output files must already exist with the same prefix.

## Candidate read extraction

Stage 0 keeps only the reads likely to come from V, D, J or C genes, using the `-f` file:

- With `-b`, `bam-extractor` uses the gene coordinates in `-f` to pull candidate reads out of the alignment.
- With `-1`/`-2` or `-u`, `fastq-extractor` screens the reads against the gene sequences in `-f`, and applies `--barcode`, `--UMI`, `--readFormat`, `--barcodeWhitelist` and `--barcodeTranslate`.

The output is `PREFIX_toassemble_1.fq` and `PREFIX_toassemble_2.fq` for paired-end data, or `PREFIX_toassemble.fq` for single-end data, plus `PREFIX_toassemble_bc.fa` and `PREFIX_toassemble_umi.fa` when barcodes or UMIs are given.

With `--noExtraction`, this stage is skipped and the `-1 -2`/`-u` files go straight to assembly.

## Assembly

Stage 1 runs `trust4` on the candidate reads. It assembles them with the `-f` file, or with the `--ref` file when `--assembleWithRef` is set. The options `-k`, `--minHitLen`, `--contigMinCov`, `--skipMateExtension`, `--cgeneEnd` and `--repseq` take effect here.

## Annotation

Stage 2 runs `annotator` on `PREFIX_final.out` against the `--ref` file:

- By default the annotator also realigns the assembled reads, `PREFIX_assembled_reads.fa`, to the annotated contigs. `--skipReadRealign` turns this off, which is useful for reducing the computation cost of barcode/UMI-based repseq.
- If no `--ref` is given but the `-f` file is in IMGT format, `run-trust4` prints a warning and uses `-f` as the reference. Otherwise the annotator runs on `-f` with `--notIMGT`.
- `--imgtAdditionalGap` and `--outputReadAssignment` are passed to this stage.

## Reporting

Stage 3 turns the annotation into reports.

Without barcodes:

```bash
perl trust-simplerep.pl PREFIX_cdr3.out > PREFIX_report.tsv
perl trust-airr.pl PREFIX_report.tsv PREFIX_annot.fa --airr-align PREFIX_airr_align.tsv > PREFIX_airr.tsv
```

With barcodes, `trust-barcoderep.pl` first builds `PREFIX_barcode_report.tsv`. `trust-simplerep.pl` then counts barcodes rather than reads and only summarizes the primary CDR3s of that barcode report, and `trust-airr.pl` writes both `PREFIX_airr.tsv` and `PREFIX_barcode_airr.tsv`.

## Restarting from a stage

Because every stage's input is a file on disk, a run can be resumed or partly redone. For example, to regenerate the reports after editing `PREFIX_cdr3.out`, or after updating TRUST4's reporting scripts:

```bash
run-trust4 -b sample.bam -f hg38_bcrtcr.fa --ref human_IMGT+C.fa -o TRUST_sample --stage 3
```

Use the same `-o`, `--od`, input and barcode options as the original run so that the file names line up. Points to watch:

- **Intermediate files are found by prefix.** They are not passed as inputs; `run-trust4` looks for `PREFIX_*` files in the output directory.
- **An existing file does not mean its stage finished.** A killed run can leave a partial file. Restart from the stage that was writing the last file, not the stage after it.
- **Keep `--barcode` and `--UMI`.** Leaving them out of a restarted single-cell run makes TRUST4 treat it as bulk data: all reads are assembled together, which is very slow, and the barcode reports are not made.
- **Old runs can be updated.** For example, rerunning `--stage 2` with a current version fills in the AIRR alignment columns that versions before mid-2022 left empty.

### Editing the candidate reads

You can also edit or combine the `PREFIX_toassemble*` files — for example to merge two sequencing runs of a sample, or to split a large sample by barcode — and run from assembly:

```bash
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa -o TRUST_sample --stage 1 \
  -1 sample_1.fq.gz -2 sample_2.fq.gz
```

The original read options are still required and checked for existence, even though extraction is skipped. Alternatively, give the edited files directly as input with `--noExtraction`, which also accepts gzipped files:

```bash
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa -o TRUST_sample_part1 \
  -1 part1_toassemble_1.fq.gz -2 part1_toassemble_2.fq.gz \
  --barcode part1_toassemble_bc.fa --noExtraction
```

With `--noExtraction`, each of `-1`, `-2`, `-u`, `--barcode` and `--UMI` takes one file.

## Cleaning up

| `--clean` | Files removed at the end of the run |
|-----------|-------------------------------------|
| `0` | None (default). |
| `1` | Intermediate files: `PREFIX_toassemble_*`, `PREFIX_assembled_reads.fa`, `PREFIX_final.out`, `PREFIX_raw.out`, `PREFIX_airr_align.tsv`. |
| `2` | Everything from `1`, plus `PREFIX_annot.fa`, `PREFIX_report.tsv`, `PREFIX_cdr3.out` and `PREFIX_barcode_report.tsv`, leaving only the AIRR files. |

:::caution Cleaning prevents restarting
Once intermediate files are removed, `--stage` cannot restart from the stages that needed them. Leave `--clean` at `0` while you are still tuning a run.
:::
