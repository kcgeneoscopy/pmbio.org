---
feature_text: |
  ## Precision Medicine
title: Germline SNV/Indel Filtering/Annotation/Review
categories:
    - Module-05-Germline
feature_image: "assets/genvis-dna-bg_optimized_v1a.png"
date: 0005-02-02
---

### Module objectives

- Perform filtering of germline SNVs and indels 

The raw output of GATK HaplotypeCaller will include many variants with varying degrees of quality. For various reasons we might wish to further filter these to a higher confidence set of variants. The recommended approach is to use GATK VQSR. However, this requires a large (i.e., at least 30-50), preferably platform-matched (i.e., similar sequencing strategy), set of samples with variant calls. For our purposes, we will first demonstrate the less optimal hard filtering strategy. 

It is strongly recommended to read the following documentation from GATK:
- [How to run VQSR](https://software.broadinstitute.org/gatk/documentation/article?id=2805)
- [How to apply hard filters](https://software.broadinstitute.org/gatk/documentation/article?id=2806) 

### Perform hard filtering

```
# Extract the SNPs from the call set
gatk --java-options '-Xmx64g' SelectVariants -R /home/ubuntu/data/reference/GRCh38_full_analysis_set_plus_decoy_hla.fa -V /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.vcf -select-type SNP -O /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.snps.vcf

# Extract the indels from the call set
gatk --java-options '-Xmx64g' SelectVariants -R /home/ubuntu/data/reference/GRCh38_full_analysis_set_plus_decoy_hla.fa -V /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.vcf -select-type INDEL -O /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.indels.vcf

# Apply basic filters to the SNP call set
gatk --java-options '-Xmx64g' VariantFiltration -R /home/ubuntu/data/reference/GRCh38_full_analysis_set_plus_decoy_hla.fa -V /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.snps.vcf --filter-expression "QD < 2.0 || FS > 60.0 || MQ < 40.0 || MQRankSum < -12.5 || ReadPosRankSum < -8.0" --filter-name "basic_snp_filter" -O /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.snps.filtered.vcf

# Apply basic filters to the INDEL call set
gatk --java-options '-Xmx64g' VariantFiltration -R /home/ubuntu/data/reference/GRCh38_full_analysis_set_plus_decoy_hla.fa -V /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.indels.vcf --filter-expression "QD < 2.0 || FS > 200.0 || ReadPosRankSum < -20.0" --filter-name "basic_indel_filter" -O /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.indels.filtered.vcf

# Merge filtered SNP and INDEL vcfs back together
gatk --java-options '-Xmx64g' MergeVcfs -I /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.snps.filtered.vcf -I /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.indels.filtered.vcf -O /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.vcf

# Extract PASS variants only
gatk --java-options '-Xmx64g' SelectVariants -R /home/ubuntu/data/reference/GRCh38_full_analysis_set_plus_decoy_hla.fa -V /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.vcf -O /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.PASS.vcf --exclude-filtered

```

### Perform VQSR filtering of variants that were joint called with 1KG exomes

When using VQSR filtering on exome data, there will be a smaller number of variants per sample compared to WGS. These are typically insufficient to build a robust recalibration model. If running on only a few samples, GATK recommends that you analyze samples jointly in cohorts of at least 30 samples. If necessary, add exomes from 1000G Project or comparable. Ideally, these should be processed with similar technical generation (technology, capture, read length, depth).

See also: https://drive.google.com/drive/folders/0BzI1CyccGsZiWlU5SXNvNnFGbVE (GATK Workshops -> 1709 -> GATKwr19-05-Variant_filtering.pdf)

```

#Build SNP recalibration model
#Note: Any annotations specified ("-an XX") below must actually be present in your VCF. They should have been added at an earlier step in the GATK workflow. If not, they could be added with VariantAnnotator
#Note: For exome data, exclude "-an DP" as this coverage metric should only be used if extreme deviations in coverage are not expected and indicative of errors
#Note: The "-an InbreedingCoeff" option is for a population level statistic that requires at least 10 samples in order to be computed (When?). For projects with fewer samples, or that includes many closely related samples (such as a family) please omit this annotation from the command line.

gatk --java-options '-Xmx64g' VariantRecalibrator -R /home/ubuntu/data/reference/GRCh38_full_analysis_set_plus_decoy_hla.fa -V /home/ubuntu/data/germline_variants/Exome_GGVCFs_jointcalls.vcf --resource hapmap,known=false,training=true,truth=true,prior=15.0:/home/ubuntu/data/reference/hapmap_3.3.hg38.vcf.gz --resource omni,known=false,training=true,truth=true,prior=12.0:/home/ubuntu/data/reference/1000G_omni2.5.hg38.vcf.gz --resource 1000G,known=false,training=true,truth=false,prior=10.0:/home/ubuntu/data/reference/1000G_phase1.snps.high_confidence.hg38.vcf.gz --resource dbsnp,known=true,training=false,truth=false,prior=2.0:/home/ubuntu/data/reference/Homo_sapiens_assembly38.dbsnp138.vcf.gz -an QD -an FS -an SOR -an MQ -an MQRankSum -an ReadPosRankSum --mode SNP -tranche 100.0 -tranche 99.9 -tranche 99.0 -tranche 90.0 -O recalibrate_SNP.recal --tranches-file recalibrate_SNP.tranches --rscript-file recalibrate_SNP_plots.R 

#Apply recalibration to SNPs 
gatk --java-options '-Xmx64g' ApplyVQSR -R /home/ubuntu/data/reference/GRCh38_full_analysis_set_plus_decoy_hla.fa -V /home/ubuntu/data/germline_variants/Exome_GGVCFs_jointcalls.vcf --mode SNP --truth-sensitivity-filter-level 99.0 --recal-file /home/ubuntu/data/germline_variants/recalibrate_SNP.recal --tranches-file /home/ubuntu/data/germline_variants/recalibrate_SNP.tranches -O /home/ubuntu/data/germline_variants/Exome_GGVCFs_jointcalls_recalibrated_snps_raw_indels.vcf

#Build Indel recalibration model
gatk --java-options '-Xmx64g' VariantRecalibrator -R /home/ubuntu/data/reference/GRCh38_full_analysis_set_plus_decoy_hla.fa -V /home/ubuntu/data/germline_variants/Exome_GGVCFs_jointcalls_recalibrated_snps_raw_indels.vcf --resource mills,known=false,training=true,truth=true,prior=12.0:/home/ubuntu/data/reference/Mills_and_1000G_gold_standard.indels.hg38.vcf.gz --resource dbsnp,known=true,training=false,truth=false,prior=2.0:/home/ubuntu/data/reference/Homo_sapiens_assembly38.dbsnp138.vcf.gz -an QD -an FS -an SOR -an MQRankSum -an ReadPosRankSum --mode INDEL -tranche 100.0 -tranche 99.9 -tranche 99.0 -tranche 90.0 --max-gaussians 4 -O recalibrate_INDEL.recal --tranches-file recalibrate_INDEL.tranches --rscript-file recalibrate_INDEL_plots.R  

#Apply recalibration to SNPs
gatk --java-options '-Xmx64g' ApplyVQSR -R /home/ubuntu/data/reference/GRCh38_full_analysis_set_plus_decoy_hla.fa -V /home/ubuntu/data/germline_variants/Exome_GGVCFs_jointcalls_recalibrated_snps_raw_indels.vcf --mode INDEL --truth-sensitivity-filter-level 99.0 --recal-file /home/ubuntu/data/germline_variants/recalibrate_INDEL.recal --tranches-file /home/ubuntu/data/germline_variants/recalibrate_INDEL.tranches -O /home/ubuntu/data/germline_variants/Exome_GGVCFs_jointcalls_recalibrated.vcf

# Extract PASS variants only and only variants actually called and non-reference in our sample of interest
gatk --java-options '-Xmx64g' SelectVariants -R /home/ubuntu/data/reference/GRCh38_full_analysis_set_plus_decoy_hla.fa -V /home/ubuntu/data/germline_variants/Exome_GGVCFs_jointcalls_recalibrated.vcf -O /home/ubuntu/data/germline_variants/Exome_Norm_GGVCFs_jointcalls_recalibrated.PASS.vcf --exclude-filtered --exclude-non-variants --remove-unused-alternates --sample-name HCC1395BL_DNA

```


### Perform VEP annotation of filtered results

```
#Annotate hard-filtered results
#Output VEP VCF
~/bin/ensembl-vep/vep --cache --dir_cache /home/ubuntu/data/vep_cache --dir_plugins /home/ubuntu/data/vep_cache/Plugins --fasta /home/ubuntu/data/vep_cache/homo_sapiens/91_GRCh38/Homo_sapiens.GRCh38.dna.toplevel.fa.gz --assembly=GRCh38 --offline --vcf --plugin Downstream --everything --terms SO --pick --coding_only --transcript_version -i /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.PASS.vcf -o /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.PASS.vep.vcf

#Output tabular VEP
~/bin/ensembl-vep/vep --cache --dir_cache /home/ubuntu/data/vep_cache --dir_plugins /home/ubuntu/data/vep_cache/Plugins --fasta /home/ubuntu/data/vep_cache/homo_sapiens/91_GRCh38/Homo_sapiens.GRCh38.dna.toplevel.fa.gz --assembly=GRCh38 --offline --tab --plugin Downstream --everything --terms SO --pick --coding_only --transcript_version -i /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.PASS.vcf -o /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.PASS.vep.tsv

#Annotate VQSR filtered results
#Output VEP VCF
~/bin/ensembl-vep/vep --cache --dir_cache /home/ubuntu/data/vep_cache --dir_plugins /home/ubuntu/data/vep_cache/Plugins --fasta /home/ubuntu/data/vep_cache/homo_sapiens/91_GRCh38/Homo_sapiens.GRCh38.dna.toplevel.fa.gz --assembly=GRCh38 --offline --vcf --plugin Downstream --everything --terms SO --pick --coding_only --transcript_version -i /home/ubuntu/data/germline_variants/Exome_Norm_GGVCFs_jointcalls_recalibrated.PASS.vcf -o /home/ubuntu/data/germline_variants/Exome_Norm_GGVCFs_jointcalls_recalibrated.PASS.vep.vcf

#Output tabular VEP
~/bin/ensembl-vep/vep --cache --dir_cache /home/ubuntu/data/vep_cache --dir_plugins /home/ubuntu/data/vep_cache/Plugins --fasta /home/ubuntu/data/vep_cache/homo_sapiens/91_GRCh38/Homo_sapiens.GRCh38.dna.toplevel.fa.gz --assembly=GRCh38 --offline --tab --plugin Downstream --everything --terms SO --pick --coding_only --transcript_version -i /home/ubuntu/data/germline_variants/Exome_Norm_GGVCFs_jointcalls_recalibrated.PASS.vcf -o /home/ubuntu/data/germline_variants/Exome_Norm_GGVCFs_jointcalls_recalibrated.PASS.vep.tsv

```

### Filter VEP annotated VCF further for variants of potential clinical relevance

Use the `filter_vep` tool to filter to interesting variants.

Note - I'm obtaining different results depending on the order of filters supplied if using separate "--filter" options. This should not be the case. Combining into a single expression seems to work though.

Note - there is a difference between the information available for filtering in VEP VCF output vs VEP tabular output. Namely some VCF fields (e.g., FILTER) are not included in the tabular output. Therefore, if you wish to use filter_vep on tabular output (for ease of reading) make sure to complete any vcf-specific filtering first (e.g., using GATK SelectVariants). Alternatively, it may be possible to proceed with filtering on the VCF and then reannotate with VEP specifying tabular output to convert. Yet another option would be to use VCF annotation tools ([vatools.org](http://vatools.org)).

Note - the filter_vep tool immediately and automatically limits to variants with a CSQ entry when filtering VCFs. Keep in this mind if you have annotated your VCF with the coding_only option which only adds CSQ entries for coding region alterations.


```

#Filter hard-filtered results
#Filter VEP VCF
~/bin/ensembl-vep/filter_vep --format vcf -i /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.PASS.vep.vcf -o /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.PASS.vep.interesting.vcf --force_overwrite --filter "(MAX_AF < 0.001 or not MAX_AF) and ((IMPACT is HIGH) or (IMPACT is MODERATE and (SIFT match deleterious or PolyPhen match damaging)))"

#Filter tabular VEP
~/bin/ensembl-vep/filter_vep --format tab -i /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.PASS.vep.tsv -o /home/ubuntu/data/germline_variants/Exome_Norm_HC_calls.filtered.PASS.vep.interesting.tsv --force_overwrite --filter "(MAX_AF < 0.001 or not MAX_AF) and ((IMPACT is HIGH) or (IMPACT is MODERATE and (SIFT match deleterious or PolyPhen match damaging)))"

#Filter VQSR-filtered results
#Filter VEP VCF
~/bin/ensembl-vep/filter_vep --format vcf -i /home/ubuntu/data/germline_variants/Exome_Norm_GGVCFs_jointcalls_recalibrated.PASS.vep.vcf -o /home/ubuntu/data/germline_variants/Exome_Norm_GGVCFs_jointcalls_recalibrated.PASS.vep.interesting.vcf --force_overwrite --filter "(MAX_AF < 0.001 or not MAX_AF) and ((IMPACT is HIGH) or (IMPACT is MODERATE and (SIFT match deleterious or PolyPhen match damaging)))"

#Filter tabular VEP
~/bin/ensembl-vep/filter_vep --format tab -i /home/ubuntu/data/germline_variants/Exome_Norm_GGVCFs_jointcalls_recalibrated.PASS.vep.tsv -o /home/ubuntu/data/germline_variants/Exome_Norm_GGVCFs_jointcalls_recalibrated.PASS.vep.interesting.tsv --force_overwrite --filter "(MAX_AF < 0.001 or not MAX_AF) and ((IMPACT is HIGH) or (IMPACT is MODERATE and (SIFT match deleterious or PolyPhen match damaging)))"

```

### Explore the filtered results

The above filtering has limited the results to approximately 200 germline variants of potential interest. 

- Are there any variants with known clinical significance (See CLIN_SIG column)? Use their HGVS IDs (See HGVSc or HGVSp columns) to search the [ClinGen Allele Registry](http://reg.clinicalgenome.org), and then follow the links (if any) to ClinVar or other resources of interest.
- Are there any variants in known cancer genes? Try intersecting the remaining genes with variants with the [Cancer Gene Census](https://cancer.sanger.ac.uk/census) genes?
- Manually review any variants of interest in IGV


