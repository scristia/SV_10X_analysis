# *Snakemake workflow for structural rearrangement analysis of 10X Genomics linked-read WGS*

## Description
This workflow will run the [SvABA](https://github.com/walaj/svaba) structural variation (SV) analysis for a set of tumor-normal pairs, starting from the BAM files aligned using [Long Ranger](https://support.10xgenomics.com/genome-exome/software/pipelines/latest/what-is-long-ranger) software. The analysis includes structural variant prediction and assessment of barcode (BX) overlap from 10X linked-reads. It will also look for [TitanCNA](https://github.com/gavinha/TitanCNA) results and combine these results to output SV classes.  
This analysis was used and described in the publication:  
Viswanathan SR*, Ha G*, Hoff A*, et al. Structural Alterations Driving Castration-Resistant Prostate Cancer Revealed by Linked-Read Genome Sequencing. *Cell* 174, 433–447.e19 (2018).

## Contact
Gavin Ha  
Fred Hutchinson Cancer Research Center  
contact: <gavinha@gmail.com> or <gha@fredhutch.org>  
Date: August 7, 2018  

## Requirements
### Software packages or libraries
 - R-3.4
	-optparse  
	-data.table  
	-GenomicRanges  
	-GenomeInfoDb  
	-VariantAnnotation  
	-plyr  
	-ggplot2  
	-reshape2  
	-stringr  
 - Python 3.4 
   - snakemake-3.12.0
 - [samtools-1.3.1](http://www.htslib.org/)

# Files in the workflow
### Scripts used by the workflow
The following scripts are used by this snakemake workflow:
 - [barCodeOverlap.R](code/barCodeOverlap.R) - 
 - [combineSVABAandTITAN.R](code/combineSVABAandTITAN.R) - 
 - [plotTitanSvaba.R](code/plotTitanSvaba.R) - 
 - [svaba_utils.R](code/svaba_utils.R) - 
 - [tenX_utils.R](code/tenX_utils.R) - 
 - [plotting.R](code/plotting.R) - 

### Tumour-Normal sample list [config/samples.yaml](config/samples.yaml)
The list of tumour-normal paired samples should be defined in a YAML file. In particular, the [Long Ranger](https://support.10xgenomics.com/genome-exome/software/pipelines/latest/what-is-long-ranger) (v2.2.2) analysis directory is listed under samples.  See `config/samples.yaml` for an example.  Both fields `samples` and `pairings` must to be provided.  `pairings` key must match the tumour sample while the value must match the normal sample.
```
samples:
  tumor_sample_1:  /path/to/tumor/longranger/dir
  normal_sample_1:  /path/to/normal/longranger/dir


pairings:
  tumor_sample_1:  normal_sample_1
```

# Running the analysis

## 1. Invoking the snakemake workflow for SvABA on a local machine
The first snakemake file [svaba.snakefile](svaba.snakefile) will 
  a. Run the full [SvABA](https://github.com/walaj/svaba) analysis
  b. Compute barcode overlap values for barcode rescue in the next step.
```
# show commands and workflow
snakemake -s svaba.snakefile -np
# run the workflow locally using 5 cores
snakemake -s svaba.snakefile --cores 5
```

## 2. Integrating SV and copy number results ([TitanCNA](https://github.com/gavinha/TitanCNA_10X_snakemake))
The second snakemake file [combineSvabaTitan.snakefile](combineSvabaTitan.snakefile) will 
  a. Combine SvABA and [TitanCNA (10X analysis)](https://github.com/gavinha/TitanCNA_10X_snakemake) results to assign copy number to the breakpoints and rearrangement classes to the SV events.
  b. Plot copy number and SV for each chromosome. 
  ```
  snakemake -s combineSvabaTitan.snakefile --cores 5
  ```
  
The third snakemake file [plotSVandCNAzoom.snakefile](plotSVandCNAzoom.snakefile) generates custom plots zoomed in for a region of interest. Users can specify the coordinates in the [configPlotZoom.yaml](config/configPlotZoom.yaml) file. See the description of the configuration below.
  ```
  snakemake -s plotSVandCNAzoom.snakefile --cores 5
  ```

## 3.  Invoking the snakemake workflow for SvABA on a cluster
If you are using a cluster, then these are resource settings for memory and runtime limits, and parallel environments.  
These are the default settings for all tasks that do not have rule-specific resources.  
There are two cluster configurations provided: `qsub` and `slurm`

#### i. `qsub`
There are 2 separate files in use for `qsub`, which are provided as a template:
	`config/cluster_qsub.sh` - This file contains other `qsub` parameters. *Note that these settings are used for the Broad's UGER cluster so users will need to modify this for their own clusters.*  
	`config/cluster_qsub.yaml` - This file contains the memory, runtime, and number of cores for certain tasks.  

To invoke the snakemake pipeline for `qsub`:
```
snakemake -s svaba.snakefile --jobscript config/cluster_qsub.sh --cluster-config config/cluster_qsub.yaml --cluster-sync "qsub -l h_vmem={cluster.h_vmem},h_rt={cluster.h_rt} -pe {cluster.pe} -binding {cluster.binding}" -j 100
```
Here, the `h_vmem` (max memory), `h_rt` (max runtime) are used. For `runSvaba` task, the default setting is to use 4 cores and this can be set with `-pe` and `-binding`. Your SGE settings may be different and users should adjust accordingly.

#### ii. `slurm`
There is only one file in use for `slurm`:
	`config/cluster_slurm.yaml` - This file contains the memory, runtime, and number of cores for certain tasks. 
To invoke the snakemake pipeline for `slurm`:
```
snakemake -s svaba.snakefile --cluster-config config/cluster_slurm.yaml --cluster "sbatch -p {cluster.partition} --mem={cluster.mem} -t {cluster.time} -c {cluster.ncpus} -n {cluster.ntasks} -o {cluster.output}" -j 50
```

# Configuration and settings
All settings for the workflow are contained in [config/config.yaml](config/config.yaml). The settings are organized by paths to scripts and reference files and then by each step in the workflow.

### 1. Path to tools
- `svaba_exe` is the compiled SvABA executable 
```
svaba_exe:  /path/to/svaba
samTools:  /path/to/samtools
```

### 2. Path to scripts
These are provided in this repo under [code/](code/).  
```
tenX_funcs:  code/tenX_utils.R
svaba_funcs:  code/svaba_utils.R
plot_funcs:  code/plotting.R
bxRescue_script:  code/barCodeOverlap.R
combineSVCN_script:  code/combineSVABAandTITAN.R
plotSVCN_script:  code/plotTitanSvaba.R
```

### 3. Path to R package files
Specify the directory in which [TitanCNA](https://github.com/gavinha/TitanCNA) is installed.  
*Set these if the R files in these libraries have been modified or updated but not yet installed or updated in R*.
```
titan_libdir:  /path/to/TitanCNA/
```

### 4. Path to TitanCNA 10X snakemake results
Specifies the [TitanCNA 10X snakemake](https://github.com/gavinha/TitanCNA_10X_snakemake) location containing result files to be merged with SvABA results in [combineSvabaTitan.snakefile](combineSvabaTitan.snakefile).
```
titan_results:  /path/to/TitanCNA/snakemake_results/
```

### 5. Reference files and settings
Global reference files used by the `snakefiles` and scripts.  
- `refGenome` specify the reference genome used in the Long Ranger analysis
- `genomeStyle` specifies the chromosome naming convention to used for **output** files. Input files can be any convention as long as it is the same genome build. Only use `UCSC` (e.g. chr1) or `NCBI` (e.g. 1). 
- `cytobandFile` is used for plotting the chromosome idiograms and only needs to specify [data/cytoBand_hg38.txt](data/cytoBand_hg38.txt) if using hg38.
- `chrs` specifies the chromosomes to analyze; users do not need to be concerned about chromosome naming convention here as the code will handle it based on the `genomeStyle`.
```
refGenome:  /path/to/ref/genome.fasta
genomeBuild:  hg38
genomeStyle:  UCSC
cytobandFile:  data/cytoBand_hg38.txt # only need if hg38
chrs:  c(1:22, \"X\")
```

### 6. Long Ranger bam files
Set this to the filenames that are used for the BAM files generated by Long Ranger. The current filenames are ones generated by [Long Ranger](https://support.10xgenomics.com/genome-exome/software/pipelines/latest/what-is-long-ranger) v2.2.2
```
bamFileName:  phased_possorted_bam.bam
```

### 7. [svaba.snakefile](svaba.snakefile) settings: SvABA
The cluster resources for SvABA are set here. 3G of memory for each of the 4 cores totals to 12G set as the limit. 
- `svaba_dbSNPindelVCF` specifies a VCF file containing known indels to use for germline filtering [Homo_sapiens_assembly38.known_indels.vcf.gz](https://storage.cloud.google.com/genomics-public-data/resources/broad/hg38/v0/Homo_sapiens_assembly38.known_indels.vcf.gz?_ga=2.113507335.-1633399588.1531762721)

```
svaba_dbSNPindelVCF:  /path/to/dbsnp_indel.vcf
svaba_numThreads:  4  # should match cluster settings for number of cores
``` 

### 8. [svaba.snakefile](svaba.snakefile) settings: Barcode counting
Minimum thresholds and settings for barcode (BX) counting.  
- `bxRescue_minLength` sets minimum length of the intra-chromosomal SV event to be consider for barcode counting. Shorter events are excluded, except for fold-back inversions.
- `bxRescue_windowSize` sets the window size region to the left or right of the breakpoint for barcode counting.
```
bxRescue_minMapQ:  20
bxRescue_minLength:  10000
bxRescue_windowSize:  1000
bxRescue_minReadOverlapSupport:  2
```

### 9. [combineSvabaTitan.snakefile](combineSvabaTitan.snakefile) settings: Plotting
Settings used for plotting copy number and SV results.  
- `plot_geneFile` is a text file listing the regions to annotate in the plot. 4 column file: name, chr, start, stop.
```
plot_zoom:  FALSE
plot_chrs:  c(1:22, \"X\") # "None" will also plot all chromosomes
plot_startPos:  None
plot_endPos:  None
plot_geneFile:  data/AR_coord.txt ## include list of genes to annotate on plot
plot_ylim:  c(-2,6)
plot_size:  c(8,4)
plot_type:  titan ## titan - will include haplotype fraction
plot_format:  png
```

### 10. [plotSVandCNAzoom.snakefile](plotSVandCNAzoom.snakefile) settings in [configPlotZoom.yaml](config/configPlotZoom.yaml)
- `plot_id` to use for naming the output directory containing the zoomed plots for each sample.
- `plot_zoom` indicates the plot should be focused on a specific region smaller than a whole chromosome. If set to `TRUE`, then `plot_chrs` (should only be a single chr), `plot_startPos`, `plot_endPos` should be set.
- 'plot_type` is set to `ichor` for total copy number only (black dots). Set it to `titan` to also include the allelic fraction panel as a second track.
```
plot_id: AR_enhancer_zoom
plot_zoom:  TRUE
plot_chrs:  c(\"X\") 
plot_startPos:  66500000
plot_endPos:  67900000
plot_geneFile:  data/AR_coord.txt ## include list of genes to annotate on plot
plot_ylim:  c(-2,6)
plot_size:  c(8,4)
plot_type:  ichor ## use "titan" to also plot the allelic fraction panel 
plot_format:  png
```


