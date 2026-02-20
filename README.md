# Introduction to Symbiosis Evolution Group bioinformatics pipeline for Multitarget-Marker-Genes-Analysis
We hope that this script will help you navigate through the analyses of an example amplicon dataset - a dozen libraries from our Greenland project.

## Contents - what we will cover here ---
1. Some basic Linux commands.
2. Accessing example data
3. Workflow overview
4. Data splitting into bins corresponding to different targets: symbio_split
5. The core amplicon analysis workflow: symbio_core
6. Mitochondrial data analysis: symbio_barcode
7. Decontamination and quantification: symbio_quant


## 1. Before we start, let's get familiar with some Linux commands!
- `pwd` --- where are you? (prints the PATH to your current position).
- `ls` --- listing directory contents.
  - `ls -l` --- lists directory contents while displaying their characteristics  
- `cd` --- changing directories:
  - `cd Workshop` --- change working directory to the directory "Workshop" that is in the current directory
  - `cd ..` --- will move you one level up in your directory tree 
  - `cd` --- by default, typing just 'cd' will take you to your home directory
- `cp` --- copying a file, needs to be followed by the item you want to copy and a path to the directory:
  - `cp /path/to/file.txt .` --- copies a remote "file.txt" to your current working directory, symbolised by a dot (".")
  - `cp Sequences.fasta Workshop/` --- copies a file "Sequences.fasta" from the current working directory to subdirectory "Workshop"
  - `cp -r /path/to/directory ~` --- copy the whole directory/file structure recursively from a remote location to your home directory ("~")
- `mkdir` --- make a new directory
  - `mkdir MyNewDirectory` --- will create a directory with the requested name
- `mv` --- move, need to be followed by the name of the item you want to move and a path to the destination directory. Can also be used to rename items.
  - `mv file.txt Workshop/` --- will move a file to the "Workshop" subdirectory
  - `mv OldName.txt NewName.txt` --- will rename your text document
- `rm` --- delete item
  - `rm FileNotNeededAnymore.txt`
  - `rm *.fasta` --- will remove all files with the extension "fasta". Asterisk ("*") 

### ...and some additional Linux tools! 
- `screen` --- a very useful tool that can be used to multiplexes a physical console between several processes by creating virtual sessions that you can connect to or disconnect from, as desired. Generally, you want to run your proccesses within a screen to ensure processes are not disrupted by network connection issues! ):
  - `screen -S MyNewSession` --- will create a new session with the requested name. Make it informative!
  - `screen -r MyRunningSession` --- will re-attach you to your session.
  - `screen -ls` --- will list all the sessions.
  - `ctr + a + d` --- will detach you from a session, without killing it
- `htop` --- starts a 
- `gunzip` --- will uncompress your gzip-compressed files (.fastq.gz ---> .fastq) **our tools are fine with gzipped files**

**There are many more useful commands and tools - you do want to learn them!**




## 2. Copying example data to your folder (THIS IS OPTIONAL AS YOU DO NOT HAVE TO COPY ANY FILES LOCALLY).
- First, log in to your account on *azor* cluster.
- Then, copy the prepared sample data to the directory of your choosing (we recommend using your home directory):
```
cp -r /mnt/storage/users/symbio/workshop_march_2022 ~/
```
- Now you have folder "workshop_march_2022" containing R1.fastq and R2.fastq files for each sample.

- Go to the copied directory, display the contents:
```
cd ~/workshop_march_2022
ls
ls -l
```

## 3. Amplicon data analysis overview

### Dependencies
To successfully use our tools, you need some external programs and packages. Below are the programs and versions that we are using:
- _usearch v11.0.667_i86linux32_
- _vsearch v2.22.1_linux_x86_64_
- _PEAR v0.9.11_
- _Python 3.13.2_ and its packages (you can install them using the command **pip install <name_of_the_package>**):
    - os
    - sys
    - re
    - csv
    - json
    - tqdm
    - gzip
    - collections
    - time
    - logging
    - datetime
    - shutil
    - subprocess
    - pathlib
    - concurrent.futures
    - glob
    - questionary
    - numpy
    - pandas

### symbio_split:
- Splits reads of marker genes into separate subdirectories,
- Produces a summary file with the information about the number of reads for each of the targets.

