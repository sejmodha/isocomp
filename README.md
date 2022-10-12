# Isocomp: comparing high-quality IsoSeq3 isoforms between samples

![](images/logo.png)

## Contributors
1. Yutong Qiu (Carnegie Mellon)
2. Chia Sin	Liew (University of Nebraska-Lincoln)
3. Chase Mateusiak (Washington University)
4. Rupesh Kesharwani (Baylor College of Medicine)
5. Bida	Gu (University of Southern California)
6. Muhammad Sohail Raza (Beijing Institute of Genomics, Chinese Academy of Sciences/China National Center for Bioinformation)
6. Evan	Biederstedt (HMS)

## Detailed project overview
https://github.com/collaborativebioinformatics/isocomp/blob/main/FinalPresentation_BCM_Hackathon_12Oct2022.pdf 


## Introduction
NGS targeted sequencing and WES have become routine for clinical diagnosis of Mendelian disease [CITATION]. Family sequencing (or "trio sequencing") involves sequencing a patient and parents (trio) or other relatives. This improves the diagnostic potential via the interpretation of germline mutations and enables detection of de novo mutations which underlie most Mendelian disorders. 

Transcriptomic profiling has been gaining used over the past several decades. However, this endeavor has been hampered by short-read sequencing, especially for inferring alternative splicing, allelic imbalance, and isoform variation. 

The promise of long-read sequencing has been to overcome the inherent uncertainties of short-reads. 

Something something Isoseq3: https://www.pacb.com/products-and-services/applications/rna-sequencing/

Provides high-quality, polished, assembled full isoforms. With this, we will be able to identify alternatively-spliced isoforms and detect gene fusions. 

Since the advent of HiFi reads, the error rates have plummeted. 

The goal of this project will be to extend the utility of long-read RNAseq for investigating Mendelian diseases between multiple samples. 

And what about gene fusions? We detect these in the stupidest possible way with short-read sequencing, and we think they're cancer-specific. What about the germline?


## Goals

Given high-quality assembled isoforms from 2-3 samples, we want to algorithmically (definitively) characterize the "unique" (i.e. differing) isoforms between samples.

## Methods

### Methods overview
[Isoform set comparison problem] Given two sets of isoform sam, find two subsets so that each subset of isoform is unique to each sample.

To solve this problem, the naive solution is to perform sequence match between two sets of isoforms. However, this method is time consuming due to the size of the isoform sets in human genome (give an example number (way larger than 10,000, e.g.)). 

We note that it is unnecessary to perform all-against-all alignment between complete sets of isoforms. In fact, we only need to compare the isoforms that are aligned to the same genomic region. We extract region windows from the genome that contain at least one isoform from any sample, then, we divide the set of isoforms into smaller subsets by their origin in the extracted genomic regions.

For each pair of samples that we are comparing, we perform intersection between two subsets of isoforms within each genomic window and identify isoforms that are shared by both samples and unique to each sample. 

For each unique isoform S from sample A, we further investigate the differences between S and other isoforms from sample B within the same genomic window. 

### Aligning isoforms to the reference genome

For every sample, we first prepend sample name to FASTA sequences in final corrected FASTA generated by SQANTI to make sure that all sequence names are unique, then align the renamed FASTA to the human Telomere-to-Telomere genome assembly of the CHM13 cell line (T2T-CHM13v2.0; RefSeq - GCF_009914755.1) using minimap2 (v2.24-r1122). The resulting SAM file is converted to BAM and sorted by samtools (v1.15.1; Danecek et al, 2021).
Dividing isoforms into subsets

We extract regions from the CHM13v2.0 genome that overlap with at least one isoform from any sample. We first obtain the average coverage of isoform per base by using samtools mpileup (citation, version). We next extract the 20,042 annotated protein coding gene regions from the reference genome and take the union of overlapping regions to create windows. Finally, windows were filtered to those which contained a per-base coverage greater than 0.05, which reduced our final set of windows to 11936.


In addition to the annotated gene regions, each sample contains more than 100,000 isoforms (Table 1) of isoforms that aligned to intron region. These isoforms are usually regarded as novel and may be important to the phenotypes of interest. Therefore, in addition to known gene regions, we divide the genome into 100 bp windows and retain the ones that has per-base coverage higher than 0.05. 

After that, we merge the gene windows and 100bp windows to obtain a complete set of windows that overlap with any isoform.

### Intersecting subsets of isoforms

