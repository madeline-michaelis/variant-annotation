# Long-read Variant Calling Annotation Pipeline

# Purpose
The goal of this pipeline is to perform variant calling on long-read (e.g. Oxford Nanopore ONT technologies) sequencing data against a reference bacterial genome of interests. The program generated multiple alignment files in the .SAM and .BAM format. Additionally, it includes two steps to functionally annotate the bacterial genomes using multiple reference genome databases. This can help match in which genes the variants tends to occur.

# Cyberinfrastructure & Implementation

These scripts are meant to be run by HTCondor, a workflow manager that takes in an executable file and a submit file. They are meant to be run on the UW-Madison shared campus-computing infrastructure CHTC, but could work on other systems with few modifications. The pipeline takes advantage of the high-throughput scaling abilities of HTCondor to submit multiple jobs at the same time. For example, if we had an experimental design of 3 treatment, 3 replicate and 3 reference genomes and 10 time points, we would have to perform the steps 3 * 3 * 3 * 10 = 270 times. Instead, we write the metadata (information about the sample design) in a comma separated file containing 270 rows, and HTCondor will submit 270 jobs at the same time for us.

# Workflow steps

![Workflow design](https://github.com/UW-Madison-Bacteriology-Bioinformatics/variant-annotation/blob/main/workflow.png)

1. Filters using `Filtlong v0.2.1`, keeping 95% of best reads with the flag `-p 95`.

2. Maps the filtered data to reference genomes using `minimap2 2.22-r1101`, a mapper for long-read sequences. It uses the flag `-x map-ont`.

3. Using samtools `samtools 1.13` to perform a alignment file format conversions.

4. Uses `Bakta version 10.0.3` to functionally annotate the reference genomes.

5. Uses `EGGNOG mapper 2.1.12` to annotate the bakta-annotated proteins (`.faa`) into different functional annotations like COG, KEGG, etc.

> [!NOTE]
> You can run steps 4 and 5 without running steps 1, 2, and 3.

# Repository files

This repository contains 2 folders: `recipes` and `scripts`.
The `recipes` folder contains the Apptainer definition files needed to create the Apptainer sif files. 
The `scripts` folder contains the HTcondor `.sh` and `.sub` files.

# Input folder set-up
The folder structure is a main folder, with a subfolder called "reads", and any reference file with the format {REFERENCE}_assembly.fasta

For example:
```
folder
-- reads
---- sample1.fastq.gz
---- sample2.fastq.gz
---- sample3.fastq.gz
reference1_assembly.fasta
reference2_assembly.fasta
reference3_assembly.fasta
```

# Expected output directory structure
The program will create the following files/output, namely 2 folders `filtlong` and `mapping` with all the relevant output files.
```
folder
-- reads
---- sample1.fastq.gz
---- sample2.fastq.gz
---- sample3.fastq.gz
reference1_assembly.fasta
reference2_assembly.fasta
reference3_assembly.fasta
-- filtlong
---- {sample}_filtlong.fastq.gz
-- mapping
---- .sam, .bam, .coverage.tsv, .mapped.bam, .mapped.sorted
```

# Instructions

Log into chtc:
```
ssh [netid]@ap2001.chtc.wisc.edu
# enter password
```

Clone this directory:
```
git clone https://github.com/patriciatran/variant-annotation.git
cd variant-annotation
```

Make the scripts executable:
```
chmod +x scripts/*.sh
```

##  Building containers

To build the software containers, you will need to start an interactive job, build the container, test it, and move it to a location accessible by the working nodes (e.g. staging, not home).
For detailed instructions, visit https://github.com/UW-Madison-Bacteriology-Bioinformatics/chtc-containers. 

Brief instructions:
```
cd recipes
nano build.sub
# change the file listed in the transfer_input_files line
condor_submit -i build.sub
# replace "container" with the name of your choice
apptainer build container.sif container.def
apptainer shell -e container.sif
# test container by typing the -h --help command.
exit
mv container.sif /staging/netid/apptainer/.
exit
cd ..
```

## Run code

Enter the scripts directory, and create 2 metadata tables (comma-separated values).
The first metadata table (`Samples_and_Ref.txt`) should contain 3 columns: the sample name, the reference name, and the path to the staging folder containing these files.
The 2nd metadata table (`references.txt`) should contain 2 columns: the reference name, and the path to the staging folder containing these files.

```
cd scripts
nano Samples_and_Ref.txt
# enter your data, quit, save
nano references.txt
# enter your data, quit, save.
```

Submit your htcondor jobs:
```
condor_submit 01_filtlong.sub
```
Repeat for 02, 03, 04 and 05.

## Next steps
This workflow will create large files. I recommend using Globus.org to transfer files to your ResearchDrive or to your personal endpoint.
For instructions, please visit: https://chtc.cs.wisc.edu/uw-research-computing/globus

## Importing to Geneious for SNP Identification

If you plan on using this in Geneious to use their SNP identification tool I suggest these steps:
1) Make a folder for each reference.
2) Load the gbff file from bakta - this ensures that you will have the gene annotations.
3) Import the sorted.bam files from Step 03 (samtools output) to the corresponding folders.
4) Use the SNP identification tool, minimum cov 10x and 95% coverage.
5) Once you have the output click on Annotations > Variants, and Columns > Manage Columns. Make sure all columns, including the gene names, are shown in the table. Export to CSV or TSV.
6) Use Python or R or process all the tables.

## References
If you find this pipeline helpful, please cite this GitHub Repository:
1. Tran, P. Q. (2025). variant-annotation (Version ???) \[Computer software\]. https://github.com/UW-Madison-Bacteriology-Bioinformatics/variant-annotation/

This workflow relies on the following softwares, please cite them as well:

1. Filtlong: Wick, R. (2017). *Filtlong* [Computer software]. https://github.com/rrwick/Filtlong
2. Minimap: Li, H. (2018). *Minimap2: pairwise alignment for nucleotide sequences*. Bioinformatics, 34(18), 3094–3100. https://doi.org/10.1093/bioinformatics/bty191
3. Samtools: Li, H., Handsaker, B., Wysoker, A., et al. (2009). *The Sequence Alignment/Map format and SAMtools*. Bioinformatics, 25(16), 2078–2079. https://doi.org/10.1093/bioinformatics/btp352
4. Bakta: Schwengers O, Jelonek L, Alfred D, Goesmann A (2021) *Bakta: rapid and standardized annotation of bacterial genomes via alignment-free sequence identification.* Microbial Genomics, 7(11). https://doi.org/10.1099/mgen.0.000685
5. Eggnog-mapper: Cantalapiedra CP, Hernández-Plaza A, Letunic I, Bork P, Huerta-Cepas J. (2021). *eggNOG-mapper v2: Functional Annotation, Orthology Assignments, and Domain Prediction at the Metagenomic Scale.* Mol Biol Evol, 38(12):5825–5829. https://doi.org/10.1093/molbev/msab293

## Help and additional information:
Patricia Q. Tran, ptran5@wisc.edu, University of Wisconsin-Madison Get Help:
- For people at UW-Madison, please visit the [departmental bioinformatics research support service main website](https://bioinformatics.bact.wisc.edu/). If you are part of the Department of Bacteriology please make an 1-on-1 individual appointment, others please attend one of my weekly office hours.
- For external people, please submit an issue via the github page.
