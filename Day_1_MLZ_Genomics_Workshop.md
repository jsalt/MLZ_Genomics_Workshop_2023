# MLZ genomics workshop 2023

## Day 1: Preparing our data with Illumiprocessor

Our goal today is to go from raw sequencing reads to cleaned reads that we can start analyzing. What does that mean? When we prepared these samples (i.e., library preparation and enrichment) we tagged each invididual with a unique combination of indexes called i5 and i7 adapters. These are eight-nucleotide sequences of DNA that were added onto the end of all the fragments of DNA in each sample, and because each indidvual sample has a unique combination of i5 and i7 sequences we were able to pool many samples together and sequence them all at once. When we get sequence data back from the sequencer, the first step is using those i5/i7 indexes to sort all the reads into each individual sample. This was already done for us by the sequencing facility. The next step is to remove those eight nucleotides at the end of all the reads, because they're not actually the DNA of the organisms that we care about (aka raw reads (with i5/i7 adapters still on) vs. clean reads (i5/i7 adapters removed).

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

* Start a new file in atom where you will record all your notes for these analyses. Save it to the `UCE_workshop` folder you created on your desktop. Name it something helpful like `UCE_worshop_notes_2023`.

* From here on out - every step that is outlined in these intstructions should be noted in your own notes. I like to use bullet points to give a brief explanation of what each step is doing and why, followed by the actual code we are running. I also keep track of the directry (and computer) where I am running each step - it helps you find files later.

### Step 2: Set up your illumiprocessor conf file and slurm file

The program we will use to remove the i5/i7 indexes is called [Illumiprocessor](https://github.com/faircloth-lab/illumiprocessor/). The basic workflow is giving illumiprocessor a list of all the samples and what i5/i7 adapters they have attached, as well as the actual eight-nucleotide sequence for each of those adapters is so the program can find and remove those sequences from the end of all our reads.

**@ your desktop**

* Set up your configuration file for Illumiprocessor. This file is the map connecting each of your samples to the unique combination of i5 and i7 indexes, as well as a place for us to rename each sample to something more useful. These files are not difficult to set up but it can be tedious, so to save time I have created your files ahead of time. But let's walk through an example file:

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

* You can download your config file from the workshop github page. Save it to: **~/Desktop/UCE_workshop**

* I use atom to prepare my config file on my computer, but once it's ready we need to get it onto Bletchley into our `UCE_workshop_hpc` directory. We will use `rsync` for this again. **~/Desktop/UCE_workshop**

```
rsync ~/Desktop/UCE_workshop/illumiprocessor.conf jsalter@bletchley.oxy.edu:~/UCE_workshop_hpc
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
conda activate phyluce-1.7.2
illumiprocessor --help
```

* I have already uploaded all the sequence data for this workshop into a shared directory on Bletchley. Copy the path for your `--input` argument from the list below:

```
Selasphorus (Atthis) hummingbirds:
Steller's Jays:
Stripe-headed Sparrow:
Green Jays:
Grosbeaks:
Parrots:
Red Warblers:
```

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
conda activate phyluce-1.7.2

illumiprocessor \
    --input [path to directory with raw reads] \
    --output [path to directory for clean reads] \
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

* If everything is working correctly, we should see our output directory **~/UCE_workshop_hpc/cleaned-reads** appear and start to fill up with folders for each of our samples.
