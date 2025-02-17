# Seattle Flu Assembly Pipeline

- [Seattle Flu Assembly Pipeline](#seattle-flu-assembly-pipeline)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Get the pipeline](#get-the-pipeline)
  - [Installation](#installation)
  - [Check fastq file naming](#check-fastq-file-naming)
    - [Lane merged fastqs](#lane-merged-fastqs)
    - [Non-lane merged fastqs](#non-lane-merged-fastqs)
  - [Setup config file](#setup-config-file)
    - [Check read pairs](#check-read-pairs)
  - [NWGC ID/SFS UUID key-value pair](#nwgc-idsfs-uuid-key-value-pair)
  - [Configuration](#configuration)
  - [Dryrun](#dryrun)
  - [Full run](#full-run)
    - [Local](#local)
    - [Deployed to SLURM on the ESR cluster](#deployed-to-slurm-on-the-esr-cluster)
  - [Basic Steps](#basic-steps)
    - [1. Index reference genomes](#1-index-reference-genomes)
    - [2. Merge lanes](#2-merge-lanes)
    - [3. Trim fastqs](#3-trim-fastqs)
    - [4. Post trim fastqc](#4-post-trim-fastqc)
    - [5. Map](#5-map)
    - [6. Sort](#6-sort)
    - [7. Bamstats](#7-bamstats)
    - [8. Mapped reads checkpoint](#8-mapped-reads-checkpoint)
    - [9. Not mapped](#9-not-mapped)
    - [10. Pileup](#10-pileup)
    - [11. Create bed file](#11-create-bed-file)
    - [12. Call Consensus](#12-call-consensus)
    - [13. Zip VCF](#13-zip-vcf)
    - [14. Index BCF](#14-index-bcf)
    - [15. VCF to consensus](#15-vcf-to-consensus)
    - [16. FASTA headers](#16-fasta-headers)
    - [17. Aggregate](#17-aggregate)
    - [Clean](#clean)
  - [Advanced Steps (*not currently supported*)](#advanced-steps-not-currently-supported)
    - [Upload to id3c](#upload-to-id3c)
      - [Send Slack alerts on failed uploads](#send-slack-alerts-on-failed-uploads)

## Introduction

Adapted from Louise Moncla's [Illumina pipline](https://github.com/lmoncla/illumina_pipeline) for influenza snp calling and [seattleflu/assembly](https://github.com/seattleflu/assembly)

## Workflow

![rulegraph_1](rulegraph_1.png)

## Prerequisites

- [Conda install](https://docs.conda.io/projects/conda/en/latest/user-guide/install/linux.html)
- [Mamba install](https://mamba.readthedocs.io/en/latest/installation.html)

## Get the pipeline

Clone the repo/pipeline

```bash
git clone https://github.com/leahkemp/assembly.git
```

## Installation

Install dependencies via mamba by running:

```bash
mamba env create -f envs/seattle-flu-environment.yaml
```

Once installed, the environment needs to be activated to use assembly by running:

```bash
conda activate seattle-flu
```

## Check fastq file naming

### Lane merged fastqs

The pipeline expects 4 total FASTQ files for each individual sample, with Read 1 (R1) and Read 2 (R2) from 4 different lanes.

Expected filename format: `318375_R1.fastq.gz` where `318375` is the NWGC sample ID.

### Non-lane merged fastqs

The pipeline expects 8 total FASTQ files for each individual sample, with Read 1 (R1) and Read 2 (R2) from 4 different lanes.

Expected filename format: `318375_S12_L001_R1_001.fastq.gz` where `318375` is the NWGC sample ID.

If the filename does not contain the NWGC sample ID, rename the FASTQ files by using:

```bash
python scripts/rename_fastq.py
usage: rename_fastq.py [-h]
                       [Sample-ID-Map] [Barcode-column] [ID-column]
                       [FASTQ-directory]

Rename fastq files to contain NWGC sample ID

positional arguments:
  Sample-ID-Map    File path to csv file that matches random barcode to NWGC
                   ID (default: None)
  Barcode-column   Column name of column containing random barcodes (default:
                   None)
  ID-column        Column name of column containing NWGC sample IDs (default:
                   None)
  FASTQ-directory  File path to directory of FASTQ files that need to be
                   renamed (default: None)

optional arguments:
  -h, --help       show this help message and exit
```

## Setup config file

By default, the pipeline will generate combinations of all samples and references and try to create consensus genomes for all combinations.

Avoid this by specifying specific sample-reference pairs and ignored samples in the config file by using:

```bash
python scripts/setup_config_file.py
usage: setup_config_file.py [-h] [--sample [Sample column]]
                            [--target [Target column]]
                            [--max_difference [Max percent difference]]
                            [FASTQ directory] [Config file]
                            [Sample/target map]

Edits the Snakemake config file by adding sample reference pairs and ignored
samples.

positional arguments:
  FASTQ directory       File path to directory containing fastq files
                        (default: None)
  Config file           File path to config file to be used with Snakemake
                        (default: None)
  Sample/target map     File path to Excel file containing samples and present
                        targets (default: None)

optional arguments:
  -h, --help            show this help message and exit
  --sample [Sample column]
                        Column name of column containg NWGC sample IDs
                        (default: sample)
  --target [Target column]
                        Column name of column containing target identifiers
                        (default: target)
  --max_difference [Max percent difference]
                        The maximum difference acceptable between R1 and R2
                        reads (default: 20)
```

__Note__: This currently only works for flu-positive samples. To add more targets/references, edit `references/target_reference_map.json`.

### Check read pairs

Embedded in this script is a check to ensure that the reads in R1 and R2 of each sample match. This is to avoid the demultiplexing error described in [this paper](https://www.nature.com/articles/s41588-019-0349-3). If the reads differ by over 20% then the sample will be added to the ignored samples. Redirect the output to a file to keep track of read differences and ignored samples.

## NWGC ID/SFS UUID key-value pair

Consensus genome FASTA headers are generated with the NWGC sample ID which then needs to be converted to the SFS UUID in order to be paired with stored metadata.

This is done with the [seqkit replace](https://bioinf.shenwei.me/seqkit/usage/#replace) method, which requires a tab-delimited key-value file for replacing the key with the value. The key-value file can be generated using the following:

```bash
python scripts/id_barcode_key_value.py
usage: id_barcode_key_value.py [-h]
                               [NWGC-SFS-Match] [NWGC-column] [SFS-column]
                               [Output]

Create key/value pairs of NWGC ID and SFS ID

positional arguments:
  NWGC-SFS-Match  File path to Excel file that matches NWGC ID to SFS ID
                  (default: None)
  NWGC-column     Column name of column containing NWGC sample IDs (default:
                  None)
  SFS-column      Column name of column containing SFS UUIDs (default: None)
  Output          File path for output key/value file (default: None)

optional arguments:
  -h, --help      show this help message and exit
```

__Note__: Remember to add the path of the output file to the config file after running this script.

## Configuration

Currently, `config/config-flu-only.json` has the most updated version of the configuration needed to run assembly.

* `fastq_directory`: the path to the directory containing the fastq files
* `not_pooled`: an object specifying whether or not the input fastq files are lane merged or not lane-merged, takes either `"pooled"` or `"not_pooled"`
* `ignored_samples`: an object containing keys that specify samples to be ignored
* `sample_reference_pairs`: an object containing specific sample/reference pairs, with the keys refering to the samples and an array of references as the values.
* `barcode_match`: the path to the file containing tab-delimited key-value pairs used to replace NWGC sample IDs with SFS UUIDs
* `min_align_rate`: the minimum overall align rate needed to generate a consensus genome (currently arbitrarily set to `1.00` so that `vcf_to_consensus` does not error out)
* `reference_virusus`: an object containing keys that specify references to be used
* `params`: parameters for the various tools used in the pipeline (currently set by Louise's recommendations)

## Dryrun

Snakemake commands need to be run from the pipeline directory. Move to the pipeline directory, for example:

```bash
cd /home/leah/assembly/
```

Run a dryrun

```bash
snakemake --configfile config/config-flu-only.json -k -j 16 -n
```

If you get no jobs that will be run such as...

```bash
localrule all:
    jobid: 0
    resources: tmpdir=/tmp

Job stats:
job      count    min threads    max threads
-----  -------  -------------  -------------
all          1              1              1
total        1              1              1

This was a dry-run (flag -n). The order of jobs does not reflect the order of execution.
```

...check the path to the fastq directory in the config file

Should get something like:

```bash
Job stats:
job                       count    min threads    max threads
----------------------  -------  -------------  -------------
aggregate                    16              1              1
all                           1              1              1
bamstats                     16              1              1
index_reference_genome        4              1              1
map                          16              1              1
mapped_reads                 16              1              1
post_trim_fastqc              4              1              1
sort                         16              1              1
trim_fastqs                   4              1              1
total                        93              1              1

This was a dry-run (flag -n). The order of jobs does not reflect the order of execution.
```

## Full run

Again make sure you're in the pipeline directory, for example:

```bash
cd /home/leah/assembly/
```

### Local

```bash
snakemake --configfile config/config-flu-only.json -k -j 16
```

### Deployed to SLURM on the ESR cluster

`-j20` sets the maximum number of threads to use at one time to 20

- Pass your username to the `-A` flag (for example `-A lkemp`)

```bash
snakemake -j 20 -k -w 60 \
--configfile config/config-flu-only.json \
--cluster-config config/cluster.json \
--cluster "sbatch -A lkemp -p prod --nodes=1 --tasks=1 --mem={cluster.memory} --cpus-per-task={cluster.cores} --time={cluster.time} -o all_output.out"
```

## Basic Steps

### 1. Index reference genomes

Using [bowtie2-build](http://bowtie-bio.sourceforge.net/bowtie2/manual.shtml#the-bowtie2-build-indexer) to build a Bowtie index for each reference genome. These will later be used in the mapping step.

### 2. Merge lanes

Concatenates 8 FASTQ files for each sample into 2 files (R1 and R2).

### 3. Trim fastqs

Trim raw FASTQ files with [Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic), which cuts out adapter/illumina-specific sequences, trims the reads based on quality scores, and removes short reads.

### 4. Post trim fastqc

Generates summary statistics using [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/). Allows for some quality control checks on raw sequence data coming from high throughput sequencing pipelines.

### 5. Map

Map trimmed reads to reference genome using [bowtie2](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml). Outputs a BAM file that represents aligned sequences and a log file that contains the alignment summary.

### 6. Sort

Sort the BAM file into "genome order" using [samtools sort](http://www.htslib.org/doc/samtools.html).

### 7. Bamstats

Generates statistics for coverage using [BAMStats](http://bamstats.sourceforge.net/).

### 8. Mapped reads checkpoint

[Checkpoints](https://snakemake.readthedocs.io/en/stable/snakefiles/rules.html#data-dependent-conditional-execution) allow for data-dependent conditional execution, forcing Snakemake to re-evaluate the DAG at this point.

Determines if all segments within the reference genome has coverage summary generated from BAMStats.
Then calculates the minimum number of reads required to meet the minimum coverage depth set for `varscan mpileup2snp` and checks the number of mapped reads from the bowtie2 alignment summary. The minimum number of reads required is calculated as:

```bash
min_reads = (reference_genome_length * min_cov) / (raw_read_length)
```

If not all segments are represented or if `mapped_reads` is less than `min_reads`, then stop the process for this sample/reference pair.

### 9. Not mapped

This rule is necessary for the checkpoint above to work, because it must send the data down one of two paths. This is the "dead end" where a consensus genome is __not__ generated.

Also a placeholder for how to handle sample/reference pairs that do not meet the `min_align_rate` threshold. Currently it just outputs the overall align rate of the sample/reference pair so that we can check that it did not meet the minimum.

### 10. Pileup

Generates [Pileup](https://en.wikipedia.org/wiki/Pileup_format) for BAM file using [samtools mpileup](http://www.htslib.org/doc/samtools.html).

_Important Flags_:
* `-a` to print out all positions, including zero depth (this is necessary for generating the low coverage Bedfile later)
* `-A` to not discard anomalous read pairs
* `-Q` to set the minimum base quality to consider a read (set to match the minimum in `varscan mpileup2snp` so that their coverage depths match)

### 11. Create bed file

Creates a BED file for positions that need to be masked in the consensus genome. Positions need to be masked if they are below the minimum coverage depth.

### 12. Call Consensus

Calls consensus (SNPs/indels) from the Pileup based on parameters set in the config file using [varscan mpileup2cns](http://varscan.sourceforge.net/using-varscan.html#v2.3_mpileup2cns)

### 13. Zip VCF

Compress VCF using [bgzip](http://www.htslib.org/doc/bgzip.html), which allows indexes to be built against the file and allows the file be used without decompressing it.

### 14. Index BCF

Creates index for compressed VCF using [bcftools index](https://samtools.github.io/bcftools/bcftools.html#index). This index is necessary for creating the consensus genome using the compressed VCF.

### 15. VCF to consensus

Create consensus genome by applying VCF variants to the reference genome using [bcftools consensus](https://samtools.github.io/bcftools/bcftools.html#consensus).
This does not account for coverage, so we pass it the BEDfile of low coverage sites with the `--mask` option to mask these sites with `N`.

### 16. FASTA headers

Edits the FASTA headers to fit the pattern needed for downstream analysis.
Example FASTA header: `>SFS-UUID|SFS-UUID-PB2|H1N1pdm|PB2`
1. Replace reference sequence name with the NWGC sample ID using Perl to perform "lookaround" regex matches
2. Uses [seqkit replace](https://bioinf.shenwei.me/seqkit/usage/#replace) to replace the NWGC sample ID with the SFS UUID.
3. Create the UUID-gene combination using AWK

### 17. Aggregate

This is the last rule of the pipeline that prints out the final result of each sample/reference pair. If a consensus genome is generated, then this will also add it to the final combined FASTA.

The input of this rule differs based on the result of the checkpoint, so this rule dictates the final outcome of each sample/reference pair.

### Clean

Deletes all of the intermediate directories and files generated from running the pipeline. The only one that it does not delete is the `consensus_genome` directory.

## Advanced Steps (*not currently supported*)

### Upload to id3c

Consensus genomes can be uploaded to [id3c] with the correct id3c permissions stored as environment
variables. Note: id3c uses basic authentication. More information can be found in the [id3c
documentation].

#### Send Slack alerts on failed uploads

An assembly can utilize the ID3C Bot Slack application to send notifications when an id3c upload
fails. The Slack webhook for the #id3c-alerts Slack channel should be stored as an environment
variable. Slack webhook URLs are found at the [Slack Apps API site]. Make sure you're logged into
the Seattle Flu Study workspace. If you cannot view the link above, contact another member of the
id3c-team to add you as a collaborator to the ID3C Bot app.

[id3c]: http://github.com/seattleflu/id3c
[id3c documentation]: https://github.com/seattleflu/id3c/blob/master/README.md
[Slack Apps API site]: https://api.slack.com/apps/ALR131CG0/incoming-webhooks?
