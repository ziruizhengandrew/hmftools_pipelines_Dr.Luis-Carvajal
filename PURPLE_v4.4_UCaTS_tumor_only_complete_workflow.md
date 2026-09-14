---
title: "PURPLE v4.4 Tumor-Only WGS Workflow — UCaTS Organoids"
author: "UCaTS WGS analysis"
date: "2026-09-14"
output:
  html_document:
    toc: true
    toc_float: true
    number_sections: true
  pdf_document:
    toc: true
    number_sections: true
---

> **Scope.** This notebook documents the complete PURPLE portion of the UCaTS Organoids tumor-only WGS workflow: AMBER/COBALT inputs, the final PURPLE v4.4 rerun, post-run R plots, Circos generation, validation, provenance, troubleshooting, and the final directory naming convention.
>
> **Safety.** All code chunks are documentation and use `eval=FALSE` so exporting this notebook will not rerun the pipeline. Historical `.command.txt`, program logs, and console logs should not be edited because they record the exact paths and commands that were actually executed at the time.

# 1. Final directory organization

After cleanup/rename, the PURPLE area is:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/
├── Amber_Cobalt_OldPurple/
│   ├── <SAMPLE>/amber/
│   ├── <SAMPLE>/cobalt/
│   └── <SAMPLE>/purple/           # historical/old PURPLE output
├── Purple_plot_regen_with_charts/
│   └── <SAMPLE>/                  # isolated rerun used only to generate Circos
├── Purple_plot_tools/
│   ├── copyNumberPlots.PURPLE_v4.4.R
│   └── circos_purple_wrapper.sh
├── Purple_reference/
├── Purple_rerun/
│   └── <SAMPLE>/purple/           # FINAL / CURRENT PURPLE output
└── Purple_tools/
    └── purple_v4.4.jar
```

Rename history:

```text
Old name                                  Current name
----------------------------------------------------------------
Purple_output                             Amber_Cobalt_OldPurple
Purple_rerun_PAVE_ESVEE_noMSI            Purple_rerun
```

The **formal current PURPLE output** is:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun
```

`Amber_Cobalt_OldPurple` is retained because its AMBER and COBALT outputs are reused as inputs. Its old `purple/` output is historical and is not the current final result.

`Purple_plot_regen_with_charts` is an auxiliary directory used to safely regenerate Circos. It is not the formal result directory.

## Historical logs after rename

Old logs may still show paths such as:

```text
.../10_Purple/Purple_output/...
.../10_Purple/Purple_rerun_PAVE_ESVEE_noMSI/...
```

That is intentional. Do not rewrite those historical records.

# 2. What PURPLE does

PURPLE estimates tumor purity, ploidy, allele-specific copy number, copy-number segments, and driver-related copy-number information from WGS data.

```text
AMBER
  └── tumor B-allele frequencies

COBALT
  └── read-depth ratios

PAVE-annotated SAGE somatic VCF
  └── SNV / INDEL information

ESVEE somatic VCF
  └── structural variants

                ↓

            PURPLE v4.4

                ↓

purity / ploidy / CNV / gene CN / enriched VCF / QC / plots
```

This project uses **tumor-only PURPLE**, so no matched normal/reference sample is supplied.

# 3. Versions used

```text
AMBER       v4.3
COBALT      v3.0
PURPLE      v4.4
PAVE        v1.9
ESVEE       v2.0
REDUX       v2.0.4
```

Java:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

PURPLE JAR:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_tools/purple_v4.4.jar
```

Final Circos environment:

```text
/home/zzr123/.conda/envs/purple_plot/bin/perl
/home/zzr123/.conda/envs/purple_plot/bin/circos
```

Observed working versions:

```text
Perl    5.32.1
Circos  0.69-8
```

# 4. Reference resources

Shared HMFtools resource bundle:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8
```

GRCh38 FASTA:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

Key resources:

```text
GC profile:
.../dna/copy_number/GC_profile.1000bp.38.cnp

Driver panel:
.../common/DriverGenePanel.38.tsv

Ensembl cache:
.../common/ensembl_data

Somatic hotspots:
.../dna/variants/KnownHotspots.somatic.38.vcf.gz

AMBER loci:
.../dna/copy_number/AmberGermlineSites.38.tsv.gz

Tumor-only diploid regions:
.../dna/copy_number/DiploidRegions.38.bed.gz
```

