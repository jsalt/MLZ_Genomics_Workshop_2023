# MLZ genomics workshop 2023

# Day 2

**Note:** Before we dive into these steps, make sure you have completed everything listed on the Day 1 tutorial, including starting assembly and installing GATK and vcftools.

## Generate a "pseudoreference" sequence

[background on what we're doing & why]

### Download a toy dataset of clean reads to test our code with

While we're waiting for our assemblies to finish running (which may take a few days), we can start getting our code ready for the next steps. This will be easier to do if we have a small example dataset to play with. We're going to use a few samples from a [paper I published last year](https://www.sciencedirect.com/science/article/abs/pii/S1055790322001725) (along with a few folks from the Moore Lab) - these are four samples of California Quail (*Callipepla californica*) plus one invidivual from their sister species, Gambel's Quail (*C. gambellii*), for an outgroup. We will download the data from GitHub.

### Step 1: Match contigs to probes

As I mentioned on Day 1, we have already taken steps during the labwork phase to select the regions of the genome that we are interested - in this case, UCEs - that process is called enrichment. But as I mentioned, some DNA outside of our targeted UCEs will always sneak through and get sequenced, so the next thing we need to do with our assembled data is to weed out those (hopefully slim) parts of our data that are not the UCEs we care about. We will use a tool in phyluce to do this called `match_contigs_to_probes`, which - you guessed it - matches our assembled contigs to the probe sequences that we used to pull out those UCEs during the enrichment process. If our labwork process was 100% efficient this step would be unneccessary (but it never is, so here we are).

**@ bletchley**

* Login to bletchley and create a new directory called `callipepla-example` in your home directory (use your notes to write this code!) **~/**

* Move into the new directory using `cd` **~/**

* Now we're going to download the example dataset directly on bletchley: **~/callipepla-example**

```
wget https://github.com/jsalt/MLZ_Genomics_Workshop_2023/raw/main/callipepla-contigs.zip
```

* Unzip the file: **~/callipepla-example**

```
unzip callipepla-contigs.zip
```

This should extract a folder called `callipepla-contigs` containing five files ending with `.contigs.fasta`. This is the output we will get when we finish assembling our data with spades.

* We also need to download the UCE probe set that we will match to our assembled contigs. We will use `wget` again: **~/callipepla-example**

```
wget https://raw.githubusercontent.com/faircloth-lab/uce-probe-sets/master/uce-5k-probe-set/uce-5k-probes.fasta
```

* Finally, we need to download one extra file to work with these example data. Enter the following command:

```
wget https://raw.githubusercontent.com/jsalt/MLZ_Genomics_Workshop_2023/main/phyluce.conf -P ~/miniconda3/envs/phyluce-1.7.2/config
```

This is downloading a configuration file from github directly to our phyluce conda environment, so everything will run smoothly.

* Now that we have our example data and our probes, we can start building our `match_contigs_to_probes` command in phyluce. Here's the basic syntax:

```
phyluce_assembly_match_contigs_to_probes \
    --contigs ~/callipepla-example/callipepla-contigs \
    --probes ~/callipepla-example/uce-5k-probes.fasta \
    --output name-of-output-directory
```
Name your output directory something sensible like `uces-callipepla`.

* This code runs fairly quickly (approx. 5 seconds / sample), so we can get away with running it directly on command line without a slurm file. When your command is ready, **activate your phyluce conda environment** and copy the command directly onto bletchley: **~/callipepla-example**

### Step 2: Generate your pseudoreference sequence

Because our datasets focus on population-level sampling (i.e., shallow evolutionary timescales) we are going to further refine our UCE data to focus on single nucleotide polymorphisms (SNPs). These are single nucleotides that are variable in the genome between  individuals. Imagine a 100 bp chunk of homologous DNA from two different individuals - across those 100 bp there may only be 7 nucleotides that are actually different (i.e., individual 1 has an 'A' where individual 2 has a 'C'). A SNP dataset *only* includes those 7 nucleotides. This significantly reduces the amount of data we are analyzing while retaining the important evolutionary signal in our data, and it allows us to run more computationally intensive analyses for fine-scale population structure that wouldn't be feasible with whole UCE contigs.

The process of finding these variable sites in our data is known as *calling SNPs* - this is because rather than assembling the raw read data for each sample indepenently, we instead map the cleaned reads from each individual to a reference sequence and look for differences between each sample and the reference sequence (i.e., SNPs). If we had whole genome data we would use a reference genome to map our samples and call SNPs, but because we have UCE data we just need a reference sequence containing UCEs. We can generate this "pseudoreference" from our own UCE data. The idea is to use the output from our last command, `match_contigs_to_probes` to see which of the samples in our dataset has the best quality data, and use that single sample as the reference sequence for calling SNPs in our other samples.

**@ bletchely**

* Let's look at the output from `match_contigs_to_probes`. We want to select the sample with the highest number of UCE loci. We will print out the logflie to the screen, and then use atom to clean up the output into something more manageable: **~/callipepla-example**

```
cat phyluce_assembly_match_contigs_to_probes.log
```

* Copy the following lines of the output from your terminal window and paste them into your text editor on your computer: **~/callipepla-example**

```
2023-03-01 09:45:07,320 - phyluce_assembly_match_contigs_to_probes - INFO - callipepla_californica_achrustera_81488: 4416 (99.46%) uniques of 4440 contigs, 0 dupe probe matches, 13 UCE loci removed for matching multiple contigs, 14 contigs removed for matching multiple UCE loci
2023-03-01 09:45:19,067 - phyluce_assembly_match_contigs_to_probes - INFO - callipepla_californica_brunnescens_29626: 4318 (99.45%) uniques of 4342 contigs, 0 dupe probe matches, 13 UCE loci removed for matching multiple contigs, 14 contigs removed for matching multiple UCE loci
2023-03-01 09:45:30,757 - phyluce_assembly_match_contigs_to_probes - INFO - callipepla_californica_californica_17959: 4369 (99.41%) uniques of 4395 contigs, 0 dupe probe matches, 13 UCE loci removed for matching multiple contigs, 15 contigs removed for matching multiple UCE loci
2023-03-01 09:45:41,526 - phyluce_assembly_match_contigs_to_probes - INFO - callipepla_californica_canfieldae_34531: 4176 (99.33%) uniques of 4204 contigs, 0 dupe probe matches, 14 UCE loci removed for matching multiple contigs, 16 contigs removed for matching multiple UCE loci
2023-03-01 09:45:53,117 - phyluce_assembly_match_contigs_to_probes - INFO - callipepla_gambelii_gambelii_62399: 4397 (99.39%) uniques of 4424 contigs, 0 dupe probe matches, 15 UCE loci removed for matching multiple contigs, 15 contigs removed for matching multiple UCE loci
```

**@ your desktop**

* We're going to use regular expressions again (yay!) to tidy this up into something we can actually interpret. We'll do three sets of find and replace (enter `ctrl f` to pull up the find and replace bar):

```
# gets rid of date/time stamp
Find: (.+-\s)(.+)
Replace: $2

#replaces spaces with tabs:
Find: [:|,]\s
Replace: \t

# get rid of text
Find: \s\(.*\) uniques of \d+ contigs
Replace:  UCE loci
```

Now we have a tab-delimited file with stats about how many loci we assembled for each of our samples. It's a good idea to save this to our files under a sensible name like `callipepla-match-counts.txt`. **This is especially important when you start working on your real data.**  

In this example dataset, the sample with the most UCE loci is *callipepla_californica_achrustera_81488*. We will use this as our pseudoreference!

**@ bletchley**

* Download the UCE contigs for the example dataset:

```
wget https://github.com/jsalt/MLZ_Genomics_Workshop_2023/blob/main/uces-callipepla.zip
```

* Now that we've chosen our pseudoreference we need to output the full UCE loci sequences for this individual so we can use it to map reads against from the rest of our samples. First, we'll create a data configuration file for this sample. We could do this in our text editor and then move it onto bletchley, or we could learn how to create and edit files directly on the command line. Enter the command `nano` followed by the name of the file we want to create: **~/callipepla-example**

```
nano cc_achrustera_81488_single.conf
```
This will open a blank file in your terminal window.

* The text below (the contents of our conf file) and paste it into the new file open on your terminal window (simple `ctrl v` will work).

```
[pseudoreference]
callipepla_californica_achrustera_81488
```

* Once your text is pasted, close the file by pressing `ctrl x`.

* You will be prompted at the bottom of the screen to press `Y` to save the file. After you enter `Y`, you need to press the `enter` key again to confirm that you want to save the file to the name you entered into nano (i.e., `pseudoref.conf`).

* Once you have completed these steps, use the `ls` command to confirm that your new conf file is in your directory. **~/callipepla-example**

* You can double-check that contents of the file using the `less command`: **~/callipepla-example**

* Now that we have our conf file listing which sample we want to output, we can set up our phyluce command to get a much more complicated configuration file to output our fasta sequence: **~/callipepla-example**

```
phyluce_assembly_get_match_counts \
    --locus-db uces-callipepla/probe.matches.sqlite \
    --taxon-list-config cc_achrustera_81488_single.conf \
    --taxon-group 'pseudoreference' \
    --output pseudoref.conf
```

This will also run very quickly (<5 seconds), so we can run it straight on on bletchley without a slurm file.

If we take a peek inside that `pseudoref.conf` file using the `less` command, we can see that it contains a list of the sample (we only have the one), along with the list of all 4000+ loci we need to output.

* Let's write that fasta file! Here's the basic syntax for our phyluce command: **~/callipepla-example**

```
phyluce_assembly_get_fastas_from_match_counts \
    --contigs  callipepla-contigs \
    --locus-db uces-callipepla/probe.matches.sqlite \
    --match-count-output pseudoref.conf \
    --output cc_achrustera_81488_pseudoref.fasta
```

Once again, because this will run in a mattter of seconds, we can run the command directly on bletchley without a slurm file.

Use `less` to look inside that `cc_achrustera_81488_pseudoref.fasta` file - we have our pseudoreference!
