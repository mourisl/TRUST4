---
title: Troubleshooting
description: Error messages and exit codes, what causes them, and how to fix them; plus memory, speed and version advice.
---

`run-trust4` logs every command it runs to standard error as a `SYSTEM CALL` line, and a failure is reported as `system ... failed: N`. The last `SYSTEM CALL` line tells you which stage failed, and the lines just above it usually tell you why.

## Error messages

| Message | Cause | Fix |
|---------|-------|-----|
| `Unknown parameter --...` | `run-trust4` does not know the option. Often `--read-format` (the option is `--readFormat`) or an option copied from an old version. | Check the spelling on the [CLI page](/reference/cli/). |
| `trust4: unrecognized option '--readFormat'` (or `--ref`) | The `trust4` assembler binary was called directly. | Run `run-trust4`, which drives the whole pipeline. |
| `Can't exec .../fastq-extractor: No such file or directory` | TRUST4 was not compiled. | Run `make` in the TRUST4 directory. |
| `Could not find file ...` | An input or reference path is wrong, or a wildcard matches nothing. | Check the path. |
| `Unknown genome name: X` | With BAM input, the chromosome names in the `-f` file do not match the BAM — for example `chr7` vs `7`, a Cell Ranger reference with `GRCh38_` prefixes, a BAM aligned to a transcriptome, or an IMGT file given as `-f`. | See [BAM input problems](/guides/bulk-rna-seq/#bam-input-problems). |
| `Two reads from the unaligned fragment are not showing up together. Please use -u(--abnormalUnmapFlag from wrapper) option.` | The mates of an unmapped pair are not adjacent in the BAM, or their IDs differ by a suffix. | Add `--abnormalUnmapFlag`; if the read IDs end in `/1`, `/2` or `.1`, `.2`, add `--mateIdSuffixLen 2`. |
| `The two mate-pair read files have different number of reads.` | `-1` and `-2` files do not match: one is truncated, or `-2` is missing. | Check the files with `gzip -t` and compare read counts. If read 1 holds only a barcode/UMI, use `-u` for read 2 instead. |
| `Read file and barcode file  have different number of reads.` (or `UMI file`) | The `--barcode`/`--UMI` file does not match the read file, or `-2` was left out. | As above. |
| `Barcode X does not exist in the translation table.` | A (whitelist-corrected) barcode is missing from the `--barcodeTranslate` file. | Make the translation table cover every barcode, including all combinations for combinatorial barcoding. |
| `Need to use -a to specify the assembly file.` | No VDJ reads were assembled, so the annotator got nothing. | Check that `_toassemble*` files are not empty. Causes: no immune reads (3′ data, low infiltration, shallow depth), a wrong `-f` file, or barcode options that drop every read. |
| `WARNING: ... is of IMGT format, automatically add --ref option.` | `-f` is an IMGT file and no `--ref` was given. | Informational. |
| `WARNING: IMGT may introduce additional gaps in ...` | The `--ref` file is not IMGT-gapped for that chain, or the gaps do not match `--imgtAdditionalGap`. CDR1 and CDR2 are not annotated for that chain. | Use a gapped IMGT reference from `BuildImgtAnnot.pl`; for mouse TRAV add `--imgtAdditionalGap TRAV:7,83`. |

## Exit codes

| Exit code in `failed: N` | Meaning |
|--------------------------|---------|
| `9` | The system killed the process, almost always for running out of memory. |
| `11`, `139`, `35584` | Segmentation fault. |
| `256` and other values | The program exited with an error; read the message above the `SYSTEM CALL` line. |

For a segmentation fault:

1. Update TRUST4. Several crashes in older versions have been fixed; v1.1.6 in particular had segfault bugs fixed in v1.1.6.1.
2. Check that `--ref` is an IMGT-gapped file from `BuildImgtAnnot.pl`, not a genome FASTA or an ungapped file. Wrong `--ref` files have caused annotator crashes and `bad_array_new_length` errors.
3. Check for FASTQ problems: quality strings of a different length from the sequence, or FASTA-style `>` headers in a FASTQ file.
4. Check for very long reads (more than about 10 kb), which older versions could not annotate.
5. If it still fails, rerun from the failing stage with `--stage` and attach the command, the log and, if possible, the `_toassemble*` or `_final.out` file to a [GitHub issue](https://github.com/liulab-dfci/TRUST4/issues).

## Memory

Memory depends on the number of **candidate reads** — the reads in `_toassemble*` — not on the input file size. As a guide, the maintainer suggested 16–20 GB for about 8 million candidate reads. Runs with very many candidate reads, such as amplified libraries or many hashed samples in one run, can need much more.

When a run is killed for memory:

- Rerun from assembly with `--stage 1` so extraction is not repeated.
- For amplified or TCR/BCR-seq data without UMIs, add `--repseq`.
- For barcoded data, make sure the barcode options are right. If `--barcode` is missing, the run is treated as bulk and all reads are assembled together, which is slow and memory-hungry. If `r1:` is missing when read 1 carries barcode and UMI bases, those bases are assembled too.

## Speed

- **Threads help extraction and annotation, not assembly.** The assembly step is essentially sequential. The maintainer usually uses 8 threads; the gain plateaus after about 16.
- **BAM input is faster than FASTQ**, because reads outside the V, D, J and C genes are skipped. Results are almost identical.
- **Barcoded data is assembled per barcode**, which is much faster than bulk assembly of the same reads (substantially faster since v1.1.0).
- **Amplified libraries are slow.** A 10x VDJ-kit run taking about 4 hours where Cell Ranger takes 30 minutes is normal, because TRUST4 is designed for unamplified data.
- **Reading and counting k-mers** should take about 1–3 seconds per 100K candidate reads. Much slower usually means a busy or slow node.

To parallelize a large barcoded sample, run extraction once, split the `_toassemble*` files by barcode, and run `run-trust4 --noExtraction` on each part.

## Versions and installation

- Check the version on the first log line: `TRUST4 v1.1.11-r641 begins.` A log line without a version number means a very old installation.
- Bioconda packages have at times lagged behind the GitHub source, and old conda versions (such as 1.0.5) lack `--readFormat` and have since-fixed bugs. If a conda install misbehaves, pin a recent version or build from source.
- `version GLIBCXX_... not found` means the binaries were built in a different environment from the one they run in. Rebuild with `make clean; make` in the environment you run TRUST4 from.
- On Windows (for example under Cygwin), the compiled programs get an `.exe` suffix that `run-trust4` does not expect. Remove the suffix or edit `run-trust4`.
- There is no GPU support.

## Empty results

If `_report.tsv` has only a header:

- For single-cell data, check `_toassemble_bc.fa`. Many `missing_barcode` lines mean the barcode is not being extracted or does not match the whitelist; see the [single-cell guide](/guides/single-cell/#checking-barcode-extraction).
- If the candidate reads fall only in C genes, no reads covered the V(D)J region. This is expected for many 3′ data sets.
- If no contig had a complete CDR3, the report and AIRR files are empty although `_cdr3.out` may have partial CDR3s.