Use the shared resource bundle instead of making repeated per-module copies.

# 5. Samples

```r
samples <- c(
  "I_26166_S_29146","I_26227_S_29149","I_26395_S_29148",
  "I_26784_S_29156","I_26785_S_29154","I_26786_S_29155",
  "I_27483_S_29161","I_27619_S_29166","I_27621_S_29160",
  "I_27622_S_29150","I_27623_S_29147","I_27660_S_29145",
  "I_27661_S_29153","I_27662_S_29152","I_27663_S_29151",
  "I_27664_S_29157","I_27665_S_29158","I_27666_S_29159",
  "I_27670_S_29162","I_27671_S_29163","I_27673_S_29165",
  "I_27674_S_29167","I_27675_S_29168"
)
```

Total: **23 tumor samples**.

# 6. Upstream inputs used by final PURPLE

## 6.1 REDUX BAMs

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam
```

PURPLE does not directly need the BAM when completed AMBER and COBALT outputs are reused.

## 6.2 AMBER

Current location:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple/<SAMPLE>/amber
```

Expected files:

```text
<SAMPLE>.amber.qc
<SAMPLE>.amber.baf.tsv.gz
<SAMPLE>.amber.baf.pcf
```

## 6.3 COBALT

Current location:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple/<SAMPLE>/cobalt
```

Expected files:

```text
<SAMPLE>.cobalt.ratio.tsv.gz
<SAMPLE>.cobalt.ratio.pcf
```

## 6.4 PAVE somatic VCF

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/09_PAVE/PAVE_output/<SAMPLE>/<SAMPLE>.sage.somatic.pave.vcf.gz
```

Passed to PURPLE as:

```text
-somatic_vcf
```

## 6.5 ESVEE somatic SV VCF

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/08_ESVEE/ESVEE_output/<SAMPLE>/<SAMPLE>.esvee.somatic.vcf.gz
```

Passed to PURPLE as:

```text
-somatic_sv_vcf
```

# 7. MSI decision

REDUX MSI prediction was investigated separately. The final formal PURPLE run intentionally used:

```text
NO -redux_tumor
NO REDUX MSI prediction
```

The former descriptive directory name was therefore:

```text
Purple_rerun_PAVE_ESVEE_noMSI
```

It has now simply been renamed:

```text
Purple_rerun
```

The rename does not change the biological results.

# 8. AMBER and COBALT — historical tumor-only setup

These steps are retained for reproducibility. They do **not** need to be rerun for the current final analysis.

## 8.1 Common setup

```r
BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"
REDUX_DIR <- file.path(BASE,"06_REDux","Redux_BAMs")
PURPLE_BASE <- file.path(BASE,"10_Purple")
AC_ROOT <- file.path(PURPLE_BASE,"Amber_Cobalt_OldPurple")

JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"
AMBER_JAR <- file.path(PURPLE_BASE,"Purple_tools","amber_v4.3.jar")
COBALT_JAR <- file.path(PURPLE_BASE,"Purple_tools","cobalt_v3.0.jar")

HMF_REF <- "/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"
AMBER_LOCI <- file.path(HMF_REF,"dna","copy_number","AmberGermlineSites.38.tsv.gz")
GC_PROFILE <- file.path(HMF_REF,"dna","copy_number","GC_profile.1000bp.38.cnp")
DIPLOID_REGIONS <- file.path(HMF_REF,"dna","copy_number","DiploidRegions.38.bed.gz")

THREADS <- 8L
JAVA_MEMORY <- "48G"
```

## 8.2 AMBER tumor-only command

```r
sample <- "I_26166_S_29146"

bam <- file.path(
  REDUX_DIR,
  sample,
  paste0(sample,".redux.bam")
)

amber_out <- file.path(
  AC_ROOT,
  sample,
  "amber"
)

dir.create(
  amber_out,
  recursive=TRUE,
  showWarnings=FALSE
)

amber_args <- c(
  paste0("-Xmx",JAVA_MEMORY),
  "-jar",AMBER_JAR,
  "-tumor",sample,
  "-tumor_bam",bam,
  "-loci",AMBER_LOCI,
  "-ref_genome_version","38",
  "-threads",as.character(THREADS),
  "-output_dir",amber_out
)

system2(
  JAVA,
  amber_args
)
```

## 8.3 COBALT tumor-only command

```r
cobalt_out <- file.path(
  AC_ROOT,
  sample,
  "cobalt"
)

