# PAVE v1.9 Tumor-Only Complete Workflow — UCaTS Organoids

> **Project:** UCaTS Organoid WGS  
> **Pipeline stage:** `SAGE -> PAVE -> PURPLE`  
> **Mode:** tumor-only WGS  
> **PAVE version:** `v1.9`  
> **Samples:** 23  
> **HMF resource bundle:** `hmf_pipeline_resources.38_v3.0.0--8`  
> **Java heap:** `48G`  
> **Threads:** `4`

## 1. Purpose

PAVE annotates and filters the SAGE somatic small-variant VCF before the result is passed to PURPLE.

Current project chain:

```text
REDUX
  |
  v
SAGE
  |
  v
<SAMPLE>.sage.somatic.vcf.gz
  |
  v
PAVE v1.9
  |
  +--> PON annotation/filtering
  +--> gnomAD annotation
  +--> ClinVar annotation
  +--> mappability annotation
  +--> germline blacklist annotation
  |
  v
<SAMPLE>.sage.somatic.pave.vcf.gz
  |
  v
PURPLE
```

The installed PAVE v1.9 JAR used by this workflow was explicitly checked to support the option names:

```text
-input_vcf
-output_vcf
-gnomad_freq_dir
```

and the script verifies those options from the actual installed JAR before any sample is run.

---

## 2. Current project paths

Project root:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids
```

SAGE input:

```text
07_SAGE/SAGE_output/
```

PAVE project:

```text
09_PAVE/
├── PAVE_tools/
├── PAVE_output/
└── PAVE_output_pre_tumoronly/
```

Current PAVE JAR:

```text
09_PAVE/PAVE_tools/pave_v1.9.jar
```

Java:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Reference FASTA:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

---

## 3. Java

The production code uses:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Check:

```bash
/home/zzr123/.conda/envs/purple/bin/java -version
```

If a separate Java environment is required:

```bash
conda create   -n pave_java   -c conda-forge   openjdk=21   -y

conda activate pave_java

java -version
```

For reproducibility, the actual project code continues to use the absolute Java path.

---

## 4. PAVE v1.9 JAR

The supplied workflow expects:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/
09_PAVE/PAVE_tools/pave_v1.9.jar
```

Create the directory if needed:

```bash
mkdir -p /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/09_PAVE/PAVE_tools
```

Before production use, record:

```bash
JAR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/09_PAVE/PAVE_tools/pave_v1.9.jar"
JAVA="/home/zzr123/.conda/envs/purple/bin/java"

ls -lh "$JAR"
sha256sum "$JAR"

"$JAVA" -jar "$JAR" -help
```

The actual script performs an additional option check and stops if the installed JAR does not expose the expected v1.9 arguments.

---

## 5. Reference FASTA

Current FASTA:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

Required files:

```text
hg38.fa
hg38.fa.fai
```

Check:

```bash
REF="/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"

ls -lh "$REF"
ls -lh "$REF.fai"

sha256sum "$REF"
```

If `.fai` is missing:

```bash
samtools faidx "$REF"
```

The same GRCh38 reference context should be maintained across SAGE, PAVE, and PURPLE.

---

## 6. Shared HMF reference bundle

The code uses the shared laboratory HMF resource bundle:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8
```

The base variant-resource directory is:

```text
dna/variants/
```

The code uses:

```text
common/DriverGenePanel.38.tsv
common/ensembl_data/

dna/variants/hmf_wgs_sage_pon_1000.38.tsv.gz
dna/variants/gnomad/
dna/variants/clinvar.38.vcf.gz
dna/variants/mappability_150.38.bed.gz
dna/variants/KnownBlacklist.germline.38.bed
```

---

## 7. Download HMF resources 3.0.0--8

If the shared resource bundle is not already available:

```bash
set -euo pipefail

REF_BASE="/quobyte/luisccgrp/REFERENCE_DATA/hmftools"
BUNDLE="hmf_pipeline_resources.38_v3.0.0--8.tar.gz"

