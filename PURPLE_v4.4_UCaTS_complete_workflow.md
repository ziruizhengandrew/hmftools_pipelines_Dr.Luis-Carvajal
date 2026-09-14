# PURPLE v4.4 Tumor-only WGS Workflow — UCATS Organoids

**Project:** UCATS Organoids tumor-only WGS  
**Cluster:** UC Davis HIVE  
**Genome:** GRCh38 / hg38  
**Final formal PURPLE output:** `10_Purple/Purple_rerun/`  
**Historical AMBER/COBALT + old PURPLE:** `10_Purple/Amber_Cobalt_OldPurple/`  
**PURPLE version:** 4.4  
**AMBER version:** 4.3  
**COBALT version:** 3.0  
**Final upstream variant inputs:** PAVE-annotated SAGE somatic VCF + ESVEE somatic SV VCF  
**Redux MSI prediction used in final PURPLE:** No

> This notebook documents the complete PURPLE workflow used for this project, including the original tumor-only AMBER/COBALT/PURPLE setup, the final PURPLE rerun with corrected PAVE + ESVEE inputs, post-run R plots, Circos regeneration, validation, provenance, and final directory organization.

---

## 1. Final workflow logic

```text
REDUX tumor BAM
      │
      ├── AMBER v4.3 ───────────────┐
      │                             │
      └── COBALT v3.0 ──────────────┤
                                    │
SAGE → PAVE annotated somatic VCF ──┤
                                    ├── PURPLE v4.4
ESVEE somatic SV VCF ───────────────┤
                                    │
HMF references + hg38 FASTA ────────┘
                                    │
                                    ▼
                     10_Purple/Purple_rerun/
                                    │
                     ┌──────────────┴──────────────┐
                     │                             │
              copyNumberPlots.R              Circos plots
                     │                             │
          4 standard PURPLE plots       2 Circos plots
```

The final formal PURPLE results are in:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun
```

The old `Purple_output/` directory was renamed to:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple
```

This directory is retained because the final rerun reused its successful AMBER and COBALT outputs.

---

## 2. Final directory organization

```text
10_Purple/
├── Amber_Cobalt_OldPurple/          # Historical AMBER + COBALT + old PURPLE
├── Purple_plot_regen_with_charts/   # Isolated PURPLE rerun used only to regenerate Circos
├── Purple_plot_tools/               # copyNumberPlots.R + Circos wrapper
├── Purple_reference/                # Historical local reference directory; shared refs preferred
├── Purple_rerun/                    # FINAL formal PURPLE output
└── Purple_tools/                    # AMBER/COBALT/PURPLE JARs
```

**Important:** old `.command.txt`, logs, and provenance files may still contain the historical directory names `Purple_output` or `Purple_rerun_PAVE_ESVEE_noMSI`. Do not edit those historical records. They document the paths that were actually used at execution time.

---

## 3. Shared reference resources

Use the shared HMF resource bundle instead of making another large local copy:

```r
HMF_REF <- "/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"
REF_FASTA <- "/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"
AMBER_LOCI <- file.path(HMF_REF,"dna","copy_number","AmberGermlineSites.38.tsv.gz")
GC_PROFILE <- file.path(HMF_REF,"dna","copy_number","GC_profile.1000bp.38.cnp")
DIPLOID_REGIONS <- file.path(HMF_REF,"dna","copy_number","DiploidRegions.38.bed.gz")
DRIVER_PANEL <- file.path(HMF_REF,"common","DriverGenePanel.38.tsv")
ENSEMBL_DIR <- file.path(HMF_REF,"common","ensembl_data")
SOMATIC_HOTSPOTS <- file.path(HMF_REF,"dna","variants","KnownHotspots.somatic.38.vcf.gz")
```

Reference bundle source used for this project:

```text
https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/hmf_pipeline_resources.38_v3.0.0--8.tar.gz
```

Optional fresh installation:

```bash
REF_BASE="/quobyte/luisccgrp/REFERENCE_DATA/hmftools"
cd "$REF_BASE"
wget -c "https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/hmf_pipeline_resources.38_v3.0.0--8.tar.gz"
tar -xzf hmf_pipeline_resources.38_v3.0.0--8.tar.gz
```

---

## 4. Tools

```r
BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"
PURPLE_BASE <- file.path(BASE,"10_Purple")
TOOL_DIR <- file.path(PURPLE_BASE,"Purple_tools")
JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"
AMBER_JAR <- file.path(TOOL_DIR,"amber_v4.3.jar")
COBALT_JAR <- file.path(TOOL_DIR,"cobalt_v3.0.jar")
PURPLE_JAR <- file.path(TOOL_DIR,"purple_v4.4.jar")
```