dir.create(
  cobalt_out,
  recursive=TRUE,
  showWarnings=FALSE
)

cobalt_args <- c(
  paste0("-Xmx",JAVA_MEMORY),
  "-jar",COBALT_JAR,
  "-tumor",sample,
  "-tumor_bam",bam,
  "-gc_profile",GC_PROFILE,
  "-tumor_only_diploid_bed",DIPLOID_REGIONS,
  "-ref_genome_version","38",
  "-threads",as.character(THREADS),
  "-output_dir",cobalt_out
)

system2(
  JAVA,
  cobalt_args
)
```

Completion should always be validated from required files, not only from process exit code.

# 9. Final PURPLE v4.4 core rerun

The formal result is now:

```text
10_Purple/Purple_rerun/
```

It reuses AMBER/COBALT and uses the current PAVE and ESVEE outputs.

## 9.1 Current paths

```r
BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"
PURPLE_BASE <- file.path(BASE,"10_Purple")
AMBER_COBALT_ROOT <- file.path(PURPLE_BASE,"Amber_Cobalt_OldPurple")
PAVE_DIR <- file.path(BASE,"09_PAVE","PAVE_output")
ESVEE_DIR <- file.path(BASE,"08_ESVEE","ESVEE_output")
PURPLE_ROOT <- file.path(PURPLE_BASE,"Purple_rerun")

JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"
PURPLE_JAR <- file.path(PURPLE_BASE,"Purple_tools","purple_v4.4.jar")

REF_FASTA <- "/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"
HMF_REF <- "/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"
GC_PROFILE <- file.path(HMF_REF,"dna","copy_number","GC_profile.1000bp.38.cnp")
DRIVER_PANEL <- file.path(HMF_REF,"common","DriverGenePanel.38.tsv")
ENSEMBL_DIR <- file.path(HMF_REF,"common","ensembl_data")
SOMATIC_HOTSPOTS <- file.path(HMF_REF,"dna","variants","KnownHotspots.somatic.38.vcf.gz")

JAVA_MEMORY <- "48G"
THREADS <- 8L
```

## 9.2 Provenance helpers

```r
valid_file <- function(x){
  length(x)==1 &&
    !is.na(x) &&
    file.exists(x) &&
    !is.na(file.info(x)$size) &&
    file.info(x)$size > 0
}

sha256_file <- function(x){
  if(!valid_file(x)) return(NA_character_)
  z <- suppressWarnings(system2("sha256sum",x,stdout=TRUE,stderr=TRUE))
  if(length(z)==0) return(NA_character_)
  strsplit(z[1],"\\s+")[[1]][1]
}

cmd_string <- function(exe,args){
  paste(shQuote(exe),paste(shQuote(args),collapse=" "))
}

run_recorded <- function(exe,args,command_file,log_file,metadata=character()){
  dir.create(dirname(command_file),recursive=TRUE,showWarnings=FALSE)
  dir.create(dirname(log_file),recursive=TRUE,showWarnings=FALSE)

  cmd <- cmd_string(exe,args)

  writeLines(
    c(
      paste0("DATE=",format(Sys.time(),"%Y-%m-%d %H:%M:%S %Z")),
      metadata,
      "",
      "ACTUAL_COMMAND:",
      cmd
    ),
    command_file
  )

  cat("\nACTUAL COMMAND:\n",cmd,"\n",sep="")
  cat("Command record:",command_file,"\n")
  cat("Program log:",log_file,"\n")

  start <- Sys.time()
  status <- system2(exe,args=args,stdout=log_file,stderr=log_file)
  runtime <- round(as.numeric(difftime(Sys.time(),start,units="mins")),2)

  cat("Exit status:",status,"\n")
  cat("Runtime:",runtime,"min\n")

  list(status=status,runtime=runtime,log=log_file,command=command_file)
}
```

## 9.3 Preflight

```r
required_files <- c(
  JAVA,
  PURPLE_JAR,
  REF_FASTA,
  paste0(REF_FASTA,".fai"),
  GC_PROFILE,
  DRIVER_PANEL,
  SOMATIC_HOTSPOTS
)

required_dirs <- c(
  AMBER_COBALT_ROOT,
  PAVE_DIR,
  ESVEE_DIR,
  ENSEMBL_DIR
)

