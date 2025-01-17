# Dorado

Dorado is a high-performance, easy-to-use, open source basecaller for Oxford Nanopore reads.

## Features

* One executable with sensible defaults, automatic hardware detection and configuration.
* Runs on Apple silicon (M1/2 family) and Nvidia GPUs including multi-GPU with linear scaling (see [Platforms](#platforms)).
* [Modified basecalling](#modified-basecalling).
* [Duplex basecalling](#duplex) (watch the following video for an introduction to [Duplex](https://youtu.be/8DVMG7FEBys)).
* Support for aligned read output in SAM/BAM.
* [POD5](https://github.com/nanoporetech/pod5-file-format) support for highest basecalling performance.
* Based on libtorch, the C++ API for pytorch.
* Multiple custom optimisations in CUDA and Metal for maximising inference performance.

If you encounter any problems building or running Dorado, please [report an issue](https://github.com/nanoporetech/dorado/issues).

## Installation

 - [dorado-0.3.4-linux-x64](https://cdn.oxfordnanoportal.com/software/analysis/dorado-0.3.4-linux-x64.tar.gz)
 - [dorado-0.3.4-linux-arm64](https://cdn.oxfordnanoportal.com/software/analysis/dorado-0.3.4-linux-arm64.tar.gz)
 - [dorado-0.3.4-osx-arm64](https://cdn.oxfordnanoportal.com/software/analysis/dorado-0.3.4-osx-arm64.tar.gz)
 - [dorado-0.3.4-win64](https://cdn.oxfordnanoportal.com/software/analysis/dorado-0.3.4-win64.zip)

## Platforms

Dorado is heavily-optimised for Nvidia A100 and H100 GPUs and will deliver maximal performance on systems with these GPUs.

Dorado has been tested extensively and supported on the following systems:

| Platform | GPU/CPU                      |
| -------- |------------------------------|
| Windows  | (G)V100, A100, H100          |
| Apple    | M1, M1 Pro, M1 Max, M1 Ultra |
| Linux    | (G)V100, A100, H100          |

Systems not listed above but which have Nvidia GPUs with ≥8 GB VRAM and architecture from Pascal onwards (except P100/GP100) have not been widely tested but are expected to work. If you encounter problems with running on your system, please [report an issue](https://github.com/nanoporetech/dorado/issues)

AWS Benchmarks on NVIDIA GPUs are available [here](https://aws.amazon.com/blogs/hpc/benchmarking-the-oxford-nanopore-technologies-basecallers-on-aws/).

## Roadmap

Dorado is Oxford Nanopore's recommended basecaller for offline basecalling. We are working on a number of features which we expect to release soon:

1. DNA barcode multiplexing
2. Adapter trimming
3. Python API
4. Statically linked binary

## Performance tips

1. For optimal performance, Dorado requires POD5 file input. Please [convert your .fast5 files](https://github.com/nanoporetech/pod5-file-format) before basecalling.
2. Dorado will automatically detect your GPU's free memory and select an appropriate batch size.
3. Dorado will automatically run in multi-GPU `cuda:all` mode. If you have a hetrogenous collection of GPUs, select the faster GPUs using the `--device` flag (e.g `--device cuda:0,2`). Not doing this will have a detrimental impact on performance.

## Running

The following are helpful commands for getting started with Dorado.
To see all options and their defaults, run `dorado -h` and `dorado <subcommand> -h`.

### Simplex basecalling

To run Dorado basecalling, download a model and point it to POD5 files _(.fast5 files are supported but will not be as performant)_.

```
$ dorado download --model dna_r10.4.1_e8.2_400bps_hac@v4.1.0
$ dorado basecaller dna_r10.4.1_e8.2_400bps_hac@v4.1.0 pod5s/ > calls.bam
```

If basecalling is interrupted, it is possible to resume basecalling from a BAM file. To do so, use the `--resume-from` flag to specify the path to the incomplete BAM file. For example:

```
$ dorado basecaller dna_r10.4.1_e8.2_400bps_hac@v4.1.0 pod5s --resume-from incomplete.bam > calls.bam
```

`calls.bam` will contain all of the reads from `incomplete.bam` plus the new basecalls *(`incomplete.bam` can be discarded after basecalling is complete)*.

**Note: it is important to choose a different filename for the BAM file you are writing to when using `--resume-from`**. If you use the same filename, the interrupted BAM file will lose the existing basecalls and basecalling will restart from the beginning.

### Modified basecalling

To call modifications, add `--modified-bases` to the basecaller command:

```
$ dorado basecaller dna_r10.4.1_e8.2_400bps_hac@v4.1.0 pod5s/ --modified-bases 5mCG_5hmCG > calls.bam
```

Refer to the [modified base models](#modified-base-models) section to see available modifications.

### Duplex

To run Duplex basecalling, run the command:

```
$ dorado duplex dna_r10.4.1_e8.2_400bps_sup@v4.1.0 pod5s/ > duplex.bam
```

This command will output both simplex and duplex reads. The `dx` tag in the output BAM can be used to distinguish between them:
* `dx:i:1` for duplex reads.
* `dx:i:0` for simplex reads which don't have duplex offsprings.
* `dx:i:-1` for simplex reads which have duplex offsprings.

Dorado duplex previously required a separate tool to perform duplex pair detection and read splitting, but this is now integrated into Dorado.

Note that modified basecalling is not yet supported in duplex mode.

### Alignment

Dorado supports aligning existing basecalls or producing aligned output directly.

To align existing basecalls, run:

```
$ dorado aligner <index> <reads> 
```
where `index` is a reference to align to in (FASTQ/FASTA/.mmi) format and `reads` is a file in any HTS format.

To basecall with alignment with duplex or simplex, run with the `--reference` option:

```
$ dorado basecaller <model> <reads> --reference <index>
```

Alignment uses [minimap2](https://github.com/lh3/minimap2) and by default uses the `map-ont` preset. This can be overridden with the `-k` and `-w` options to set kmer and window size respectively.

### Sequencing Summary

The `dorado summary` command outputs a tab-separated file with read level sequencing information from the BAM file generated during basecalling. To create a summary, run:

```
$ dorado summary <bam>
```

Note that summary generation is only available for reads basecalled from POD5 files. Reads basecalled from .fast5 files are not compatible with the summary command.

## Available basecalling models

To download all available Dorado models, run:

```
$ dorado download --model all
```

### **Simplex models:**

v4.2.0 models are recommended for our latest released condition (5 kHz).

* dna_r10.4.1_e8.2_400bps_fast@v4.2.0
* dna_r10.4.1_e8.2_400bps_hac@v4.2.0
* dna_r10.4.1_e8.2_400bps_sup@v4.2.0

The following simplex models are also available (all for 4 kHz data):

* dna_r9.4.1_e8_fast@v3.4
* dna_r9.4.1_e8_hac@v3.3
* dna_r9.4.1_e8_sup@v3.3
* dna_r9.4.1_e8_sup@v3.6
* dna_r10.4.1_e8.2_260bps_fast@v3.5.2
* dna_r10.4.1_e8.2_260bps_hac@v3.5.2
* dna_r10.4.1_e8.2_260bps_sup@v3.5.2
* dna_r10.4.1_e8.2_400bps_fast@v3.5.2
* dna_r10.4.1_e8.2_400bps_hac@v3.5.2
* dna_r10.4.1_e8.2_400bps_sup@v3.5.2
* dna_r10.4.1_e8.2_260bps_fast@v4.0.0
* dna_r10.4.1_e8.2_260bps_hac@v4.0.0
* dna_r10.4.1_e8.2_260bps_sup@v4.0.0
* dna_r10.4.1_e8.2_400bps_fast@v4.0.0
* dna_r10.4.1_e8.2_400bps_hac@v4.0.0
* dna_r10.4.1_e8.2_400bps_sup@v4.0.0
* dna_r10.4.1_e8.2_260bps_fast@v4.1.0
* dna_r10.4.1_e8.2_260bps_hac@v4.1.0
* dna_r10.4.1_e8.2_260bps_sup@v4.1.0
* dna_r10.4.1_e8.2_400bps_fast@v4.1.0
* dna_r10.4.1_e8.2_400bps_hac@v4.1.0
* dna_r10.4.1_e8.2_400bps_sup@v4.1.0

### **RNA models:**

* rna002_70bps_fast@v3
* rna002_70bps_hac@v3
* rna004_130bps_fast@v3
* rna004_130bps_hac@v3
* rna004_130bps_sup@v3

### **Modified base models**

* dna_r9.4.1_e8_fast@v3.4_5mCG@v0.1
* dna_r9.4.1_e8_hac@v3.3_5mCG@v0.1
* dna_r9.4.1_e8_sup@v3.3_5mCG@v0.1
* dna_r9.4.1_e8_fast@v3.4_5mCG_5hmCG@v0
* dna_r9.4.1_e8_hac@v3.3_5mCG_5hmCG@v0
* dna_r9.4.1_e8_sup@v3.3_5mCG_5hmCG@v0
* dna_r10.4.1_e8.2_260bps_fast@v3.5.2_5mCG@v2
* dna_r10.4.1_e8.2_260bps_hac@v3.5.2_5mCG@v2
* dna_r10.4.1_e8.2_260bps_sup@v3.5.2_5mCG@v2
* dna_r10.4.1_e8.2_400bps_fast@v3.5.2_5mCG@v2
* dna_r10.4.1_e8.2_400bps_hac@v3.5.2_5mCG@v2
* dna_r10.4.1_e8.2_400bps_sup@v3.5.2_5mCG@v2
* dna_r10.4.1_e8.2_260bps_fast@v4.0.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_260bps_hac@v4.0.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_260bps_sup@v4.0.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_400bps_fast@v4.0.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_400bps_hac@v4.0.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_400bps_sup@v4.0.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_260bps_fast@v4.1.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_260bps_hac@v4.1.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_260bps_sup@v4.1.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_400bps_fast@v4.1.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_400bps_hac@v4.1.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_400bps_sup@v4.1.0_5mCG_5hmCG@v2
* dna_r10.4.1_e8.2_400bps_fast@v4.2.0_5mCG_5hmCG@v2 (5 kHz)
* dna_r10.4.1_e8.2_400bps_hac@v4.2.0_5mCG_5hmCG@v2 (5 kHz)
* dna_r10.4.1_e8.2_400bps_sup@v4.2.0_5mCG_5hmCG@v2 (5 kHz)
* dna_r10.4.1_e8.2_400bps_sup@v4.2.0_5mC@v2 (5 kHz)
* dna_r10.4.1_e8.2_400bps_sup@v4.2.0_6mA@v2 (5 kHz)

## Decoding Dorado model names

The names of Dorado models are systematically structured, each segment corresponding to a different aspect of the model, which include both chemistry and run settings. Below is a sample model name explained:

`dna_r10.4.1_e8.2_400bps_hac@v4.2.0`

- **Analyte Type (`dna`)**: This denotes the type of analyte being sequenced. For DNA sequencing, it is represented as `dna`. If you are using the Direct RNA Sequencing Kit, this will be `rna`.

- **Pore Type (`r10.4.1`)**: This section corresponds to the type of flow cell used. For instance, FLO-MIN114/FLO-FLG114 is indicated by `r10.4.1`, while FLO-MIN106D/FLO-FLG001 is signified by `r9.4.1`.

- **Chemistry Type (`e.8.2`)**: This represents the chemistry type, which corresponds to the kit used for sequencing. For example, Kit 14 chemistry is denoted by `e.8.2`.

- **Translocation Speed (`400bps`)**: This parameter, selected at the run setup in MinKNOW, refers to the speed of translocation. Prior to starting your run, a prompt will ask if you prefer to run at 260 bps or 400 bps. The former yields more accurate results but provides less data. As of MinKNOW version 23.04, the 260 bps option has been deprecated.

- **Model Type (`hac`)**: This represents the size of the model, where larger models yield more accurate basecalls but take more time. The three types of models are `fast`, `hac`, and `sup`. The `fast` model is the quickest, `sup` is the most accurate, and `hac` provides a balance between speed and accuracy. For most users, the `hac` model is recommended.

- **Model Version Number (`v4.2.0`)**: This denotes the version of the model. Model updates are regularly released, and higher version numbers typically signify greater accuracy.

## Developer quickstart

### Linux dependencies

The following packages are necessary to build Dorado in a barebones environment (e.g. the official ubuntu:jammy docker image).

```
$ apt-get update && apt-get install -y --no-install-recommends \
        curl \
        git \
        ca-certificates \
        build-essential \
        nvidia-cuda-toolkit \
        libhdf5-dev \
        libssl-dev \
        libzstd-dev \
        cmake \
        autoconf \
        automake
```

### Clone and build

```
$ git clone https://github.com/nanoporetech/dorado.git dorado
$ cd dorado
$ cmake -S . -B cmake-build
$ cmake --build cmake-build --config Release -j
$ ctest --test-dir cmake-build
```

The `-j` flag will use all available threads to build Dorado and usage is around 1-2 GB per thread. If you are constrained
by the amount of available memory on your system, you can lower the number of threads i.e.` -j 4`.

After building, you can run Dorado from the build directory `./cmake-build/bin/dorado` or install it somewhere else on your
system i.e. `/opt` *(note: you will need the relevant permissions for the target installation directory)*.

```
$ cmake --install cmake-build --prefix /opt
```

### Pre-commit

The project uses pre-commit to ensure code is consistently formatted; you can set this up using pip:

```bash
$ pip install pre-commit
$ pre-commit install
```

## Troubleshooting Guide

### Library Path Errors

Dorado comes equipped with the necessary libraries (such as CUDA) for its execution. However, on some operating systems, the system libraries might be chosen over Dorado's. This discrepancy can result in various errors, for instance,  `CuBLAS error 8`.

To resolve this issue, you need to set the `LD_LIBRARY_PATH` to point to Dorado's libraries. Use a command like the following on Linux (change path as appropriate):

```
$ export LD_LIBRARY_PATH=<PATH_TO_DORADO>/dorado-x.y.z-linux-x64/lib:$LD_LIBRARY_PATH
```

On macOS, the equivalent export would be (change path as appropriate):

```
$ export DYLD_LIBRARY_PATH=<PATH_TO_DORADO>/dorado-x.y.z-osx-arm64/lib:$DYLD_LIBRARY_PATH
```

This will let the Dorado binary pick up the shipped libraries and you will not need to manually install `libaec` and `zstd`. 

### Improving the Speed of Duplex Basecalling

Duplex basecalling is an IO-intensive process and can perform poorly if using networked storage or HDD. This can generally be improved by splitting up POD5 files appropriately.

Firstly install the POD5 python tools:

The POD5 documentation can be found [here](https://pod5-file-format.readthedocs.io/en/latest/docs/tools.html).


```
$ pip install pod5
```

Then run `pod5 view` to generate a table containing information to split on specifically, the "channel" information.

```
$ pod5 view /path/to/your/dataset/ --include "read_id, channel" --output summary.tsv
```

This will create "summary.tsv" file which should look like:

```
read_id channel
0000173c-bf67-44e7-9a9c-1ad0bc728e74    109
002fde30-9e23-4125-9eae-d112c18a81a7    463
...
```

Now run `pod5 subset` to copy records from your source data into outputs per-channel. This might take some time depending on the size of your dataset
```
$ pod5 subset /path/to/your/dataset/ --summary summary.tsv --columns channel --output split_by_channel
```

The command above will create the output directory `split_by_channel` and write into it one pod5 file per unique channel.  Duplex basecalling these split reads will now be much faster.

### Running Duplex Basecalling in a Distributed Fashion

If running duplex basecalling in a distributed fashion (e.g. on a SLURM or Kubernetes cluster) it is important to split POD5 files as described above. The reason is that duplex basecalling requires aggregation of reads from across a whole sequencing run, which will be distributed over multiple POD5 files.
The splitting strategy described above ensures that all reads which need to be aggregated are in the same POD5 file. Once the split is performed one can execute multiple jobs against smaller subsets of POD5 (e.g one job per 100 channels). This will allow basecalling to be distributed across nodes on a cluster. 
This will generate multiple BAMs which can be merged. This apporach also offers some resilience as if any job fails it can be restarted without having to re-run basecalling against the entire dataset.

### GPU Out of Memory Errors

Dorado operates on a broad range of GPUs but it is primarily developed for Nvidia A100/H100 and Apple Silicon. Dorado attempts to find the optimal batch size for basecalling. Nevertheless, on some low-RAM GPUs, users may face out of memory crashes.

A potential solution to this issue could be setting a manual batch size using the following command:

`dorado basecaller --batchsize 64 ...`

To determine the batch size picked by `dorado`, run it in verbose mode by adding the `-v` option.

**Note:** Reducing memory consumption by modifying the `chunksize` parameter is not recommended as it influences the basecalling results.

### Low GPU Utilization

Low GPU utilization can lead to reduced basecalling speed. This problem can be identified using tools such as `nvidia-smi` and `nvtop`. Low GPU utilization often stems from I/O bottlenecks in basecalling. Here are a few steps you can take to improve the situation:

1. Opt for POD5 instead of .fast5: POD5 has superior I/O performance and will enhance the basecall speed in I/O constrained environments.
2. Transfer data to the local disk before basecalling: Slow basecalling often occurs because network disks cannot supply Dorado with adequate speed. To mitigate this, make sure your data is as close to your host machine as possible.
3. Choose SSD over HDD: Particularly for duplex basecalling, using a local SSD can offer significant speed advantages. This is due to the duplex basecalling algorithm's reliance on heavy random access of data.


## Licence and Copyright

(c) 2023 Oxford Nanopore Technologies PLC.

Dorado is distributed under the terms of the Oxford Nanopore
Technologies PLC.  Public License, v. 1.0.  If a copy of the License
was not distributed with this file, You can obtain one at
http://nanoporetech.com