Expected versions:

```text
AMBER  4.3
COBALT 3.0
PURPLE 4.4
```

Basic verification:

```r
system2(JAVA,"-version")
system2(JAVA,c("-jar",AMBER_JAR,"-version"))
system2(JAVA,c("-jar",COBALT_JAR,"-version"))
system2(JAVA,c("-jar",PURPLE_JAR,"-version"))
```

SHA256 helper:

```r
sha256_file <- function(x){if(!file.exists(x)) return(NA_character_); y <- system2("sha256sum",x,stdout=TRUE,stderr=TRUE); strsplit(y[1],"\\s+")[[1]][1]}
data.frame(tool=c("AMBER","COBALT","PURPLE"),path=c(AMBER_JAR,COBALT_JAR,PURPLE_JAR),sha256=sapply(c(AMBER_JAR,COBALT_JAR,PURPLE_JAR),sha256_file))
```

---

## 5. Samples

```r
samples <- c("I_26166_S_29146","I_26227_S_29149","I_26395_S_29148","I_26784_S_29156","I_26785_S_29154","I_26786_S_29155","I_27483_S_29161","I_27619_S_29166","I_27621_S_29160","I_27622_S_29150","I_27623_S_29147","I_27660_S_29145","I_27661_S_29153","I_27662_S_29152","I_27663_S_29151","I_27664_S_29157","I_27665_S_29158","I_27666_S_29159","I_27670_S_29162","I_27671_S_29163","I_27673_S_29165","I_27674_S_29167","I_27675_S_29168")
```

---

# PART I — Original tumor-only AMBER + COBALT + PURPLE workflow

## 6. Input BAMs

REDUX BAMs were used for tumor-only AMBER/COBALT:

```r
REDUX_DIR <- file.path(BASE,"06_REDux","Redux_BAMs")
```

Expected BAM:

```text
06_REDux/Redux_BAMs/SAMPLE/SAMPLE.redux.bam
```

The original BAMs were **not** used for the final HMFtools tumor-only pipeline once REDUX BAMs were available.

---

## 7. AMBER v4.3

AMBER provides tumor BAF information for PURPLE.

```r
run_amber <- function(sample,output_root){
  bam <- file.path(REDUX_DIR,sample,paste0(sample,".redux.bam")); out <- file.path(output_root,sample,"amber"); dir.create(out,recursive=TRUE,showWarnings=FALSE)
  args <- c("-Xmx48G","-jar",AMBER_JAR,"-tumor",sample,"-tumor_bam",bam,"-loci",AMBER_LOCI,"-ref_genome_version","38","-threads","8","-output_dir",out)
  system2(JAVA,args)
}
```

Key outputs:

```text
SAMPLE.amber.qc
SAMPLE.amber.baf.tsv.gz
SAMPLE.amber.baf.pcf
```

---

## 8. COBALT v3.0

COBALT provides read-depth / copy-number ratio information.

```r
run_cobalt <- function(sample,output_root){
  bam <- file.path(REDUX_DIR,sample,paste0(sample,".redux.bam")); out <- file.path(output_root,sample,"cobalt"); dir.create(out,recursive=TRUE,showWarnings=FALSE)
  args <- c("-Xmx48G","-jar",COBALT_JAR,"-tumor",sample,"-tumor_bam",bam,"-gc_profile",GC_PROFILE,"-tumor_only_diploid_bed",DIPLOID_REGIONS,"-ref_genome_version","38","-threads","8","-output_dir",out)
  system2(JAVA,args)
}
```

Key outputs:

```text
SAMPLE.cobalt.ratio.tsv.gz
SAMPLE.cobalt.ratio.pcf
```

---

## 9. Final somatic inputs

PAVE-annotated SAGE SNV/indel VCF:

```r
PAVE_DIR <- file.path(BASE,"09_PAVE","PAVE_output")
pave_vcf <- file.path(PAVE_DIR,sample,paste0(sample,".sage.somatic.pave.vcf.gz"))
```

ESVEE somatic SV VCF:

```r
ESVEE_DIR <- file.path(BASE,"08_ESVEE","ESVEE_output")
esvee_vcf <- file.path(ESVEE_DIR,sample,paste0(sample,".esvee.somatic.vcf.gz"))
```

---