fc <- data.frame(
  path=required_files,
  exists=file.exists(required_files),
  size_bytes=ifelse(file.exists(required_files),file.info(required_files)$size,NA_real_)
)

dc <- data.frame(
  path=required_dirs,
  exists=dir.exists(required_dirs)
)

print(fc,row.names=FALSE)
print(dc,row.names=FALSE)

if(!all(fc$exists) || !all(dc$exists)){
  stop("Missing required PURPLE input/reference.")
}

cat("\nJAVA:\n")
system2(JAVA,"-version")
cat("\nPURPLE JAR SHA256:",sha256_file(PURPLE_JAR),"\n")
```

## 9.4 Per-sample final PURPLE command

```r
run_purple <- function(sample){

  amber_dir <- file.path(AMBER_COBALT_ROOT,sample,"amber")
  cobalt_dir <- file.path(AMBER_COBALT_ROOT,sample,"cobalt")

  pave_vcf <- file.path(
    PAVE_DIR,
    sample,
    paste0(sample,".sage.somatic.pave.vcf.gz")
  )

  esvee_vcf <- file.path(
    ESVEE_DIR,
    sample,
    paste0(sample,".esvee.somatic.vcf.gz")
  )

  sample_root <- file.path(PURPLE_ROOT,sample)
  purple_out <- file.path(sample_root,"purple")
  log_dir <- file.path(sample_root,"logs")
  command_dir <- file.path(sample_root,"commands")

  dir.create(purple_out,recursive=TRUE,showWarnings=FALSE)
  dir.create(log_dir,recursive=TRUE,showWarnings=FALSE)
  dir.create(command_dir,recursive=TRUE,showWarnings=FALSE)

  required_output <- c(
    file.path(purple_out,paste0(sample,".purple.qc")),
    file.path(purple_out,paste0(sample,".purple.purity.tsv")),
    file.path(purple_out,paste0(sample,".purple.purity.range.tsv")),
    file.path(purple_out,paste0(sample,".purple.segment.tsv")),
    file.path(purple_out,paste0(sample,".purple.cnv.somatic.tsv")),
    file.path(purple_out,paste0(sample,".purple.cnv.gene.tsv")),
    file.path(purple_out,paste0(sample,".purple.driver.catalog.somatic.tsv")),
    file.path(purple_out,paste0(sample,".purple.somatic.vcf.gz")),
    file.path(purple_out,paste0(sample,".purple.somatic.vcf.gz.tbi")),
    file.path(purple_out,paste0(sample,".purple.sv.vcf.gz")),
    file.path(purple_out,paste0(sample,".purple.sv.vcf.gz.tbi"))
  )

  if(all(vapply(required_output,valid_file,logical(1)))){
    cat("[",sample,"] PURPLE already complete -> SKIP\n",sep="")
    return(data.frame(Sample=sample,Status="COMPLETE_ALREADY",Runtime_min=0))
  }

  input_required <- c(
    file.path(amber_dir,paste0(sample,".amber.qc")),
    file.path(amber_dir,paste0(sample,".amber.baf.tsv.gz")),
    file.path(amber_dir,paste0(sample,".amber.baf.pcf")),
    file.path(cobalt_dir,paste0(sample,".cobalt.ratio.tsv.gz")),
    file.path(cobalt_dir,paste0(sample,".cobalt.ratio.pcf")),
    pave_vcf,
    esvee_vcf
  )

  if(!all(vapply(input_required,valid_file,logical(1)))){
    stop("Missing input for sample: ",sample)
  }

  args <- c(
    paste0("-Xmx",JAVA_MEMORY),
    "-jar",PURPLE_JAR,
    "-tumor",sample,
    "-amber_dir",amber_dir,
    "-cobalt_dir",cobalt_dir,
    "-somatic_vcf",pave_vcf,
    "-somatic_sv_vcf",esvee_vcf,
    "-gc_profile",GC_PROFILE,
    "-ref_genome",REF_FASTA,
    "-ref_genome_version","38",
    "-ensembl_data_dir",ENSEMBL_DIR,
    "-driver_gene_panel",DRIVER_PANEL,
    "-somatic_hotspots",SOMATIC_HOTSPOTS,
    "-threads",as.character(THREADS),
    "-no_charts",
    "-output_dir",purple_out
  )

  rr <- run_recorded(
    JAVA,
    args,
    file.path(command_dir,paste0(sample,".purple.command.txt")),
    file.path(log_dir,"purple.log"),
    metadata=c(
      paste0("SAMPLE=",sample),
      "TOOL=PURPLE v4.4",
      "MODE=TUMOR_ONLY",
      "REDUX_MSI_PREDICTION=NOT_USED",
      paste0("PURPLE_JAR_SHA256=",sha256_file(PURPLE_JAR))
    )
  )

  output_ok <- all(vapply(required_output,valid_file,logical(1)))

  data.frame(
    Sample=sample,
    Status=ifelse(rr$status==0 && output_ok,"COMPLETE","FAILED"),
    Runtime_min=rr$runtime,
    stringsAsFactors=FALSE
  )
}
```

## 9.5 Run all samples with stop-on-failure

```r
results <- data.frame()

