### Comparative gene expression between tongue and hard palate tissues in *Carollia perspicillata*
Authors: Georgiana Wood, Alexis M. Brown, Liliana M. Daválos

Last Update: 02/20/2024

GitHub repository: https://github.com/georgianawood/c.perspicillata

---
**Objectives:**

## **Pipeline**
#### In Seawulf, activate custom pipeline environment
```
module load anaconda/3

source activate /gpfs/projects/DavalosGroup/Lexi/envs/rnaseq
```


Change to project directory:
```
cd /gpfs/projects/DavalosGroup/carollia_project
```



## **1) QC and read trimming: *fastp***
**1. Run fastQC script on samples to be used.**

FastQC script:

```
!/bin/bash
#SBATCH --ntasks-per-node=40
#SBATCH --time=48:00:00
#SBATCH --nodes=2
#SBATCH --job-name=fastqc
#SBATCH --output=/gpfs/scratch/gnwood/carollia_project/error.txt/%j.out.txt
#SBATCH --error=/gpfs/scratch/gnwood/carollia_project/error.txt/%j.err.txt
#SBATCH -p long-40core
#SBATCH --mail-user=georgiana.wood@stonybrook.edu
#SBATCH --mail-type=begin
#SBATCH --mail-type=end

module load anaconda/3
source activate activate /gpfs/projects/DavalosGroup/Lexi/envs/genomics

Read1=/gpfs/projects/DavalosGroup/Georgie/carollia_project_original/rena_tissues/hardpal/*1.fq.gz;
Read2=/gpfs/projects/DavalosGroup/Georgie/carollia_project_original/rena_tissues/hardpal/*2.fq.gz;

fastqc --outdir /gpfs/scratch/gnwood/carollia_project/fastqc $Read1 $Read2;

done
```

**2. Run fastp script on all samples to be used.**

Fastp script:

```
#!/bin/bash
#SBATCH --ntasks-per-node=40
#SBATCH --time=48:00:00
#SBATCH --nodes=2
#SBATCH --job-name=fastp
#SBATCH --output=/gpfs/scratch/gnwood/carollia_project/error.txt/%j.out.txt
#SBATCH --error=/gpfs/scratch/gnwood/carollia_project/error.txt/%j.err.txt
#SBATCH -p long-40core
#SBATCH --mail-user=georgiana.wood@stonybrook.edu
#SBATCH --mail-type=begin
#SBATCH --mail-type=end

module load anaconda/3
source activate /gpfs/projects/DavalosGroup/Lexi/envs/rnaseq

cd /gpfs/projects/DavalosGroup/Georgie/carollia_project_original/rena_tissues/hardpal

for f1 in *1.fq.gz*
do

        f2="${f1/1.fq.gz/2.fq.gz}"
        fastp -i "$f1" -I "$f2" -o "trimmed-$f1" -O "trimmed-$f2"


done
```
*This script specifies the hard palate tissues, but would be altered to a directory with all trimmed fastp outputs.*

**3. Combining forward and reverse reads for cohesive Trinity inputs**

Code to combine forward and reverse reads:

```
gunzip *gz ## Unzips any gzipped files

cat *_1*fq > All_R1.fq ## combine all forward reads
cat *_2*fq > All_R2.fq ## combine all reverse reads
```

At this point, our samples are trimmed, preprocessed, and ready for the assembly stage. All_R1.fq and All_R2.fq will be the inputs for Trinity.

## **2) *De novo* transcriptome assembly: Trinity**

**1. Running Trinity** 

*All tongue & hard palate samples used.*

Script for running Trinity:

