# UCaTS Organoids WGS — Final HMFtools Commands

> **Purpose:** Consolidated command reference for the UCaTS Organoids tumor-only WGS pipeline.  
> **Project root:** `/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids`  
> **Java:** `/home/zzr123/.conda/envs/purple/bin/java`  
> **Genome:** GRCh38 / hg38  
> **Shared HMF reference bundle:**  
> `/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8`
>
> **Important:** Where a run generated a per-sample `.command.txt`, that file is the authoritative record of the exact historical command actually executed. The commands below are the resolved command templates corresponding to the final project scripts.

---

## 1. REDUX v2.0.4 — Main preprocessing

### Input

```text
05_ASCAT_CN/BAMs/<SAMPLE>_tumor.bam
```

### Output

```text
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx40G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_tools/redux_v2.0.4.jar \
    -sample SAMPLE \
    -input_bam /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BAMs/SAMPLE_tumor.bam \
    -ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa \
    -ref_genome_version V38 \
    -sequencing_type ILLUMINA \
    -unmap_regions /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_reference/unmap_regions.38.tsv \
    -ref_genome_msi_file /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_reference/msi_jitter_sites.38.tsv.gz \
    -form_consensus \
    -bamtool /path/to/samtools \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_BAMs/SAMPLE \
    -log_level INFO \
    -threads 24
```

If the original tumor BAM did not already have an index:

```bash
samtools index -@ 24 \
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BAMs/SAMPLE_tumor.bam
```

---

## 2. REDUX v2.0.4 — MSI-only prediction rerun

> This stage does **not** regenerate the REDUX BAM. It runs on the existing REDUX BAM and attempts to generate `<SAMPLE>.redux.msi_prediction.tsv`.

### Input

```text
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam
```

### Output workspace

```text
06_REDux/MSI_only/<SAMPLE>/
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx40G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_tools/redux_v2.0.4.jar \
    -sample SAMPLE \
    -input_bam /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_BAMs/SAMPLE/SAMPLE.redux.bam \
    -ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa \
    -ref_genome_version V38 \
    -sequencing_type ILLUMINA \
    -ref_genome_msi_file /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/variants/msi_jitter_sites.38.tsv.gz \
    -msi_model_coefficients /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/variants/ms_model_coefficients.hmf_wgs.tsv \
    -msi_model_error_rates /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/variants/ms_model_error_rates.tso500.37.tsv \
    -bamtool /path/to/samtools \
    -bqr_jitter_msi_only \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/MSI_only/SAMPLE \
    -threads 24 \
    -log_level INFO
```

### Important caveat

The MSI model combination used here was:

```text
HMF WGS coefficients
+
TSO500.37 error rates
```

This was an **experimental test performed per mentor instruction** and should not be described as a standard matched WGS/GRCh38 MSI model configuration.

---

## 3. SAGE v5.0.2

### Input

```text
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam
```

### Output

```text
07_SAGE/SAGE_output/<SAMPLE>/<SAMPLE>.sage.somatic.vcf.gz
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx32G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/07_SAGE/SAGE_tools/sage_v5.0.2.jar \
    -tumor SAMPLE \
    -tumor_bam /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_BAMs/SAMPLE/SAMPLE.redux.bam \
    -ref_sample_count 0 \
    -ref_genome_version 38 \
    -ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa \
    -hotspots /path/to/KnownHotspots.somatic.38.vcf.gz \
    -high_confidence_bed /path/to/HG001_GRCh38_GIAB_highconf.bed.gz \
    -ensembl_data_dir /path/to/ensembl_data \
    -driver_gene_panel /path/to/DriverGenePanel.38.tsv \
    -output_vcf /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/07_SAGE/SAGE_output/SAMPLE/SAMPLE.sage.somatic.vcf.gz \
    -threads 16
```

### Tumor-only setting

```text
-ref_sample_count 0
```

---

## 4. PAVE v1.9

### Input

```text
07_SAGE/SAGE_output/<SAMPLE>/<SAMPLE>.sage.somatic.vcf.gz
```

### Output

