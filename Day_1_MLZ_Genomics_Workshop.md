# MLZ genomics workshop 2023

# Day 1

## Preparing our data with Illumiprocessor

Our first goal today is to go from raw sequencing reads to cleaned reads that we can start analyzing. What does that mean? When we prepared these samples (i.e., library preparation and enrichment) we tagged each invididual with a unique combination of indexes called i5 and i7 adapters. These are eight-nucleotide sequences of DNA that were added onto the end of all the fragments of DNA in each sample, and because each indidvual sample has a unique combination of i5 and i7 sequences we were able to pool many samples together and sequence them all at once. When we get sequence data back from the sequencer, the first step is using those i5/i7 indexes to sort all the reads into each individual sample. This was already done for us by the sequencing facility. The next step is to remove those eight nucleotides at the end of all the reads, because they're not actually the DNA of the organisms that we care about (aka raw reads (with i5/i7 adapters still on) vs. clean reads (i5/i7 adapters removed).

### Step 0: Bash 101 + file management

**@ your desktop**

Over the next few days, we are going to be using atom to edit files on our laptops and then transferring them to Bletchley using `rsync` or `scp`. This will be made much easier if we keep everything organized in a single directory (or folder) on our computers. I suggest creating a folder on your desktop called `UCE_workshop`. You all know how to create a new folder, but let's learn how to do it on the command line!

* Open a new terminal window. By default, a new terminal window will place you in your computer's home directory, which is represented by the tilda symbol `~`. You should see a prompt in your new window that looks something like this:

```
jsalt at lemoncurd in ~ $
```

