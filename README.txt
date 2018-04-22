----------
RNA-MuTect
----------

*****************************************
The pipeline uses the following software:
1. Samtools
2. bamtools 
3. PLINK/SEQ
4. zlib
5. HiSat2
6. MuTect
7. Matlab
*****************************************

Input: 
------
1. A matched-normal DNA BAM file
2. RNA BAM file aligned with STAR. Parameters used to run STAR are found in Supplementary Table 13.
3. A MAF file listing inital set of mutations called by MuTect with the -U ALLOW_N_CIGAR_READS flag. Other callers may be used as well as long as the output file is in a MAF format.

To run the RNA-MuTect pipeline please follow these steps:
---------------------------------------------------------
1. Realignment step:

  a. sh HiSat_realign_preprocess.sh <Dir> <MAF_file>  <RNA_BAM_file>  <Reference_genome_fasta_format> <case_sample_ID>
  b. sh HiSat_realign_preprocess.sh <Dir> <MAF_file>  <Matched-normal_DNA_BAM_file>  <Reference_genome_fasta_format> <control_sample_ID>
     
     Each of these steps will extract from the corresponding BAM file all the reads that are aligned within the called mutations regions and will write them into fastq files.
     A text file with the path to the new fastq files will be generated.
     
     <Dir> - directory with jar files (FilterSamReads; GenomeAnalysisTK; picard; SamToFastq)
     <MAF_file> - input MAF file
     <RNA_BAM_file> - BAM file of the RNA sample
     <Reference_genome_fasta_format> - reference genome in a fasta format that matches the input BAM file
     <case_sample_ID>/<control_sample_ID> - ID of input sample
     
    Output:
    -------
    1. sample_ID.rna_reads_fastq_list.list - A text file with the path to the two generated fastq files

  c. run the HiSat2 aligner for the case sample with the parameters specified in Supplementary Table 13 and fastq files list generated in (a)
  d. run the HiSat2 aligner for the control sample with the parameters specified in Supplementary Table 13 and fastq files list generated in (b)
  
    Output:
    -------
    1. case_sample_ID.aligned.sorted_by_coord.hisat2.bam - A HiSat2 aligned BAM file for the case sample
    2. control_sample_ID.aligned.sorted_by_coord.hisat2.bam - A HiSat2 aligned BAM file for the control sample
  
  
  e. Run MuTect with:
     1. The HiSat BAMs (generated in (c) and (d))
     2. the -U ALLOW_N_CIGAR_READS flag
     3. An interval list containing  chromosome and position of mutations listed in the input MAF file (see code directory for formatting)
   
     Output:
     -------
     1. pair_ID.call_stats.txt - A call_stats file which is used as put for the next step.  
     
2. Filtering steps (includes two matlab scripts):    
  a. ./run_FilterRNAMutations.sh <Matlab_Runtime_Location> <pair_ID> <MAF_file> <call_stats_file> <MIN_ALT_COUNT> <PON_THR> <Darned_mat.mat> <Radar_mat.mat> <Exac_mat.mat> <PoN_GTEx>
     <pair_ID> - ID of the input pair 
     <MAF_file> - input MAF file
     <call_stats_file> - output of Step 1
     <MIN_ALT_COUNT> - the minimal number of reads required supporting the alternate allele (MIN_ALT_COUNT=3 was used in the paper for TCGA samples and MIN_ALT_COUNT=4 for GTEx samples)
     <PON_THR> - A threshold for PoN filtering (PON_THR=-3 was used in the paper)
     <Darned_mat.mat> - input file found in directory (mat_files)
     <Radar_mat.mat> - input file found in directory (mat_files)
     <Exac_mat.mat> - input file found in directory (mat_files)
     <PoN_GTEx> - input file that should be retreived from dbGAP. See below for an option to run this step with this file
     
     Output:
     -------
     1. pair_ID.intersect.txt - A MAF file containing variants detcted but both aligners (STAR and HiSat2)
     2. pair_ID.post_filtering.txt - A MAF file containing variants remained after applying various filtering criteria
     3. pair_ID.pre_filtering_plus_info.txt - A MAF file containing all input variants with additional columns per filtering criteria, indicating whether a variant was filtered (1) or not (0) by each criteria
     
  b. ./run_FilterRNAMutationsBasedOnDuplicateReads <Matlab_Runtime_Location> <pair_ID> <RNA_BAM_file> <post_filtering_MAF_file> <MIN_ALT_COUNT>
     <pair_ID> - ID of the input pair
     <RNA_BAM_file> - BAM file of the RNA sample
     <post_filtering_MAF_file> - A MAF file with variants remained after applying carious filtering criteria - output (2) of the previous step
     <MIN_ALT_COUNT> - the minimal number of reads required supporting the alternate allele (MIN_ALT_COUNT=3 was used in the paper for TCGA samples and MIN_ALT_COUNT=4 for GTEx samples)

     Output:
     -------
     1. pair_ID.post_filtering_remove_duplicates.txt - A MAF file containing variants remained after applying the duplicate reads filtering. This is the final MAF file.
	
*** In case the PoN is not available, (a) can be run as follows:
    ./run_FilterRNAMutationsNoPoN <Matlab_Runtime_Location> <pair_ID> <MAF_file> <call_stats_file> <MIN_ALT_COUNT> <Darned_mat.mat> <Radar_mat.mat> <Exac_mat.mat>
    <pair_ID> - ID of the input pair 
    <MAF_file> - input MAF file
    <call_stats_file> - output of Step 1
    <MIN_ALT_COUNT> - the minimal number of reads required supporting the alternate allele (MIN_ALT_COUNT=3 was used in the paper for TCGA samples and MIN_ALT_COUNT=4 for GTEx samples)
    <Darned_mat.mat> - input file found in directory
    <Radar_mat.mat> - input file found in directory
    <Exac_mat.mat> - input file found in directory