for(i in seq_along(samples)){
  sample <- samples[i]
  cat("\n[",i,"/",length(samples),"] ",sample,"\n",sep="")

  x <- run_purple(sample)
  results <- rbind(results,x)

  write.csv(
    results,
    file.path(PURPLE_ROOT,"PURPLE_rerun_23_samples_status.csv"),
    row.names=FALSE
  )

  if(!x$Status %in% c("COMPLETE","COMPLETE_ALREADY")){
    stop("PURPLE failed for ",sample,". Remaining samples were NOT started.")
  }
}
```

# 10. Why `Purple_rerun` is the final output

Use:

```text
10_Purple/Purple_rerun/<SAMPLE>/purple/
```

for downstream analysis because it uses the corrected/current PAVE and ESVEE inputs.

Do not use:

```text
10_Purple/Amber_Cobalt_OldPurple/<SAMPLE>/purple/
```

as the current final PURPLE output.

# 11. Important PURPLE outputs

Per sample:

```text
<SAMPLE>.purple.qc
<SAMPLE>.purple.purity.tsv
<SAMPLE>.purple.purity.range.tsv
<SAMPLE>.purple.segment.tsv
<SAMPLE>.purple.cnv.somatic.tsv
<SAMPLE>.purple.cnv.gene.tsv
<SAMPLE>.purple.driver.catalog.somatic.tsv
<SAMPLE>.purple.somatic.vcf.gz
<SAMPLE>.purple.somatic.vcf.gz.tbi
<SAMPLE>.purple.sv.vcf.gz
<SAMPLE>.purple.sv.vcf.gz.tbi
purple.version
```

Interpretation:

```text
purple.purity.tsv
  → best-fit purity, ploidy, sex/gender inference, fit summary

purple.purity.range.tsv
  → candidate purity/ploidy solutions and scores

purple.segment.tsv
  → fitted genomic segments

purple.cnv.somatic.tsv
  → genome-wide somatic copy-number segments

purple.cnv.gene.tsv
  → gene-level copy number

purple.driver.catalog.somatic.tsv
  → somatic driver catalog

purple.somatic.vcf.gz
  → somatic SNV/INDEL VCF enriched by PURPLE

purple.sv.vcf.gz
  → structural variant VCF enriched by PURPLE

purple.qc
  → PURPLE QC
```

# 12. Post-run R plots

The formal core run used `-no_charts`, so the standard R plots were generated afterward from the successful formal PURPLE outputs without recomputing the biological fit.

Expected R plots:

```text
<SAMPLE>.copynumber.png
<SAMPLE>.map.png
<SAMPLE>.purity.range.png
<SAMPLE>.segment.png
```

The exact plotting script was extracted from the actual PURPLE v4.4 JAR:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_plot_tools/copyNumberPlots.PURPLE_v4.4.R
```

Observed SHA256:

```text
5208bd4bda0c03ff9fcd47f0fd7656d2cf8dad84214131ed9c567f33676806bd
```

## 12.1 Extract plotting script from JAR

```r
jar_contents <- utils::unzip(PURPLE_JAR,list=TRUE)
plot_entry <- jar_contents$Name[grepl("copyNumberPlots\\.R$",jar_contents$Name)]
stopifnot(length(plot_entry)==1)

PLOT_TOOL_DIR <- file.path(PURPLE_BASE,"Purple_plot_tools")
COPYNUMBER_PLOTS <- file.path(PLOT_TOOL_DIR,"copyNumberPlots.PURPLE_v4.4.R")

tmp <- tempfile("purple_plot_extract_")
dir.create(tmp,recursive=TRUE)

utils::unzip(
  PURPLE_JAR,
  files=plot_entry,
  exdir=tmp,
  overwrite=TRUE
)

file.copy(
  file.path(tmp,plot_entry),
  COPYNUMBER_PLOTS,
  overwrite=TRUE
)

unlink(tmp,recursive=TRUE,force=TRUE)
```