This is telling you which user (`jsalt`) is **at** which computer (`lemoncurd` - this is what I've named my laptop) **in** what directory (`~`). If you ever get lost on the terminal, you can enter `pwd` and it will print out the full path to the directory you are currently in.

```
pwd
```
Output:
```
/Users/jsalt
```

This is my home directory, and can abbreviated as `~`.

* We can see the contents of the directory we're in using the `ls` command. Enter the following:

```
ls
```

We should see a list of all the files and directories in our home directory print out to the screen.

* We want to move over to the desktop so we can create a new directory to keep our workshop materials organized. To change directories, we use the `cd` command and provide the path to where we want to move. Enter the following:

```
cd ~/Desktop
```

We should see our terminal prompt update accordingly, and we can always double check by entering `pwd`.

```
jsalt at lemoncurd in ~/Desktop $
```

* Now let's create a new directory on our desktop using the `mkdir` command. We'll call this new directory `UCE_workshop`. Enter the following command:

```
mkdir UCE_workshop
```

It's good practice to keep track of *where* you are when you're working on an analysis, because it's surprisingly easy to lose track and waste time later looking for files. So from here on out, we are going to get in the habit of writing down the path the directory we're working in for each step. I like to do this by adding the path at the end of each step, like so:

* Check that your directory was created successfully using the `ls` command **~/Desktop**

**@ bletchley**

* Now we need to do a bit of file management on Bletchley. Open a new terminal window and login (here's a reminder on the command)

```
ssh jsalter@bletchley.oxy.edu
```

* Let's create a new directory for this workshop on Bletchley. Notice that by default we are in our home directory `~` after logging in. Create a new directory using the `mkdir` command and then move into that directory using `cd`: **~/**

```
mkdir UCE_workshop_hpc
cd ~/UCE_workshop_hpc
```

For the sake of clarity, I decided to name this something slightly different that the directory on our desktops (remember, HPC stands for high-performance computer cluster). Now we have a place to keep our files organized on our desktops and on Bletchley.

### Step 1: Start taking notes

For any analysis, it is *so important* to take thorough, detailed notes. Big picture, these notes are how you will keep track of what you did so you can write it up later (e.g., for a presentation or publication), and they will also help you keep track of where you are in a given project when you inevitably start and stop working on it. It's amazing how much you can forget about what you were doing in just a few days or weeks - good notes help you dive back in and not waste time retracing your steps. These notes will also become a great resource for how to set up specific analyses - instead of starting from scratch the next time you, say, need to set up a slurm file (more on this later), you can just look back in your own notes and adapt from there. Your notes are the place where you'll set up the code you intend to run, so you can get everything right before you enter it onto the command line. Finally, detailed notes help you troubleshoot - if you can't remember what you tried before, good luck trying to figure out why something's not working. **I cannot stress this enough: good notes are everything!**

I use atom to take notes and I like to have a single document where I write down *every single step* of what I did for a project. I take notes in markdown format `.md` because it allows you to do some nice formatting (fyi these instructions are written in markdown - you can find a basic guide to the syntax [here](https://www.markdownguide.org/cheat-sheet/)), but even a simple `.txt` file will do.

* Start a new file in atom where you will record all your notes for these analyses. Save it to the `UCE_workshop` folder you created on your desktop. Name it something helpful like `UCE_workshop_notes_2023`.

* From here on out - every step that is outlined in these intstructions should be noted in your own notes. I like to use bullet points to give a brief explanation of what each step is doing and why, followed by the actual code we are running. I also keep track of the directry (and computer) where I am running each step - it helps you find files later.

### Step 2: Set up your illumiprocessor conf file and slurm file

The program we will use to remove the i5/i7 indexes is called [Illumiprocessor](https://github.com/faircloth-lab/illumiprocessor/). The basic workflow is giving illumiprocessor a list of all the samples and what i5/i7 adapters they have attached, as well as the actual eight-nucleotide sequence for each of those adapters is so the program can find and remove those sequences from the end of all our reads.

**@ your desktop**

* First things first, we need to upload our raw sequencing data onto Bletchley. You should all have received an email from me with the link to donwload your dataset from dropbox - place the folder containing your data into **~/Desktop/UCE_workshop**. We'll use `rsync` to transfer the files into our newly created **UCE_workshop_hpc** directory on Bletchely *(don't forget to change the command to include your folder name / username)*:

```
rysnc -r --progress ~/Desktop/UCE_workshop/selasphorus jsalter@bletchley.oxy.edu:~/UCE_workshop_hpc
```

Because we're transferring a directory instead of a single file, we add the `-r` flag, which stands for **recursive** - this tells `rsync` to copy the directory and all its contents. We also add the `--progress` flag so it will print out the name of each file as it uploads, which tells us how it's going (this will take a few minutes).

If `rsync` is not working for you, you can use `scp` instead. We still need to add that `-r` flag to copy the contents of the directory:

```
scp -r ~/Desktop/UCE_workshop/selasphorus jsalter@bletchley.oxy.edu:~/UCE_workshop_hpc
```

* In the meantime, set up your configuration file for Illumiprocessor. This file is the map connecting each of your samples to the unique combination of i5 and i7 indexes, as well as a place for us to rename each sample to something more useful. These files are not difficult to set up but it can be tedious, so to save time I have created your files ahead of time. But let's walk through an example file:

*illumiprocessor.conf*
```
# this is the section where you list the adapters you used. the asterisk
# will be replaced with the appropriate index for the sample.
[adapters]
i7:GATCGGAAGAGCACACGTCTGAACTCCAGTCAC*ATCTCGTATGCCGTCTTCTGCTTG
i5:AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT*GTGTAGATCTCGGTGGTCGCCGTATCATT

# this is the list of all the indexes we used for this dataset
[tag sequences]
i7_103_10:AAGAAGGC
i7_103_11:AGGTTCGA
i7_103_12:CATGTTCC
i5_09_B:ACGGACTT
i5_09_E:CGTCTAAC
i5_09_F:AACACTGG
i5_09_G:TCAGACAC

# this is how each index maps to each set of reads
[tag map]
aegolius_funereus_28868:i7_103_10,i5_09_B
asio_flammeus_21727:i7_103_11,i5_09_F
asio_otus_15937:i7_103_12,i5_09_E
athene_cunicularia_31327:i7_103_10,i5_09_G

# we may want to rename our read files something a bit more helpful. if you
# don't want to change the names, you would keep them the same after the colon.
[names]
aegolius_funereus_28868:aegolius_funereus_28868_mongolia
asio_flammeus_21727:asio_flammeus_21727_kansas
asio_otus_15937:asio_otus_15937_kansas
athene_cunicularia_31327:athene_cunicularia_31327_missouri
```

* Your config file was in the folder you downloaded from Dropbox (along with the raw read data). Save it to: **~/Desktop/UCE_workshop**

* I use atom to prepare my config file on my computer, but once it's ready we need to get it onto Bletchley into our `UCE_workshop_hpc` directory. We will use `rsync` for this again. **~/Desktop/UCE_workshop**

```
rsync --progress ~/Desktop/UCE_workshop/illumiprocessor.conf jsalter@bletchley.oxy.edu:~/UCE_workshop_hpc
```

**OR**

```
scp ~/Desktop/UCE_workshop/illumiprocessor.conf jsalter@bletchley.oxy.edu:~/UCE_workshop_hpc
```

* Let's start preparing our code to run illumiprocessor. Here's the basic syntax:

```
illumiprocessor \
    --input [path to directory with raw reads] \
    --output [path to directory for clean reads] \
    --config [illumiprocessor.conf] \
    --r1-pattern '{}_(R1|READ1|Read1|read1)_\d+.fastq(?:.gz)*' \
    --r2-pattern '{}_(R2|READ2|Read2|read2)_\d+.fastq(?:.gz)*' \
    --cores [# of cores]
```

The **command** we are running is `illumiprocessor` and we are specifying how we want to run it using **arguments** like `--input` and `--output`. We can learn more about what each of these arguments means by entering `illumiprocessor --help`. But first, we need to activate the conda environment we created for phyluce:

```
source activate phyluce-1.7.2
illumiprocessor --help
```

* The `--input` argument is the path to the directory of raw sequence data we just uploaded to Bletchely. It should be something along the lines of: `~/UCE_workshop_hpc/selasphorus-raw-reads` *(don't forget to sub in your group name)*.

* For the `--output` argument we need to list the directory we want to store the output in. Name this something sensible like `cleaned-reads`. By default, if we just provide the name of the directory illumiprocessor will create it in the directoy where we run the code from (which should be our `UCE_workshop_hpc` folder).

* We're going to leave the `--r1-pattern` and `--r2-pattern` arguments as above - this is basically telling illumiprocessor how to recognize the reads from the sequencer.

* For `--config` we need to provide the name of our configuration file (e.g., `parrots-illumiprocessor.conf`). By default, if we just provide the name of the file illumiprocessor will look for it in the directoy where we run the code from (which should be our `UCE_workshop_hpc` folder).

* That just leaves `--cores` - this is how much computational power we want to devote to this task. We'll go with `12`.

* Make sure to write your final illumiprocessor command in your notes with all the arguments customized to your analysis.

* Finally, we need to set up the the `slurm` file that will actually run our code. A little background here (taken from the [Bletchley Users Guide](https://github.com/justinnhli/bletchley-docs/blob/main/users.md#running-your-code)):

```
Almost all high-performance clusters like Bletchley have a workload manager or job scheduling system that manages what programs are running and where they run. On Bletchley, this program is called Slurm. Slurm's role is to make sure that whenever someone wants to use Bletchley, they can be assigned a CPU (or multiple CPUs) with enough memory to run their program, without monopolizing the cluster from other people. Although you can run programs directly on Bletchley, this is highly discouraged, as you may prevent other users from doing their work; anyone found doing this may have their access to Bletchley removed. Running programs directly from the command line should be reserved for testing your code, and once you are confident your code works, you should then ask Slurm to assign your program to a CPU.

In general, your workflow for using Bletchley will look like this:

1. Develop the code on your own computer, and make sure that it works there (on a small example).
2. Upload your code to Bletchley.
3. Run your code from the command line in Bletchley on a small example, to make sure it still works.
4. Modify your code to do the actual work instead of the small example.
5. Submit your code to Slurm and wait for it to finish
6. Download any result files to be analyzed.
```

This is the general workflow we will be following, because we want to be considerate users of Bletchley. Here's an example slurm file that you can adapt for running illumiprocessor *(replace your customized illumiprocessor code below)*:

*run-illumiprocessor.slurm*
```
#!/bin/sh

# Note: Comment lines have a space following the #. No space indicates a Slurm command

# Set up your job below:

# ------------------------
# Job name:
#SBATCH --job-name=illumiprocessor
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
#SBATCH --nodes=1
#SBATCH -c 12
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

# activate our conda environment for phyluce
source activate phyluce-1.7.2

illumiprocessor \
    --input [path to directory with raw reads] \
    --output cleaned-reads \
    --config [illumiprocessor.conf] \
    --r1-pattern '{}_(R1|READ1|Read1|read1)_\d+.fastq(?:.gz)*' \
    --r2-pattern '{}_(R2|READ2|Read2|read2)_\d+.fastq(?:.gz)*' \
    --cores [cores]

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

* Save your customized slurm file to **~/Desktop/UCE_workshop**. Once again we'll use `rsync` or `scp` to upload it to Bletchely. Use your notes to adapt the code you used to move your illumiprocessor conf file.

### Step 3: Run illumiprocessor

**@ bletchley**

* Go back to the terminal window that's open to Bletchley. Move into the **~/UCE_workshop_hpc** directory and check that the conf file and slurm files are there using `ls`.

* It's finally time to run our first piece of code on Bletchley! Here's how to submit our job using slurm: **~/UCE_workshop_hpc**

```
sbatch run-illumiprocessor.slurm
```

* We can check to see that our code is running by running the `squeue` command. If we simply enter `squeue`, we will get a list of all the jobs currently running on bletchley - this can be useful to get a sense of how busy the cluster is. If you just want to check on the jobs that you have submitted, enter `squeue -u [your username]` (e.g., I would enter `squeue -u jsalter`). This shows us what partition we're running on, the current status of the job, how long it's been running for, etc.

If everything is working correctly, we should see our output directory **~/UCE_workshop_hpc/cleaned-reads** appear and start to fill up with folders for each of our samples.

*This should run fairly quickly, but we have a few things to do in the meantime...*

### Assemble our UCE data

[background on assembly]

**@ bletchley**

* First we need to set up another configuration file that will point phyluce towards the cleaned reads for each sample. This time you're going to create the conf file yourself! Here's the basic format:

```
[samples]
sample_name:/path/to/sample_name/split-adapter-quality-trimmed
```

This is where a good text editor can come in really handy, because if we were to type out each sample name and path by hand it would very time consuming and extrememly likely that we would make an error. Instead, we can use some simple command line + text editor magic to speed up the process. Let's start by getting a list of our sample names. Conveniently, the output from illumiprocessor created a directory for each sample with our new sample names, so we can simply print out the contents of our **~/UCE_workshop_hpc/cleaned-reads** directory:

```
ls -1 ~/UCE_workshop_hpc/cleaned-reads
```

By adding the `-1` flag we're telling it to print a single entry on each line.

**@ your dekstop**

* Copy the list and paste it into a new file on your text editor. Because the format for each sample in our assembly conf file is the same, we can use a magical tool called **regular expressions** to automate the process of creating each entry. In brief, regular expressions (or regex for short) allow you to do find and replace for all charcters meeting certain conditions - so instead of finding a specific phrase (like "quail_123") we can find all phrases that follow the pattern "word underscore numbers" - i.e., we can pick up "quail_123" and "duck_4567". Learning the basics of regex is super useful - you can read up on the basic synatx [here](https://quickref.me/regex), but I'll walk us through the steps for today.

* We're going to use regex to grab the name of each sample and build the rest of the line around it. In the new file where you pasted your list of sample names, pull up the find and replace bar by entering `command F`. Click the little button with the `.*` symbol on the find bar to turn on regex. In the find bar, enter the following:

```
(\w+)
```

We're telling atom to find all lines that have alphanumeric characters (including `_`) and place them in a capture group (i.e., between parentheses). This will allow us to use that capture group in other parts of our expression.

* In the replace bar, we're going to build the complete line we need for each sample `sample_name:/path/to/sample_name/split-adapter-quality-trimmed`. Enter the following into the replace bar and hit `Replace All`:

```
$1:~/UCE_workshop_hpc/cleaned-reads/$1/split-adapter-quality-trimmed
```

Et voilà! We have our assembly conf file. That `$1` refers back to the capture group `(\w+)` we defined in the find bar.

* At the very top of the file, add `[samples]` and hit a new line. The format should look like this:

```
[samples]
sample_name1:/path/to/sample_name1/split-adapter-quality-trimmed
sample_name2:/path/to/sample_name2/split-adapter-quality-trimmed
...
```

* Save the file to the workshop folder on your desktop **~/Desktop/UCE_workshop** - name it something sensible like `assembly.conf`.

* Move the `assembly.conf` file in your **~/UCE_workshop_hpc** directory on Bletchely using `rsync` or `scp`.

**@ bletchley**

* Next, let's set up our phyluce command. Just like we did with illumiprocessor, we'll work out all the syntax here before pasting it into a slurm file. We can get a list of the required arguments by running the command with `--help` at the end *(don't forget to activate our phyluce environment first!)*: **~/UCE_workshop_hpc**

```
source activate phyluce-1.7.2
phyluce_assembly_assemblo_spades --help
```

This prints out a list of all the arguments we can pass to phyluce and what they should be. You can do this for any phyluce command.

* The arguments are pretty simple here since we've created a config file. We just need to specify the path to our output directory `--output`, the path to our config file `--config`, and the number of cores `--cores`. Pick a sensible name for the output directory, like `spades-assemblies`. We're going to up the number of cores to 24.

```
phyluce_assembly_assemblo_spades \
    --output [name of output directory] \
    --config [name of conf file] \
    --cores 24
```

* Now let's create a new slurm file to run this job. We can use our illumiprocessor slurm file as a template, but there a few key things we need to change:

```
#SBATCH --job-name=spades
```

```
#SBATCH -c 24
```

```
slurm_startjob(){
# THIS IS WHERE YOUR JOB SCRIPT GOES

# activate our conda environment for phyluce
source activate phyluce-1.7.2

[paste your phyluce command here]

}
```

Save this slurm file as `run-spades.slurm` in **~/UCE_workshop_hpc**.

* Time to run! Submit the job using:

```
sbatch run-spades.slurm
```

You should see the output directory appear if it's working properly. You can also check on the status using `squeue`. This will take many hours to run, but should be finished in time for our next session on Wednesday.

## Installing software for Wednesday

While we're waiting for our assembly to complete, we can get a few things ready for the next steps - mainly, installing the software we will use to call SNPs from our UCE data. The main program we need is called the Genome Analysis ToolKit (GATK for short, and it is a serious piece of bioinformatic software. It can do all manner of fancy analyses, but we're just going to scratch the surface here. You can read more about GATK [here](https://gatk.broadinstitute.org/hc/en-us/articles/360036194592-Getting-started-with-GATK4). GATK is *not* installed through conda, but it is a fairly easy process.

### Download & install GATK

**@ your desktop**

* Download the latest version of gatk (v4.3.0) [here](https://github.com/broadinstitute/gatk/releases/download/4.3.0.0/gatk-4.3.0.0.zip) and save it to your desktop.

* Transfer the file into your home directory **~/** on bletchley. **~/Desktop**

**@ bletchley**

* Back on bletchley, unzip the gatk file in your home directory: **~/**

```
unzip gatk-4.3.0.0.zip
```

This will print out a lot of stuff to your terminal window - just wait a few seconds for it to finish.

* Once the unzip is complete, we need to do a couple of things to ensure we can run `gatk` from anywhere inside bletchley. First, we'll add the directory to our path by entering the following command: **~/**

```
export PATH="~/gatk-4.3.0.0/:$PATH"
```

* Finally, we will add an *alias* to bletchley - this is like a keyboard shortcut, so that instead of typing out the full path to `gatk` everytime we want to run it (i.e., `~/gatk-4.3.0.0/gatk`), we can just type `gatk`. To do this, we need to edit our *bash profile*, which is like a settings file. We can edit files directly on the command line using `nano`, which is a command line text editor. **~/**

```
nano .bash_profile
```

* This will open up your `.bash_profile` file, which should look like the example below. Add the last two lines as follows:

```
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/.local/bin:$HOME/bin

export PATH

# aliases

alias gatk='~/gatk-4.3.0.0/gatk'
```

* To save your changes to the file hold down `control` and `X`, hit `Y` to save the changes, then `return` to exit the file. To make the changes stick, we need to log out and then back into bletchley. We can log out by simply entering:

```
exit
```

* Log back onto bletchley and test that the installation worked: **~/**

```
gatk --help
```

This should print out a list of options to the screen if the installation was successful.

### Create a new environment + install bwa & vcftools through conda

The last piece of software we need to install is

**@ bletchley**

* Let's create and a new environment to keep our SNP-calling software organized (minus GATK, which runs outside of conda).

```
conda create --name UCE_workshop -c bioconda
```

That `-c bioconda` flag is telling conda that we want the environment to link up with the bioconda repository, where all the software we'll be using is stored. You don't really need to worry about this.

* [BWA](https://bio-bwa.sourceforge.net/) is a software package for aligning read data to a reference sequence. Typically, this means a reference genome, but in our case we are going to be aligning our reads to the assembled UCE loci for our best sample (more on this Wednesday). For now, we just need to install it in our new conda environment:

```
source activate UCE_workshop
conda install -c bioconda bwa
```

Follow the prompts on screen to get this installed (this may take a few minutes).

* The last piece of software we need to install for now is [VCFtools](https://vcftools.github.io/examples.html). VCFtools is used to manipulate SNP data stored in variant call format (VCF), which will be the final output that we use to run our actual phylogenetic / population analyses (more on this Wednesday). Let's install it with conda *(make sure the UCE_workshop environment is still activated)*:

```
conda install -c bioconda vcftools
```

Follow the prompts on screen to get this installed (this may take a few minutes).

That's it for today! We'll let our assemblies finish running and start the SNP calling process on Wednesday.
