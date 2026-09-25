---
title: Annotating sequences
description: Use TRUST4's annotator on its own to assign V, D, J, C genes and CDRs to any sequences, like IgBLAST or IMGT/V-QUEST.
---

You can use the `annotator` from TRUST4 to annotate the V, D, J, C genes and CDRs for any given sequences, just like using IgBLAST or IMGT/V-QUEST. No assembly is involved: the annotator reads your sequences and aligns them to the reference.

## Annotate a FASTA file in AIRR format

To obtain the annotation in AIRR format for human sequences with eight threads:

```bash
./annotator -f human_IMGT+C.fa -a input.fa --fasta -t 8 \
  --needReverseComplement --noImpute --outputFormat 1 > annotation.tsv
```

| Option | Why it is there |
|--------|-----------------|
| `-f human_IMGT+C.fa` | The IMGT reference for the species. |
| `-a input.fa` | The sequences to annotate. |
| `--fasta` | The input is plain FASTA, not a TRUST4 assembly file. |
| `-t 8` | Eight threads. |
| `--needReverseComplement` | Reverse complement sequences on another strand, so input in either orientation is annotated. |
| `--noImpute` | Do not impute CDR3 sequence for TCR. |
| `--outputFormat 1` | Write AIRR format instead of the default annotated FASTA. |

The result is written to standard output.

:::caution Spelling of --needReverseComplement
The annotator only recognises `--needReverseComplement`. A misspelling such as `--needReveserComplement` is an unrecognised option, and the annotator prints its usage message and exits.
:::

## Annotated FASTA output

Without `--outputFormat 1`, the annotator writes annotated FASTA in the layout of TRUST4's `_annot.fa` file — a header of `consensus_id consensus_length average_coverage annotations` followed by the sequence. See [output formats](/reference/output-formats/#trust-annot-fa).

```bash
./annotator -f human_IMGT+C.fa -a input.fa --fasta -t 8 --needReverseComplement > annotation.fa
```

For FASTQ input use `--fastq` instead of `--fasta`.

## Long reads

For long reads, such as PacBio HiFi, assembly adds little: each read already spans the V(D)J region. The maintainer's recipe annotates the candidate reads directly:

```bash
./fastq-extractor -t 8 -f human_IMGT+C.fa -o tmp -u long_reads.fq.gz
./annotator -f human_IMGT+C.fa -a tmp.fq --fastq -t 8 -o tmp \
  --outputCDR3File --needReverseComplement > tmp_annot.fa
perl trust-simplerep.pl tmp_cdr3.out > long_reads_report.tsv
```

`--outputCDR3File` makes the annotator write `tmp_cdr3.out` without reads for realignment, which `trust-simplerep.pl` then summarizes.

Long reads are supported but not optimized: indel errors are not modeled, so Nanopore reads should be error-corrected first. `--repseq` has not been tested on long reads. Very long reads (more than about 10 kb) caused crashes in older versions; trim or filter them if you hit one. For running the full pipeline on long reads, `--minHitLen 53` was suggested for PacBio HiFi, but has not been tested.

## Non-IMGT references

The `-f` file is expected to be in IMGT format. If it is not — for example a `*_bcrtcr.fa` file — add `--notIMGT`. This is what `run-trust4` does itself when no `--ref` file is given.

## All annotator options

The full list is on the [command-line interface](/reference/cli/#annotator) page.
