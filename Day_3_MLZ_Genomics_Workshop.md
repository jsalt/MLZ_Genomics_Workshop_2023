# MLZ genomics workshop 2023

# Day 3

**Note:** Before we dive into these steps, make sure you have completed everything listed on the Day 1 & 2 tutorials, especially starting assemblies with your data.

## First, we install more software!

We need a couple additional programs to complete the next steps, and we can install these through conda inside the environment we already created for this workshop:

**@bletchely**

* [Samtools](http://www.htslib.org/doc/samtools.html) is a suite of programs for interacting with high-throughput sequencing data:

```
source activate UCE_workshop
conda install -c bioconda samtools
```

* [GNU parallel](https://www.gnu.org/software/parallel/) is a tool that allows us to run our code in parallel - meaning we can take full advantage of Bletchely's HPC design to tackle computationally intensive tasks:

*Make sure you're still inside the UCE_workshop environment before installing*

```
conda install -c conda-forge parallel
```

## Align raw reads to pseudoreference + prepare bam files for calling SNPs

Now that our software is installed, we can start re-aligning each of our samples to our pseudoreference genome and preparing to call SNPs across all our samples. But first, we need to do a couple more steps to prepare our pseudoreference.  

### Index + organize our pseudoreference

Before we can use our pseudoreference, we need to **index** it. This is like adding an index to a big book - it allows us to quickly find the information we need without flipping through every page. This is especially important when we're working with a whole reference genome, but even though we're working with a much smaller sequence we still need to index it so we can run the next steps.

**@ bletchley**

* To help keep everything organized, we'll create a new directory and move our psuedoreference sequence into it: **~/callipepla-example**

```
mkdir pseudoref
mv cc_achrustera_81488_pseudoref.fasta pseudoref
```

* Move into the new directory and index the pseudoreference with bwa and samtools: **~/callipepla-example**

```
cd pseudoref
samtools faidx cc_achrustera_81488_pseudoref.fasta
bwa index cc_achrustera_81488_pseudoref.fasta
```

### Map each sample to the pseudoreference

Now that our psuedoreference is indexed, we're going to map our sample reads back to the pseudoreference. Even though our example dataset only have five individuals, we're going to use GNU Parallel to run these jobs, because it will take a lot longer with your actual datasets. The basic set-up for running a job through parallel involves three parts: 1) a bash script with the code we want to run on each sample; 2) a slurm file that uses parallel to manage running that code on more than one of our samples at once; and 3) a list of the samples we want to process.

**@ your desktop**

