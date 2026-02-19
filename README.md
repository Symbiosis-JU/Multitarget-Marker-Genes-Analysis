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

**Let's use some of those beautiful commands in action!**


## 2. Copying example data to your folder.
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
  - 16S zotu table (produced by symbio_core),
- Decontaminates 16S data based on (automatically recognised) negative controls,
- Creates:
  - Table with zOTUs assigned as: symbiont, other or contaminants,
  - Decontaminated zotu and otu tables (both with reads and abundances),
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
**You don't have to download any scripts. They all are stored in our cluster, and as long as you have our software dictionary in your PATH**
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
**ET VOILÀ!!!**
![alt text](https://github.com/Symbiosis-JU/Multitarget-Marker-Genes-Analysis/blob/main/split_summary.png?raw=true)


Now you can see that you have four new subdirectories: **16SV1-V2  16SV4  COIBF3-BR2  unassigned  primer_binner_20260219_120817.log  primer_binner_summary.tsv**
- COIBF3-BR2 --- contains mitochondrial COI reads,
- 16SV1-V2 --- containes bacterial 16S V1-V2 reads,
- 16SV4 --- containes bacterial 16S V4 reads,
- unassigned --- with sequences that were unrecognised by the script.

**Let's proceed with COI data analysis!**


## 5. symbio_core
This is the core amplicon analysis workflow. 
This script joins F and R (R1 and R2) reads, passing only high-quality ones. 
Next, it converts fastq to fasta file, dereplicates and denoise sequences in each library separately.
Joins all the libraries into one table and assigns all the sequences to taxonomy.
**This is the first step of analysis of bacterial 16S and COI (or other) data!**



## MAO script
This script is simple, but brilliant at the same time.
It uses ```zotu_table_expanded.txt``` of COI data as an in input and produces:
- barcode.txt that contains info about most abundand COI barcode, taxonomy and bacteria presence
- barcode.fasta, containing most abundand Eucaryotic COI barcode per each library/sample
- euc5.txt, containing information about Eucaryotic COI sequences that represents at least 5% of total Eucaryotic reads per sample
- bac.txt, containing information about bacterial COI sequences

Just do our trick with creating an ampty file with ```nano MAO.py```, paste the script and close the file with saving.
Make script executable with ```chmod +x MAO.py``` and run it with ```./MAO.py zotu_table_expanded.txt```.

## QUACK script
This script is used for decontamination of 16S data.
**Be aware that we are working with insects. If you are working with a different data type, consider applying some changes to fit your needs.**
For example, this script deletes all reads characterized as chloroplasts by default, so if you are working with lichens, that may cause a lot of damage (happened before).

**Before running this script you need to run LSD for you 16S data (eg. V4)**
Go to the directory with V4 reads:
```
cd ~/workshop_march_2022/split/V4_trimmed
```
Create a **sample_list** in similar manner as with COI data:
```
for file in *_F_V4.fastq; do
    SampleName=`basename $file _F_V4.fastq `
    SampleNameMod=$(echo "$SampleName" | sed 's/-/_/g' | sed 's/_S[0-9]\+$//g')
    echo $SampleNameMod "$SampleName"_F_V4.fastq "$SampleName"_R_V4.fastq >> sample_list_V4.txt
done
```
You can copy the LSD script used for COI data, as it is the same for 16S:
```
cp ~/workshop_march_2022/split/COI_trimmed/LSD.py ~/workshop_march_2022/split/V4_trimmed
```
**Remember that when you copy executable script, you don't need to make it executable again**

Now, run LSD for you 16S V4:
```
./LSD.py sample_list_V4.txt ~/workshop_march_2022/split/V4_trimmed 16SV4
```

OK, we have all inputs to run **QUACK**

To run Quack you need:
- the actual script! Copy it from [QUACK.py](https://github.com/Symbiosis-JU/Bioinformatic-pipelines/blob/main/QUACK.py) and change permissions to make executable, as before!
- zotu table produced by LSD (zotu_table_expanded.txt),
- otus.tax, also produced by LSD,
- list of blanks --- tab separated text file with names of blank (negative control) libraries with description (PCR/Extraction_blank). 
In our case, you may want to create a file "blank_list.txt" that looks like this:
```
NegExtr_GRE0619  blank_extr
NegExtr_GRE0643  blank_extr
NegPCR_GRE0692  blank_PCR
NegPCR_GRE1092	blank_PCR
```
...or perhaps like this? Depending on your actual sample names. Make sure that they are correct!
```
GRE0619_Neg_extr  blank_extr
GRE0643_Neg_extr	blank_extr
GRE0692_Neg_PCR blank_PCR
GRE1092_Neg_PCR blank_PCR
```

- list of spikeins used --- tab separated text file with names of used spikeins with description (PCR/Extraction_spikein).
In our case, you may want to create a file "spikeins.txt" and with the contents as below.
```
Ec5502	Extr_Spikein
Ec5001	PCR_Spikein
```
- ThresholdA --- a unique genotype will be assigned as a contaminant UNLESS the maximum relative abundance it attains in at least one experimental library is more than ThresholdA * of the maximum relative abundance it attains in any blank library. Recommended value: 10
- ThresholdB --- a unique genotype assigned previously as a symbiont will be assigned as "Other" UNLESS the maximum relative abundance it attains in at least one experimental library is more than ThresholdB. Recommended value: 0.001
- ThresholdC --- a library will be deleted UNLESS the % of contamination will be lower than ThresholdC. Recommended value: 30

OK, everything checked? Let's run this baby!:
```
./QUACK.py zotu_table_expanded.txt blank_list.txt spikein.txt otus.tax 10 0.001 30
```
If everything went well, script should print to a screen following message:
```
				---------- WELCOME TO QUACK (Beta_version): ----

					Q -Quantification 

					U - Utility 

					A - And 

					C - Contamination 

					K - Killer 

				-------------------------------------------------

Opening OTU table..................... OK!
Opening List of blanks..................... OK!
Blank list proceeded succesfully,
I am going to use 2 of PCR blanks:
GRE0692_Neg_PCR, GRE1092_Neg_PCR, 

I am going to use 3 of Extraction blanks:
GRE0619_Neg_extr, GRE0643_Neg_extr, GRE0667_Neg_extr, 
Opening List of Spikeins..................... OK!

Spikein list proceeded succesfully!
I am going to use following as PCR spikein:  Ec5001
I am going to use following as Extraction spikein:  Ec5502
Searching for Non-bacteria taxa..................... 
Chimeras, Eukaryota, Chloroplast, Mitochondria and Archea reads has been recognized!
Saving Table with classes....................... OK!
Saving Table with statistics....................... OK!
Saving Decontaminated zOTU Table....................... OK!
Adding sequences...................... OK!
Saving Decontaminated OTU Table....................... OK!

DONE! May QUACK bring you luck!
         ,-.
       ,--' ~.).
     ,'         `.
    ; (((__   __)))
    ;  ( (#) ( (#)
    |   \_/___\_/|
   ,"  ,-'    `__".
  (   ( ._   ____`.)--._        _
   `._ `-.`-' \(`-'  _  `-. _,-' `-/`.
    ,')   `.`._))  ,' `.   `.  ,','  ;
  .'  .     `--'  /     ).   `.      ;
 ;     `-        /     '  )         ;
 \                       ')       ,'
  \                     ,'       ;
   \               `~~~'       ,'
    `.                      _,'
      `.                ,--'
        `-._________,--') 
```

   
