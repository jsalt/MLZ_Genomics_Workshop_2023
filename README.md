# MLZ Genomics Workshop 2023

This repository will hold code and general information for how to analyze UCE data using Oxy's high-performance computing cluster Bletchley.

## To-do *before* our first meeting

Follow the instructions for [Day 0](https://github.com/jsalt/MLZ_Genomics_Workshop_2023/blob/main/Day_0_MLZ_Genomics_Workshop.md). This will walk you through four steps:
1. Accessing Bletchley, the Oxy cluster
2. Installing Miniconda on Bletchley
3. Installing Phyluce on Bletchley
4. Installing the text editor Atom on your own computer

It is also a good idea to read through the [Bletchley Users Guide](https://github.com/justinnhli/bletchley-docs/blob/main/users.md) because we want to be considerate users of this shared resource!


## [Day 1:](https://github.com/jsalt/MLZ_Genomics_Workshop_2023/blob/main/Day_1_MLZ_Genomics_Workshop.md) From raw reads to assembled contigs

Today we're going to cover the big picture of what we're doing when we analyze UCE data for phylogenetic / population genetic analyses. Then we're going to tackle four main steps:
1. Getting our files organized and starting our all-important notes
2. Trimming the adapters off our raw sequence data with [Illumiprocessor](https://github.com/faircloth-lab/illumiprocessor/)
3. Assembling our data in [Phyluce](https://phyluce.readthedocs.io/en/latest/purpose.html)
4. Installing a few more pieces of software for the next steps

## [Day 2:](https://github.com/jsalt/MLZ_Genomics_Workshop_2023/blob/main/Day_2_MLZ_Genomics_Workshop.md) Generating our pseduoreference
Today we'll pick up where we left off on Day 1 with the main goal of getting everyone's data assembling. Once that's running, we'll talk about the analytical approach we're using with our datasets and tackle the following:
1. Download a small example dataset that we'll use to test our code
2. Develop and test the code to generate a pseudoreference sequence from our assembled data