mkdir -p "$REF_BASE"
cd "$REF_BASE"

wget -c "https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/$BUNDLE"

tar -xzf "$BUNDLE"
```

Expected root:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
```

Check the PAVE resources:

```bash
HMF_REF="/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"

ls -lh "$HMF_REF/common/DriverGenePanel.38.tsv"
ls -lh "$HMF_REF/common/ensembl_data/ensembl_gene_data.csv"
ls -lh "$HMF_REF/dna/variants/hmf_wgs_sage_pon_1000.38.tsv.gz"
ls -lh "$HMF_REF/dna/variants/clinvar.38.vcf.gz"
ls -lh "$HMF_REF/dna/variants/mappability_150.38.bed.gz"
ls -lh "$HMF_REF/dna/variants/KnownBlacklist.germline.38.bed"

ls "$HMF_REF/dna/variants/gnomad/" | head
```

---

## 8. gnomAD requirement

The code expects the gnomAD directory:

```text
dna/variants/gnomad/
```

to contain exactly:

```text
24
```

chromosome files matching:

```text
gnomad_variants_chr.+_v38.csv.gz
```

The script checks:

```r
gnomad_files <- list.files(
  gnomad_dir,
  pattern="^gnomad_variants_chr.+_v38\.csv\.gz$",
  full.names=TRUE
)

if(length(gnomad_files)!=24)
  stop(...)
```

Therefore PAVE does not start if the gnomAD reference set is incomplete.

---

## 9. PON resource and tumor-only filters

The PON file used is:

```text
hmf_wgs_sage_pon_1000.38.tsv.gz
```

The filtering rule is:

```text
HOTSPOT:6:5;PANEL:3:3;UNKNOWN:3:0
```

In R:

```r
PON_FILTERS <- "HOTSPOT:6:5;PANEL:3:3;UNKNOWN:3:0"
```

The code intentionally passes:

```r
shQuote(PON_FILTERS)
```

because semicolons have special meaning to the shell.

This prevents:

```text
;
```

inside the PON filter definition from being interpreted as shell command separators.

---

## 10. Tumor-only annotations

The current tumor-only rerun adds or uses:

```text
PON
PON filtering
gnomAD
ClinVar
Mappability
Germline blacklist
```

The command also supplies:

```text
DriverGenePanel.38.tsv
Ensembl data
ILLUMINA sequencing type
```

---

## 11. Old PAVE output backup

This workflow deliberately preserves the earlier PAVE result.

On the first run:

```text
PAVE_output
```

is renamed to:

```text
PAVE_output_pre_tumoronly
```

Then a fresh:

```text
PAVE_output
```

is created for the corrected tumor-only rerun.

The logic is:

```text
if old PAVE_output exists
AND backup does not yet exist
    rename old output to backup
```

If the script is restarted after interruption and the backup already exists, the new `PAVE_output` is retained so completed samples can be resumed/skipped.

---

## 12. Fixed 23-sample cohort

The code explicitly defines the 23 UCaTS tumor samples.

This is useful because the PAVE workflow will not accidentally annotate an unrelated SAGE VCF that happens to be present elsewhere in `SAGE_output`.

The expected sample count is:

```text
23
```

---

## 13. Verify PAVE options from the installed JAR

Before any sample starts, the script executes:

```bash
java -jar pave_v1.9.jar -help
```

and verifies that the installed JAR supports:

```text
-input_vcf
-output_vcf
-pon_file
-pon_filters
-gnomad_freq_dir
-clinvar_vcf
-mappability_bed
-blacklist_bed
-sequencing_type
```

This is particularly important because an earlier configuration issue involved using an option name not registered by the installed PAVE JAR.

If any expected option is absent:

```text
PAVE does not start.
```

---

## 14. SAGE input discovery

The workflow searches recursively for:

```text
<SAMPLE>.sage.somatic.vcf.gz
```

under:

```text
07_SAGE/SAGE_output
```

For every expected sample:

```text
exactly one matching SAGE VCF
```

must be found.

The script stops if:

```text
a sample is missing
or
multiple VCFs exist for the same sample
```

---

## 15. SAGE input integrity check

Before PAVE runs, all 23 SAGE VCFs must pass:

```text
BGZF integrity
TBI index
```

Utilities required:

```text
bgzip
tabix
```

Check:

```bash
which bgzip
which tabix

bgzip --help | head
tabix --help | head
```

If these are unavailable:

```bash
conda install   -c bioconda   htslib   -y
```

The code checks BGZF using:

```bash
bgzip -t <VCF>
```

If the `.tbi` is missing, it creates it using:

```bash
tabix -f -p vcf <VCF>
```

PAVE does **not** start until all 23 input VCFs pass both checks.

---

## 16. PAVE command used

For each sample the core command resolves to the equivalent of:

```bash
java -Xmx48G   -jar pave_v1.9.jar   -sample SAMPLE   -input_vcf SAMPLE.sage.somatic.vcf.gz   -ref_genome hg38.fa   -ref_genome_version 38   -ensembl_data_dir ensembl_data   -driver_gene_panel DriverGenePanel.38.tsv   -sequencing_type ILLUMINA   -pon_file hmf_wgs_sage_pon_1000.38.tsv.gz   -pon_filters 'HOTSPOT:6:5;PANEL:3:3;UNKNOWN:3:0'   -gnomad_freq_dir gnomad   -clinvar_vcf clinvar.38.vcf.gz   -mappability_bed mappability_150.38.bed.gz   -blacklist_bed KnownBlacklist.germline.38.bed   -output_dir SAMPLE_OUTPUT   -output_vcf SAMPLE.sage.somatic.pave.vcf.gz   -threads 4
```

Compute:

```text
Java heap = 48G
Threads   = 4
```

---

## 17. Output files

Per sample:

```text
09_PAVE/PAVE_output/<SAMPLE>/
├── <SAMPLE>.sage.somatic.pave.vcf.gz
├── <SAMPLE>.sage.somatic.pave.vcf.gz.tbi
├── <SAMPLE>.pave.log
├── <SAMPLE>.pave.err
└── .PAVE_COMPLETE
```

Project-level:

```text
09_PAVE/PAVE_output/PAVE_run_summary.tsv
```

Historical pre-tumor-only PAVE output:

```text
09_PAVE/PAVE_output_pre_tumoronly/
```

---

## 18. Resume behavior

A sample is skipped only when all three are present:

```text
.PAVE_COMPLETE
output VCF
output VCF .tbi
```

and the VCF/index are non-empty.

This is safer than skipping merely because a partial VCF exists.

---

## 19. Failed sample behavior

If a sample does not already meet the completion rule, its current sample output directory is deleted before rerun:

```r
if(dir.exists(sample_out))
  unlink(sample_out,recursive=TRUE,force=TRUE)
```

This removes partial output from a previous failed PAVE attempt.

A failure does not mark the sample complete.

---

## 20. Output validation

A successful Java exit alone is not sufficient.

The output must pass:

```text
Java exit status = 0
output VCF exists
output VCF non-empty
BGZF integrity passes
TBI creation succeeds
TBI non-empty
```

Only then is:

```text
.PAVE_COMPLETE
```

written.

---

## 21. Completion marker

The `.PAVE_COMPLETE` file records:

```text
Sample
completion time
PAVE version
sequencing type
Java heap
threads
PON
PON filters
gnomAD directory
ClinVar
mappability
blacklist
BGZF PASS
TBI PASS
```

This makes the completed sample easier to audit later.

---

## 22. Downstream use

The corrected current PAVE VCF is:

```text
09_PAVE/PAVE_output/<SAMPLE>/
<SAMPLE>.sage.somatic.pave.vcf.gz
```

The later PURPLE rerun should consume this current PAVE-annotated SAGE VCF.

The workflow is:

```text
SAGE
  |
  v
PAVE_output/<SAMPLE>/<SAMPLE>.sage.somatic.pave.vcf.gz
  |
  v
PURPLE
```

The historical:

```text
PAVE_output_pre_tumoronly
```

should remain archived rather than being used as the current downstream input.

---

## 23. Final complete R code

The code below is preserved from the supplied PAVE v1.9 tumor-only final rerun.

```r
# ============================================================
# PAVE v1.9 - TUMOR-ONLY FINAL RERUN
# 23 tumor WGS samples
#
# SAGE -> PAVE -> PURPLE
#
# Tumor-only annotations:
#   PON
#   PON filtering
#   gnomAD
#   ClinVar
#   Mappability
#   Germline blacklist
#
# IMPORTANT:
# Installed PAVE v1.9 JAR confirmed to use:
#   -input_vcf
#   -output_vcf
#   -gnomad_freq_dir
#
# Compute:
#   Java heap = 48G
#   Threads = 4
# ============================================================

# -------------------- PATHS --------------------
BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"
SAGE_OUT <- file.path(BASE,"07_SAGE","SAGE_output")
PAVE_BASE <- file.path(BASE,"09_PAVE")
PAVE_OUT <- file.path(PAVE_BASE,"PAVE_output")
BACKUP_OUT <- file.path(PAVE_BASE,"PAVE_output_pre_tumoronly")
java_bin <- "/home/zzr123/.conda/envs/purple/bin/java"
pave_jar <- file.path(PAVE_BASE,"PAVE_tools","pave_v1.9.jar")
ref_fasta <- "/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"

# -------------------- HMF REFERENCES --------------------
HMF_REF <- "/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"
VARIANT_REF <- file.path(HMF_REF,"dna","variants")
driver_panel <- file.path(HMF_REF,"common","DriverGenePanel.38.tsv")
ensembl_dir <- file.path(HMF_REF,"common","ensembl_data")
pon_file <- file.path(VARIANT_REF,"hmf_wgs_sage_pon_1000.38.tsv.gz")
gnomad_dir <- file.path(VARIANT_REF,"gnomad")
clinvar_file <- file.path(VARIANT_REF,"clinvar.38.vcf.gz")
mappability_file <- file.path(VARIANT_REF,"mappability_150.38.bed.gz")
blacklist_bed <- file.path(VARIANT_REF,"KnownBlacklist.germline.38.bed")

# IMPORTANT: shQuote() is required because ; is special to the shell
PON_FILTERS <- "HOTSPOT:6:5;PANEL:3:3;UNKNOWN:3:0"

# -------------------- COMPUTE --------------------
MEMORY <- "48G"
THREADS <- 4

# ============================================================
# 1. BACK UP OLD PAVE OUTPUT
#
# First run:
#   PAVE_output -> PAVE_output_pre_tumoronly
#
# If this script is rerun after interruption:
#   backup already exists, so current new PAVE_output is kept
#   and completed samples can be skipped.
# ============================================================

if(dir.exists(PAVE_OUT) && !dir.exists(BACKUP_OUT)){
  cat("Backing up previous PAVE output...\n")
  ok <- file.rename(PAVE_OUT,BACKUP_OUT)
  if(!ok) stop("Could not rename old PAVE_output.")
  cat("Old output saved as:\n",BACKUP_OUT,"\n")
}

dir.create(PAVE_OUT,recursive=TRUE,showWarnings=FALSE)

# ============================================================
# 2. SAMPLE LIST
# ============================================================

samples <- c(
"I_26166_S_29146","I_26227_S_29149","I_26395_S_29148","I_26784_S_29156","I_26785_S_29154",
"I_26786_S_29155","I_27483_S_29161","I_27619_S_29166","I_27621_S_29160","I_27622_S_29150",
"I_27623_S_29147","I_27660_S_29145","I_27661_S_29153","I_27662_S_29152","I_27663_S_29151",
"I_27664_S_29157","I_27665_S_29158","I_27666_S_29159","I_27670_S_29162","I_27671_S_29163",
"I_27673_S_29165","I_27674_S_29167","I_27675_S_29168")

cat("\nSamples:",length(samples),"\n")

# ============================================================
# 3. RESOURCE CHECK
# ============================================================

required_files <- c(
JAVA=java_bin,
PAVE_JAR=pave_jar,
FASTA=ref_fasta,
FASTA_INDEX=paste0(ref_fasta,".fai"),
DRIVER_PANEL=driver_panel,
ENSEMBL_GENE=file.path(ensembl_dir,"ensembl_gene_data.csv"),
PON=pon_file,
CLINVAR=clinvar_file,
MAPPABILITY=mappability_file,
BLACKLIST=blacklist_bed)

required_dirs <- c(
ENSEMBL_DIR=ensembl_dir,
GNOMAD_DIR=gnomad_dir)

cat("\n========== RESOURCE CHECK ==========\n")
for(nm in names(required_files)) cat(sprintf("%-16s %-5s %s\n",nm,file.exists(required_files[nm]),required_files[nm]))
for(nm in names(required_dirs)) cat(sprintf("%-16s %-5s %s\n",nm,dir.exists(required_dirs[nm]),required_dirs[nm]))

if(!all(file.exists(required_files))) stop("Missing file(s):\n",paste(required_files[!file.exists(required_files)],collapse="\n"))
if(!all(dir.exists(required_dirs))) stop("Missing directory/directories:\n",paste(required_dirs[!dir.exists(required_dirs)],collapse="\n"))

gnomad_files <- list.files(gnomad_dir,pattern="^gnomad_variants_chr.+_v38\\.csv\\.gz$",full.names=TRUE)
cat("\ngnomAD chromosome files:",length(gnomad_files),"\n")
if(length(gnomad_files)!=24) stop("Expected 24 gnomAD chromosome files, found ",length(gnomad_files))

# ============================================================
# 4. VERIFY OPTIONS FROM ACTUAL INSTALLED JAR
# ============================================================

pave_help <- suppressWarnings(system2(java_bin,c("-jar",pave_jar,"-help"),stdout=TRUE,stderr=TRUE))
expected_opts <- c("-input_vcf","-output_vcf","-pon_file","-pon_filters","-gnomad_freq_dir",
                   "-clinvar_vcf","-mappability_bed","-blacklist_bed","-sequencing_type")
option_ok <- sapply(expected_opts,function(x) any(grepl(x,pave_help,fixed=TRUE)))

cat("\n========== PAVE OPTION CHECK ==========\n")
print(option_ok)

if(!all(option_ok)) stop("Installed PAVE JAR missing option(s): ",paste(names(option_ok)[!option_ok],collapse=", "))

# ============================================================
# 5. FIND SAGE VCFs
# ============================================================

all_vcfs <- list.files(SAGE_OUT,pattern="\\.sage\\.somatic\\.vcf\\.gz$",recursive=TRUE,full.names=TRUE)
input_vcfs <- setNames(rep(NA_character_,length(samples)),samples)

for(s in samples){
  hit <- all_vcfs[basename(all_vcfs)==paste0(s,".sage.somatic.vcf.gz")]
  if(length(hit)==1) input_vcfs[s] <- hit
  if(length(hit)>1) stop("Multiple SAGE VCFs found for ",s)
}

if(any(is.na(input_vcfs))) stop("Missing SAGE VCF(s):\n",paste(names(input_vcfs)[is.na(input_vcfs)],collapse="\n"))
cat("\nFound all",length(samples),"SAGE VCFs.\n")

# ============================================================
# 6. INPUT VCF QC
#
# Check:
#   BGZF integrity
#   .tbi index
#
# Prevents corrupted SAGE VCFs from reaching PAVE.
# ============================================================

bgzip <- Sys.which("bgzip")
tabix <- Sys.which("tabix")

if(!nzchar(bgzip)) stop("bgzip not found.")
if(!nzchar(tabix)) stop("tabix not found.")

input_qc <- data.frame(Sample=samples,BGZF_OK=FALSE,TBI_OK=FALSE,stringsAsFactors=FALSE)

cat("\n========== SAGE INPUT QC ==========\n")

for(i in seq_along(samples)){
  s <- samples[i]
  vcf <- input_vcfs[s]
  tbi <- paste0(vcf,".tbi")
  cat("[",i,"/23] ",s," : ",sep="")

  bgzf_status <- suppressWarnings(system2(bgzip,c("-t",vcf),stdout=FALSE,stderr=FALSE))
  input_qc$BGZF_OK[i] <- as.integer(bgzf_status)==0L

  if(!input_qc$BGZF_OK[i]){
    cat("BGZF FAILED\n")
    next
  }

  if(file.exists(tbi) && !is.na(file.info(tbi)$size) && file.info(tbi)$size>0){
    input_qc$TBI_OK[i] <- TRUE
  } else {
    cat("TBI missing -> creating | ")
    idx <- suppressWarnings(system2(tabix,c("-f","-p","vcf",vcf),stdout=FALSE,stderr=FALSE))
    input_qc$TBI_OK[i] <- as.integer(idx)==0L && file.exists(tbi) && file.info(tbi)$size>0
  }

  cat("BGZF PASS | TBI ",ifelse(input_qc$TBI_OK[i],"PASS","FAILED"),"\n",sep="")
}

cat("\n========== INPUT QC SUMMARY ==========\n")
print(input_qc,row.names=FALSE)

bad <- input_qc[!input_qc$BGZF_OK | !input_qc$TBI_OK,]

if(nrow(bad)>0) stop("PAVE HAS NOT STARTED. Failed SAGE input(s):\n",paste(bad$Sample,collapse="\n"))

cat("\nALL 23 SAGE VCFs PASSED QC.\n")
cat("PAVE WILL NOW START.\n")

# ============================================================
# 7. RESULTS TABLE
# ============================================================

results <- data.frame(
Sample=samples,
Status=NA_character_,
Exit_status=NA_integer_,
Runtime_min=NA_real_,
Output_size_MB=NA_real_,
stringsAsFactors=FALSE)

summary_file <- file.path(PAVE_OUT,"PAVE_run_summary.tsv")

# ============================================================
# 8. RUN PAVE
# ============================================================

for(i in seq_along(samples)){
  sample <- samples[i]
  input_vcf <- input_vcfs[sample]
  sample_out <- file.path(PAVE_OUT,sample)
  output_vcf <- file.path(sample_out,paste0(sample,".sage.somatic.pave.vcf.gz"))
  output_tbi <- paste0(output_vcf,".tbi")
  log_file <- file.path(sample_out,paste0(sample,".pave.log"))
  err_file <- file.path(sample_out,paste0(sample,".pave.err"))
  complete_marker <- file.path(sample_out,".PAVE_COMPLETE")

  cat("\n============================================================\n")
  cat("[",i,"/",length(samples),"] PAVE TUMOR-ONLY: ",sample,"\n",sep="")
  cat("============================================================\n")

  # Resume support
  if(file.exists(complete_marker) && file.exists(output_vcf) && file.info(output_vcf)$size>0 &&
     file.exists(output_tbi) && file.info(output_tbi)$size>0){
    cat("Already completed -> SKIP\n")
    results$Status[i] <- "SKIPPED_COMPLETED"
    results$Output_size_MB[i] <- round(file.info(output_vcf)$size/1024^2,2)
    write.table(results,summary_file,sep="\t",quote=FALSE,row.names=FALSE,na="")
    next
  }

  # Clean partial failed output for this sample
  if(dir.exists(sample_out)) unlink(sample_out,recursive=TRUE,force=TRUE)
  dir.create(sample_out,recursive=TRUE,showWarnings=FALSE)

  # IMPORTANT:
  # shQuote(PON_FILTERS) prevents the shell from interpreting
  # semicolons as separate commands.
  args <- c(
    paste0("-Xmx",MEMORY),"-jar",pave_jar,
    "-sample",sample,
    "-input_vcf",input_vcf,
    "-ref_genome",ref_fasta,
    "-ref_genome_version","38",
    "-ensembl_data_dir",ensembl_dir,
    "-driver_gene_panel",driver_panel,
    "-sequencing_type","ILLUMINA",
    "-pon_file",pon_file,
    "-pon_filters",shQuote(PON_FILTERS),
    "-gnomad_freq_dir",gnomad_dir,
    "-clinvar_vcf",clinvar_file,
    "-mappability_bed",mappability_file,
    "-blacklist_bed",blacklist_bed,
    "-output_dir",sample_out,
    "-output_vcf",output_vcf,
    "-threads",as.character(THREADS)
  )

  cat("Input :",input_vcf,"\n")
  cat("Output:",output_vcf,"\n")
  cat("Memory:",MEMORY,"| Threads:",THREADS,"\n")
  cat("PON:",basename(pon_file),"\n")
  cat("PON filters:",PON_FILTERS,"\n")
  cat("gnomAD directory:",gnomad_dir,"\n")

  start <- Sys.time()

  status <- tryCatch(
    system2(java_bin,args,stdout=log_file,stderr=err_file),
    error=function(e){
      cat("system2 ERROR:",conditionMessage(e),"\n")
      999L
    }
  )

  runtime <- round(as.numeric(difftime(Sys.time(),start,units="mins")),2)
  results$Exit_status[i] <- as.integer(status)
  results$Runtime_min[i] <- runtime

  basic_success <- as.integer(status)==0L && file.exists(output_vcf) &&
                   !is.na(file.info(output_vcf)$size) && file.info(output_vcf)$size>0

  if(!basic_success){
    results$Status[i] <- "FAILED"
    cat("\nFAILED:",sample,"| status:",status,"|",runtime,"min\n")

    if(file.exists(err_file) && file.info(err_file)$size>0){
      err <- readLines(err_file,warn=FALSE)
      cat("----- ERROR -----\n",paste(tail(err,30),collapse="\n"),"\n-----------------\n",sep="")
    }

    write.table(results,summary_file,sep="\t",quote=FALSE,row.names=FALSE,na="")
    next
  }

  # Validate output BGZF
  bgzf_out <- suppressWarnings(system2(bgzip,c("-t",output_vcf),stdout=FALSE,stderr=FALSE))

  if(as.integer(bgzf_out)!=0L){
    results$Status[i] <- "FAILED_OUTPUT_BGZF"
    cat("FAILED OUTPUT BGZF:",sample,"\n")
    write.table(results,summary_file,sep="\t",quote=FALSE,row.names=FALSE,na="")
    next
  }

  # Create output .tbi
  idx <- suppressWarnings(system2(tabix,c("-f","-p","vcf",output_vcf),stdout=FALSE,stderr=FALSE))

  if(as.integer(idx)!=0L || !file.exists(output_tbi) || file.info(output_tbi)$size==0){
    results$Status[i] <- "FAILED_OUTPUT_INDEX"
    cat("FAILED OUTPUT INDEX:",sample,"\n")
    write.table(results,summary_file,sep="\t",quote=FALSE,row.names=FALSE,na="")
    next
  }

  # Final success
  size_mb <- round(file.info(output_vcf)$size/1024^2,2)
  results$Status[i] <- "SUCCESS"
  results$Output_size_MB[i] <- size_mb

  writeLines(c(
    paste("Sample:",sample),
    paste("Completed:",Sys.time()),
    "PAVE_version: 1.9",
    "Sequencing_type: ILLUMINA",
    paste("Java_heap:",MEMORY),
    paste("Threads:",THREADS),
    paste("PON:",pon_file),
    paste("PON_filters:",PON_FILTERS),
    paste("gnomAD_dir:",gnomad_dir),
    paste("ClinVar:",clinvar_file),
    paste("Mappability:",mappability_file),
    paste("Blacklist:",blacklist_bed),
    "BGZF_integrity: PASS",
    "TBI_index: PASS"
  ),complete_marker)

  cat("\nSUCCESS:",sample,"|",runtime,"min |",size_mb,"MB\n")
  write.table(results,summary_file,sep="\t",quote=FALSE,row.names=FALSE,na="")
}

# ============================================================
# 9. FINAL SUMMARY
# ============================================================

cat("\n========== PAVE TUMOR-ONLY FINAL SUMMARY ==========\n")
print(results,row.names=FALSE)

cat(
"\nSUCCESS:",sum(results$Status=="SUCCESS",na.rm=TRUE),
"\nSKIPPED:",sum(results$Status=="SKIPPED_COMPLETED",na.rm=TRUE),
"\nFAILED:",sum(grepl("^FAILED",results$Status),na.rm=TRUE),
"\nTOTAL:",nrow(results),"\n")

write.table(results,summary_file,sep="\t",quote=FALSE,row.names=FALSE,na="")
cat("\nSummary:",summary_file,"\n")
cat("Current output:",PAVE_OUT,"\n")
cat("Old output backup:",BACKUP_OUT,"\n")

```

