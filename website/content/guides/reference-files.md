---
title: Reference gene files
description: The -f and --ref files, the ones bundled with TRUST4, and how to build them for another species or genome.
---

Every TRUST4 run takes two reference files describing the V, D, J and C genes:

| Option | Role | Bundled human file | Bundled mouse file |
|--------|------|--------------------|--------------------|
| `-f` | The genomic sequence and coordinate of V, J, C genes. Used to extract candidate reads and to assemble. | `hg38_bcrtcr.fa`, `hg19_bcrtcr.fa` | `mouse/GRCm38_bcrtcr.fa`, `mouse/GRCm39_bcrtcr.fa` |
| `--ref` | Detailed V/D/J/C gene reference from the IMGT database. Used to annotate genes and CDRs. Optional but recommended. | `human_IMGT+C.fa` | `mouse/mouse_IMGT+C.fa` |

## Which -f file to use

It depends on the input:

- **BAM input (`-b`)**: the `-f` file must contain the genomic coordinates of the V, D, J and C genes, which are crucial for extracting candidate reads in alignment BAM files. Use the `*_bcrtcr.fa` file that matches the genome your reads were aligned to.
- **FASTQ/FASTA input (`-1`/`-2` or `-u`)**: the coordinate information is not needed, so you can directly use the IMGT sequence file for `-f` as well. This is useful when analyzing species without reference genomes or genome annotations.

```bash
# FASTQ input: the IMGT file serves as both -f and --ref
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa -1 r1.fq.gz -2 r2.fq.gz
```

:::note --ref is added for you when -f is an IMGT file
If you omit `--ref` and the `-f` file is in IMGT format (its sequences contain IMGT gap characters, `.`), `run-trust4` prints a warning and uses the `-f` file as `--ref` automatically.
:::

`--assembleWithRef` makes TRUST4 conduct the assembly with the `--ref` file instead of the `-f` file. It requires `--ref`.

With FASTQ input, `hg38_bcrtcr.fa` still works as `-f`: it is faster than the IMGT file, results are almost identical, and contigs can be slightly longer because the genomic sequences include UTRs.

## Requirements for the --ref file

`--ref` must be an **IMGT-format, gapped** reference, such as the output of `BuildImgtAnnot.pl`. The `.` gap characters in the V genes are what TRUST4 uses to locate CDR1 and CDR2.

- A genome FASTA, a `*_bcrtcr.fa` file or an ungapped sequence file given to `--ref` has caused annotator crashes (segmentation faults, `bad_array_new_length`). Give ungapped sequences to `-f` only.
- If the gaps are missing or do not match for a chain, the log shows `WARNING: IMGT may introduce additional gaps in ...` and CDR1 and CDR2 are not annotated for that chain; the CDR3 is then found from motifs.
- Sequences must be upper case. Lower-case reference sequences gave empty results; `BuildImgtAnnot.pl` upper-cases its output.
- Gene names must start with the standard locus and segment prefix, such as `TRBV`, `TRAJ` or `IGHC`, which is how TRUST4 recognises the chain and the gene type.

The bundled `human_IMGT+C.fa` has constant genes taken from GENCODE, which are longer than IMGT's and anchor the assembly better. IMGT does not need to be refreshed often. `hg38_bcrtcr.fa` is built from GENCODE rather than IMGT, which may matter for IMGT's licence terms.

### Custom and extra genes

You can add sequences — for example pseudogenes, or alleles missing from IMGT — to an IMGT+C file, using the same naming. For a sequence you cannot gap, remove its `.` characters; the CDR3 is then found from motifs. Without any `--ref`, CDR3s are also found from motifs, and CDR1 and CDR2 are not annotated.

To analyse only TCRs or only BCRs, you can subset the reference, e.g. `grep -A 1 --no-group-separator ">T" human_IMGT+C.fa > human_TR_IMGT+C.fa`.

## Build the --ref file from IMGT

Normally, the file specified by `--ref` is downloaded from the IMGT website. For example, for human, you can use the command:

```bash
perl BuildImgtAnnot.pl Homo_sapiens > IMGT+C.fa
```

The script downloads the IMGT/GENE-DB reference sequences with `wget` and keeps the entries whose species matches the name you give, with spaces written as underscores. The available species names can be found on the [IMGT FTP](http://www.imgt.org//download/V-QUEST/IMGT_V-QUEST_reference_directory/). For example, `Macaca_mulatta` for rhesus macaque.

## Build the -f file for BAM input

If your input data for TRUST4 is raw FASTQ files, you can use the IMGT file for the `-f` option. If your input data is BAM files, you need to generate another file for `-f`. To do that, you need:

- the reference genome of the species you are interested in (e.g. hg38 for human, or mm10 for mouse), and
- the corresponding genome annotation GTF file (e.g. GENCODE v35 for human, or GENCODE vM21 for mouse).

The GTF must have `gene_name` (and `transcript_name`) attributes, which GENCODE files have and some Ensembl files lack. If the GTF is missing some receptor genes, as happened with a dog GTF lacking TRBC and TRAJ genes, use the IMGT file with FASTQ input instead.

Then run:

```bash
perl BuildDatabaseFa.pl reference.fa annotation.gtf bcr_tcr_gene_name.txt > bcrtcr.fa
```

The gene-name list `bcr_tcr_gene_name.txt` is provided for human as `human_vdjc.list` in the repository. For another species, the `IMGT+C.fa` file can be used to generate it:

```bash
grep ">" IMGT+C.fa | cut -f2 -d'>' | cut -f1 -d'*' | sort | uniq > bcr_tcr_gene_name.txt
```

## A complete recipe for a new species

Putting the steps together, for a species with a reference genome and GTF:

```bash
perl BuildImgtAnnot.pl Mus_musculus > IMGT+C.fa
grep ">" IMGT+C.fa | cut -f2 -d'>' | cut -f1 -d'*' | sort | uniq > bcr_tcr_gene_name.txt
perl BuildDatabaseFa.pl genome.fa annotation.gtf bcr_tcr_gene_name.txt > bcrtcr.fa

run-trust4 -b sample.bam -f bcrtcr.fa --ref IMGT+C.fa
```

For a species without a reference genome or annotation, stop after the first command and run TRUST4 on FASTQ files with `-f IMGT+C.fa --ref IMGT+C.fa`.

## Mouse TRAV genes and IMGT gaps

IMGT has additional gaps on the TRAV genes, so the conventional coordinate system does not apply. If you need the CDR1 and CDR2 information on the TRAV genes, please add the option `--imgtAdditionalGap TRAV:7,83`. The usage text gives this value as the example for mouse:

```bash
run-trust4 -b sample.bam -f mouse/GRCm38_bcrtcr.fa --ref mouse/mouse_IMGT+C.fa \
  --imgtAdditionalGap TRAV:7,83
```

The positions are 0-based codon positions of the additional gaps.
