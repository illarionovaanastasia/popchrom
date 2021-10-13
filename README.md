# Reference Panel Generator for Diverse Sequencing Data Analysis

<img align="right" width="320" height="230" src="https://github.com/collaborativebioinformatics/RPG_Pikachu/blob/main/others/RPG.png">

## Contributors 
-  Elbay Aliyev - Lead 
-  Muhammad Sohail Raza - Writer
-  Shangzhe Zhang, Anastasia Illarionova, Prashant Ranjan, Tiancheng Xu, Aditi Sammi - Tech support 
-  Bryce Kille, ChunHsuan LO - SysAdmin

## Introduction 

After two decades of refinements, the human reference genome (GRCh38) has become more accurate and mostly complete. However, there are still hundreds of unresolved gaps persist, and no single chromosome has been finished from end to end because of the existence of highly repeated sequences (called transposable elements). Foutunatedly, by using the high-coverage & ultra-long-read technologies, several scientific groups have presented a new human genome assembly that surpasses the continuity of GRCh38, along with a gapless, telomere-to-telomere assembly of a human chromosome, which is called CHM13 reference genome.

## Goal 

Designing a portable pippeline which performs the following steps (the tool can be applied to any genomes in fasta format and any VCF files)
- To introduce common variants into CHM13 reference genome.
- To propose a more representative reference genome by 1000 Genomes Project.
- To screen out in-frame stop codons sites that disagree with Ribo-seq analysis for validting the annotation.

## Flowchart

<img width="1200" alt="flowchart" src="https://github.com/collaborativebioinformatics/popchrom/blob/main/others/flowchart_version6.png">

Credits: Anastasia Illarionova
## Usage

### Dependencies:

- conda (python3)
- snakemake
- [ClinVar](https://www.ncbi.nlm.nih.gov/clinvar/)
- [bcftools](https://samtools.github.io/bcftools/)
- [gffread](https://github.com/gpertea/gffread)
- [GATK](https://gatk.broadinstitute.org)

### Input:

- vcf files (using CHM13_v1.0 as reference already)

  https://s3-us-west-2.amazonaws.com/human-pangenomics/index.html?prefix=T2T/CHM13/assemblies/variants/1000_Genomes_Project/by_chr/
- CHM13_v1.0 Fasta file

  https://s3-us-west-2.amazonaws.com/human-pangenomics/T2T/CHM13/assemblies/chm13.draft_v1.0.fasta.gz

- CHM13_v1.0 GFF file

  https://s3-us-west-2.amazonaws.com/human-pangenomics/T2T/CHM13/assemblies/annotation/chm13.draft_v1.0.gene_annotation.v4.gff3.gz

### Outputs: 

- Positions of in-frame stop codons in CHRM13 reference sequence
- New reference Fasta file (for all samples in 1GP)
- (Optional) New reference Fasta files for 5 subpopulations
- ClinVar annotation of common alleles

### Set up:

- Install snakemake

```
conda install -n base -c conda-forge mamba
mamba create -c conda-forge -c bioconda -n snakemake snakemake
```
- Clone the Git repo

```
git clone https://github.com/collaborativebioinformatics/RPG_Pikachu.git
```
- Go inside the RPG_Pikachu

```
cd RPG_Pikachu
```

- Dry run (See what will happen)

```
snakemake -np
```

- Use snakemake to run the workfolw

```
snakemake --cores 10 --use-conda
```

## Detailed Workflow and demonstration of the core steps

### I. Data Acquisition and Preprocessing:

1. Download [CHM13 v1.0 draft fasta file](https://s3-us-west-2.amazonaws.com/human-pangenomics/T2T/CHM13/assemblies/chm13.draft_v1.0.fasta.gz).

2. Download [population based VCF](https://s3-us-west-2.amazonaws.com/human-pangenomics/index.html?prefix=T2T/CHM13/assemblies/variants/1000_Genomes_Project) (CHM13 v1.0 draft). Run GATK FixVcfHeader before launching the pipeline!

3. Download [gene annotation file](https://s3-us-west-2.amazonaws.com/human-pangenomics/T2T/CHM13/assemblies/annotation/chm13.draft_v1.0.gene_annotation.v4.gff3.gz) (gff3, for CHM13 v1.0 draft).

4. Modify config.yaml file
  ```
  ref_genome: data/hm13.draft_v1.0.fasta                  # Reference genome 
  gff: data/chm13.draft_v1.0.gene_annotation.v4.gff3      # Reference genome annotation
  vcf: 1KG_CHM13.vcf                                      # Variant call dataset with allele frequencies 
  vcf_clinvar: clinvar_liftover_tochm13_v0draft.vcf       # Variant call dataset for additional annotation of filtered variants
  af_flag: "AF"                                           # Allele frequency flag in the INFO field to use for the variant filtering
  ```

### II. Core tasks:

**1.** Identification of common variants from VCF file

  -  Variant call filtering criteria (biallelic SNP, AAF > 5%, custom AF flag)

  ```
  bcftools view --max-alleles 2 --exclude-types indels -i 'INFO/AF > 0.05' {input.vcf} > {input.vcf}_filtered.vcf
  ```

**2.** Insertion of common variants into the Reference genome

  ```
  cat {input.reference} | vcf-consensus {input.vcf} > {input.gff}
  ```

**3.** Identifying stop codon sites in CHM13 fasta file (Shangzhe, Muhamad, Bryce, ChunHsuan)

  -  Extract the protein sequences from GFF annotation and FASTA file

  ```
  gffread -g {input.reference} {input.gff} -y pep.fasta
  ```

  -  Custom script to extract the positions of in-frame stop codons

  ```
  python3 determine.in-frame.stop-codon.py pep.fasta {input.gff} > {output_stop-codons.bed}
  ```
  
  -  Riboseq-validation (for the ORF & inframe-stop-codon sites where variants located)

  ```
  (Pseudocodes)
  
  To download paired RNASeq.fastq and RiboSeq.fastq
  
  To quality control fastq files 
  (adapter trimming, remove low quality reads, etc.) 
  
  To align the reads to CHM13 refgenome 
  (get RNASeq.bam and RiboSeq.bam)

  To peak calling for read coverage of aligned bam files 
  (This will reveal real ORF and stop-codon sites by comparing RNAseq & Riboseq at same time.)

  To validate our targeted variant sites by the peak calling results, 
  and to clasify them into true ones and false ones.
  ```
  
  ```
  Rscript ./RiboSeq_pipeline.R
  (Details were described in https://github.com/collaborativebioinformatics/RPG_Pikachu/blob/main/others/RiboSeq_validation/RiboSeq_pipeline.R)
  ```

### III. Results:

**1. Statistical visualization**

6,728,753 common alleles were identified, 17 of which are overlapping with in-frame stop codons. 

<figure>
<img width="500" height= "500" alt="image" src="https://user-images.githubusercontent.com/19209239/137164957-a034549f-c778-461f-b420-97705780eeb8.jpeg">
<figcaption align = "center">Fig.1 - Distribution of ClinVar-annotated common allele frequency</figcaption>
</figure><br><br>

**2. Postions of in-frame stop codons**

There are 3,166 in-frame stop codons for 603 transcripts and 334 genes. 


**3. Biologically annotated variants (CHM13 based).**


## References

- Sergey Aganezov et.al, A complete reference genome improves analysis of human genetic variation. bioRxiv 2021.07.12.452063; doi: https://doi.org/10.1101/2021.07.12.452063
- T2T Consortium: https://sites.google.com/ucsc.edu/t2tworkinggroup/technology?authuser=0
- Lifting over variants from GRCh38 to T2T-CHM13: https://github.com/mccoy-lab/t2t-variants/tree/main/liftover_vcfs
- Ribosome-Profiling: https://github.com/FDA/Ribosome-Profiling

## Acknowledgement

- Fritz J. Sedlazeck
- Ben Busby
