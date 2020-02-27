----------
RNA-MuTect
----------

Please visit https://github.com/broadinstitute/RNA_MUTECT_1.0-1 for the updates version of RNA_MUTECT

*****************************************
The pipeline uses the following software:
1. samtools (1.2+)
2. bamtools (1.0.2+)
3. PLINK/SEQ (0.08+)
4. zlib (1.2.6+)
5. HiSat2 (2.0.3-beta)
6. MuTect (1.1.6)
7. Matlab (2016a+)
*****************************************

* Example outputs for a TCGA sample are provided in the 'example_output' directory
* The pipeline is built for genome build hg19/GRCh37

Input: 
------
1. A matched-normal DNA BAM file
2. RNA BAM file aligned with STAR. Parameters used to run STAR are found in Supplementary Table 12.
3. A MAF file listing inital set of mutations called by MuTect with the '-U ALLOW_N_CIGAR_READS' flag. Following MuTect, Oncotator should be run to get the final MAF file: https://software.broadinstitute.org/cancer/cga/oncotator
Other callers and filters may be used, as long as the output file is in a MAF format.

* For example MAF see: example_RNA_vs_DNA_somatic_pair_RNA.txt in 'example_output' directory

To run the RNA-MuTect pipeline please follow these steps:
---------------------------------------------------------
1. Realignment step:

  a. sh HiSat_realign_preprocess.sh <Dir> <MAF_file>  <RNA_BAM_file>  <Reference_genome_fasta_format> <case_sample_ID>
  b. sh HiSat_realign_preprocess.sh <Dir> <MAF_file>  <Matched-normal_DNA_BAM_file>  <Reference_genome_fasta_format> <control_sample_ID>
     
  Each of these steps will extract from the corresponding BAM file all the reads that are aligned with the called mutations and will write them into fastq files.
  A text file with the path to the new fastq files will be generated.
     
  <Dir> - directory with jar files (FilterSamReads; GenomeAnalysisTK; picard; SamToFastq) This jar files were taken from: The Genome Analysis Toolkit: a MapReduce framework for analyzing next-generation DNA sequencing data McKenna A, Hanna M, Banks E, Sivachenko A, Cibulskis K, Kernytsky A, Garimella K, Altshuler D, Gabriel S, Daly M, DePristo MA, 2010 GENOME RESEARCH 20:1297-303 
  <MAF_file> - input MAF file
  <RNA_BAM_file> - BAM file of the RNA sample
  <Reference_genome_fasta_format> - reference genome in a fasta format that matches the input BAM file (hg19)
  <case_sample_ID>/<control_sample_ID> - ID of input sample
     
  Output:
  -------
  1. sample_ID.rna_reads_fastq_list.list - A text file with the path to the two generated fastq files
  Running time: ~1 hour, for ~2000 initially called variants

  c. run the HiSat2 aligner for the case sample with the parameters specified in Supplementary Table 12 and fastq files list generated in (a)
  d. run the HiSat2 aligner for the control sample with the parameters specified in Supplementary Table 12 and fastq files list generated in (b)
  
  Output:
    -------
  1. case_sample_ID.aligned.sorted_by_coord.hisat2.bam - A HiSat2 aligned BAM file for the case sample
  2. control_sample_ID.aligned.sorted_by_coord.hisat2.bam - A HiSat2 aligned BAM file for the control sample
  Running time: ~10 minutes, for ~2000 initially called variants
  
  e. Run MuTect with:
     1. The HiSat BAMs (generated in (c) and (d))
     2. The '-U ALLOW_N_CIGAR_READS' flag
     3. The '--force_output' flag
     4. An interval list containing chromosome and position of mutations listed in the input MAF file (format = 'chromosome:position', each mutation in a different line)
     
  Output:
  -------
  1. pair_ID.call_stats.txt - A call_stats file which is used as put for the next step.  
  Running time: ~1 hour, for ~2000 initially called variants
  * For example MAF see: example_RNA_vs_DNA_somatic_pair_RNA.call_stats.no_germline.txt in 'example_output' directory. Note that suspected germline variants were removed from this file.
  
     