## 10. PURPLE v4.4 tumor-only command

PURPLE integrates AMBER BAF, COBALT depth ratios, PAVE-annotated SAGE SNV/indels, ESVEE SVs, GRCh38 references, driver panel, and hotspots.

```r
purple_args <- c("-Xmx48G","-jar",PURPLE_JAR,"-tumor",sample,"-amber_dir",amber_dir,"-cobalt_dir",cobalt_dir,"-somatic_vcf",pave_vcf,"-somatic_sv_vcf",esvee_vcf,"-gc_profile",GC_PROFILE,"-ref_genome",REF_FASTA,"-ref_genome_version","38","-ensembl_data_dir",ENSEMBL_DIR,"-driver_gene_panel",DRIVER_PANEL,"-somatic_hotspots",SOMATIC_HOTSPOTS,"-threads","8","-no_charts","-output_dir",purple_out)
system2(JAVA,purple_args)
```

Important decisions:

```text
DO use:
-somatic_vcf       PAVE-annotated SAGE VCF
-somatic_sv_vcf    ESVEE somatic SV VCF
-driver_gene_panel DriverGenePanel.38.tsv
-somatic_hotspots  KnownHotspots.somatic.38.vcf.gz

DO NOT use:
-run_drivers       not required for PURPLE v4.4
-redux_tumor       not used in final run
REDUX MSI prediction not supplied
```

---

# PART II — Final PURPLE rerun

## 11. Why PURPLE was rerun

The final PURPLE rerun was performed after PAVE and ESVEE were finalized. AMBER and COBALT did not need to be rerun, so the successful historical AMBER/COBALT outputs were reused.

Final formal output:

```text
10_Purple/Purple_rerun/
```

Historical AMBER/COBALT source:

```text
10_Purple/Amber_Cobalt_OldPurple/
```

---

## 12. Final reproducible PURPLE rerun script