```text
09_PAVE/PAVE_output/<SAMPLE>/<SAMPLE>.sage.somatic.pave.vcf.gz
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx48G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/09_PAVE/PAVE_tools/pave_v1.9.jar \
    -sample SAMPLE \
    -input_vcf /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/07_SAGE/SAGE_output/SAMPLE/SAMPLE.sage.somatic.vcf.gz \
    -ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa \
    -ref_genome_version 38 \
    -ensembl_data_dir /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/common/ensembl_data \
    -driver_gene_panel /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/common/DriverGenePanel.38.tsv \
    -sequencing_type ILLUMINA \
    -pon_file /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/variants/hmf_wgs_sage_pon_1000.38.tsv.gz \
    -pon_filters 'HOTSPOT:6:5;PANEL:3:3;UNKNOWN:3:0' \
    -gnomad_freq_dir /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/variants/gnomad \
    -clinvar_vcf /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/variants/clinvar.38.vcf.gz \
    -mappability_bed /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/variants/mappability_150.38.bed.gz \
    -blacklist_bed /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/variants/KnownBlacklist.germline.38.bed \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/09_PAVE/PAVE_output/SAMPLE \
    -output_vcf /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/09_PAVE/PAVE_output/SAMPLE/SAMPLE.sage.somatic.pave.vcf.gz \
    -threads 4
```

---

## 5. ESVEE v2.0

### Input

```text
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam
```

### Output

```text
08_ESVEE/ESVEE_output/<SAMPLE>/<SAMPLE>.esvee.somatic.vcf.gz
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx32G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/08_ESVEE/ESVEE_tools/esvee_v2.0.jar \
    -tumor SAMPLE \
    -tumor_bam /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_BAMs/SAMPLE/SAMPLE.redux.bam \
    -ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa \
    -ref_genome_version 38 \
    -known_hotspot_file /path/to/known_fusions.38.bedpe \
    -pon_sgl_file /path/to/sgl_pon.38.bed.gz \
    -pon_sv_file /path/to/sv_pon.38.bedpe.gz \
    -repeat_mask_file /path/to/repeat_mask_data.38.fa.gz \
    -unmap_regions /path/to/unmap_regions.38.tsv \
    -bamtool /path/to/samtools \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/08_ESVEE/ESVEE_output/SAMPLE \
    -threads 24
```

If the native BWA library was found, the command additionally included:

```bash
-bwa_lib /path/to/libbwwwa.so
```

---

## 6. AMBER v4.3

### Input

```text
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx48G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_tools/amber_v4.3.jar \
    -tumor SAMPLE \
    -tumor_bam /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_BAMs/SAMPLE/SAMPLE.redux.bam \
    -loci /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/copy_number/AmberGermlineSites.38.tsv.gz \
    -ref_genome_version 38 \
    -threads 8 \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple/SAMPLE/amber
```

---

## 7. COBALT v3.0

### Input

```text
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx48G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_tools/cobalt_v3.0.jar \
    -tumor SAMPLE \
    -tumor_bam /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_BAMs/SAMPLE/SAMPLE.redux.bam \
    -gc_profile /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/copy_number/GC_profile.1000bp.38.cnp \
    -tumor_only_diploid_bed /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/copy_number/DiploidRegions.38.bed.gz \
    -ref_genome_version 38 \
    -threads 8 \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple/SAMPLE/cobalt
```

---

## 8. PURPLE v4.4 — Final formal core run

### Inputs

```text
AMBER:
10_Purple/Amber_Cobalt_OldPurple/<SAMPLE>/amber

COBALT:
10_Purple/Amber_Cobalt_OldPurple/<SAMPLE>/cobalt

PAVE:
09_PAVE/PAVE_output/<SAMPLE>/<SAMPLE>.sage.somatic.pave.vcf.gz

ESVEE:
08_ESVEE/ESVEE_output/<SAMPLE>/<SAMPLE>.esvee.somatic.vcf.gz
```

### Output

