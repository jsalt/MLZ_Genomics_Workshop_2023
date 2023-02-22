# MLZ genomics workshop 2023

## To-do *before* the workshop

Hello! This tutorial is designed to get you up to speed accessing the Oxy cluster (Bletchley) and installing the key pieces of software we will need to analyze our sequence data. A little housekeeping note: it can be tricky to keep track of where you are in the computing environment when you run each step, so I have included little reminders before each section. We will be working through a terminal window, but sometimes we will need to access files on your local computer (i.e., your laptop), whereas most of the time we will be using the terminal to connect to Bletchley. I have noted this as follows:

**@ your desktop**
Any steps following this will be run on your personal computer (through terminal)

**@ bletchley**
Any steps following this will be run on Bletchley, meaning you need to have a terminal window open and logged into Bletchley (more info on how to do that below)

Let's get started!

### Step 1: Access Bletchley

Some of you already had a Bletchley account. If not, you should  have received an email from David Dellinger confirming your access to Bletchley, Oxy's high-performance computing (HPC) cluster, last week. To access Bletchley, open a terminal window and enter your username to login (for the purposes of this tutorial I will use my username):

*Note: you must be on campus to access Bletchley*

```
ssh jsalter@bletchley.oxy.edu
```

You will be prompted to enter your password - you can find it in the email from David Dellinger with your account info. You can reset the password once you have logged in using the `passwd` command. **Make sure you write down your password somewhere safe!**

### Step 2: Install miniconda

Miniconda is a blah blah blah...

**@ your desktop**

* Download the Miniconda3 Linux 64-bit installation package here and save it to your desktop: https://docs.conda.io/en/latest/miniconda.html#linux-installers

* We need to move the installation package from your local computer onto Bletchley. We will do this using `rysnc`, which transfers files between local and remote machines ("remote" meaning you can access it through `ssh`). Open a new terminal window and enter the following (swap in your username):

```
rsync ~/Desktop/Miniconda3-py310_23.1.0-1-Linux-x86_64.sh jsalter@bletchley.oxy.edu:~
```

You will be prompted to enter your Bletchley password again. With this command, we are telling rsync where to find the Miniconda file `~/Desktop/Miniconda3-py310_23.1.0-1-Linux-x86_64.sh` and where to send it on Bletchley `jsalter@bletchley.oxy.edu:~`, in this case our home directory `~/`.

**@ bletchley**

* Log in to Bletchley in a new terminal window (or click back to the window you already have open). We can check to see that our Miniconda file made it by entering the `ls` command.

* Time to install Miniconda. In the terminal window on Bletchley, enter the following command and follow the prompts on screen (this may take a few minutes):

```
bash Miniconda3-py310_23.1.0-1-Linux-x86_64.sh
```

* Once installation is complete, close your terminal window. Open a new window and log back onto Bletchley. You should notice that your terminal prompt now includes `(base)` before your username, like so:

```
(base) [jsalter@bletchley ~]$
```

That `base` refers to the conda *environment* - this is basically a way of keeping your software organized as you work on different projects. We are going to create a number of different conda environments for analyzing our UCE data.

### Step 3: Install Phyluce (including Illumiprocessor)

For the first part of this workshop, we will be using Phyluce (pronounced *phy-loo-chee*), which is a software package designed to analyze UCE data. Phyluce also includes the software Illumiprocessor, which we will use to remove the unique i5/i7 adapters that we labeled each sample with during library preparation.

**@ bletchley**

* Download the latest release of Phyluce v1.7.2 by entering the following command in the terminal window where you're logged into Bletchley:

```
wget https://raw.githubusercontent.com/faircloth-lab/phyluce/v1.7.2/distrib/phyluce-1.7.2-py36-Linux-conda.yml
```

* Create a new conda environment and install phyluce in it (this will take a few minutes):

```
conda env create -n phyluce-1.7.2 --file phyluce-1.7.2-py36-Linux-conda.yml
```

* Once the installation is complete, we can activate the new environment by entering this command:

```
conda activate phyluce-1.7.2
```

Now our terminal prompt should look like this, telling us we have the phyluce-1.7.2 environment activated:

```
(phyluce-1.7.2) [jsalter@bletchley ~]$
```

*Note: we must activate the phyluce environment before we run any analyses with phyluce. I will include reminders in the remaining tutorials.*

### Step 4: Download a good text editor

The last thing we need to do is get a good text editor. There are a number of popular options - I personally use Atom, which is made by GitHub. You can download Atom here: https://github.com/atom/atom.

Why do you need a good text editor? Because you need a place to take notes on the analyses you're running and prepare files you're going to copy onto Bletchley. Sure, you *could* do this in word or Google docs, but a good text editor has a lot of tricks up its sleeve that can make your life MUCH easier. More on this later...

That's everything we need to get started!
