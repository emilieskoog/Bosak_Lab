# For Elise

#### Below, I walk through how I blasted for genes within my MAGs in order to later create a presence/absence "heatmap" in R. 

## :memo: Acquiring gene hits from dataset

#### Before we can begin plotting our results, we need to first identify hits for each gene of interest within each MAG. I did this using  NCBI BLAST v. 2.6.0 on an HPC.

### Step 1: Place all MAGs into one directory

Drag and drop or use the command line (whatever makes you happiest!) to move all of your genomes or MAGs of interest into this folder. Navigate to this folder in your terminal. 


### Step 2: Make a file with all names of MAGs in them
Now that we are in this folder, we want to create a text file with the names of each of our MAGs. We can do this with a simple command:
```
ls > mags.txt
```
So now we have created a new file with a list of everything in this folder. However, that also includes the name of this newly created text file, in this case mags.txt. We do not want to have this text file name within this file, so we can simply "nano" into the text file 

```
nano mags.txt
```
and remove that one line (mags.txt). Make sure to save the modified file.

### Step 3: Create a database of all MAGs

Next, we want to create a BLASTable database for each of our MAGs. This following example is an array job where we are submitting many similar jobs using a single script. In my case, I have 84 genomes, so I am creating 84 jobs in a single execution. 

:exclamation: An important note, we are creating protein databases out of our NUCLEOTIDE MAG files to blast for PROTEIN queries. :exclamation: 
```
#!/bin/bash
#SBATCH -p your/partition/here      
#SBATCH -N 1                          
#SBATCH -n 1            
#SBATCH --mem=10G                     
#SBATCH --time=1:00:00                
#SBATCH -J build_db            
#SBATCH --output=dbbuild_%a.out  
#SBATCH --error=dbbuild__%a.err
#SBATCH --array=1-84 #change this last number to reflect your number of MAGs or genomes


FILENAME=$(awk "NR==$SLURM_ARRAY_TASK_ID" mags.txt) #change to reflect your job scheduling system

module add engaging/ncbi-blast/2.6.0+  #loads BLAST into your working directory

makeblastdb -in $FILENAME -parse_seqids -dbtype nucl -out $FILENAME  
```
You can visit [this site](https://www.unix.com/shell-programming-and-scripting/118605-awk-whats-nf-nr.html) to understand more about awk and NR.

### Step 4: Create a fasta file for each desired protein you are BLASTing

For example, I acquired the sulfotransferase PROTEIN sequence that I wanted to search for within each of my MAGs. 

```
>sulfo_PF13469
PIFIVGLPRSGTTLLHRLLAAHPQVRPPEETVIPILALLQSGRELRRLLDALTREDAELPHGPEECWQLLRQAFASFILEALARVSYARWLCDKSPSHLFYLDLLLRLFPDAKFIHLVRPDVISSYCSLSYSDFLDQIlARWARAYMAARARLPPDRFLDVRYEDLVADPEGTLRRIYDFLGLPWDD
```
I created a new file

```
nano sulfo.fasta
```
and pasted the above sequence into the file.

My file was in the same directory as my MAGs and mags.txt file.

### Step 5: BLAST proteins agains (nucleotide) MAGs

Once again, I created a job array while blasting (just makes things faster).
```
#!/bin/bash
#SBATCH -p your/partition/here      
#SBATCH -N 1                          
#SBATCH -n 1            
#SBATCH --mem=10G                     
#SBATCH --time=1:00:00                
#SBATCH -J blasts              
#SBATCH --output=blasts_%a.out     
#SBATCH --error=blasts__%a.err      
#SBATCH --array=1-84

module add engaging/ncbi-blast/2.6.0+  #loads BLAST into your working directory

FILENAME=$(awk "NR==$SLURM_ARRAY_TASK_ID" mags.txt) 

tblastn -query sulfo.fasta -db $FILENAME -out $FILENAME+"hits_sulfotransferase" -outfmt "7 std"
```

BLAST is now going through each of our MAG databases that we created in Step 3 and searching for our protein query, in this case sulfotransferase. And each sulfotransferase hit for each MAG will be outputted into a single file that will include the filename for each MAG and append the "+hits_sulfotransferase" to the file name. Here we chose the standard output format which includes percent identity, e-value, and many other details for each hit found. The output files will have the following structure: 


```
# TBLASTN 2.6.0+
# Query: sulfo_PF13469
# Database: SB_biofilm_MAG_10.fa
# Fields: query acc.ver, subject acc.ver, % identity, alignment length, mismatches, gap opens, q. start, q. end, s. start, s. end, evalue, bit score
# 13 hits found
sulfo_PF13469   k141_434019     32.381  210     117     6       2       186     62714   63343   1.64e-14        68.6
sulfo_PF13469   k141_117837     28.497  193     123     5       1       183     87658   87095   8.31e-09        52.0
sulfo_PF13469   k141_117837     31.111  45      31      0       50      94      67863   67729   1.3     27.3
sulfo_PF13469   k141_221753     46.667  30      16      0       156     185     22597   22508   0.011   33.5
sulfo_PF13469   k141_51103      45.161  31      17      0       1       31      12128   12220   0.24    29.6
sulfo_PF13469   k141_51103      54.545  22      10      0       4       25      6887    6952    2.6     26.2
sulfo_PF13469   k141_51103      27.586  29      21      0       157     185     12686   12772   3.7     25.8
sulfo_PF13469   k141_51103      45.455  22      12      0       4       25      5825    5890    3.8     25.8
sulfo_PF13469   k141_81607      39.394  33      20      0       153     185     52643   52545   0.27    29.3
sulfo_PF13469   k141_386313     57.895  19      8       0       2       20      95397   95453   2.8     26.2
sulfo_PF13469   k141_546304     33.333  24      16      0       2       25      32506   32435   3.1     26.2
sulfo_PF13469   k141_291400     56.250  16      7       0       2       17      186     139     4.5     25.4
sulfo_PF13469   k141_282143     44.000  25      14      0       2       26      22593   22667   8.9     24.6
# BLAST processed 1 queries
```