```text
10_Purple/Purple_rerun/<SAMPLE>/purple
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx48G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_tools/purple_v4.4.jar \
    -tumor SAMPLE \
    -amber_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple/SAMPLE/amber \
    -cobalt_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple/SAMPLE/cobalt \
    -somatic_vcf /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/09_PAVE/PAVE_output/SAMPLE/SAMPLE.sage.somatic.pave.vcf.gz \
    -somatic_sv_vcf /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/08_ESVEE/ESVEE_output/SAMPLE/SAMPLE.esvee.somatic.vcf.gz \
    -gc_profile /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/copy_number/GC_profile.1000bp.38.cnp \
    -ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa \
    -ref_genome_version 38 \
    -ensembl_data_dir /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/common/ensembl_data \
    -driver_gene_panel /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/common/DriverGenePanel.38.tsv \
    -somatic_hotspots /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/variants/KnownHotspots.somatic.38.vcf.gz \
    -threads 8 \
    -no_charts \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun/SAMPLE/purple
```

### Important

The final formal PURPLE run used:

```text
NO -redux_tumor
NO REDUX MSI prediction
```

---

## 9. PURPLE v4.4 — Chart / Circos regeneration

> This was an isolated plot-generation rerun. It was not used as the biological PURPLE result.

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx48G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_tools/purple_v4.4.jar \
    -tumor SAMPLE \
    -amber_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple/SAMPLE/amber \
    -cobalt_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple/SAMPLE/cobalt \
    -somatic_vcf /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/09_PAVE/PAVE_output/SAMPLE/SAMPLE.sage.somatic.pave.vcf.gz \
    -somatic_sv_vcf /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/08_ESVEE/ESVEE_output/SAMPLE/SAMPLE.esvee.somatic.vcf.gz \
    -gc_profile /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/copy_number/GC_profile.1000bp.38.cnp \
    -ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa \
    -ref_genome_version 38 \
    -ensembl_data_dir /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/common/ensembl_data \
    -driver_gene_panel /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/common/DriverGenePanel.38.tsv \
    -somatic_hotspots /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/variants/KnownHotspots.somatic.38.vcf.gz \
    -threads 8 \
    -circos /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_plot_tools/circos_purple_wrapper.sh \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_plot_regen_with_charts/SAMPLE/purple
```

Unlike the formal core run, this chart-generation command does **not** use `-no_charts`.

---

## 10. LINX v2.3.1

### Input

```text
10_Purple/Purple_rerun/<SAMPLE>/purple
```

### Output

```text
11_LINX/LINX_rerun/<SAMPLE>
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx16G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/11_LINX/LINX_tools/linx_v2.3.1.jar \
    -sample SAMPLE \
    -ref_genome_version 38 \
    -sv_vcf /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun/SAMPLE/purple/SAMPLE.purple.sv.vcf.gz \
    -purple_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun/SAMPLE/purple \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/11_LINX/LINX_rerun/SAMPLE \
    -ensembl_data_dir /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/common/ensembl_data \
    -known_fusion_file /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/dna/sv/known_fusion_data.38.csv \
    -driver_gene_panel /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/common/DriverGenePanel.38.tsv
```

---

## 11. CHORD v2.1.2

### Inputs

```text
10_Purple/Purple_rerun/<SAMPLE>/purple/<SAMPLE>.purple.somatic.vcf.gz
10_Purple/Purple_rerun/<SAMPLE>/purple/<SAMPLE>.purple.sv.vcf.gz
```

### Output

```text
13_CHORD/CHORD_rerun/<SAMPLE>
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx64G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/13_CHORD/CHORD_tools/chord_v2.1.2.jar \
    -sample SAMPLE \
    -snv_indel_vcf_file /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun/SAMPLE/purple/SAMPLE.purple.somatic.vcf.gz \
    -sv_vcf_file /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun/SAMPLE/purple/SAMPLE.purple.sv.vcf.gz \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/13_CHORD/CHORD_rerun/SAMPLE \
    -ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa \
    -threads 1 \
    -log_level INFO
