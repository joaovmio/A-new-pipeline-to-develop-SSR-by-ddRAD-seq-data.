# A-new-pipeline-to-develop-SSR-by-ddRAD-seq-data
### This pipeline was made to develop molecular markers of microsatellites. Here, you will find the code and instructions on how to use it. 

This pipeline was developed using ddRAD-seq (double digestion restriction-site associated DNA sequence) data from Agave sp accessions and the reference genome. This code crosses these two types of information, first generating the microsatellites (SSR), then filtering only the reads containing these markers and finally filtering again only the polymorphic regions in relation to the reference genome, identifying insertions and deletions using the CIGAR alignment pattern. The design of primers for the in vitro validation of polymorphic regions can be done using the PRIMER3 platform (recommended). 

![image](https://github.com/user-attachments/assets/fd882611-bfb5-4544-ad7e-747f8bb46a4c)


## 1. Microsatellite marker identification with MISA (Beier et al., 2017)
This is the first step in identifying microsatellite markers. The software used is the Microsatellite Identification tool (MISA), provided by the Leibniz Institute of Plant Genetics and Crop Plant Research (IPK) through the platform: [MISA Web](https://webblast.ipk-gatersleben.de/misa/).

After installation, you can easily execute it in your terminal with the following command:
```bash
perl misa.pl reference_genome.fa > reference_genome.misa
```
For this approach, microsatellites categorized as "p1" (mononucleotides) and "c" (compound microsatellites) were removed. To filter them, run the following command in the terminal:
```bash
grep -Ev "c|p1" reference_genome.fa.misa > reference_genome_filter.misa
```

## 1.1 Flanking primer design with PRIMER3 (Untergasser et al., 2012)
At this stage, you can generate flanking primers for the microsatellite sequences obtained with MISA by running the program in your terminal or by accessing PRIMER3 through a web browser to design primers after selecting polymorphic regions at the end of this pipeline.

Execution via terminal:
Generating the input file for Primer3:
```bash
perl p3_in_alterado.pl reference_genome.misa
```
Running Primer3 to design primers:
```bash
your/path/primer3 < reference_genome.fasta.p3in > reference_genome.fasta.p3out
```
Processing the generated primers:
```bash
perl p3_out.pl reference_genome.fasta.p3out reference_genome.fasta.misa
```

## 2. Alignment of ddRAD reads to SSR loci
### 2.1 Filtering genome sequences to retain only SSR-containing regions
First, filter the sequences in `reference_genome.fasta` using the SSR-containing sequences from the `.misa` file. At the end of this step, you will have a FASTA file containing only the genome regions with microsatellite markers.

### 2.2 Read alignment with BWA (Li & Durbin, 2009)
For this pipeline, the BWA aligner was used to map the reference genome with ddRAD sequencing data. SAMTools was used to manipulate `.sam` and `.bam` files.
```bash
# Indexing the genome (only SSR regions are used)
bwa index GENOME.fa

# Aligning reads against the genome (loop for multiple files)
for i in /path/to/reads/*.gz ; do bwa mem loci_ssr_ah_sp1_scomp.txt.fasta ${i} -t 10 > ${i}.sam; done

# Converting .SAM to .BAM using SAMTools (reducing memory usage)
samtools view -bS -o [OUT.bam] [IN.sam]

# Converting from SAM to BAM (loop for all files)
for i in *.bt2.sam; do samtools view -bS -o ${i}.bam ${i}; done

# Sorting BAM files
samtools sort [IN.bam] -o [out_sorted.bam]

# Sorting BAM files in a loop
for i in *.bam; do samtools sort ${i} -o ${i}_sorted.bam; done
```

### 2.3 Converting BAM files to BED format using Bedtools (Quinlan & Hall, 2010)
```bash
for file in *.bam ; do bedtools bamtobed -i $file -cigar > $file.bed ; done
```

### 3. Identifying SSR Loci in ddRAD Data

For identifying ddRAD-seq reads containing microsatellites, the subtract module from Bedtools is used. The schematic below summarizes this step, and at the end, you obtain the start and end positions of reads containing microsatellites within the sequences.

The code was developed in Python (ssr_ddrad.py) and it is highly recommended to execute it using parallel for time optimization and processing efficiency of each microsatellite. The file ssr.misa is the input file containing the microsatellites, and list_beds.txt contains the paths and names of the BED files corresponding to the sequenced individuals.

To run the analysis, use the following command:

```bash
parallel -a ssr.misa --delay 0.2 -j 50 --joblog "parallel.log" --resume-failed srun -c 1 --mem="20G" python3 ssr_ddrad.py '{}' list_beds.txt
```

Explanation of Parameters:

```bash
-a # → Input file for parallel

--delay # → Time interval between job submissions

-j # → Number of jobs executed simultaneously

--joblog # → Log file storing job execution details

--resume-failed # → Helps resume jobs in case of unexpected interruptions
```