```r
main <- function(){
BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"; PURPLE_BASE <- file.path(BASE,"10_Purple")
AMBER_COBALT_ROOT <- file.path(PURPLE_BASE,"Amber_Cobalt_OldPurple"); OUT_ROOT <- file.path(PURPLE_BASE,"Purple_rerun")
PAVE_DIR <- file.path(BASE,"09_PAVE","PAVE_output"); ESVEE_DIR <- file.path(BASE,"08_ESVEE","ESVEE_output"); TOOL_DIR <- file.path(PURPLE_BASE,"Purple_tools")
JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"; PURPLE_JAR <- file.path(TOOL_DIR,"purple_v4.4.jar")
REF_FASTA <- "/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"; HMF_REF <- "/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"
GC_PROFILE <- file.path(HMF_REF,"dna","copy_number","GC_profile.1000bp.38.cnp"); DRIVER_PANEL <- file.path(HMF_REF,"common","DriverGenePanel.38.tsv"); ENSEMBL_DIR <- file.path(HMF_REF,"common","ensembl_data"); SOMATIC_HOTSPOTS <- file.path(HMF_REF,"dna","variants","KnownHotspots.somatic.38.vcf.gz")
THREADS <- 8L; JAVA_MEMORY <- "48G"; dir.create(OUT_ROOT,recursive=TRUE,showWarnings=FALSE)
samples <- c("I_26166_S_29146","I_26227_S_29149","I_26395_S_29148","I_26784_S_29156","I_26785_S_29154","I_26786_S_29155","I_27483_S_29161","I_27619_S_29166","I_27621_S_29160","I_27622_S_29150","I_27623_S_29147","I_27660_S_29145","I_27661_S_29153","I_27662_S_29152","I_27663_S_29151","I_27664_S_29157","I_27665_S_29158","I_27666_S_29159","I_27670_S_29162","I_27671_S_29163","I_27673_S_29165","I_27674_S_29167","I_27675_S_29168")
valid_file <- function(x) length(x)==1&&!is.na(x)&&file.exists(x)&&!is.na(file.info(x)$size)&&file.info(x)$size>0
sha256_file <- function(x){if(!valid_file(x)) return(NA_character_); z <- system2("sha256sum",x,stdout=TRUE,stderr=TRUE); strsplit(z[1],"\\s+")[[1]][1]}
cmd_string <- function(exe,args) paste(shQuote(exe),paste(shQuote(args),collapse=" "))
run_recorded <- function(exe,args,command_file,log_file,metadata=character()){dir.create(dirname(command_file),recursive=TRUE,showWarnings=FALSE); dir.create(dirname(log_file),recursive=TRUE,showWarnings=FALSE); cmd <- cmd_string(exe,args); writeLines(c(paste0("DATE=",format(Sys.time(),"%Y-%m-%d %H:%M:%S %Z")),metadata,"","ACTUAL_COMMAND:",cmd),command_file); cat("\nACTUAL COMMAND:\n",cmd,"\n",sep=""); start <- Sys.time(); status <- system2(exe,args=args,stdout=log_file,stderr=log_file); runtime <- round(as.numeric(difftime(Sys.time(),start,units="mins")),2); cat("Exit status:",status,"| Runtime:",runtime,"min\n"); list(status=status,runtime=runtime,log_file=log_file)}
required_output <- function(sample,purple)c(file.path(purple,paste0(sample,".purple.driver.catalog.somatic.tsv")),file.path(purple,paste0(sample,".purple.qc")),file.path(purple,paste0(sample,".purple.purity.tsv")),file.path(purple,paste0(sample,".purple.purity.range.tsv")),file.path(purple,paste0(sample,".purple.segment.tsv")),file.path(purple,paste0(sample,".purple.cnv.somatic.tsv")),file.path(purple,paste0(sample,".purple.cnv.gene.tsv")),file.path(purple,paste0(sample,".purple.somatic.vcf.gz")),file.path(purple,paste0(sample,".purple.somatic.vcf.gz.tbi")),file.path(purple,paste0(sample,".purple.sv.vcf.gz")),file.path(purple,paste0(sample,".purple.sv.vcf.gz.tbi")),file.path(purple,"purple.version"))
console_log <- file.path(OUT_ROOT,paste0("PURPLE_rerun_console_",format(Sys.time(),"%Y%m%d_%H%M%S"),".log")); sink(console_log,split=TRUE); run_status <- "INTERRUPTED_OR_FAILED"
on.exit({cat("\nPURPLE RUN END | STATUS:",run_status,"|",format(Sys.time(),"%Y-%m-%d %H:%M:%S %Z"),"\n"); while(sink.number(type="output")>0) sink(type="output")},add=TRUE)
cat("PURPLE v4.4 FINAL RERUN\nOutput:",OUT_ROOT,"\nMSI prediction: NOT USED\n")
preflight_files <- c(JAVA,PURPLE_JAR,REF_FASTA,paste0(REF_FASTA,".fai"),GC_PROFILE,DRIVER_PANEL,SOMATIC_HOTSPOTS); if(!all(file.exists(preflight_files))||!dir.exists(ENSEMBL_DIR)) stop("Missing tool/reference")
cat("PURPLE SHA256:",sha256_file(PURPLE_JAR),"\n"); system2(JAVA,"-version")
results <- data.frame(Sample=samples,Status="NOT_STARTED",Runtime_min=NA_real_,stringsAsFactors=FALSE)
for(i in seq_along(samples)){
sample <- samples[i]; cat("\n[",i,"/23] ",sample,"\n",sep="")
amber <- file.path(AMBER_COBALT_ROOT,sample,"amber"); cobalt <- file.path(AMBER_COBALT_ROOT,sample,"cobalt"); pave <- file.path(PAVE_DIR,sample,paste0(sample,".sage.somatic.pave.vcf.gz")); esvee <- file.path(ESVEE_DIR,sample,paste0(sample,".esvee.somatic.vcf.gz"))
purple <- file.path(OUT_ROOT,sample,"purple"); logs <- file.path(OUT_ROOT,sample,"logs"); cmds <- file.path(OUT_ROOT,sample,"commands"); dir.create(purple,recursive=TRUE,showWarnings=FALSE); dir.create(logs,recursive=TRUE,showWarnings=FALSE); dir.create(cmds,recursive=TRUE,showWarnings=FALSE)
amber_req <- c(file.path(amber,paste0(sample,".amber.qc")),file.path(amber,paste0(sample,".amber.baf.tsv.gz")),file.path(amber,paste0(sample,".amber.baf.pcf"))); cobalt_req <- c(file.path(cobalt,paste0(sample,".cobalt.ratio.tsv.gz")),file.path(cobalt,paste0(sample,".cobalt.ratio.pcf")))
if(!all(vapply(c(amber_req,cobalt_req,pave,esvee),valid_file,logical(1)))) stop("Missing input for ",sample)
req <- required_output(sample,purple); if(all(vapply(req,valid_file,logical(1)))){cat("[SKIP] validated PURPLE outputs already complete\n"); results$Status[i] <- "COMPLETE_ALREADY"; results$Runtime_min[i] <- 0; next}
args <- c(paste0("-Xmx",JAVA_MEMORY),"-jar",PURPLE_JAR,"-tumor",sample,"-amber_dir",amber,"-cobalt_dir",cobalt,"-somatic_vcf",pave,"-somatic_sv_vcf",esvee,"-gc_profile",GC_PROFILE,"-ref_genome",REF_FASTA,"-ref_genome_version","38","-ensembl_data_dir",ENSEMBL_DIR,"-driver_gene_panel",DRIVER_PANEL,"-somatic_hotspots",SOMATIC_HOTSPOTS,"-threads",as.character(THREADS),"-no_charts","-output_dir",purple)
rr <- run_recorded(JAVA,args,file.path(cmds,paste0(sample,".purple.command.txt")),file.path(logs,"purple.log"),c(paste0("SAMPLE=",sample),"TOOL=PURPLE v4.4","MODE=TUMOR_ONLY","REDUX_MSI_PREDICTION=NOT_USED",paste0("PURPLE_JAR_SHA256=",sha256_file(PURPLE_JAR)),paste0("PAVE=",pave),paste0("ESVEE=",esvee)))
ok <- rr$status==0&&all(vapply(req,valid_file,logical(1))); results$Status[i] <- ifelse(ok,"COMPLETE","FAILED"); results$Runtime_min[i] <- rr$runtime; write.csv(results,file.path(OUT_ROOT,"PURPLE_rerun_23_samples_status.csv"),row.names=FALSE)
if(!ok){cat("\nLAST 100 LOG LINES:\n"); if(file.exists(rr$log_file)) cat(tail(readLines(rr$log_file,warn=FALSE),100),sep="\n"); stop("PURPLE failed: ",sample)}
}
writeLines(capture.output(sessionInfo()),file.path(OUT_ROOT,paste0("PURPLE_sessionInfo_",format(Sys.time(),"%Y%m%d_%H%M%S"),".txt"))); run_status <- "COMPLETED"; print(results,row.names=FALSE)
}
main()
```