## 12.2 Generate R plots

```r
R_SCRIPT <- "/cvmfs/hpc.ucdavis.edu/sw/conda/environments/r-4.4.2/bin/Rscript"

sample <- "I_26166_S_29146"
purple_dir <- file.path(PURPLE_ROOT,sample,"purple")
plot_dir <- file.path(purple_dir,"plot")
dir.create(plot_dir,recursive=TRUE,showWarnings=FALSE)

system2(
  R_SCRIPT,
  c(
    COPYNUMBER_PLOTS,
    sample,
    purple_dir,
    plot_dir
  )
)
```

Final result:

```text
R plots complete: 23/23
```

# 13. Circos environment

Initial Circos attempts failed because the executable was being launched with the wrong/system Perl and could not find the required Perl modules.

Final working environment:

```text
/home/zzr123/.conda/envs/purple_plot/bin/perl
/home/zzr123/.conda/envs/purple_plot/bin/circos
```

Critical rule: do not rely on `Sys.which("circos")` in Jupyter because PATH may resolve to another installation.

## 13.1 Circos wrapper

PURPLE expects an executable path for `-circos`. The final wrapper forces Circos to use the matching Perl:

```bash
#!/usr/bin/env bash
set -euo pipefail
exec '/home/zzr123/.conda/envs/purple_plot/bin/perl' \
     '/home/zzr123/.conda/envs/purple_plot/bin/circos' "$@"
```

Saved as:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_plot_tools/circos_purple_wrapper.sh
```

# 14. Why an isolated Circos regeneration was needed

The formal core PURPLE run used:

```text
-no_charts
```

In this actual project run, usable Circos configs/images were not produced in the formal output directory.

Therefore:

1. Keep the formal PURPLE result untouched.
2. Generate the four R plots directly from the formal core output.
3. Rerun PURPLE only in a separate plotting directory with `-circos`.
4. Do **not** use `-no_charts` in that isolated Circos regeneration.
5. Compare core output hashes before accepting the regenerated Circos plots.

Isolated directory:

```text
10_Purple/Purple_plot_regen_with_charts/
```

# 15. Isolated Circos PURPLE regeneration

```r
CIRCOS_REGEN_ROOT <- file.path(
  PURPLE_BASE,
  "Purple_plot_regen_with_charts"
)

CIRCOS_WRAPPER <- file.path(
  PURPLE_BASE,
  "Purple_plot_tools",
  "circos_purple_wrapper.sh"
)

sample <- "I_26166_S_29146"

amber_dir <- file.path(AMBER_COBALT_ROOT,sample,"amber")
cobalt_dir <- file.path(AMBER_COBALT_ROOT,sample,"cobalt")

pave_vcf <- file.path(
  PAVE_DIR,
  sample,
  paste0(sample,".sage.somatic.pave.vcf.gz")
)

esvee_vcf <- file.path(
  ESVEE_DIR,
  sample,
  paste0(sample,".esvee.somatic.vcf.gz")
)

regen_purple <- file.path(
  CIRCOS_REGEN_ROOT,
  sample,
  "purple"
)

dir.create(regen_purple,recursive=TRUE,showWarnings=FALSE)

args <- c(
  "-Xmx48G",
  "-jar",PURPLE_JAR,
  "-tumor",sample,
  "-amber_dir",amber_dir,
  "-cobalt_dir",cobalt_dir,
  "-somatic_vcf",pave_vcf,
  "-somatic_sv_vcf",esvee_vcf,
  "-gc_profile",GC_PROFILE,
  "-ref_genome",REF_FASTA,
  "-ref_genome_version","38",
  "-ensembl_data_dir",ENSEMBL_DIR,
  "-driver_gene_panel",DRIVER_PANEL,
  "-somatic_hotspots",SOMATIC_HOTSPOTS,
  "-threads","8",

  # IMPORTANT: no -no_charts here

  "-circos",CIRCOS_WRAPPER,
  "-output_dir",regen_purple
)