### symbio_core (can be used for every marker gene in your dataset):
- Analyses each library (sample) separately,
- Merges R1 and R2 reads, passes only high-quality reads,
- Converts fastq to fasta files,
- Dereplicates and denoises samples,
- Assigns taxonomy affiliation to reads,
- Produces zOTU/OTU tables used by other scripts.

### symbio_barcode:
- Uses COI zotu table (produced by symbio_core) as an input,
- creates:
  - table with info about the most abundant COI barcode, taxonomy and bacteria presence,
  - fasta file, containing the most abundant Eucaryotic COI barcode for each library/sample,
  - table containing information about Eucaryotic COI sequences that represent at least 5% of total Eucaryotic reads per sample,
  - table containing information about bacterial COI sequences.

### symbio_quant:
- Uses as an input:
  - 16S zOTU table (produced by symbio_core),
- Decontaminates 16S data based on (automatically recognised) negative controls,
- Creates:
  - Table with zOTUs assigned as: symbiont, other or contaminants,
  - Decontaminated zOTU and OTU tables (both with reads and abundances),
  - Statistics table,
  - **optionally** quantifies your data based on the provided information about the proportion of homogenate taken for the DNA extraction and the number of spike-ins added.

### Master.info:
Most of our scripts require information from the user. Instead of re-typing them each time, we created a file containing that information for our standard regions: 
![alt text](https://github.com/Symbiosis-JU/Proof_of_Concept/blob/main/master.info.png?raw=true)

**You can create your own master.info (tab-delimited) with the following headers:**
_Target_organism	Gene	Gene_fragment	Description	Primer_Set	Primer_Forward	Primer_Reverse	Ref	Min_insert_size	Max_insert_size	Database_path_


- **Target_organism** - the info about what organisms your primers are targeting (Insect, Plant, Bacteria, Fungi),
- **Gene** - what gene do your primers target (e.g., COI, 16S, ITS),
- **Gene_fragment** - what fragment of the gene is amplified during PCR (for example, in bacterial 16S, it can be V4 or V1-V2 hypervariable region),
- **Description** - as other users might not be familiar with crazy gene names, it is a good idea to describe what a particular gene is,
- **Primer_Set** - what primer set are you using to amplify the marker gene fragment (extremely important for writing a paper),
- **Primer_Forward** - the sequence of the forward primer used in PCR, IUPAC nucleotide code for degenerate primers is welcomed, 
- **Primer_Reverse** - the same as above, but for the reverse primer (pay attention to how your data are organised - if the reverse primer is reversed complement or not), 
- **Ref** - scientific reference for the primers you are using (crucial for writing a publication),
- **Min_insert_size** - the minimum size (in bp) of your target (**without primers and Illumina adaptors**),
- **Max_insert_size** - the same as above, but with the maximum size of your insert size (remember about the constraints of the sequencing machine, e.g., 2x300 bp),
- **Database_path** - the path to your database file. 

You can create your own master.info, but please remember that if you are going to use a customised database, it has to be compatible with sintax algorithm as follows:
_**>AY846382.1.1778;tax=d:Eukaryota,p:Chloroplastida,c:Chlorophyta,o:Chlorophyceae,f:Sphaeropleales,g:Monoraphidium,s:Monoraphidium_contortum**_

### Symlinks:
**You don't have to download any scripts. They are all stored in our cluster, and as long as you have our software dictionary in your PATH**
**the only thing you need to do to run the analysis is to type the name of the tool, e.g., symbio_split and hit enter**


## 4. symbio_split - splitting libraries into datasets corresponding to different target regions

First, we want to split the data for each of the libraries into bins representing our target genes.

type **symbio_split**, press enter. What you'll see is:
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/split_start.png?raw=true)

So let's proceed with the analysis, and the script will ask what we want to do with master.info if we want to:
- see default
- use default
- use custom

Let's see what the default master.info looks like:
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/split_master.info.png?raw=true)

You can inspect the default master.info if the target of your interest is already there. 
After inspection, you can choose to either go with the default or with your custom master file.
Now the fun begins! Choose which targets you want the script to look for (use space to select):
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/split_targets_selection.png?raw=true)

Next, you need to select in which mode you want to analyse your data:
![alt tekst](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/split_mode.png?raw=true)

**A)** - all the R1 and R2 pairs in the directory will be split,

**B)** - all the R1 and R2 pairs across the dictionaries you specify will be split,

**C)** - if you want to analyse particular samples scattered across different dictionaries, choose this option. You will be asked to provide a sample list. 

