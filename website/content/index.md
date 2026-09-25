---
title: TRUST4 — TCR and BCR repertoire reconstruction from RNA-seq data
description: Reconstruct T-cell and B-cell receptor repertoires, including CDR3, from unselected bulk or single-cell RNA sequencing data.
---

<section class="hero"><div class="hero-inner"><h1>TRUST4</h1><p class="tagline">Reconstruct <strong>T-cell and B-cell receptor</strong> repertoires — V, D, J, C genes and CDR1, 2, 3 — directly from unselected bulk or single-cell RNA-seq data.</p><div class="cta"><a class="btn primary" href="getting-started/introduction/">Get started</a><a class="btn ghost" href="getting-started/quick-start/">Run the example</a><a class="btn ghost" href="https://github.com/liulab-dfci/TRUST4" target="_blank" rel="noopener">View on GitHub</a></div></div></section>

<section class="home-section">

## What is TRUST4?

TRUST4 is a computational tool to analyze TCR and BCR sequences using unselected RNA sequencing data, profiled from fluid and solid tissues, including tumors. TRUST4 performs **de novo assembly** on V, J, C genes including the hypervariable complementarity-determining region 3 (CDR3) and reports consensus contigs of BCR/TCR sequences. It then realigns the contigs to IMGT reference gene sequences to identify the corresponding gene and CDR3 details.

TRUST4 supports both single-end and paired-end, bulk or single-cell sequencing data with any read length.

<div class="stats"><div class="stat"><div class="value">7</div><div class="label">Receptor chains: IGH, IGK, IGL, TRA, TRB, TRG, TRD</div></div><div class="stat"><div class="value">Any</div><div class="label">Read length, single-end or paired-end</div></div><div class="stat"><div class="value">BAM or FASTQ</div><div class="label">Aligned reads or raw sequencing files</div></div><div class="stat"><div class="value">AIRR</div><div class="label">Standard rearrangement output format</div></div></div>

</section>

<section class="home-section">

## Why TRUST4

<div class="cards"><div class="card"><h3>No enrichment needed</h3><p>TRUST4 works on ordinary RNA-seq, so repertoires can be recovered from existing bulk expression data of tissues and tumors, not only from targeted TCR-seq or BCR-seq.</p></div><div class="card"><h3>De novo assembly</h3><p>Candidate reads are assembled into consensus contigs of the V, J and C genes, including the hypervariable CDR3, and then annotated against IMGT reference genes.</p></div><div class="card"><h3>10x Genomics and barcoded data</h3><p>With cell barcodes, reads are assembled per cell and each cell gets a representative chain pair. Barcode whitelists, translation and combinatorial barcoding are supported.</p></div><div class="card"><h3>UMI-aware abundance</h3><p>For 10x Genomics-like data, TRUST4 supports UMI-based abundance estimation, or treating a UMI as a molecule barcode.</p></div><div class="card"><h3>SMART-seq and annotation only</h3><p>A wrapper processes plate-based SMART-seq cells, and the <code>annotator</code> can annotate any given sequences, like IgBLAST or IMGT/V-QUEST.</p></div><div class="card"><h3>Reports other tools understand</h3><p>The report table is compatible with repertoire tools such as VDJTools, and the AIRR output follows the AIRR Community rearrangement schema.</p></div></div>

</section>

<section class="quickstart"><div class="home-section">

## Quick start

Install from Bioconda and run on your alignment file, using the human reference files from the [TRUST4 repository](https://github.com/liulab-dfci/TRUST4):

```bash
conda install -c bioconda trust4

run-trust4 -b sample.bam -f hg38_bcrtcr.fa --ref human_IMGT+C.fa -t 8
```

Or run on raw paired-end FASTQ files, in which case the IMGT file can be used for `-f` as well:

```bash
run-trust4 -f human_IMGT+C.fa --ref human_IMGT+C.fa \
  -1 sample_1.fq.gz -2 sample_2.fq.gz -o TRUST_sample -t 8
```

The full walkthrough, including the bundled example data, is in the [quick start guide](/getting-started/quick-start/).

</div></section>

<section class="home-section">

## Citation

<div class="citation"><p>Song, L., Cohen, D., Ouyang, Z. et al. <strong>TRUST4: immune repertoire reconstruction from bulk and single-cell RNA-seq data.</strong> <em>Nature Methods</em> (2021).</p><p><a href="https://doi.org/10.1038/s41592-021-01142-2" target="_blank" rel="noopener">doi:10.1038/s41592-021-01142-2</a></p></div>

TRUST4 is copyright &copy; 2018–present, Li Song, X. Shirley Liu. Questions and bug reports are welcome on the [GitHub issue tracker](https://github.com/liulab-dfci/TRUST4/issues).

</section>