```
!/bin/bash
#SBATCH --ntasks-per-node=28
#SBATCH --time=48:00:00
#SBATCH --nodes=1
#SBATCH --job-name=trinity
#SBATCH --output=/gpfs/scratch/gnwood/carollia_project/error.txt/%j.out.txt
#SBATCH --error=/gpfs/scratch/gnwood/carollia_project/error.txt/%j.err.txt
#SBATCH -p long-40core
#SBATCH --mail-user=georgiana.wood@stonybrook.edu
#SBATCH --mail-type=begin
#SBATCH --mail-type=end

module load anaconda/3
source activate /gpfs/projects/DavalosGroup/Lexi/envs/trinity

cd /gpfs/scratch/gnwood/carollia_project/all_R1R2

f1=All_R1.fq
f2=All_R2.fq

Trinity --seqType fq \
        --left $f1 \
        --right $f2 \
        --min_contig_length 300 \
        --CPU 36 \
        --max_memory 100G \
        --output /gpfs/scratch/gnwood/carollia_project/trinity \
        --full_cleanup
```

**2. Transrate quality assesment on assembly**

Now that we have our assembly output,  (```trinity.Trinity.fasta```) we will quickly assess the quality of our assessment.

Results were lost in the erasure of doom, but if this is deemed a neccessary step it runs relatively quickly (~2 hours). Quality assessment was poor initially.

Script for running transrate:

```
!/bin/bash
#SBATCH --ntasks-per-node=28
#SBATCH --time=48:00:00
#SBATCH --nodes=1
#SBATCH --job-name=transrate
#SBATCH --output=/gpfs/scratch/gnwood/carollia_project/error.txt/%j.out.txt
#SBATCH --error=/gpfs/scratch/gnwood/carollia_project/error.txt/%j.err.txt
#SBATCH -p long-40core
#SBATCH --mail-user=georgiana.wood@stonybrook.edu
#SBATCH --mail-type=begin
#SBATCH --mail-type=end

cd /gpfs/scratch/gnwood/carollia_project/trinity

transrate --assembly trinity.Trinity.fasta \
        --left /gpfs/scratch/gnwood/carollia_project/all_R1R2/All_R1.fq \
        --right /gpfs/scratch/gnwood/carollia_project/all_R1R2/All_R2.fq \
        --threads 16

```

## **3) Consolidate redundant transcripts with CDHIT-EST**

![no-it-doesnt-freakin-work-chris-rock](https://hackmd.io/_uploads/By0jlDKi6.gif)

Due to many headaches and tears, we have decided to move on without this step.

## **4) Quantification with RSEM**

Script for RSEM quantification:
```
#!/bin/bash
#SBATCH --ntasks-per-node=40
#SBATCH --time=48:00:00
#SBATCH --nodes=6
#SBATCH --job-name=trinity_rsem
#SBATCH --output=/gpfs/scratch/gnwood/carollia_project/error.txt/%j.out.txt
#SBATCH --error=/gpfs/scratch/gnwood/carollia_project/error.txt/%j.err.txt
#SBATCH -p long-40core
#SBATCH --mail-user=georgiana.wood@stonybrook.edu
#SBATCH --mail-type=begin
#SBATCH --mail-type=end

module load anaconda/3
source activate /gpfs/projects/DavalosGroup/Lexi/envs/rnaseq

/gpfs/projects/DavalosGroup/Lexi/envs/rnaseq/bin/align_and_estimate_abundance.pl --seqType fq \
        --left gpfs/scratch/gnwood/carollia_project/fastp/trimmed-11.8_1.fq \ #SUBMIT FOR EACH TISSUE
        --right gpfs/scratch/gnwood/carollia_project/fastp/trimmed-11.8_2.fq \ #SUBMIT FOR EACH TISSUE
        --transcripts gpfs/scratch/gnwood/carollia_project/trinity/trinity.Trinity.fasta \
        --est_method RSEM --aln_method bowtie2 \
        --trinity_mode --prep_reference \
        --output_dir gpfs/scratch/gnwood/carollia_project/rsem
```
*Each tissue must be submitted as separate jobs*

## **Project Notes:**

At this point, the project will be turned over to Lexi.

## **5) Differential Expression: Tongue vs. Hard Palate**
## **6) Gene Ontology Annotation and Enrichment******