For each isoform S in the subset of sample A, we perform exact string matching with all isoforms in the subset of sample B. If no isoform in sample B in the same genomic window matches S exactly, we say that S is unique.

### Comparing unique isoforms with other isoforms
For each unique isoform U, we perform Needleman-Wunch alignment between U and other isoforms within the same genomic window. We measure similarities between isoforms by the percentage of matched bases in U.
Annotating the differences between unique isoforms and the other sequences

We categorize differences between isoforms into [TODO] SNPs (<5bp), large-scale variants (>5bp), gene fusion, different exon usage and completely novel. Similar categories was used by SQANTI in annotating differences between sample isoforms and reference transcriptome. Note that we extend the categories by SQANTI by adding SNPs and large-scale variants.

### Iso-Seq analysis
 
Isoseq3 (v3.2.2) generated HQ (Full-length high quality) transcripts [Table 1] were mapped to GRCh38 (v33 p13) using Minimap2 long read alignment tools [1] (v2.24-r1122; commands: minimap2 -t 8 -ax splice:hq -uf --secondary=no -C5 -O6,24 -B4 GRCh38.v33p13.primary_assembly.fa sample.polished.hq.fastq.gz). The table 2 shows the basic statistics of the alignment of each sample [NA24385 /HG002, NA24143/HG004 and NA24631/HG005]. Next, we performed cDNA_cupcake [https://github.com/Magdoll/cDNA_Cupcake] workflow to collapse the redundant isoforms from bam, followed by filtering the low counts isoforms by 10 and filter away 5' degraded isoforms that might not be biologically significant. Next, sqanti3 [2] tool was used to generate final corrected fasta [Table 3a] transcripts and gtf [Table 3b] along with the isoform classification reports. The external databases including reference data set of transcription start sites (refTSS), list of polyA motif, tappAS-annotation and genecode hg38 annotation were utilized. Finally, IsoAnnotLite (v2.7.3) analysis was performed to annotate the gtf file from sqanti3.


## Description

## Flowchart
![](images/workflow.png)
### To extract sets of unique isoforms
![](images/workflow_part1.png)
### To annotate the unique isoforms
![](images/workflow_part2.png)

## Example Output

For each isoforom that is unique to at least one sample, we output the information of the read and the similarity between that isoform and the most similiar isoform in the same window.

```
win_chr win_start       win_end total_isoform   isoform_name    sample_from     sample_compared_to      mapped_start    isoform_sequence        selected_alignments
NC_060925.1     255178  288416  4       PB.6.2  HG004   HG002   255173  GGATTATCCGGAGCCAAGGTCCGCTCGGGTGAGTGCCCTCCGCTTTTTGTGGCCAAACCCAGCCACGCAGTCCCCTCCCTGCGGCGTCCTCCACACCCGGGGTCTGCTGGTCTCCGCGGATGTCACAGGCTCGGCAACCGCCCTCCTGTCGGCGGGGAGTCCCGCGACGCCCGGAAATGCTCCGAAGCCTGTCGCTCAGCTGCCAGATCTGCGTCTGTGTCCGGTTCCGTCACTGAGGTCGCCCCTGTCCGGCCCTTCCACCCTAGTTCTCTTCACCGTCCGCCCATCCTATCGCGCGCGGCCTCAGGTCCCCATTCGGCATGTGGCTTGTCTTCCATCGTCCCCACCCTCGCCCCTCTTGGCCCCTCAGGGCAGCCCTGGGATTCGGCAGACGCCAGTCCTCCCTGAGATGCTTCCCCGTCCTTCCCTCCGCCAGGCCCTACGTCTCCGCAAACCCCACGCTTCGGGGTGGCCGCCTCAGACAGGACCCTGAGTCCGAGACTGGGGTAGGGGACCTGCCCGATCCTGTAACAACCCTCGTGCTTCTGCACAATCGCCTCCCACTAGCGGTGACTGTTGGGTGTCTACCTTCCCGGTGTCCCACTGAGAAGCGGGCTCCTCCTTGGCAGGGGCTTCTTCATTGCCTCGCTGTGGATGTCGAGGTGGGGCAGGAGAGTGAGGAGAAAACAGAGAGGAGGGAGGTAGAGCCAACGAGCGAGAAAAGGGGAGGGAAGTTTAGATGGGAAGTGGATGGGTCTGAGGAATTTGAACAAACACCGACAGTGAAGGAGAGTGACCTGAGCAAGCAGTAGTGGGGTAAATGGAAATAGACAAAATGGGAATCAGCAGAGATATGGAGGACAGAATACAATGAGGAGGCCTTGACCGTCAGTAGCAGAGAGGGCAGCAGAAGCCTAATTCCCAAATTCCTTAGTGGTTTTCTGATTTCCAAATTAGTTTCCCTTTTAAATTTATTGTGTCAGGTTCAGCTTATGAGGCCTCAATACTTTTCAGTCTTAATTGTATATTGAAAATACTTTTTGTTTACTAAATGCTTTTTACATTAATTCAGTGTGCACTCCGTAAGGATATTGATGATTTGAGTTAGTTTAGTATTCAACAGCTTCCTCTATTCCTTTGTATGATCTCTGTATTTAATGGCTGTGGCATAAAGTTTCCAACTAAGTATAAGTATCAAGTTTTCTTTGTGATGTTTTCTGCAAATATTGAAGGATGACCTGGATTGTCCTAGAACTTTGTTCCAACAGATTACATGTGTTCATAACGAATAAATTGCTCAAAGATATTTC      0.02_HG002_PB.6.2_3=6I1=3I1286=11I
NC_060925.1     255178  288416  4       PB.6.2  HG004   HG005   255173  GGATTATCCGGAGCCAAGGTCCGCTCGGGTGAGTGCCCTCCGCTTTTTGTGGCCAAACCCAGCCACGCAGTCCCCTCCCTGCGGCGTCCTCCACACCCGGGGTCTGCTGGTCTCCGCGGATGTCACAGGCTCGGCAACCGCCCTCCTGTCGGCGGGGAGTCCCGCGACGCCCGGAAATGCTCCGAAGCCTGTCGCTCAGCTGCCAGATCTGCGTCTGTGTCCGGTTCCGTCACTGAGGTCGCCCCTGTCCGGCCCTTCCACCCTAGTTCTCTTCACCGTCCGCCCATCCTATCGCGCGCGGCCTCAGGTCCCCATTCGGCATGTGGCTTGTCTTCCATCGTCCCCACCCTCGCCCCTCTTGGCCCCTCAGGGCAGCCCTGGGATTCGGCAGACGCCAGTCCTCCCTGAGATGCTTCCCCGTCCTTCCCTCCGCCAGGCCCTACGTCTCCGCAAACCCCACGCTTCGGGGTGGCCGCCTCAGACAGGACCCTGAGTCCGAGACTGGGGTAGGGGACCTGCCCGATCCTGTAACAACCCTCGTGCTTCTGCACAATCGCCTCCCACTAGCGGTGACTGTTGGGTGTCTACCTTCCCGGTGTCCCACTGAGAAGCGGGCTCCTCCTTGGCAGGGGCTTCTTCATTGCCTCGCTGTGGATGTCGAGGTGGGGCAGGAGAGTGAGGAGAAAACAGAGAGGAGGGAGGTAGAGCCAACGAGCGAGAAAAGGGGAGGGAAGTTTAGATGGGAAGTGGATGGGTCTGAGGAATTTGAACAAACACCGACAGTGAAGGAGAGTGACCTGAGCAAGCAGTAGTGGGGTAAATGGAAATAGACAAAATGGGAATCAGCAGAGATATGGAGGACAGAATACAATGAGGAGGCCTTGACCGTCAGTAGCAGAGAGGGCAGCAGAAGCCTAATTCCCAAATTCCTTAGTGGTTTTCTGATTTCCAAATTAGTTTCCCTTTTAAATTTATTGTGTCAGGTTCAGCTTATGAGGCCTCAATACTTTTCAGTCTTAATTGTATATTGAAAATACTTTTTGTTTACTAAATGCTTTTTACATTAATTCAGTGTGCACTCCGTAAGGATATTGATGATTTGAGTTAGTTTAGTATTCAACAGCTTCCTCTATTCCTTTGTATGATCTCTGTATTTAATGGCTGTGGCATAAAGTTTCCAACTAAGTATAAGTATCAAGTTTTCTTTGTGATGTTTTCTGCAAATATTGAAGGATGACCTGGATTGTCCTAGAACTTTGTTCCAACAGATTACATGTGTTCATAACGAATAAATTGCTCAAAGATATTTC      0.02_HG002_PB.6.2_3=6I1=3I1286=11
```


## Computational Resources / Operation

## Citations
[1] https://www.pacb.com/products-and-services/applications/rna-sequencing/