* First, we need to upload the cleaned reads for our example dataset to `~/callipepla-example` on bletchley (you should have gotten this file in an email). We can use `rsync`, `scp,`, or [FileZilla](https://filezilla-project.org/download.php?show_all=1).


**@ bletchley**

* Once you have your file on bletchley, unzip the reads into a directory: **~/callipepla-example**

```
tar -xzvf callipepla-reads.tar.gz
```

* Now let's create a new directory for our mapped reads to go into, and move into that directory: **~/callipepla-example**

```
mkdir bam-files
cd bam-files
```

* First, let's create the bash script that contains the code for mapping our reads to the pseudoreference sequence. We can set up this file in our notes, and then copy and paste it onto the command line using `nano`: **~/callipepla-example/bam-files**

*bwa-align.sh*

```
#!/bin/bash

source activate UCE_workshop

## set this manually
CORES_PER_JOB=4

## set path to psuedoreference
PSEUDOREF=~/callipepla-example/pseudoref/cc_achrustera_81488_pseudoref.fasta

## DO NOT EDIT BELOW THIS LINE - this comes as input from GNU parallel on STDIN, via the sample.list file

SAMPL=$(basename $1)
READ1=$2
READ2=$3

## Here are the specific commands we are running

# run bwa and output BAM
bwa mem -t $CORES_PER_JOB $PSEUDOREF $READ1 $READ2 | samtools view -bS - > $SAMPL.bam
```

* Once we have created this file using `nano`, we need to make it **executable** - this means we can run it as a piece of code: **~/callipepla-example/bam-files**

```
chmod +x bwa-align.sh
```

We should see the file change color if we enter `ls` - this means we can execute this script

* Next, let's set up our slurm file that will parallelize this script (again, we can use `nano`): **~/callipepla-example/bam-files**

*bwa-align.slurm*  

```
#!/bin/sh

# Note: Comment lines have a space following the #. No space indicates a Slurm command

# Set up your job below:

# ------------------------
# Job name:
#SBATCH --job-name=bwa-align
# ------------------------

# ------------------------
# Partition name:
#SBATCH --partition=medium
#
# List of job submission partitions:
# Name:          TimeLimit:
# teaching       5:00
# short          5:00:00
# medium         2-00:00:00
# long           7-00:00:00
# unlimited      infinite
# demo           1:00
# cscomps        5:00
# gpu            infinite
# ------------------------

# ------------------------
# Resource specification:
#SBATCH --nodes=2
#SBATCH -c 4
#
# Notes on requesting resources:
# In general you only need to change the number of CPU's,
# i.e., the number following SBATCH -c
# Each node has a maximum of 36 CPU's
# CPU's requested is PER NODE requested, so the total number of cores is the product of
# the number of nodes and the number of CPU's listed!
# e.g., nodes=2, -c 4 requests 8 total processors.
# You may add other resource requests as desired.
# ------------------------

# ------------------------
slurm_startjob(){
# THIS IS WHERE YOUR JOB SCRIPT GOES

# Number of Cores per job (needs to be multiple of 2)
export CORES_PER_JOB=4

# move into the directory containing this script
cd $SLURM_SUBMIT_DIR

# set the number of Jobs per node based on $CORES_PER_JOB
export JOBS_PER_NODE=$(($SLURM_CPUS_PER_TASK / $CORES_PER_JOB))

parallel --colsep '\,' \
        --progress \
        --joblog logfile.$SLURM_JOBID \
        -j $JOBS_PER_NODE \
        --workdir $SLURM_SUBMIT_DIR \
        -a callipepla.list \
        ./bwa-align.sh {$1} {$2} {$3}
}

# Load any environment variables, etc. here


# Prep output to slurm-<jobno>.out
slurm_infoutput(){
echo "================================== SLURM JOB =================================="
date
echo
echo "Slurm User:         $SLURM_JOB_USER"
echo "Job-id:             $SLURM_JOB_ID"
echo "Jobname:            $SLURM_JOB_NAME"
echo "Queue:              $SLURM_JOB_PARTITION"
echo "Number of nodes:    $SLURM_JOB_NUM_NODES"
echo "#CPU's per node:    $SLURM_CPUS_PER_TASK"
echo "Start Directory:    $SLURM_SUBMIT_HOST:$SLURM_SUBMIT_DIR"
echo "================================== SLURM JOB =================================="
echo
echo "--- Slurm job-script output ---"
}

# Output job info
slurm_infoutput

# Run the job:
slurm_startjob

#
# End of file
#
```

There's a lot going on in this slurm file, but here are the key points. Notice that the command in the file is not `bwa mem`, which is how we're mapping the reads, it's just a `parallel` command. We've already got the `bwa` command in our *bwa-align.sh* file, and here we're using parallel to call that file. The only other thing we need to give parallel is a list of our samples - that's what `-a callipepla.list` stands for.

* Let's create our sample list. For our example dataset, we only have five samples. Notice how in the slurm file the line calling our bash script is `./bwa-align.sh {$1} {$2} {$3}` - this means there are three values we need to feed into our bash script for each of our samples. We can see what they are by looking back at the bash script:

```
SAMPL=$(basename $1)
READ1=$2
READ2=$3
```

First, we need give it the name of our sample (`$1`); second, we need the path to the READ1 file (`$2`); third we need the path to the READ2 file (`$3`). Notice that in the parallel command it says `--colsep '\,'` - this means it is expecting our three arguments to be separated by a comma. Here's the basic syntax of what we need for each sample:

*callipepla.list*
```
callipepla_californica_brunnescens_LSU_29626,/home/jsalter/UCE_workshop/callipepla-example/callipepla-reads/callipepla_californica_brunnescens_LSU_29626-READ-1.fastq,/home/jsalter/UCE_workshop/callipepla-example/callipepla-reads/callipepla_californica_brunnescens_LSU_29626-READ-2.fastq
```

First we have the sample name, second we have the full path the READ1, third the full path to READ2.

**IMPORTANT!!! For some reason, bwa may not like if you abbreviate your full patch using the `~` symbol, so use `/home/yourname/` instead.***

* With only five samples it's concieveable that we could create this file by hand, but once you start working with your real datasets that's going to get unruly. We can use a couple of command line tricks + our text editors to do the heavy lifting. First, let's get a list of the read1 samples using `ls`: **~/callipepla-example**

```
ls -1 callipepla-reads/*READ-1.fastq
```

**@ your desktop**

* Copy and paste that list into a new document in your text editor. We'll use regular expressions again to transform this into the three values we need:

*Make sure you turn regular expressions on in your search bar*

```
Find: ((callipepla-reads/(\w+))-READ-1.fastq)
Replace All: $3,~/callipepla-example/$1,~/callipepla-example/$2-READ-2.fastq
```

**@ bletchley**

* Now that we have our completed list, copy and save it into a new text file on bletchley called `callipepla.list`: **~/callipepla-example/bam-files**

```
nano callipepla.list
```

* Now that we have our sample list, our bash script, and our slurm file, we are ready to submit the run using sbatch. **~/callipepla-example/bam-files**

*Make sure that all three of these files are in the same directory!*

```
sbatch bwa_align.slurm
```

### Sort the resulting bam files

* Create new sample list for the bams **~/callipepla-example/bam-files**

```
ls -d $PWD/*.bam > bams.list

```        

* Create a bash script to sort the bame files **~/callipepla-example/bam-files**   

*chmod +x bam-sort.sh*

```
#!/bin/bash

source activate UCE_workshop

## set this manually
CORES_PER_JOB=4

## DO NOT EDIT BELOW THIS LINE - this comes as input from GNU parallel on STDIN, via the sample.list file

SAMPL=$(basename $1 .bam)
BAM=$(basename $1)

## Here are the specific commands we are running

# sort bam files
samtools sort -@ $CORES_PER_JOB -o $SAMPL.sorted.bam $BAM
```

* Create a slurm file to parallelize the job **~/callipepla-example/bam-files**

*bam_sort.slurm*

```
#!/bin/sh

# Note: Comment lines have a space following the #. No space indicates a Slurm command

# Set up your job below:

# ------------------------
# Job name:
#SBATCH --job-name=bam-sort
# ------------------------

# ------------------------
# Partition name:
#SBATCH --partition=medium
#
# List of job submission partitions:
# Name:          TimeLimit:
# teaching       5:00
# short          5:00:00
# medium         2-00:00:00
# long           7-00:00:00
# unlimited      infinite
# demo           1:00
# cscomps        5:00
# gpu            infinite
# ------------------------

# ------------------------
# Resource specification:
#SBATCH --nodes=2
#SBATCH -c 4
#
# Notes on requesting resources:
# In general you only need to change the number of CPU's,
# i.e., the number following SBATCH -c
# Each node has a maximum of 36 CPU's
# CPU's requested is PER NODE requested, so the total number of cores is the product of
# the number of nodes and the number of CPU's listed!
# e.g., nodes=2, -c 4 requests 8 total processors.
# You may add other resource requests as desired.
# ------------------------

# ------------------------
slurm_startjob(){
# THIS IS WHERE YOUR JOB SCRIPT GOES

# Number of Cores per job (needs to be multiple of 2)
export CORES_PER_JOB=4

# move into the directory containing this script
cd $SLURM_SUBMIT_DIR

# set the number of Jobs per node based on $CORES_PER_JOB
export JOBS_PER_NODE=$(($SLURM_CPUS_PER_TASK / $CORES_PER_JOB))

parallel --colsep '\,' \
        --progress \
        --joblog logfile.$SLURM_JOBID \
        -j $JOBS_PER_NODE \
        --workdir $SLURM_SUBMIT_DIR \
        -a bams.list \
        ./bam-sort.sh {$1}

}

# Load any environment variables, etc. here


# Prep output to slurm-<jobno>.out
slurm_infoutput(){
echo "================================== SLURM JOB =================================="
date
echo
echo "Slurm User:         $SLURM_JOB_USER"
echo "Job-id:             $SLURM_JOB_ID"
echo "Jobname:            $SLURM_JOB_NAME"
echo "Queue:              $SLURM_JOB_PARTITION"
echo "Number of nodes:    $SLURM_JOB_NUM_NODES"
echo "#CPU's per node:    $SLURM_CPUS_PER_TASK"
echo "Start Directory:    $SLURM_SUBMIT_HOST:$SLURM_SUBMIT_DIR"
echo "================================== SLURM JOB =================================="
echo
echo "--- Slurm job-script output ---"
}

# Output job info
slurm_infoutput

# Run the job:
slurm_startjob

#
# End of file
#
```

* Submit the job **~/callipepla-example/bam-files**

```
sbatch bam_sort.slurm
```

### Add read groups

* Add read groups (catalog numbers) in gatk (picard): **~/callipepla-example/bam-files**

* Create sample list: **~/callipepla-example/bam-files**

```
ls -d $PWD/*.sorted.bam | sed -E "s/(.*_([0-9]*)).sorted.bam/\1,\1.sorted.bam,\2/" > sorted.list      
```

* Create the bash script: **~/callipepla-example/bam-files**

*chmod +x add-rg.sh*

```
#!/bin/bash

source activate UCE_workshop

## set this manually
CORES_PER_JOB=4

## DO NOT EDIT BELOW THIS LINE - this comes as input from GNU parallel on STDIN, via the sample.list file

SAMPL=$(basename $1)
BAM=$2
CAT=$3

## Here are the specific commands we are running

gatk AddOrReplaceReadGroups \
-I $BAM \
-O $SAMPL.RG.bam \
-SORT_ORDER coordinate \
-RGPL illumina \
-RGPU unit1 \
-RGLB lib1 \
-RGSM $CAT \
-VALIDATION_STRINGENCY LENIENT

````

* Create the slurm file: **~/callipepla-example/bam-files**

*add_rg.slurm*

```
#!/bin/sh

# Note: Comment lines have a space following the #. No space indicates a Slurm command

# Set up your job below:

# ------------------------
# Job name:
#SBATCH --job-name=add-rg
# ------------------------

# ------------------------
# Partition name:
#SBATCH --partition=medium
#
# List of job submission partitions:
# Name:          TimeLimit:
# teaching       5:00
# short          5:00:00
# medium         2-00:00:00
# long           7-00:00:00
# unlimited      infinite
# demo           1:00
# cscomps        5:00
# gpu            infinite
# ------------------------

# ------------------------
# Resource specification:
#SBATCH --nodes=2
#SBATCH -c 4
#
# Notes on requesting resources:
# In general you only need to change the number of CPU's,
# i.e., the number following SBATCH -c
# Each node has a maximum of 36 CPU's
# CPU's requested is PER NODE requested, so the total number of cores is the product of
# the number of nodes and the number of CPU's listed!
# e.g., nodes=2, -c 4 requests 8 total processors.
# You may add other resource requests as desired.
# ------------------------

# ------------------------
slurm_startjob(){
# THIS IS WHERE YOUR JOB SCRIPT GOES

# Number of Cores per job (needs to be multiple of 2)
export CORES_PER_JOB=4

# move into the directory containing this script
cd $SLURM_SUBMIT_DIR

# set the number of Jobs per node based on $CORES_PER_JOB
export JOBS_PER_NODE=$(($SLURM_CPUS_PER_TASK / $CORES_PER_JOB))

parallel --colsep '\,' \
        --progress \
        --joblog logfile.$SLURM_JOBID \
        -j $JOBS_PER_NODE \
        --workdir $SLURM_SUBMIT_DIR \
        -a sorted.list \
        ./add-rg.sh {$1} {$2} {$3}

}

# Load any environment variables, etc. here


# Prep output to slurm-<jobno>.out
slurm_infoutput(){
echo "================================== SLURM JOB =================================="
date
echo
echo "Slurm User:         $SLURM_JOB_USER"
echo "Job-id:             $SLURM_JOB_ID"
echo "Jobname:            $SLURM_JOB_NAME"
echo "Queue:              $SLURM_JOB_PARTITION"
echo "Number of nodes:    $SLURM_JOB_NUM_NODES"
echo "#CPU's per node:    $SLURM_CPUS_PER_TASK"
echo "Start Directory:    $SLURM_SUBMIT_HOST:$SLURM_SUBMIT_DIR"
echo "================================== SLURM JOB =================================="
echo
echo "--- Slurm job-script output ---"
}

# Output job info
slurm_infoutput

# Run the job:
slurm_startjob

#
# End of file
#
```

* Submite the slurm file: **~/callipepla-example/bam-files**

```
sbatch add_rg.slurm
```

### Mark duplicates

* Create a new sample list: **~/callipepla-example/bam-files**

```
ls -d $PWD/*.RG.bam | sed -E "s/(.*_[0-9]*_.*).RG.bam/\1,\1.RG.bam/" > RG.list    
```

* Create the bash script: **~/callipepla-example/bam-files**

*chmod +x mark-dup.sh*

```
#!/bin/bash

source activate UCE_workshop

## set this manually
CORES_PER_JOB=4

## DO NOT EDIT BELOW THIS LINE - this comes as input from GNU parallel on STDIN, via the sample.list file

SAMPL=$(basename $1 .RG.bam)
BAM=$2

## Here are the specific commands we are running

gatk MarkDuplicates \
-I $BAM \
-O $SAMPL.RG.MD.bam \
-METRICS_FILE $SAMPL.picard-metricsfile.txt \
-MAX_FILE_HANDLES_FOR_READ_ENDS_MAP 250 \
-ASSUME_SORTED true \
-VALIDATION_STRINGENCY SILENT \
-REMOVE_DUPLICATES false
````

* Create the slurm file: **~/callipepla-example/bam-files**

*mark_dup.slurm*
```
#!/bin/sh

# Note: Comment lines have a space following the #. No space indicates a Slurm command

# Set up your job below:

# ------------------------
# Job name:
#SBATCH --job-name=mark-dup
# ------------------------

# ------------------------
# Partition name:
#SBATCH --partition=medium
#
# List of job submission partitions:
# Name:          TimeLimit:
# teaching       5:00
# short          5:00:00
# medium         2-00:00:00
# long           7-00:00:00
# unlimited      infinite
# demo           1:00
# cscomps        5:00
# gpu            infinite
# ------------------------

# ------------------------
# Resource specification:
#SBATCH --nodes=2
#SBATCH -c 4
#
# Notes on requesting resources:
# In general you only need to change the number of CPU's,
# i.e., the number following SBATCH -c
# Each node has a maximum of 36 CPU's
# CPU's requested is PER NODE requested, so the total number of cores is the product of
# the number of nodes and the number of CPU's listed!
# e.g., nodes=2, -c 4 requests 8 total processors.
# You may add other resource requests as desired.
# ------------------------

# ------------------------
slurm_startjob(){
# THIS IS WHERE YOUR JOB SCRIPT GOES

# Number of Cores per job (needs to be multiple of 2)
export CORES_PER_JOB=4

# move into the directory containing this script
cd $SLURM_SUBMIT_DIR

# set the number of Jobs per node based on $CORES_PER_JOB
export JOBS_PER_NODE=$(($SLURM_CPUS_PER_TASK / $CORES_PER_JOB))

parallel --colsep '\,' \
        --progress \
        --joblog logfile.$SLURM_JOBID \
        -j $JOBS_PER_NODE \
        --workdir $SLURM_SUBMIT_DIR \
        -a RG.list \
        ./mark-dup.sh {$1} {$2}

}

# Load any environment variables, etc. here


# Prep output to slurm-<jobno>.out
slurm_infoutput(){
echo "================================== SLURM JOB =================================="
date
echo
echo "Slurm User:         $SLURM_JOB_USER"
echo "Job-id:             $SLURM_JOB_ID"
echo "Jobname:            $SLURM_JOB_NAME"
echo "Queue:              $SLURM_JOB_PARTITION"
echo "Number of nodes:    $SLURM_JOB_NUM_NODES"
echo "#CPU's per node:    $SLURM_CPUS_PER_TASK"
echo "Start Directory:    $SLURM_SUBMIT_HOST:$SLURM_SUBMIT_DIR"
echo "================================== SLURM JOB =================================="
echo
echo "--- Slurm job-script output ---"
}

# Output job info
slurm_infoutput

# Run the job:
slurm_startjob

#
# End of file
#
```

* Submit the slurm file: **~/callipepla-example/bam-files**

```
sbatch mark_dup.slurm
```
