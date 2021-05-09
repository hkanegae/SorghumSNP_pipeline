## SorghumSNP_pipeline
A description of the analysis step of *Sorghum bicolor* variant calling pipeline.

![sorghum](https://github.com/hkanegae/SorghumSNP_pipeline/blob/main/sorghum.JPG)
***

### Analysis workflow for detection of genome-wide variations in TASUKE+ of RAP-DB was modified.

[Analysis workflow for detection of genome-wide variations in TASUKE+ of RAP-DB](https://rapdb.dna.affrc.go.jp/genome-wide_variations/Analysis_workflow_for_detection_of_genome-wide_var.html)

#### Sorghum bicolor v3.1.1
[Phytozome Sorghum bicolor v3.1.1](https://phytozome-next.jgi.doe.gov/info/Sbicolor_v3_1_1)
**Reference genome **
- Sbicolor_454_v3.0.1_M_C.fa = Sbicolor_ (454_v3.0.1 + EF115542.1 + NC_008360.1
- EF115542.1 Sorghum bicolor cultivar BTx623 chloroplast, complete genome
- NC_008360.1 Sorghum bicolor mitochondrion, complete genome

#### Sorghum bicolor Rio v2.1
[Phytozome Sorghum bicolor Rio v2.1](https://phytozome-next.jgi.doe.gov/info/SbicolorRio_v2_1)
**Reference genome **
SbicolorRio_468_v2.0.fa
*****
#### Mapping to the reference genome
1. Trimmomatic
```
java -jar trimmomatic-0.38.jar PE \
    -phred33 read.r1.fastq.gz read.r2.fastq.gz \
    read.pe.r1.fastq.gz read.se.r1.fastq.gz read.pe.r2.fastq.gz read.se.r2.fastq.gz \
    ILLUMINACLIP:adapters.fa:2:30:10 LEADING:20 TRAILING:20 SLIDINGWINDOW:10:20 MINLEN:30
```
2. Making index of the genome
```
bwa index genome.fa
samtools faidx genome.fa
java $JAVA_MEM -jar $PICARD_HOME/picard.jar CreateSequenceDictionary \
    REFERENCE=genome.fa \
    OUTPUT=genome.dict
```
3. Alignment of Illumina reads to the reference genome
```
bwa mem -M genome.fa read.pe.r1.fastq.gz read.pe.r2.fastq.gz \
    | samtools sort -o alignment_${SAMPLE_ID}_${NUM}.sort.bam -
samtools index alignment_${SAMPLE_ID}_${NUM}.sort.bam
```
4. Make clean BAM
```
java $JAVA_MEM -jar $PICARD_HOME/picard.jar MergeBamAlignment \
    ALIGNED=alignment_${SAMPLE_ID}_${NUM}.sort.bam \
    UNMAPPED=uBAM_${SAMPLE_ID}_${NUM}.bam \
    OUTPUT=alignment_${SAMPLE_ID}_${NUM}.merge.bam \
    REFERENCE_SEQUENCE=genome.fa
```
5. Remove PCR duplicates
```
java $JAVA_MEM -jar $PICARD_HOME/picard.jar MarkDuplicates \
    INPUT=alignment_${SAMPLE_ID}_${NUM}.merge.bam \
    OUTPUT=alignment_${SAMPLE_ID}_${NUM}.rmdup.bam \
    METRICS_FILE=rmdup.matrix \
    REMOVE_DUPLICATES=true \
    MAX_RECORDS_IN_RAM=1000000 \
    TMP_DIR=./tmp_${SAMPLE_ID}_${NUM}
samtools index alignment_${SAMPLE_ID}_${NUM}.rmdup.bam
```

#### Merging BAM files
```
samtools merge alignment_${SAMPLE_ID}.rmdup.bam alignment_${SAMPLE_ID}_1.rmdup.bam alignment_${SAMPLE_ID}_2.rmdup.bam ......
samtools index alignment_${SAMPLE_ID}.rmdup.bam
```

#### Variant detection and filtering by GATK
1. HaplotypeCaller
```
gatk --java-options "$JAVA_MEM" HaplotypeCaller \
    --input alignment_${SAMPLE_ID}.rmdup.bam \
    --output variants_${SAMPLE_ID}.g.vcf.gz \
    --reference genome.fa \
    -max-alternate-alleles 2 \
    --emit-ref-confidence GVCF
```
2. GenotypeGVCFs
```
gatk --java-options "$JAVA_MEM" GenotypeGVCFs \
    --variant variants_${SAMPLE_ID}.g.vcf.gz \
    --output variants_${SAMPLE_ID}.genotype.vcf.gz \
    --reference genome.fa
```
3. VariantFiltration
```
gatk --java-options "$JAVA_MEM" VariantFiltration \
    --reference genome.fa \
    --variant variants_${SAMPLE_ID}.genotype.vcf.gz \
    --output variants_${SAMPLE_ID}.filter.genotype.vcf.gz \
    --filter-expression "QD < 5.0 || FS > 50.0 || SOR > 3.0 || MQ < 50.0 || MQRankSum < -2.5 || ReadPosRankSum < -1.0 || ReadPosRankSum > 3.5" \
    --filter-name "FILTER"
```
4. SelectVariants
```
gatk --java-options "$JAVA_MEM" SelectVariants \
    --reference genome.fa \
    --variant variants_${SAMPLE_ID}.filter.genotype.vcf.gz \
    --output variants_${SAMPLE_ID}.varonly.vcf.gz \
    --exclude-filtered \
    --select-type-to-include SNP \
    --select-type-to-include INDEL
```

#### Variant calling with multi-sample 
1. GenomicsDBImport
```
array=(chr01 chr02 chr03 chr04 chr05 chr06 chr07 chr08 chr09 chr10)
for name in ${array[@]}; do echo $name; 
gatk --java-options "$JAVA_MEM" GenomicsDBImport --reference genome.fa  -V variants_A.g.vcf.gz -V variants_B.g.vcf.gz -V variants_C.g.vcf.gz  --genomicsdb-workspace-path ${SAMPLE_ID}_"$name" --intervals "$name"
done
```
2. GenotypeGVCFs
```
for name in ${array[@]}; do echo $name;
gatk --java-options "$JAVA_MEM" GenotypeGVCFs --reference genome.fa -V gendb://${SAMPLE_ID}_"$name" -G StandardAnnotation --new-qual -O variants_${SAMPLE_ID}.genotype_"$name".vcf.gz
done
```
3. VariantFiltration
```
for name in ${array[@]}; do echo $name;
gatk --java-options "$JAVA_MEM" VariantFiltration --reference genome.fa --variant variants_${SAMPLE_ID}.genotype_"$name".vcf.gz --output variants_${SAMPLE_ID}.filter.genotype_"$name".vcf.gz --filter-expression "QD < 5.0 || FS > 50.0 || SOR > 3.0 || MQ < 50.0 || MQRankSum < -2.5 || ReadPosRankSum < -1.0 || ReadPosRankSum > 3.5" --filter-name "FILTER"
done
```
4. SelectVariants
```
for name in ${array[@]}; do echo $name;
gatk --java-options "$JAVA_MEM" SelectVariants --reference genome.fa  --variant variants_${SAMPLE_ID}.filter.genotype_"$name".vcf.gz --output variants_${SAMPLE_ID}_"$name".varonly.vcf.gz --exclude-filtered --select-type-to-include SNP --select-type-to-include INDEL
done
```