Let's go with option **A** and use our Greenland sub-data (you don't have to copy any files; if you are writing the path, don't be afraid to hit tab for autofill):
![alt tekst](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/split_path.png?raw=true)

And now is the time to decide where your output folder should be located and how it should be named (by default, the output directory is time-stamped):
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/split_output.png?raw=true)

Then decide how many CPUs you want to allocate to this task, and if you are going with a dry run (no output files, just the info on how many reads are allocated to each category).
(**REMEMBER TO BOOK THE CLUSTER FIRST IN GOOGLE CALENDAR!!!**)
**ET VOILÀ!!!**
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/split_summary.png?raw=true)


Now you can see that you have four new subdirectories: **16SV1-V2  16SV4  COIBF3-BR2  unassigned  primer_binner_20260219_120817.log  primer_binner_summary.tsv**
- COIBF3-BR2 --- contains mitochondrial COI reads,
- 16SV1-V2 --- containes bacterial 16S V1-V2 reads,
- 16SV4 --- containes bacterial 16S V4 reads,
- unassigned --- with sequences that were unrecognised by the script.

**Let's proceed with 16S V4 data analysis!**

## 5. symbio_core
This is the core amplicon analysis workflow. 
This script joins F and R (R1 and R2) reads, passing only high-quality ones. 
Next, it converts fastq to fasta file, dereplicates and denoise sequences in each library separately.
Joins all the libraries into one table and assigns all the sequences to taxonomy.
**This is the first step of analysis of bacterial 16S and COI (or other) data!**

Similarly to symbio_split, the only thing you need to do to further analyse your data is to type:
**symbio_core** and hit enter! You can do it whenever you like; you don't need to be in the directory
with the data, as the script will ask you about the input and output paths. 

You will be asked similar questions as in the previous step: 
if you want to proceed, how to handle master.info and which target you are going to analyse (**this time you need to choose one!**).

Then you need to decide if you want the OUTPUT directory to be defaulted in the directory with the split data or type a custom path: 
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/core_output.png?raw=true)

And then, if you want to start the analysis from the beginning or at any of the major steps of the analysis. 
The latter is useful if, for some reason, your analysis crashed at some point, but further steps went without problems:
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/core_resume.png?raw=true)

We want to start from the beginning as we are doing our analysis for the first time!
Next, allocate how many CPUs you want to use in your analysis (**REMEMBER TO BOOK THE CLUSTER FIRST IN GOOGLE CALENDAR!!!**).
And what is the phred score you want to use for your sequences to pass the quality filter. "**What is the phred score?**" you might ask.

_A Phred score is a quantitative measure of the quality of identifying nucleobases (A, C, T, G) in DNA sequencing, representing the probability that a base call is incorrect. It ranges from 0 to 99+, where higher scores indicate greater confidence, with Q = 30 (recommended in our pipeline) indicating 99,9% accuracy._

So basically, with our thoroughly checked polymerase as well as with Illumina reads, we can confidently set the minimum Phred Quality at a minimum of 30. Therefore, our sequences would be more trustworthy.  

**And that's all!!!** 
Now the script will automatically go through all your samples, conducting all the steps listed in "Chapter 3" of this guideline.
Of course, you can inspect all the intermediate directories to check what is happening there, but your goal is to obtain two main tables:
- **OTU_table_expanded** - with the 97% simillarity clustering of your genotypes (equivalent of species).
- **zOTU_table_expanded** - with zerio-radius OTUs, each sequence that differs at least by one nucleotide will be a different zOTU.
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/core_results.png?raw=true)

As this is still contaminated data and unfiltered data, probably you are going to use the zOTU table for further steps.

## symbio_core
Type **symbio_quant** to activate the script.

First, the script will ask you to choose where to create a time-stamped output directory:
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/quant_output.png?raw=true)

Next, you have to specify where the zOTU table file is located. As you can see, you can use ./ if you happen to be in the directory
with symbio_core output:
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/quant_input.png?raw=true)

In the previous interactions of this script, you had to indicate the names of libraries that will be assigned as negative controls.
Now, we are doing that automatically. In our Lab all the negatives have a prefix "neg", so it is easy to find them. 
However, if you have negatives named differently, you can use keywords to find them:
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/quant_negs.png?raw=true)

The same situation is with positive controls:
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/quant_positiv.png?raw=true)

