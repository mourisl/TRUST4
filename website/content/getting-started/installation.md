---
title: Installation
description: Install TRUST4 from Bioconda, use the Docker container, or build it from source with make.
---

TRUST4 is distributed as source code that compiles with a single `make`, as a Bioconda package, and as a BioContainers Docker image.

## Requirements

- A Linux or macOS system with a C++ compiler and `make`.
- [pthreads](http://en.wikipedia.org/wiki/POSIX_Threads), which TRUST4 depends on.
- [zlib](http://en.wikipedia.org/wiki/Zlib), which the bundled samtools depends on.
- Perl, which runs the `run-trust4` driver and the reporting scripts.

For macOS, TRUST4 has been successfully compiled with gcc_darwin17.7.0 and gcc_9.2.0 installed by Homebrew.

## Build from source

### 1. Clone the repository

Clone the [GitHub repo](https://github.com/liulab-dfci/TRUST4):

```bash
git clone https://github.com/liulab-dfci/TRUST4.git
cd TRUST4
```

### 2. Run make

```bash
make
```

This builds four executables in the repository directory — `trust4`, `bam-extractor`, `fastq-extractor` and `annotator` — and compiles the bundled `samtools-0.1.19` library that `bam-extractor` needs. The driver script `run-trust4` calls them from the directory it lives in.

### 3. Put TRUST4 on your PATH (optional)

If you want to run TRUST4 without specifying the directory, either add the directory of TRUST4 to the environment variable `PATH`:

```bash
export PATH="/path/to/TRUST4:$PATH"
```

or create a soft link of the file `run-trust4` to a directory in `PATH`:

```bash
ln -s /path/to/TRUST4/run-trust4 ~/bin/run-trust4
```

:::note A symbolic link is enough
`run-trust4` resolves its own real location before calling `trust4`, `annotator` and the other programs, so linking just the driver script works: the other executables stay in the repository directory.
:::

## Install with Conda

TRUST4 is also available from [Bioconda](https://anaconda.org/bioconda/trust4):

```bash
conda install -c bioconda trust4
```

## Use the Docker container

A container is published by BioContainers:

```bash
docker pull quay.io/biocontainers/trust4:<tag>
```

See [trust4/tags](https://quay.io/repository/biocontainers/trust4?tab=tags) for valid values for `<tag>`.

:::tip Getting the reference files
The reference files used on this site — `hg38_bcrtcr.fa`, `human_IMGT+C.fa`, the `mouse/` files and the `example/` data — live in the [GitHub repository](https://github.com/liulab-dfci/TRUST4). If you installed TRUST4 through Conda or Docker, clone or download the repository to get them.
:::

## Verify the installation

Running `run-trust4` without arguments prints its version and usage message:

```bash
run-trust4
```

For an end-to-end check, run the bundled test from the TRUST4 source folder:

```bash
bash trust-example-test.sh
```

It runs TRUST4 on `example/example.bam` and compares the report with the pre-generated one. It prints `TRUST4 is ready to use.` on success. The [quick start](/getting-started/quick-start/) walks through the example in more detail.

## Keeping up to date

Source installations update by pulling and rebuilding:

```bash
cd /path/to/TRUST4
git pull
make
```

Conda installations update in the usual way:

```bash
conda update -c bioconda trust4
```

## Next steps

- [Quick start](/getting-started/quick-start/) — your first run on the example data.
- [Command-line interface](/reference/cli/) — every option of `run-trust4`.
