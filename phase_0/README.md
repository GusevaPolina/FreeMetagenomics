# Phase 0: from raw reads to scaffolds and mapped reads

This part of the pipeline is missing in [the original repository](https://github.com/SushiLab/magpipe). 
> However, there is a description of this in the article ([Paoli _et al._ 2022](https://www.nature.com/articles/s41586-022-04862-3#Sec7)). It would be provided for each part.

So far, this phase could be split up into several steps:
1. removing adapters, a control as of the PhiX genome, and low-quality reads;
2. merging forward and reverse reads;
3. normalising obtained files (forward, reverse, merged, and unmerged reads)
4. performing an assembly of scaffolds;
5. mapping filtered reads to scaffolds.


## Step 1: cleaning reads
> Sequencing reads from all metagenomes were quality filtered using BBMap (v.38.71) by removing sequencing adapters from the reads, 
> removing reads that mapped to quality control sequences (PhiX genome) and discarding low quality reads using the parameters trimq = 14, maq = 20, maxns = 0 and minlength = 45.

Following the original description, the first step could be performed as:
```bash
# Quality filtering using BBMap
bbduk.sh in=input.fastq out=filtered.fastq ref=adapters.fa ktrim=r k=23 mink=11 hdist=1 tpe tbo qtrim=rl trimq=14 maq=20 maxns=0 minlength=45
# Removing PhiX genome reads
bbduk.sh in=filtered.fastq out=filtered_phix_removed.fastq ref=phix_genome.fa k=31 hdist=1 stats=stats.txt
```

However, this requeres `adapters.fa` which is not provided. Therefore, it was replaced by:
```bash
# Removing adapters using illumina-utils
pip install illumina-utils
# Creating *.ini for the following step
iu-gen-configs samples.txt
# Filtering reads and merging all attempts into one forward read and one reverse read
iu-filter-quality-minoche sample_name.ini --ignore-deflines
```
Around 1 hour for total of ~25 Gb of reads, <32 Gb RAM.

The `samples.txt` TAB-separated file from the TARA OCEAN project could be found by the following [link](http://merenlab.org/data/tara-oceans-mags/files/samples.txt). It looks like:
```
sample	r1	r2
sample_name	reads1_1.fastq.gz,reads2_1.fastq.gz	reads1_2.fastq.gz,reads2_2.fastq.gz
```

Afterwards, the clean reads still should be filtered:
```bash
# Removing low-quality reads
bbduk.sh in1=sample_name-QUALITY_PASSED_R1.fastq in2=sample_name-QUALITY_PASSED_R2.fastq out1=sample_name_filtered_R1.fastq out2=sample_name_filtered_R2.fastq qtrim=rl trimq=14 maq=20 maxns=0 minlength=45
```
Around 2.5 min, for forward and reverse of ~30 Gb each, <32 Gb RAM.


And the control should be removed too:
```bash
# Removing a control
bbmap.sh in=sample_name_filtered_*.fastq out=*_phix_removed.fastq ref=phix.fa nodisk
```
Around 2 min, a file of 30 Gb, <32 Gb RAM.

## Step 2: merging
> Downstream analyses were performed on quality-controlled reads or if specified, merged quality-controlled reads (bbmerge.sh minoverlap = 16).
```bash
# Merging forward and reverse reads
bbmerge.sh in1=R1_phix_removed.fastq in2=R2_phix_removed.fastq out=merged.fastq outu=unmerged.fastq ihist=histogram.txt minoverlap=16
```
Around 4 min, for forward and reverse of ~30 Gb each, <32 Gb RAM.

## Step 3: normalising
> Quality-controlled reads were normalized (bbnorm.sh target = 40, mindepth = 0)

All files should be normalised (forward, reverse, meerged, and unmerged). The code example is for `merged.fastq`:
```bash
# Normalise reads
bbnorm.sh in=merged.fastq out=merged_normalized.fastq target=40 mindepth=0
```
Around 20 min for ~30Gb, <32 Gb RAM. Time increases linearly.

## Step 4: assembling scaffolds
> ... they were assembled with metaSPAdes (v.3.11.1 or v.3.12 if required). The resulting scaffolded contigs (hereafter scaffolds) were finally filtered by length (≥1 kb).

As even more modern versions of `metaSPAdes` require around 36-48 hours of run with at least 512 Gb RAM, so it is replaced by `MEGAHIT` (v.1.2.9) as a less resource-intensive alternative. Then, there is no need in merged and unmerged files.

```bash
# Assembling reads into scaffolds 
megahit -1 forward_normalized.fastq  -2 reverse_normalized.fastq --min-contig-len 1000 -o output_folder --num-cpu-threads 80
```
Around 7 hours for input of 27 Gb forward and reverse reads each, with ~256 Gb RAM.

## Step 5: mapping
> ... metagenomic samples were grouped into several sets and, for each sample set, the quality-controlled metagenomic reads from all samples were individually mapped against the scaffolds of each sample, resulting in the following numbers of pairwise readset to scaffold mappings
> Mapping was performed using the Burrows–Wheeler-Aligner (BWA) (v.0.7.17-r1188)54, allowing the reads to map at secondary sites (with the -a flag). 
> Alignments were filtered to be at least 45 bases in length, with an identity of ≥97% and covering ≥80% of the read sequence. 

```bash
# Indexing the reference
bwa index scaffold.fa                 # 3 min
# Mapping reads onto the reference
bwa mem -a -o mapped.sam scaffold.fa forward_normalized.fastq reverse_normalized.fastq
# Transforming SAM into BAM
samtools view -F 4 -bS -o mapped.bam mapped.sam
# Removing reads with insufficient mapping
samtools view -h mapped.bam | awk -F '\t' 'length($10) >= 45 && $3 >= 97 && ($4/length($10)) >= 0.8' | samtools view -bS - > mapped_filtered.bam
# Sorting the final file
samtools sort -o mapped_sorted.bam mapped_filtered.bam
```

Overall, the whole step is performed in 3 hours for a file with scaffolds of ~200 Mb and total reads of 60 Gb, <256 Gb RAM.