---

## 24. Expected directory structure

```text
UCaTS_Organoids/
│
├── 07_SAGE/
│   └── SAGE_output/
│       ├── I_26166_S_29146/
│       │   └── I_26166_S_29146.sage.somatic.vcf.gz
│       └── ...
│
├── 09_PAVE/
│   ├── PAVE_tools/
│   │   └── pave_v1.9.jar
│   │
│   ├── PAVE_output_pre_tumoronly/
│   │   └── historical PAVE results
│   │
│   └── PAVE_output/
│       ├── PAVE_run_summary.tsv
│       ├── I_26166_S_29146/
│       │   ├── I_26166_S_29146.sage.somatic.pave.vcf.gz
│       │   ├── I_26166_S_29146.sage.somatic.pave.vcf.gz.tbi
│       │   ├── I_26166_S_29146.pave.log
│       │   ├── I_26166_S_29146.pave.err
│       │   └── .PAVE_COMPLETE
│       └── ...
│
└── shared references/
    ├── Illumina/Dragen/hg38.fa
    └── hmftools/hmf_pipeline_resources.38_v3.0.0--8/
```

---

## 25. Reproducibility checklist

```text
[ ] PAVE v1.9 JAR exists
[ ] PAVE JAR checksum recorded
[ ] Java version recorded

[ ] hg38.fa exists
[ ] hg38.fa.fai exists

[ ] HMF resource bundle = 38_v3.0.0--8
[ ] DriverGenePanel.38.tsv exists
[ ] Ensembl data exists
[ ] hmf_wgs_sage_pon_1000.38.tsv.gz exists
[ ] gnomAD directory contains 24 v38 chromosome files
[ ] clinvar.38.vcf.gz exists
[ ] mappability_150.38.bed.gz exists
[ ] KnownBlacklist.germline.38.bed exists

[ ] PON filters recorded exactly
[ ] installed JAR options verified

[ ] all 23 SAGE VCFs found
[ ] all SAGE VCFs pass BGZF
[ ] all SAGE VCFs have valid TBI

[ ] old PAVE output backed up
[ ] current output written to PAVE_output

[ ] output VCF passes BGZF
[ ] output TBI exists
[ ] .PAVE_COMPLETE written
[ ] PAVE_run_summary.tsv written
```

---

## 26. Project summary

```text
Tool:
PAVE v1.9

Input:
07_SAGE/SAGE_output/<SAMPLE>/<SAMPLE>.sage.somatic.vcf.gz

Output:
09_PAVE/PAVE_output/<SAMPLE>/<SAMPLE>.sage.somatic.pave.vcf.gz

Historical output:
09_PAVE/PAVE_output_pre_tumoronly

Genome:
GRCh38 / hg38

Reference FASTA:
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa

HMF resources:
hmf_pipeline_resources.38_v3.0.0--8

Tumor-only annotations:
PON
PON filters
gnomAD
ClinVar
Mappability
Germline blacklist

PON filters:
HOTSPOT:6:5;PANEL:3:3;UNKNOWN:3:0

Compute:
48G Java heap
4 threads

Samples:
23
```
