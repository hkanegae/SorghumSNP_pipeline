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
2. Making index of the genome
3. Alignment of Illumina reads to the reference genome
4. Make clean BAM
5. Remove PCR duplicates
```
$ java -jar picard.jar MarkDuplicates \
    INPUT=alignment.merge.bam \
    OUTPUT=alignment.rmdup.bam \
    METRICS_FILE=rmdup.matrix \
    REMOVE_DUPLICATES=true \
    MAX_RECORDS_IN_RAM=1000000 \
    TMP_DIR=./tmp
$ samtools index alignment.rmdup.bam
```