Final core behavior:

```text
reuses AMBER/COBALT
uses final/current PAVE
uses final/current ESVEE
no -redux_tumor
no Redux MSI prediction
-no_charts for core run
validates outputs before COMPLETE
skips only validated completed samples
```

---

## 13. Final PURPLE core outputs

```text
SAMPLE.purple.qc
SAMPLE.purple.purity.tsv
SAMPLE.purple.purity.range.tsv
SAMPLE.purple.segment.tsv
SAMPLE.purple.cnv.somatic.tsv
SAMPLE.purple.cnv.gene.tsv
SAMPLE.purple.driver.catalog.somatic.tsv
SAMPLE.purple.somatic.vcf.gz
SAMPLE.purple.somatic.vcf.gz.tbi
SAMPLE.purple.sv.vcf.gz
SAMPLE.purple.sv.vcf.gz.tbi
purple.version
```

Use these from `Purple_rerun/` for downstream analysis.

---

# PART III — Standard PURPLE plots

## 14. Why plotting was separate

The final core rerun used `-no_charts`. Four standard plots were later created directly from the completed core using the exact `copyNumberPlots.R` packaged in PURPLE v4.4.

---

## 15. Extract exact `copyNumberPlots.R`

```r
PURPLE_BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple"; PURPLE_JAR <- file.path(PURPLE_BASE,"Purple_tools","purple_v4.4.jar")
PLOT_TOOL_DIR <- file.path(PURPLE_BASE,"Purple_plot_tools"); dir.create(PLOT_TOOL_DIR,recursive=TRUE,showWarnings=FALSE)
jar_contents <- utils::unzip(PURPLE_JAR,list=TRUE); plot_entry <- jar_contents$Name[grepl("copyNumberPlots\\.R$",jar_contents$Name)]
if(length(plot_entry)!=1) stop("Could not uniquely identify copyNumberPlots.R")
tmp <- tempfile("purple_plot_"); dir.create(tmp); utils::unzip(PURPLE_JAR,files=plot_entry,exdir=tmp,overwrite=TRUE)
COPYNUMBER_PLOTS <- file.path(PLOT_TOOL_DIR,"copyNumberPlots.PURPLE_v4.4.R"); file.copy(file.path(tmp,plot_entry),COPYNUMBER_PLOTS,overwrite=TRUE); unlink(tmp,recursive=TRUE)
```

---