system2(
  JAVA,
  args
)
```

Expected Circos output:

```text
<SAMPLE>.input.png
<SAMPLE>.circos.png
```

# 16. Core consistency validation

Before accepting Circos plots from the isolated rerun, compare formal vs regenerated core outputs by SHA256.

Files compared:

```text
<SAMPLE>.purple.purity.tsv
<SAMPLE>.purple.segment.tsv
<SAMPLE>.purple.cnv.somatic.tsv
```

Example:

```r
compare_names <- c(
  paste0(sample,".purple.purity.tsv"),
  paste0(sample,".purple.segment.tsv"),
  paste0(sample,".purple.cnv.somatic.tsv")
)

comparison <- do.call(
  rbind,
  lapply(
    compare_names,
    function(filename){

      main_file <- file.path(
        PURPLE_ROOT,
        sample,
        "purple",
        filename
      )

      regen_file <- file.path(
        CIRCOS_REGEN_ROOT,
        sample,
        "purple",
        filename
      )

      main_hash <- sha256_file(main_file)
      regen_hash <- sha256_file(regen_file)

      data.frame(
        File=filename,
        Main_SHA256=main_hash,
        Regen_SHA256=regen_hash,
        Match=!is.na(main_hash) && !is.na(regen_hash) && identical(main_hash,regen_hash)
      )
    }
  )
)

print(comparison,row.names=FALSE)
stopifnot(all(comparison$Match))
```

Final observed result:

```text
Core SHA256 matches: 23/23
```

Only after this check were the two Circos PNGs copied into the formal `Purple_rerun/<SAMPLE>/purple/plot/` directory.

# 17. Final plots per sample

Each formal sample now has six validated images:

```text
<SAMPLE>.copynumber.png
<SAMPLE>.map.png
<SAMPLE>.purity.range.png
<SAMPLE>.segment.png
<SAMPLE>.input.png
<SAMPLE>.circos.png
```

Final verified project status:

```text
R plots complete:          23/23
Circos copied:             23/23
Core SHA256 matches:       23/23
Final six-plot complete:   23/23
Final visualization run:   COMPLETED
```

# 18. Circos interpretation

`<SAMPLE>.input.png` shows major inputs to PURPLE, especially AMBER BAF and COBALT read-depth-ratio information.

`<SAMPLE>.circos.png` summarizes the fitted tumor genome, including copy-number changes, allele-specific changes/LOH, variants, and structural variants.

# 19. Downstream tools — always use the latest formal PURPLE

When LINX, CHORD, ORANGE, QC aggregation, purity/ploidy extraction, CNV analysis, or another downstream step needs the current PURPLE output, use:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun/<SAMPLE>/purple
```

Do **not** use:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple/<SAMPLE>/purple
```

for current final downstream analysis.

# 20. Final validation script after rename

```r
BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"
PURPLE_ROOT <- file.path(BASE,"10_Purple","Purple_rerun")

samples <- c(
  "I_26166_S_29146","I_26227_S_29149","I_26395_S_29148",
  "I_26784_S_29156","I_26785_S_29154","I_26786_S_29155",
  "I_27483_S_29161","I_27619_S_29166","I_27621_S_29160",
  "I_27622_S_29150","I_27623_S_29147","I_27660_S_29145",
  "I_27661_S_29153","I_27662_S_29152","I_27663_S_29151",
  "I_27664_S_29157","I_27665_S_29158","I_27666_S_29159",
  "I_27670_S_29162","I_27671_S_29163","I_27673_S_29165",
  "I_27674_S_29167","I_27675_S_29168"
)

valid_file <- function(x){
  file.exists(x) &&
    !is.na(file.info(x)$size) &&
    file.info(x)$size > 0
}