2. Filtering steps (include two matlab scripts):    
  a. ./run_FilterRNAMutations.sh <Matlab_Runtime_Location> <pair_ID> <MAF_file> <call_stats_file> <MIN_ALT_COUNT> <PON_THR> <Darned_mat.mat> <Radar_mat.mat> <Exac_mat.mat> <PoN_GTEx> <cytoBand.txt>
     <Matlab_Runtime_Location> - path to matlab runtime
     <pair_ID> - ID of the input pair 
     <MAF_file> - input MAF file (e.g., 'example_RNA_vs_DNA_somatic_pair_RNA.txt' in 'example_output' directory)
     <call_stats_file> - output of Step #1 (e.g., 'example_RNA_vs_DNA_somatic_pair_RNA.call_stats.no_germline.txt' in 'example_output' directory)
     <MIN_ALT_COUNT> - the minimal number of reads required supporting the alternate allele (MIN_ALT_COUNT=3 was used in the paper for TCGA samples and MIN_ALT_COUNT=4 for GTEx samples)
     <PON_THR> - A threshold for PoN filtering (PON_THR=-3 was used in the paper)
     <Darned_mat.mat> - input file found in directory (mat_files) - downloaded on September 2015 - Kiran A, O'Mahony J, Sanjeev K, Baranov PV (2013) Darned in 2013: Inclusion of model organisms and linking with Wikipedia. Nucleic Acids Res, Epub ahead of print
     <Radar_mat.mat> - input file found in directory (mat_files)  - downloaded on September 2015 - Ramaswami, G. & Li, J. B. RADAR: a rigorously annotated database of A-to-I RNA editing. Nucleic Acids Res. 42, D109–D113 (2014)
     <Exac_mat.mat> - input file found in directory (mat_files) - downloaded on June 2015 - Lek, M. et al. Analysis of protein-coding genetic variation in 60,706 humans. Nature 536, 285–291 (2016)
     <PoN_GTEx> - input file that should be retreived from dbGAP under GTEx - /Genotype Files/phg000830.v1.GTEx_WES.panel-of-normals.c1.GRU.tar. See below for an option to run this step without this file
     <cytoBand.txt> - input file found in main directory	
  
     Output:
     -------
     1. pair_ID.intersect.txt - A MAF file containing variants detcted but both aligners (STAR and HiSat2)
     2. pair_ID.post_filtering.txt - A MAF file containing variants remained after applying various filtering criteria
     3. pair_ID.pre_filtering_plus_info.txt - A MAF file containing all input variants with additional columns per filtering criteria, indicating whether a variant was filtered (1) or not (0) by each criteria
     Running time: ~5 minutes, for ~2000 initially called variants
     * For example MAFs see: (1) example_RNA_vs_DNA_somatic.intersect.txt; (2) example_RNA_vs_DNA_somatic.post_filtering.txt; (3) example_RNA_vs_DNA_somatic.pre_filtering_plus_info.txt, in 'example_output' directory
     ** NOTE: These output files should be removed from the directory between different runs with the same input files
     
  b. ./run_FilterRNAMutationsBasedOnDuplicateReads <Matlab_Runtime_Location> <pair_ID> <RNA_BAM_file> <post_filtering_MAF_file> <MIN_ALT_COUNT>
     <Matlab_Runtime_Location> - path to matlab runtime
     <pair_ID> - ID of the input pair
     <RNA_BAM_file> - BAM file of the RNA sample
     <post_filtering_MAF_file> - A MAF file with variants remained after applying various filtering criteria (output #2 of the previous step)
     <MIN_ALT_COUNT> - the minimal number of reads required supporting the alternate allele (MIN_ALT_COUNT=3 was used in the paper for TCGA samples and MIN_ALT_COUNT=4 for GTEx samples)
	
     Output:
     -------
     1. pair_ID.post_filtering_remove_duplicates.txt - A MAF file containing variants remained after applying the duplicate reads filtering. This is the final MAF file.
     Running time: ~30 minutes, for ~60 called variants
     * For example MAF see: (1) example_RNA_vs_DNA_somatic.post_filtering_remove_duplicates.txt, in 'example_output' directory
	
*** In case the PoN is not available, step #2a can be run as follows:
    ./run_FilterRNAMutationsNoPoN <Matlab_Runtime_Location> <pair_ID> <MAF_file> <call_stats_file> <MIN_ALT_COUNT> <Darned_mat.mat> <Radar_mat.mat> <Exac_mat.mat>
    <Matlab_Runtime_Location> - path to matlab runtime
    <pair_ID> - ID of the input pair (e.g., 'example_RNA_vs_DNA_somatic_pair_RNA.txt' in 'example_output' directory)
    <MAF_file> - input MAF file
    <call_stats_file> - output of Step #1 (e.g., 'example_RNA_vs_DNA_somatic_pair_RNA.call_stats.no_germline.txt' in 'example_output' directory)
    <MIN_ALT_COUNT> - the minimal number of reads required supporting the alternate allele (MIN_ALT_COUNT=3 was used in the paper for TCGA samples and MIN_ALT_COUNT=4 for GTEx samples)
    <Darned_mat.mat> - input file found in directory (mat_files)
    <Radar_mat.mat> - input file found in directory (mat_files)
    <Exac_mat.mat> - input file found in directory (mat_files)

    * For example MAFs see: (1) example_RNA_vs_DNA_somatic.intersect_no_pon.txt; (2) example_RNA_vs_DNA_somatic.post_filtering_no_pon.txt; (3) example_RNA_vs_DNA_somatic.pre_filtering_plus_info_no_pon.txt; (4) example_RNA_vs_DNA_somatic_pair_RNA.post_filtering_remove_duplicates_no_pon.txt, in 'example_output' directory
    