## 16. R dependencies

```r
R_SCRIPT <- "/cvmfs/hpc.ucdavis.edu/sw/conda/environments/r-4.4.2/bin/Rscript"
r_dependency_code <- "suppressPackageStartupMessages({library(VariantAnnotation);library(dplyr);library(ggplot2);library(cowplot)});cat('R_PLOT_DEPENDENCIES_OK\\n')"
system2(R_SCRIPT,c("-e",shQuote(r_dependency_code)))
```

---

## 17. Generate four standard plots

```r
PURPLE_ROOT <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun"
COPYNUMBER_PLOTS <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_plot_tools/copyNumberPlots.PURPLE_v4.4.R"
R_SCRIPT <- "/cvmfs/hpc.ucdavis.edu/sw/conda/environments/r-4.4.2/bin/Rscript"
for(sample in samples){
  purple_dir <- file.path(PURPLE_ROOT,sample,"purple"); plot_dir <- file.path(purple_dir,"plot"); dir.create(plot_dir,recursive=TRUE,showWarnings=FALSE)
  expected <- file.path(plot_dir,paste0(sample,c(".copynumber.png",".map.png",".purity.range.png",".segment.png")))
  if(all(file.exists(expected)&file.info(expected)$size>0)){cat("[SKIP]",sample,"\n"); next}
  log <- file.path(PURPLE_ROOT,sample,"logs","plots",paste0(sample,".purple.Rplots.log")); dir.create(dirname(log),recursive=TRUE,showWarnings=FALSE)
  status <- system2(R_SCRIPT,c(COPYNUMBER_PLOTS,sample,purple_dir,plot_dir),stdout=log,stderr=log)
  if(status!=0||!all(file.exists(expected)&file.info(expected)$size>0)) stop("R plotting failed: ",sample)
}
```

Expected standard plots:

```text
SAMPLE.copynumber.png
SAMPLE.map.png
SAMPLE.purity.range.png
SAMPLE.segment.png
```

---

# PART IV — Circos environment

## 18. Separate plotting environment

The original `purple` environment had an incomplete Circos/Perl setup, so a separate plotting environment was used instead of modifying the production PURPLE Java environment.

Successful environment:

```text
/home/zzr123/.conda/envs/purple_plot
```

Observed successful versions:

```text
Perl   5.32.1
Circos 0.69-8
```

Verify:

```bash
/home/zzr123/.conda/envs/purple_plot/bin/perl /home/zzr123/.conda/envs/purple_plot/bin/circos -version
/home/zzr123/.conda/envs/purple_plot/bin/perl /home/zzr123/.conda/envs/purple_plot/bin/circos -modules
```

Every required module must report `ok`.

---

## 19. Circos wrapper used by PURPLE

```bash
#!/usr/bin/env bash
set -euo pipefail
exec '/home/zzr123/.conda/envs/purple_plot/bin/perl' '/home/zzr123/.conda/envs/purple_plot/bin/circos' "$@"
```

Saved as:

```text
10_Purple/Purple_plot_tools/circos_purple_wrapper.sh
```

Create from R:

```r
PLOT_TOOL_DIR <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_plot_tools"
PERL <- "/home/zzr123/.conda/envs/purple_plot/bin/perl"; CIRCOS_SCRIPT <- "/home/zzr123/.conda/envs/purple_plot/bin/circos"
CIRCOS_WRAPPER <- file.path(PLOT_TOOL_DIR,"circos_purple_wrapper.sh")
writeLines(c("#!/usr/bin/env bash","set -euo pipefail",paste("exec",shQuote(PERL),shQuote(CIRCOS_SCRIPT),"\"$@\"")),CIRCOS_WRAPPER); Sys.chmod(CIRCOS_WRAPPER,"0755")
```

---

# PART V — Circos regeneration

## 20. Why an isolated PURPLE rerun was required

The final formal core run used `-no_charts`; in the actual PURPLE v4.4 execution, the Circos configs were not produced. To avoid changing the formal results, PURPLE was rerun in:

```text
10_Purple/Purple_plot_regen_with_charts/
```

The regeneration run reused exactly the same AMBER, COBALT, PAVE, ESVEE and references, did **not** use Redux MSI prediction, did **not** use `-no_charts`, and supplied `-circos <wrapper>`.

---

## 21. Circos regeneration command