validation <- do.call(
  rbind,
  lapply(
    samples,
    function(sample){

      p <- file.path(PURPLE_ROOT,sample,"purple")
      plot_dir <- file.path(p,"plot")

      required_core <- c(
        file.path(p,paste0(sample,".purple.qc")),
        file.path(p,paste0(sample,".purple.purity.tsv")),
        file.path(p,paste0(sample,".purple.purity.range.tsv")),
        file.path(p,paste0(sample,".purple.segment.tsv")),
        file.path(p,paste0(sample,".purple.cnv.somatic.tsv")),
        file.path(p,paste0(sample,".purple.cnv.gene.tsv")),
        file.path(p,paste0(sample,".purple.driver.catalog.somatic.tsv")),
        file.path(p,paste0(sample,".purple.somatic.vcf.gz")),
        file.path(p,paste0(sample,".purple.sv.vcf.gz"))
      )

      required_plots <- c(
        file.path(plot_dir,paste0(sample,".copynumber.png")),
        file.path(plot_dir,paste0(sample,".map.png")),
        file.path(plot_dir,paste0(sample,".purity.range.png")),
        file.path(plot_dir,paste0(sample,".segment.png")),
        file.path(plot_dir,paste0(sample,".input.png")),
        file.path(plot_dir,paste0(sample,".circos.png"))
      )

      data.frame(
        Sample=sample,
        Core_OK=all(vapply(required_core,valid_file,logical(1))),
        Six_plots_OK=all(vapply(required_plots,valid_file,logical(1))),
        stringsAsFactors=FALSE
      )
    }
  )
)

print(validation,row.names=FALSE)
cat("\nCore complete:",sum(validation$Core_OK),"/23\n")
cat("Six plots complete:",sum(validation$Six_plots_OK),"/23\n")
```

# 21. Provenance requirements for future runs

Every future production run should retain:

```text
1. Tool name + exact version
2. JAR path
3. JAR SHA256
4. Reference paths
5. Explicit input paths
6. Explicit output path
7. Exact resolved command
8. Per-sample .command.txt
9. Per-sample stdout/stderr log
10. Exit status
11. Runtime
12. Required-output validation
13. Final status table
14. Full console log
15. R sessionInfo()
16. Executed notebook/Rmd
```

A sample should be marked `COMPLETE` only after required outputs exist and are non-empty. A sample should be `SKIP` only if the same output validation passes.

# 22. Troubleshooting history and lessons

## 22.1 Partial plots were initially mistaken for a complete plotting run

Final rule:

```text
Explicitly validate every expected PNG.
```

## 22.2 Circos Perl modules were missing

Examples included:

```text
Config::General
Font::TTF::Font
GD
GD::Polyline
List::MoreUtils
Math::Bezier
Math::Round
Math::VecStat
Params::Validate
Readonly
Regexp::Common
SVG
Set::IntSpan
Statistics::Basic
Text::Format
```

Final fix: dedicated `purple_plot` environment.

## 22.3 Correct Circos executable but wrong Perl

Directly invoking the Circos script could still resolve the wrong/system Perl.

Final fix:

```text
/home/zzr123/.conda/envs/purple_plot/bin/perl
    ↓
/home/zzr123/.conda/envs/purple_plot/bin/circos
```

plus the wrapper used by PURPLE.

## 22.4 Formal run had no usable Circos output

The formal core run used `-no_charts`. The verified project solution was an isolated rerun using `-circos` and omitting `-no_charts`.

## 22.5 Concern that plotting rerun could alter core results

Final fix: SHA256 comparison of purity, segment, and somatic CNV files. All 23 matched.

# 23. Final project state

```text
Formal PURPLE core:             23/23 complete
R plots:                        23/23 complete
Circos input plots:             23/23 complete
Circos final plots:             23/23 complete
Core SHA256 consistency:        23/23 match
Formal core overwritten:        NO
Redux MSI prediction used:      NO
```

Formal final output:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun
```

# 24. Exporting this notebook

This is an R Markdown document. All pipeline chunks are `eval=FALSE`, so exporting does not execute the pipeline.

## RStudio

```text
Knit → Knit to HTML
```

or:

```text
Knit → Knit to PDF
```

PDF requires a working LaTeX installation.

## R command line

HTML:

```r
rmarkdown::render(
  "PURPLE_v4.4_UCaTS_tumor_only_complete_workflow.Rmd",
  output_format="html_document"
)
```

PDF:

```r
rmarkdown::render(
  "PURPLE_v4.4_UCaTS_tumor_only_complete_workflow.Rmd",
  output_format="pdf_document"
)
```

# 25. Official references

PURPLE documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/purple/README.md
```

HMFtools repository/version overview:

```text
https://github.com/hartwigmedical/hmftools
```

The official PURPLE documentation covers PURPLE inputs, tumor-only mode, `-circos`, `-no_charts`, output files, and post-run chart generation with `copyNumberPlots.R`.