And to be sure, the script will summarise what negatives and positives we have chosen:
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/quant_control_sum.png?raw=true)

To quantify our bacterial loads, we need to specify which spike-in was used at the DNA extraction step.
In this case, our extraction spike-in is: Ec5502_16S:
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/quant_spike_ins.png?raw=true)

The script will also ask about which spike-in is a PCR one; we can select it, but for now, we don't need it.

Now the script will ask about various thresholds for the decontamination steps.
It is **super important** to pay attention here. You can use the default threshold, but keep in mind that each dataset 
is different and might need extra care. 

**ThresholdA - Contaminant rule**:
_A genotype is considered a **true symbiont** only if its maximum relative abundance
  in any experimental sample is more than **A** × higher than in any blank sample._

**Threshold B — 'Other' rule**:
_A genotype classified as a symbiont will instead be labelled as **Other**
  if its maximum relative abundance in experimental samples is below this value_

**Threshold C — Library removal rule**:
  _Any library (sample) with more than **C%** combined contamination + spike-in reads
  will be marked for removal from decontaminated outputs_

This is tricky if you are working with insects or samples that possibly have no microbiome. 
**IT CAN HAPPEN**, and then they will be deleted - so again, be careful.

**Threshold D — Safe-spot rule (relative abundance %)**:
  _A contaminant genotype is flagged for review if it reaches **≥D%** relative abundance
  in a large number of experimental libraries (see next threshold)_

Sometimes, due to cross-talk in negative control, you will have, for example, 3 reads of real symbiont reads and nothing more. 
Hence, it reaches 100% relative abundance in the negative sample and would be considered a contaminant.
That is why I introduced ThresholdD - to avoid such situations. 

**Threshold E — Safe-spot rule (minimum number of libraries)**:
  _The minimum number of experimental libraries **(≥E)** in which a contaminant
  must exceed the D% threshold to trigger a Safe-spot warning_

![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/quant_threshold.png?raw=true)

As _Brachybacterium_ is a known contaminant in our kits, and sometimes avoids being recognised as a contaminant,
the script will make sure to assign it as such. 

And in fact, the script warns us that some libraries are heavily contaminated. But let's keep them for further inspection:

![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/quant_warning.png?raw=true)

Next, the script asks if you also want the relative abundance tables (not only the read ones).
Sometimes you want them, and sometimes you will do it automatically in R (for example), so it's up to you. 

Now,**QUANTIFICATION** 
The script will ask for the table with the information needed for the quantification: 
number of plasmids added and the proportion of homogenate taken for DNA extraction.

![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/quant_quant_file.png?raw=true)

Here is a table for our workshop dataset:

Sample_ID	ExtractionSpikeCopies	HomogenateFraction
16SV4_GRE0619_Neg_extr	10000	0.25
16SV4_GRE0643_Neg_extr	10000	0.25
16SV4_GRE0692_Neg_PCR	10000	0.25
16SV4_GRE0882	10000	0.25
16SV4_GRE1002	10000	0.25
16SV4_GRE1092_Neg_PCR	10000	0.25
16SV4_GRE1294_Positive_4	10000	0.25
16SV4_GRE1351	10000	0.25
16SV4_GRE1775	10000	0.25
16SV4_GRE1805	10000	0.25
16SV4_GRE2059	10000	0.25
16SV4_GRE2090	10000	0.25
16SV4_GRE2091	10000	0.25
16SV4_GRE2290	10000	0.25
16SV4_GRE2313	10000	0.25
16SV4_GRE2395_Positive_5	10000	0.25

Finally, the script will ask if you want to keep your positives in the decontaminated table.

**AND DONE**
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/quant_summary.png?raw=true)

## MAO script
This script is simple, but brilliant at the same time.
It uses ```zotu_table_expanded.txt``` of COI data as an in input and produces:
- barcode.txt that contains info about most abundand COI barcode, taxonomy and bacteria presence
- barcode.fasta, containing most abundand Eucaryotic COI barcode per each library/sample
- euc5.txt, containing information about Eucaryotic COI sequences that represents at least 5% of total Eucaryotic reads per sample
- bac.txt, containing information about bacterial COI sequences

Just do our trick with creating an ampty file with ```nano MAO.py```, paste the script and close the file with saving.
Make script executable with ```chmod +x MAO.py``` and run it with ```./MAO.py zotu_table_expanded.txt```.