```r
regen_args <- c("-Xmx48G","-jar",PURPLE_JAR,"-tumor",sample,"-amber_dir",amber_dir,"-cobalt_dir",cobalt_dir,"-somatic_vcf",pave_vcf,"-somatic_sv_vcf",esvee_vcf,"-gc_profile",GC_PROFILE,"-ref_genome",REF_FASTA,"-ref_genome_version","38","-ensembl_data_dir",ENSEMBL_DIR,"-driver_gene_panel",DRIVER_PANEL,"-somatic_hotspots",SOMATIC_HOTSPOTS,"-threads","8","-circos",CIRCOS_WRAPPER,"-output_dir",regen_purple)
system2(JAVA,regen_args)
```

Expected Circos artifacts:

```text
purple/circos/SAMPLE.input.conf
purple/circos/SAMPLE.circos.conf
purple/plot/SAMPLE.input.png
purple/plot/SAMPLE.circos.png
```

---

## 22. Core consistency validation

Because PURPLE was rerun only to regenerate chart artifacts, the regenerated core was compared against the formal `Purple_rerun/` results using SHA256.

Files checked:

```text
SAMPLE.purple.purity.tsv
SAMPLE.purple.segment.tsv
SAMPLE.purple.cnv.somatic.tsv
```

```r
compare_names <- c(paste0(sample,".purple.purity.tsv"),paste0(sample,".purple.segment.tsv"),paste0(sample,".purple.cnv.somatic.tsv"))
comparison <- do.call(rbind,lapply(compare_names,function(f){
  main_file <- file.path(main_purple,f); regen_file <- file.path(regen_purple,f)
  main_hash <- sha256_file(main_file); regen_hash <- sha256_file(regen_file)
  data.frame(File=f,Main_SHA256=main_hash,Regen_SHA256=regen_hash,Match=!is.na(main_hash)&&!is.na(regen_hash)&&identical(main_hash,regen_hash))
}))
print(comparison,row.names=FALSE); if(!all(comparison$Match)) stop("Regenerated PURPLE core differs from formal PURPLE core")
```

Final project result:

```text
Core SHA256 matches: 23/23
```

Only after the hashes matched were the Circos PNGs copied into the formal plot directory.

---

## 23. Copy only Circos PNGs to formal results

```r
final_plot <- file.path(PURPLE_ROOT,sample,"purple","plot"); dir.create(final_plot,recursive=TRUE,showWarnings=FALSE)
file.copy(file.path(regen_purple,"plot",paste0(sample,".input.png")),file.path(final_plot,paste0(sample,".input.png")),overwrite=TRUE)
file.copy(file.path(regen_purple,"plot",paste0(sample,".circos.png")),file.path(final_plot,paste0(sample,".circos.png")),overwrite=TRUE)
```

The regenerated PURPLE core files are **not** copied into `Purple_rerun/`.

---

# PART VI — Final validation

## 24. Six required plots per sample

```text
SAMPLE.copynumber.png
SAMPLE.map.png
SAMPLE.purity.range.png
SAMPLE.segment.png
SAMPLE.input.png
SAMPLE.circos.png
```

Validation:

```r
PURPLE_ROOT <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun"
validation <- do.call(rbind,lapply(samples,function(sample){
  plot_dir <- file.path(PURPLE_ROOT,sample,"purple","plot"); files <- file.path(plot_dir,paste0(sample,c(".copynumber.png",".map.png",".purity.range.png",".segment.png",".input.png",".circos.png")))
  data.frame(Sample=sample,File=basename(files),OK=file.exists(files)&file.info(files)$size>0,Size_bytes=ifelse(file.exists(files),file.info(files)$size,NA_real_))
}))
print(aggregate(OK~Sample,validation,all),row.names=FALSE)
```

Final successful status:

```text
R plots complete:        23/23
Circos copied:           23/23
Core SHA256 matches:     23/23
Final six-plot complete: 23/23
```

---

# PART VII — Provenance requirements

## 25. Required records for future runs

Keep:

```text
1. Tool + exact version
2. JAR path + SHA256
3. Shared reference paths
4. Explicit upstream input paths
5. Preflight checks
6. Exact resolved command printed before execution
7. Per-sample .command.txt
8. Per-sample stdout/stderr log
9. Exit status
10. Runtime
11. Required-output validation
12. Final run/status table
13. Full console log
14. R sessionInfo()
15. Executed notebook/Rmd with cell output
```

Historical logs must not be edited after directory renaming.

---

## 26. Reusable command-record helper

