---
title: Citation and support
description: How to cite TRUST4, where the source lives, and how to get help.
---

## Citing TRUST4

If TRUST4 contributes to work you publish, please cite the Nature Methods paper:

> Song, L., Cohen, D., Ouyang, Z. et al. **TRUST4: immune repertoire reconstruction from bulk and single-cell RNA-seq data.** *Nature Methods* (2021). doi:[10.1038/s41592-021-01142-2](https://doi.org/10.1038/s41592-021-01142-2)

The evaluation instructions and scripts used in the TRUST4 manuscript are available at [github.com/liulab-dfci/TRUST4_manuscript_evaluation](https://github.com/liulab-dfci/TRUST4_manuscript_evaluation).

If you use the IMGT-derived reference files, please also acknowledge [IMGT](https://www.imgt.org/), the source of the gene sequences.

## Support

Questions, bug reports and feature requests belong on the issue tracker:

**[github.com/liulab-dfci/TRUST4/issues](https://github.com/liulab-dfci/TRUST4/issues)**

We will typically respond within a day or two, but it could take longer, e.g. a month, for fixing bugs and adding features.

A report is much easier to act on when it includes:

- the TRUST4 version, printed on the first line of the log (`TRUST4 v... begins.`), and how it was installed (Bioconda, Docker or `make` from source);
- the exact command you ran;
- which `-f` and `--ref` files you used;
- the last lines of the standard error output, including the final `SYSTEM CALL` line;
- the type of data: bulk or single-cell, platform, read length, BAM or FASTQ.

The [FAQ](/reference/faq/) and [troubleshooting](/reference/troubleshooting/) pages cover common questions and errors, and are worth a look before filing.

## Source code and licence

TRUST4 is developed in the open at [github.com/liulab-dfci/TRUST4](https://github.com/liulab-dfci/TRUST4).

Copyright © 2018–present, Li Song, X. Shirley Liu. TRUST4 includes portions copyright from samtools — Copyright © 2008–, Genome Research Ltd, Heng Li. The licence terms are in the `LICENSE.txt` file of the repository.

## Related resources

- [Bioconda](https://anaconda.org/bioconda/trust4) — the packaged distribution.
- [BioContainers](https://quay.io/repository/biocontainers/trust4?tab=tags) — Docker images.
- [AIRR Community rearrangement schema](https://docs.airr-community.org/en/latest/datarep/rearrangements.html) — the format of the `_airr.tsv` output.
- [TCRMatch](https://github.com/IEDB/TCRMatch) — epitope prediction that accepts TRUST4 reports.

## About this documentation

This site is generated from Markdown sources in the `website/content` directory. Every page carries an **Edit this page on GitHub** link at the bottom; corrections and additions are welcome through the same repository as the code.
