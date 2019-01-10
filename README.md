# RNAIndel
RNAIndel calls coding indels and classifies them into 
somatic, germline, and artifact from tumor RNA-Seq data.
Users can also classify indels called by their own callers by
supplying a VCF file. For the indel calling part, users may use
their own callers. RNAIndel supports GRCh38 as well as GRCh37. 

## Prerequisites
The Python dependencies will be [installed](#installation). Users need to install Java.
* [python>=3.5.2](https://www.python.org/downloads/)
    * [pandas>=0.23.0](https://pandas.pydata.org/)
    * [numpy>=1.12.0](https://www.scipy.org/scipylib/download.html)
    * [scikit-learn=0.18.1](http://scikit-learn.org/stable/install.html#)
    * [pysam=0.15.1](https://pysam.readthedocs.io/en/latest/index.html)
    * [pyvcf=0.6.8](https://pyvcf.readthedocs.io/en/latest/index.html)
* [java>=1.8.0](https://www.java.com/en/download/) (required for the built-in Bambino caller)

## Download
```
git clone https://github.com/stjude/RNAIndel.git  # Clone the repo
```

## Installation
Setup a virtual python environment using [conda](https://conda.io/docs/). This step is optional but highly recommended.
```
conda create -n py36 python=3.6 anaconda    # Create a python3.6 virtual environment
source activate py36                        # Activate the virtual environment
```
Install RNAIndel in the virtual environment.
```
cd RNAIndel                 # Switch to the RNAIndel root directory
python setup.py install     # Install bambino and rna_indel from source
bambino -h                  # Check if bambino works correctly
rna_indel -h                # Check if rna_indel works correctly
```

## Data directory set up
Download datafile: [data_dir_37.tar.gz](http://ftp.stjude.org/pub/software/RNAIndel/data_dir_37.tar.gz) 
for GRCh37 and [data_dir_38.tar.gz](http://ftp.stjude.org/pub/software/RNAIndel/data_dir_38.tar.gz) for GRCh38.
Place the gzipped file under a directory of your choice and unpack it. In [Demo](#demo), 
the data directory (GRCh38 version) is located under the RNAIndel root direcotry.<br>  
```
tar xzvf data_dir_37.tar.gz  # for GRCh37
tar xzvf data_dir_38.tar.gz  # for GRCh38
```

## Usage
The RNAIndel pipeline consists of [indel calling](./Bambino) and [classification](./RNAIndel) components, which are performed by two separate excutables.<br>
The pipeline is provided in [BASH](#use-bash-pipeline) or [CWL](#use-cwl-workflow). For a Docker image, see [Docker](#docker). <br>

### Use BASH pipeline
Indels are called by the built-in caller and classified into somatic, germline, and artifact. 
```
rna_indel_pipeline.sh -b BAM \
                      -o OUTPUT_VCF \
                      -f FASTA \
                      -d DATA_DIR \
                      [other options]
```
Users can also classify indel entries in a VCF file generated by their callers (indel calling by the built-in caller will not be performed).
Specify the input VCF file by -c. <br> 
```
rna_indel_pipeline.sh -b BAM \
                      -c INPUT_VCF \
                      -o OUTPUT_VCF \
                      -f FASTA \
                      -d DATA_DIR \
                      [other options]
```
#### Options
* ```-b``` input BAM file (required)
* ```-c``` VCF file from other caller (required for using other callers, e.g., [GATK](https://software.broadinstitute.org/gatk/))
* ```-o``` output VCF file (required)
* ```-f``` reference genome (GRCh37 or 38) FASTA file (required)
* ```-d``` data directory contains trained models and databases (required) [Data directory set up](#data-directory-set-up) 
* ```-q``` STAR mapping quality MAPQ for unique mappers (default=255)
* ```-p``` number of cores (default=1)
* ```-m``` maximum heap space (default 6000m)
* ```-n``` user-defined panel of non-somatic indels in VCF format
* ```-l``` direcotry to store log files 
* ```-h``` show usage message

### Use CWL Workflow
```
cwl-runner --outdir OUT_DIR bambino_rna_indel.cwl INPUT_YML
```

### Docker
RNAIndel has a `Dockerfile` to create a Docker image with all the dependencies installed for running `bambino` and `rna_indel`.
To use this image, [install Docker](https://docs.docker.com/install/) for your platform. It is highly recommended to use
the CWL script to pull the pre-built [docker images](https://cloud.docker.com/u/adamdingliang/repository/docker/adamdingliang/rnaindel)
and run `bambino` and/or `rna_indel`. The docker image can also be used without the CWL script:
```
$ docker run -it adamdingliang/rnaindel:0.1.0 rna_indel
usage: rna_indel [-h] -b FILE (-i FILE | -c FILE) -o FILE -f FILE -d DIR
                 [-q INT] [-p INT] [-n FILE] [-l DIR]
rna_indel: error: the following arguments are required: -b/--bam, -o/--output-vcf, -f/--fasta, -d/--data-dir
```

## Demo 
A demo analysis using sample data is [HERE](./sample_data).<br>

## Input BAM file
Please prepare your BAM file as follows:<br>

1. Map your reads with the STAR 2-pass mode to GRCh38.<br>
2. Add read groups, sort, mark duplicates, and index the BAM file with Picard.<br>

Please input the BAM file from Step 2 without caller-specific preprocessing such as indel realignment.<br>
Additional processing steps may prevent desired behavior.

## Panel of non-somatic indels (PONS)
Somatic prediction can be refined by applying a user-defined indel panel. Putative somatic indels found in the panel will be 
reclassified to germline or artifact, whichever has the higher probability. Indels predicted germline or artifact are not
subject to reclassification by PONS. Such panels can be compiled:
### from normal RNA-Seq data
RNA-Seq data may be a (ideally matched) single or a pooled dataset.<br>
1. Perform variant calling on the RNA-Seq data and generate a VCF file.<br>
2. Index the VCF with Tabix. <br>
### from a cohort dataset of tumor RNA-Seq and tumor/normal-paired DNA-Seq
In this approah, non-somatic indels recurrently misclassified as somatic are collected using a large cohort.<br>
1. Apply RNAIndel on the RNA-Seq data. <br>
2. Validate indels predicted as somatic (putative somatic indels) with the DNA-Seq data. <br>
3. Collect putative somatic indels which are validated as germline or artifact in N samples or more (recurrent non-somatic indels). <br>
4. Format the recurrent non-somatic indels in a VCF file and index with Tabix.<br>

A sample panel by the second approach is included in the [data package](#data-directory-set-up), which is compiled from a 
a cohort of 330 samples with RNA-Seq and T/N-paired WES & PCR-free WGS. When no custom panel is available, apply this panel by appending the following [option](#options):<br>
```
-n path/to/data_dir/non_somatic/non_somatic.vcf.gz
```

## Citations
1. Hagiwara, K., Ding, L., Edmonson, M.N., Rice, S.V., Newman, S., Meshinchi, S., Ries, R.E., Rusch, M., Zhang, J. RNAIndel: a machine-learning framework for discovery of somatic coding indels using tumor RNA-Seq data.    

2. Edmonson, M.N., Zhang, J., Yan, C., Finney, R.P., Meerzaman, D.M., and Buetow, K.H. (2011) Bambino: A Variant Detector 
and Alignment Viewer for next-Generation Sequencing Data in 
the SAM/BAM Format. Bioinformatics 27: 865–866. 
DOI: [10.1093/bioinformatics/btr032](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3051333/)