```r
cmd_string <- function(exe,args) paste(shQuote(exe),paste(shQuote(args),collapse=" "))
run_recorded <- function(exe,args,command_file,log_file,metadata=character()){
  dir.create(dirname(command_file),recursive=TRUE,showWarnings=FALSE); dir.create(dirname(log_file),recursive=TRUE,showWarnings=FALSE)
  cmd <- cmd_string(exe,args); writeLines(c(paste0("DATE=",format(Sys.time(),"%Y-%m-%d %H:%M:%S %Z")),metadata,"","ACTUAL_COMMAND:",cmd),command_file)
  cat("\nACTUAL COMMAND:\n",cmd,"\n",sep=""); start <- Sys.time(); status <- system2(exe,args=args,stdout=log_file,stderr=log_file)
  runtime <- round(as.numeric(difftime(Sys.time(),start,units="mins")),2); cat("Exit status:",status,"| Runtime:",runtime,"min\n")
  list(status=status,runtime=runtime,log_file=log_file,command_file=command_file)
}
```

Full console capture:

```r
console_log <- file.path(OUT_ROOT,paste0("PURPLE_console_",format(Sys.time(),"%Y%m%d_%H%M%S"),".log")); sink(console_log,split=TRUE); run_status <- "INTERRUPTED_OR_FAILED"
on.exit({cat("\nSTATUS:",run_status,"\n"); while(sink.number(type="output")>0) sink(type="output")},add=TRUE)
```

At successful completion:

```r
run_status <- "COMPLETED"
```

---

# PART VIII — Quick summaries

## 27. Purity/ploidy summary

```r
PURPLE_ROOT <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun"
extract_purity <- function(sample){f <- file.path(PURPLE_ROOT,sample,"purple",paste0(sample,".purple.purity.tsv")); x <- read.delim(f,check.names=FALSE); x$Sample <- sample; x}
purity_table <- do.call(rbind,lapply(samples,extract_purity)); write.csv(purity_table,file.path(PURPLE_ROOT,"PURPLE_23_samples_purity_ploidy.csv"),row.names=FALSE)
```

## 28. QC files

PURPLE QC files are:

```text
Purple_rerun/SAMPLE/purple/SAMPLE.purple.qc
```

Inspect one before combining because the exact `.purple.qc` field structure should be preserved:

```r
qc_file <- file.path(PURPLE_ROOT,samples[1],"purple",paste0(samples[1],".purple.qc")); readLines(qc_file,n=30)
```

---

# PART IX — Downstream usage

## 29. Always use `Purple_rerun` as the formal PURPLE root

```r
PURPLE_ROOT <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun"
```

LINX:

```r
purple_dir <- file.path(PURPLE_ROOT,sample,"purple")
```

CHORD:

```r
somatic_vcf <- file.path(PURPLE_ROOT,sample,"purple",paste0(sample,".purple.somatic.vcf.gz"))
sv_vcf <- file.path(PURPLE_ROOT,sample,"purple",paste0(sample,".purple.sv.vcf.gz"))
```

ORANGE should also use the formal `Purple_rerun` output rather than `Purple_plot_regen_with_charts`.

---

# PART X — Directory meaning

## 30. `Amber_Cobalt_OldPurple`

```text
Contains historical AMBER, COBALT and old PURPLE output.
AMBER + COBALT are retained and reused by the final PURPLE rerun.
The old PURPLE result itself is not the final formal result.
```

## 31. `Purple_rerun`

```text
FINAL formal PURPLE output.
Use for purity, ploidy, CNV, QC, drivers, somatic VCF, SV VCF,
LINX, CHORD, ORANGE, downstream analysis and final plots.
```

## 32. `Purple_plot_regen_with_charts`

```text
Isolated rerun used only to produce Circos artifacts.
Do not treat as the formal output directory.
Core consistency was validated against Purple_rerun by SHA256.
```

## 33. `Purple_plot_tools`

```text
Contains copyNumberPlots.PURPLE_v4.4.R, circos_purple_wrapper.sh,
and plotting/Circos provenance records.
```

---

# PART XI — Final project status

```text
AMBER                         COMPLETE
COBALT                        COMPLETE
Final PAVE input              COMPLETE
Final ESVEE input             COMPLETE
PURPLE v4.4 core              COMPLETE 23/23
Standard PURPLE R plots       COMPLETE 23/23
Circos input plots            COMPLETE 23/23
Circos final plots            COMPLETE 23/23
Core SHA256 consistency       PASS 23/23
Formal core overwritten       NO
Redux MSI prediction used     NO
```

The formal result directory for all future work is:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun
```
