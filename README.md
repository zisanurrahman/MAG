# metag-rev-sup

#Supplementary materials for Quince et al. Metagenome analysis Review

<a name="assembly"/>
##Assembly based metagenomics analysis

This analysis regenerates the results in the left panel of Figure 3. The results will not 
be identical due to some differences in software principally the switch from idba_ud to megahit
for the assembly step. This was changed for reasons of efficiency.

We took twenty samples from healthy control children and those with Crohn's disease from a 
larger study looking at the effect of a treatment (EEN) on the Crohn's microbiota 
[Quince et al. 2015] (https://www.ncbi.nlm.nih.gov/pubmed/26526081).
The sequences comprised a combination of MiSeq 2X250bp and HiSeq 2X150bp paired end reads.

Reads were trimmed and filtered with sickle to remove sequencing adaptors and 
regions with an average quality of < 20. Human DNA was then removed using DeconSeq. 
The minimum and maximum read numbers following trimming and filtering 
were 2,456,000 and 14,720,000 respectively with a median of 7,797,000.

We are now going to perform a basic assembly based metagenomics analysis of these same samples. This will involve 
a collection of different software programs:

1. [megahit](https://github.com/voutcn/megahit): A highly efficient metagenomics assembler currently our default for most studies

2. [bwa](https://github.com/lh3/bwa): Necessary for mapping reads onto contigs

3. [samtools] (http://www.htslib.org/download/): Utilities for processing mapped files

4. [CONCOCT](https://github.com/BinPro/CONCOCT): Our own contig binning algorithm

5. [prodigal] (https://github.com/hyattpd/prodigal/releases/): Used for calling genes on contigs

6. [gnu parallel] (http://www.gnu.org/software/parallel/): Used for parallelising rps-blast

7. [standalone blast] (http://www.ncbi.nlm.nih.gov/books/NBK52640/): Needs rps-blast

8. [COG RPS database] (ftp://ftp.ncbi.nih.gov/pub/mmdb/cdd/little_endian/): Cog databases

9. [GFF python parser] (https://github.com/chapmanb/bcbb/tree/master/gff)


###Co-assembly

Create an example directory to work in:
```
mkdir ~/Example
cd ~/Example
```

Begin by downloading the reads:
```
wget ****
tar -xvzf Reads.tar.gz
```
We will then perform a co-assembly of these samples using megahit. First we 
need to get a comma separated list of forward and reverse read files:
```
ls Reads/*R1*fasta | tr "\n" "," | sed 's/,$//' > R1.csv
ls Reads/*R2*fasta | tr "\n" "," | sed 's/,$//' > R2.csv
```
Then set assembler running may take several hours...
```
nohup megahit -1 $(<R1.csv) -2 $(<R2.csv) -t 96 -o Assembly --presets meta > megahit.out&
```

We can have a look at how good the assembly was:
```
contig-stats.pl < Assembly/final.contigs.fa
```
Output should be similar too:
```
sequence #: 1316028	total length: 964326614	max length: 318273	N50: 1088	N90: 299
```
Not bad, a N50 of 1088bp. All scripts are available in this repo in the scripts dir. They will 
need to be added to your path though.

We will now perform CONCOCT binning of these contigs. As explained in 
[Alneberg et al.](http://www.nature.com/nmeth/journal/v11/n11/full/nmeth.3103.html) 
there are good reasons to cut up contigs prior to binning. We will use a script from 
CONCOCT to do this. For convenience we 
will create an environmental variables that points to the CONCOCT install directory 
change as appropriate:
```
export CONCOCT=~/Installed/CONCOCT
```

###Mapping

First we cut up contigs and place in new dir:

```bash
mkdir contigs
python $CONCOCT/scripts/cut_up_fasta.py -c 10000 -o 0 -m Assembly/final.contigs.fa > contigs/final_contigs_c10K.fa
```

Having cut-up the contigs the next step is to map all the reads from each sample back onto them. 
First index the contigs with bwa:

```bash
cd contigs
bwa index final_contigs_c10K.fa
cd ..
```

Then perform the actual mapping you may want to put this in a shell script and adjust the number of threads 
according to your resources:

```bash
mkdir Map

for file in Reads/*R1.fasta
do
    stub=${file%_R1.fasta}
    stub=${stub#Reads\/}
    
    bwa mem -t 64 contigs/final_contigs_c10K.fa $file Reads/${stub}_R2.fasta > Map/${stub}.sam
    echo $stub
done
```

Then we need to calculate our contig lengths.

```bash
python ~/bin/Lengths.py -i contigs/final_contigs_c10K.fa > contigs/final_contigs_c10K.len
```

Then we calculate coverages for each contig in each sample:

```bash
for file in Map/*.sam
do
    stub=${file%.sam}
    stub2=${stub#Map\/}
    echo $stub	
    samtools view -h -b -S $file > ${stub}.bam 
    samtools view -b -F 4 ${stub}.bam > ${stub}.mapped.bam
    samtools sort ${stub}.mapped.bam -o ${stub}.mapped.sorted
    bedtools genomecov -ibam ${stub}.mapped.sorted -g contigs/final_contigs_c10K.len > ${stub}_cov.txt
done
```

and use awk to aggregate the output of bedtools:

```bash
for i in Map/*_cov.txt 
do 
   echo $i
   stub=${i%_cov.txt}
   stub=${stub#Map\/}
   echo $stub
   awk -F"\t" '{l[$1]=l[$1]+($2 *$3);r[$1]=$4} END {for (i in l){print i","(l[i]/r[i])}}' $i > Map/${stub}_cov.csv
done
```

and finally run the following perl script to collate the coverages across samples, where we have simply adjusted the format 
from csv to tsv to be compatible with CONCOCT:

```bash
Collate.pl Map | tr "," "\t" > Coverage.tsv
```


We will only run concoct with default settings here. The only one we 
vary is the cluster number which should be at least twice the number of genomes in your 
co-assembly. 

```
mkdir Concoct
cd Concoct
mv ../Coverage.tsv .
concoct --coverage_file Coverage.tsv --composition_file ../contigs/final_contigs_c10K.fa 
cd ..
```

#Annotate genes on contigs


First we call genes on the contigs using prodigal.

```
mkdir Annotate
cd Annotate/
LengthFilter.pl ../contigs/final_contigs_c10K.fa 1000 > final_contigs_gt1000_c10K.fa
prodigal -i final_contigs_gt1000_c10K.fa -a final_contigs_gt1000_c10K.faa -d final_contigs_gt1000_c10K.fna  -f gff -p meta -o final_contigs_gt1000_c10K.gff
```

Then we assign COGs with RPSBlast. The script in the CONCOCT distro for doing this requires 
an environment variable pointing to the COG database to be set. We also set another variable 
pointing to the CONCOCT scripts for convenience:

```
export COGSDB_DIR=/home/opt/rpsblast_db
export CONCOCT=/home/chris/Installed/CONCOCT
```

Then we run this on 32 cores:
```
$CONCOCT/scripts/RPSBLAST.sh -f final_contigs_gt1000_c10K.faa -p -c 32 -r 1
```

This allows us to evaluate clusters based on single-copy core genes:
Then generate SCG frequencies in clusters:
```
$CONCOCT/scripts/COG_table.py -b final_contigs_gt1000_c10K.out -m $CONCOCT/scgs/scg_cogs_min0.97_max1.03_unique_genera.txt -c ../Concoct/clustering_gt1000.csv --cdd_cog_file $CONCOCT/scgs/cdd_to_cog.tsv > clustering_gt1000_scg.tsv
```

and a pdf:
```
$CONCOCT/scripts/COGPlot.R -s clustering_gt1000_scg.tsv -o clustering_gt1000_scg.pdf
```

54 clusters 75% pure and complete

![Single copy core gene plot](./Figures/clustering_gt1000_scg.pdf)