```

### Important

The project uses:

```text
GRCh38 / hg38
```

not the GRCh37 FASTA shown in some generic CHORD examples.

---

## 12. BamMetrics v1.6.2

> BamMetrics used the **original tumor BAMs**, not REDUX BAMs.

### Input

```text
05_ASCAT_CN/BAMs/<SAMPLE>_tumor.bam
```

### Output

```text
05_ASCAT_CN/BamMetrics_output/<SAMPLE>
```

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx16G \
    -cp /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BamMetrics_tools/bam-tools_v1.6.2.jar \
    com.hartwig.hmftools.bamtools.metrics.BamMetrics \
    -sample SAMPLE \
    -bam_file /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BAMs/SAMPLE_tumor.bam \
    -ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa \
    -ref_genome_version V38 \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BamMetrics_output/SAMPLE \
    -threads 8
```

BamMetrics is different from most other steps because it uses:

```text
-cp <jar> com.hartwig.hmftools.bamtools.metrics.BamMetrics
```

rather than:

```text
-jar <jar>
```

---

## 13. QSEE v1.0 — Multi-sample tumor-only run

QSEE was run once for the complete tumor cohort rather than once per sample.

### Final command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx32G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/QSEE_tools/qsee_v1.0.jar \
    -sample_id_file /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/qsee_sample_ids.tsv \
    -sample_data_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/QSEE_input \
    -driver_gene_panel /quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/common/DriverGenePanel.38.tsv \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/QSEE_output \
    -sequencing_type ILLUMINA \
    -threads 23 \
    -merge_plots \
    -allow_missing_input
```

### Main outputs

```text
multisample.qsee.status.tsv.gz
multisample.qsee.vis.data.tsv.gz
multisample.qsee.vis.report.pdf
```

---

## 14. ORANGE v5.0.1

### Inputs

```text
PURPLE:
10_Purple/Purple_rerun/<SAMPLE>/purple

PURPLE plots:
10_Purple/Purple_rerun/<SAMPLE>/purple/plot

LINX:
11_LINX/LINX_rerun/<SAMPLE>
```

### Output

```text
16_Orange/Orange_rerun/<SAMPLE>
```

### Final tumor-only command

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx8G \
    -jar /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/16_Orange/Orange_tools/orange_v5.0.1.jar \
    -experiment_type WGS \
    -tumor SAMPLE \
    -ref_genome_version 38 \
    -sequencing_type ILLUMINA \
    -purple_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun/SAMPLE/purple \
    -purple_plot_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun/SAMPLE/purple/plot \
    -linx_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/11_LINX/LINX_rerun/SAMPLE \
    -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/16_Orange/Orange_rerun/SAMPLE \
    -log_level INFO \
    -add_disclaimer
```

If a valid per-sample optional QSEE or VirusInterpreter directory exists, the script can append:

```bash
-qsee_dir /path/to/sample/qsee
```

and/or:

```bash
-virus_dir /path/to/sample/virus
```

The current tumor-only ORANGE workflow deliberately does **not** pass:

```text
-reference
-chord_dir
-cuppa_dir
```

---

# Pipeline overview

```text
Original tumor BAM
        |
        +----------------------> BamMetrics
        |
        v
      REDUX
       /  \
      v    v
    SAGE  ESVEE
      |
      v
    PAVE
      \
       \
        +------+
               |
AMBER ----------|
COBALT ---------|
ESVEE ----------|
PAVE -----------|--> PURPLE
                     |
                     +------> CHORD
                     |
                     +------> LINX
                                |
                                v
                              ORANGE

QSEE consumes staged QC inputs from PURPLE + BamMetrics + COBALT + ESVEE + REDUX.
```

---

# Current canonical output roots

```text
REDUX:
06_REDux/Redux_BAMs

SAGE:
07_SAGE/SAGE_output

ESVEE:
08_ESVEE/ESVEE_output

PAVE:
09_PAVE/PAVE_output

PURPLE:
10_Purple/Purple_rerun

LINX:
11_LINX/LINX_rerun

CHORD:
13_CHORD/CHORD_rerun

QSEE:
14_QSEE/QSEE_output

ORANGE:
16_Orange/Orange_rerun
```

---

# Provenance rule

For reproducibility, use the following priority when documenting an historical run:

```text
1. Per-sample .command.txt generated during the actual run
2. Program log / console log from the same run
3. Executed R/Rmd/IPYNB with outputs preserved
4. This consolidated command reference
```

Do not rewrite historical `.command.txt` files merely because a project directory was later renamed.
