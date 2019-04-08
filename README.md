# RNAIndel

RNAIndel calls coding indels and classifies them into 
somatic, germline, and artifact from tumor RNA-Seq data.
You can also use RNAIndel as a postprocessor to
classify indels called by your own caller. 
RNAIndel supports GRCh38 as well as GRCh37. 

## Dependencies
* [python>=3.5.2](https://www.python.org/downloads/)
    * [pandas>=0.23.0](https://pandas.pydata.org/) 
    * [scikit-learn>=0.18.1](http://scikit-learn.org/stable/install.html#)
    * [pysam>=0.13.0](https://pysam.readthedocs.io/en/latest/index.html)
* [java>=1.8.0](https://www.java.com/en/download/) 

## Setup
Install RNAIndel. Be sure that the [dependencies](#dependencies) are satisfied before pip installation.  
```
pip install rnaindel
```
Test the installation.
```
rnaindel -h
```

Download data package ([GRCh38](http://ftp.stjude.org/pub/software/RNAIndel/data_dir_38.tar.gz), [GRCh37](http://ftp.stjude.org/pub/software/RNAIndel/data_dir_37.tar.gz)) and 
unpack it in a convenient directory on your system. 

```
tar xzvf data_dir_38.tar.gz  # for GRCh38
```

## Usage 
RNAIndel has 3 commands:
* ```analysis``` analyze RNA-Seq data for indel discovery
* ```feature``` calculate features for training
* ```training``` train and update the models

Commands are invoked:
```
rnaindel command [command-specific options]
```

### Discover somatic indels ([demo](./sample_data))

#### Working with the built-in caller
RNAIndel calls indels by the [built-in caller](https://academic.oup.com/bioinformatics/article/27/6/865/236751), which is optimized 
for RNA-Seq indel calling, and classifies the indels into somatic, germline, and artifact. 
```
rnaindel analysis -b BAM -o OUTPUT_VCF -f FASTA -d DATA_DIR [other options]
```
#### Working with user`s caller 
RNAIndel predicts somatic indel entries in a VCF file generated by a user`s caller <br>
such as [GATK-HaplotypeCaller](https://software.broadinstitute.org/gatk/documentation/tooldocs/4.0.8.0/org_broadinstitute_hellbender_tools_walkers_haplotypecaller_HaplotypeCaller.php), 
[MuTect2](https://software.broadinstitute.org/gatk/documentation/tooldocs/4.0.8.0/org_broadinstitute_hellbender_tools_walkers_mutect_Mutect2.php)
and [FreeBayes](https://github.com/ekg/freebayes). 
Specify the input VCF file by -v. <br> 
```
rnaindel analysis -b BAM -v INPUT_VCF -o OUTPUT_VCF -f FASTA -d DATA_DIR [other options]
```
#### Options (analysis)
* ```-b``` input [STAR](https://academic.oup.com/bioinformatics/article/29/1/15/272537)-mapped BAM file (required)
* ```-v``` VCF file from other caller (required if working with user`s caller)
* ```-o``` output VCF file (required)
* ```-f``` reference genome (GRCh37 or 38) FASTA file (required)
* ```-d``` [data directory](#setup) contains trained models and databases (required)
* ```-q``` STAR mapping quality MAPQ for unique mappers (default: 255)
* ```-p``` number of cores (default: 1)
* ```-m``` maximum heap space (default: 6000m)
* ```-l``` direcotry to store log files (default: current)
* ```-n``` user-defined panel of non-somatic indels in VCF format (default: built-in validated false-positive set)

### Train RNAIndel
Advanced users can train RNAIndel with their own training sets. 
#### Step 1 (feature calculation)
Features are calculated for each indel and reported in a tab-delimited file.<br>
Using a callset by the built-in caller, 
```
rnaindel feature -b BAM -o OUTPUT_TAB -f FASTA -d DATA_DIR [other options]
```
Using user`s callset, specify the input VCF by -v.
```
rnaindel feature -b BAM -v INPUT_VCF -o OUTPUT_TAB -f FASTA -d DATA_DIR [other options]
```
See \"[analysis](#options-(analysis))\" for \"feature\" commands. 
#### Step 2 (annotation)
The output tab-delimited file has a column \"truth\". Users annotate each indel
by filling the column with either of <br> 
\"somatic\", \"germline\", or \"artifact\". 
#### Step 3 (update models)
Repeat Step 1 and 2 for N samples.<br>
Users concatenate the annotated files. Here, assuming the files are \"sample.i.tab\" (i = 1,...,N), 
```
head -1 sample.1.tab > training_set.tab           # keep the header line
```
```
tail -n +2 -q sample.*.tab > training_set.tab     # concatenate files without header
```
The concatenated file is used as a training set to update the models.
Specify the indel class to be trained by -c. 
```
rnaindel training -t TRAINING_SET -d DATA_DIR -c INDEL_CLASS [other options]
```
#### Options (training)
* ```-t``` training set with annotation (required)
* ```-d``` [data directory](#setup) contains trained models and databases (required) 
* ```-c``` indel class to be trained. s for single-nucleotide indel and m for multi-nucleotide indel (required)
* ```-k``` number of folds in k-fold cross-validation (default: 5)
* ```-p``` number of processes (default: 1)
* ```-l``` directory to ouput log files (default: current)
* ```-ds-beta``` F beta to be optimized in down sampling step. Optimized for TPR if beta > 100. (default: 10)
* ```-fs-beta``` F beta to be optimized in feature selection step. Optimized for TPR if beta > 100. (default: 10)
* ```-pt-beta``` F beta to be optimized in parameter tuning step. Optimized for TPR if beta > 100. (default: 10)

## Reference
1. Hagiwara, K., Ding, L., Edmonson, M.N., Rice, S.V., Newman, S., Meshinchi, S., Ries, R.E., Rusch, M., Zhang, J. 
RNAIndel: a machine-learning framework for discovery of somatic coding indels using tumor RNA-Seq data.
([preprint](https://www.biorxiv.org/content/early/2019/01/07/512749?rss=1))  

