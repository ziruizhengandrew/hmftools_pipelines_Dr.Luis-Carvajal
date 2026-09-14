---
title: "QSEE v1.0 Tumor-Only WGS QC Workflow — UCaTS Organoids"
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

> **Purpose.** This notebook documents the complete QSEE workflow used for the UCaTS Organoids tumor-only WGS cohort: software installation, reference/resource acquisition, upstream input requirements, directory organization, input staging, multi-sample execution, provenance capture, validation, interpretation, troubleshooting, and export.
>
> **Important.** All pipeline code chunks in this documentation use `eval=FALSE`, so knitting/exporting this notebook will **not** rerun QSEE or modify the analysis.
>
> **Current formal state.** QSEE has completed successfully for all 23 tumor samples. The validated multi-sample outputs are present and contain all 23 sample IDs.

# 1. QSEE overview

QSEE ("Qsee") is an HMFtools quality-control aggregation and visualization tool. It collects QC metrics produced by other parts of the HMF/WiGiTS pipeline and reports WARN/FAIL thresholds across multiple QC categories.

QSEE does **not** replace the upstream algorithms. It consumes their output files.

For this project:

```text
PURPLE      ─┐
BamMetrics  ─┤
COBALT      ─┤
ESVEE       ─┼──> QSEE ──> QC status table + visualization data + PDF report
REDUX       ─┘
```

The current workflow is:

```text
tumor-only
23 samples
multi-sample QSEE mode
ILLUMINA sequencing type
merge plots into one PDF
allow optional missing inputs
```

# 2. Official project documentation

Main QSEE repository:

```text
https://github.com/hartwigmedical/hmftools/tree/master/qsee
```

HMFtools repository:

```text
https://github.com/hartwigmedical/hmftools
```

HMFtools resource documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/pipeline/README_RESOURCES.md
```

Oncoanalyser reference-data documentation:

```text
https://nf-co.re/oncoanalyser/3.0.0/docs/usage/
```

The HMFtools repository currently lists QSEE as version `1.0`.

# 3. Current project directory layout

The QSEE working directory is:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE
```

Recommended/current structure:

```text
14_QSEE/
├── QSEE_tools/
│   └── qsee_v1.0.jar
├── QSEE_reference/
│   └── optional/local QSEE-specific reference copies
├── QSEE_input/
│   └── symlinks to required upstream files
├── QSEE_output/
│   ├── multisample.qsee.status.tsv.gz
│   ├── multisample.qsee.vis.data.tsv.gz
│   └── multisample.qsee.vis.report.pdf
├── commands/
│   └── exact executed QSEE command records
├── logs/
│   ├── QSEE program logs
│   └── full R console logs
├── QSEE_input_manifest.tsv
├── qsee_sample_ids.tsv
├── QSEE_multisample_output_validation.tsv
└── QSEE_multisample_sample_validation.tsv
```

# 4. Current upstream data sources

The final/current paths are:

```text
PURPLE
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Purple_rerun

COBALT
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/10_Purple/Amber_Cobalt_OldPurple

ESVEE
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/08_ESVEE

REDUX
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux

BamMetrics
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN
```

Important distinction:

```text
Purple_rerun
    = current/final PURPLE result

Amber_Cobalt_OldPurple
    = historical PURPLE directory retained because its AMBER/COBALT
      outputs are still the source used by the current workflow
```

`Purple_plot_regen_with_charts` is not used as a QSEE source.

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

Total:

```text
23 tumor samples
```

The production script derives this list directly from:

```text
10_Purple/Purple_rerun/**/*.purple.qc
```

and stops unless exactly 23 unique samples are found.

# 6. Software installation

## 6.1 Java

QSEE is a Java application. This project uses:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Observed Java:

```text
OpenJDK 21
```

Check it with:

```bash
/home/zzr123/.conda/envs/purple/bin/java -version
```

If a new Java environment is needed, one reproducible conda approach is:

```bash
conda create -n qsee_java -c conda-forge openjdk=21 -y
conda activate qsee_java
java -version
```

For production reproducibility, record the actual `java -version` output rather than relying only on the environment name.

## 6.2 QSEE JAR — current project copy

Current project JAR:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/QSEE_tools/qsee_v1.0.jar
```

Observed SHA256 in the successful run:

```text
333e4d76979aa281a35dd611a84645392d2f18dd7c8928a6d9fb76b848d50442
```

Verify:

```bash
sha256sum \
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/QSEE_tools/qsee_v1.0.jar
```

For the existing project, preserve this JAR rather than silently replacing it.

## 6.3 Downloading QSEE for a fresh installation

The official release source is:

```text
https://github.com/hartwigmedical/hmftools/releases
```

QSEE release tags use names such as:

```text
qsee-v1.0-beta.11
```

For a fresh installation, choose an explicit QSEE release tag and record that tag. Do **not** silently download "latest" for a reproducible analysis.

A robust GitHub-API method that avoids guessing the release asset filename is:

```bash
set -euo pipefail

QSEE_BASE="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE"
TOOL_DIR="$QSEE_BASE/QSEE_tools"
TAG="qsee-v1.0-beta.11"

mkdir -p "$TOOL_DIR"
cd "$TOOL_DIR"

ASSET_URL="$(
python3 - "$TAG" <<'PY'
import json
import sys
import urllib.request

tag = sys.argv[1]
api = f"https://api.github.com/repos/hartwigmedical/hmftools/releases/tags/{tag}"

with urllib.request.urlopen(api) as r:
    release = json.load(r)

jars = [
    a["browser_download_url"]
    for a in release["assets"]
    if a["name"].endswith(".jar")
]

if len(jars) != 1:
    raise SystemExit(
        f"Expected exactly one JAR asset for {tag}, found {len(jars)}: {jars}"
    )

print(jars[0])
PY
)"

echo "Resolved QSEE asset:"
echo "$ASSET_URL"

curl -L --fail \
  "$ASSET_URL" \
  -o "qsee_${TAG#qsee-v}.jar"

sha256sum "qsee_${TAG#qsee-v}.jar"
```

This section is for a **fresh install**. It should not be used to imply that a newly downloaded beta JAR is byte-identical to the project's existing `qsee_v1.0.jar` unless the SHA256 values are actually compared.

# 7. QSEE reference/resources

A useful distinction is:

```text
QSEE-specific resource
vs.
upstream pipeline output
```

QSEE itself does not require a large standalone reference database comparable to a genome FASTA. The main external resource supplied directly to QSEE in this workflow is:

```text
DriverGenePanel.38.tsv
```

An optional cohort resource is:

```text
qsee.cohort.percentiles.tsv.gz
```

Most other QSEE inputs are generated by PURPLE, BamMetrics, COBALT, ESVEE, and REDUX.

# 8. Downloading the HMFtools resource bundle

The project uses the shared GRCh38 HMFtools bundle:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
```

This bundle contains the driver gene panel used by QSEE:

```text
common/DriverGenePanel.38.tsv
```

## 8.1 Current bundle used by this project

Bundle name:

```text
hmf_pipeline_resources.38_v3.0.0--8.tar.gz
```

The Oncoanalyser 3.0.0 reference-data documentation lists this GRCh38 WiGiTS/HMFtools resource bundle.

A direct staging example:

```bash
set -euo pipefail

REF_BASE="/quobyte/luisccgrp/REFERENCE_DATA/hmftools"
BUNDLE="hmf_pipeline_resources.38_v3.0.0--8.tar.gz"

mkdir -p "$REF_BASE"
cd "$REF_BASE"

wget -c \
"https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/$BUNDLE"

tar -xzvf "$BUNDLE"

ls -lh \
"$REF_BASE/hmf_pipeline_resources.38_v3.0.0--8/common/DriverGenePanel.38.tsv"
```

After extraction, the QSEE driver panel is:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
common/DriverGenePanel.38.tsv
```

## 8.2 Recommended alternative: Oncoanalyser reference staging

If setting up the HMF resources from scratch, Oncoanalyser can stage reference files automatically.

For Oncoanalyser 3.0.0, use the reference-preparation workflow documented by nf-core/Oncoanalyser rather than manually assembling individual WiGiTS resource files.

The exact command should follow the documentation for the selected Oncoanalyser version. The important principle is:

```text
pin the Oncoanalyser version
pin GRCh38_hmf
stage HMFtools/WiGiTS resources once
reuse the shared bundle across modules
```

Do not create separate duplicate HMF resource bundles inside every algorithm directory.

# 9. Driver gene panel

QSEE receives:

```text
-driver_gene_panel
```

with:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
common/DriverGenePanel.38.tsv
```

Check:

```bash
DRIVER_PANEL="/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/common/DriverGenePanel.38.tsv"

ls -lh "$DRIVER_PANEL"
head "$DRIVER_PANEL"
```

Do not substitute a different driver panel without recording the change.

# 10. Optional cohort percentiles

QSEE can compare the current samples with a cohort using:

```text
-cohort_percentiles_file
```

This project did **not** require a cohort-percentiles file for the successful run.

If omitted, QSEE can still calculate QC metrics and generate output; cohort percentile distributions are simply unavailable.

If a sufficiently large internal cohort is available, QSEE can generate cohort percentiles. The official documentation recommends approximately 50 or more samples.

Example:

```bash
java -cp qsee.jar \
com.hartwig.hmftools.qsee.cohort.CohortPercentilesTrainer \
-sample_id_file sample_ids.txt \
-sample_data_dir inputs/ \
-driver_gene_panel DriverGenePanel.38.tsv \
-output_dir cohort_percentiles/ \
-threads 12
```

Optional:

```text
-write_cohort_features
-allow_missing_input
```

# 11. QSEE input files

Official QSEE input categories used in this workflow are:

```text
PURPLE
  <SAMPLE>.purple.purity.tsv      tumor mandatory
  <SAMPLE>.purple.qc              tumor mandatory

BamMetrics
  <SAMPLE>.bam_metric.summary.tsv       tumor mandatory
  <SAMPLE>.bam_metric.coverage.tsv      tumor mandatory
  <SAMPLE>.bam_metric.flag_counts.tsv   tumor mandatory
  <SAMPLE>.bam_metric.frag_length.tsv   optional
  <SAMPLE>.bam_metric.gene_coverage.tsv optional

COBALT
  <SAMPLE>.cobalt.gc.median.tsv         optional

ESVEE
  <SAMPLE>.esvee.prep.disc_stats.tsv    optional

REDUX
  <SAMPLE>.redux.bqr.tsv                 optional
  <SAMPLE>.redux.duplicate_freq.tsv      optional
  <SAMPLE>.redux.ms_table.tsv.gz         optional
```

For the current cohort, every expected input was found except:

```text
<SAMPLE>.bam_metric.gene_coverage.tsv
```

for all 23 samples.

This is acceptable because gene coverage is optional and QSEE was run with:

```text
-allow_missing_input
```

# 12. Why gene coverage is missing

The HMF BamMetrics tool only writes gene-coverage output when the relevant driver-gene/Ensembl configuration is supplied during BamMetrics execution.

Therefore:

```text
missing bam_metric.gene_coverage.tsv
```

does not imply that the existing BamMetrics run failed.

For this QSEE analysis, the mandatory BamMetrics files were present for all 23 tumors.

# 13. QSEE tumor-only multi-sample sample-ID file

QSEE multi-sample mode takes:

```text
-sample_id_file
```

For tumor-only mode, the `ReferenceId` column can be omitted.

Current file:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/qsee_sample_ids.tsv
```

Format:

```text
TumorId
I_26166_S_29146
I_26227_S_29149
...
I_27675_S_29168
```

R generation:

```r
write.table(
  data.frame(TumorId=samples),
  "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/qsee_sample_ids.tsv",
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)
```

# 14. Why this workflow stages inputs with symlinks

QSEE can accept tool-specific directories directly, but this project uses one staged `QSEE_input` directory containing symlinks to all selected upstream files.

Advantages:

```text
1. Exact input provenance is explicit.
2. Current PURPLE can be separated from old PURPLE.
3. COBALT can still come from Amber_Cobalt_OldPurple.
4. No large files are copied.
5. The input manifest records exactly which source file was used.
6. QSEE receives one simple -sample_data_dir.
```

Symlinks do not alter the upstream source files.

# 15. Current source mapping

```text
QSEE file                          Source
-----------------------------------------------------------------------
*.purple.purity.tsv               10_Purple/Purple_rerun
*.purple.qc                       10_Purple/Purple_rerun

*.cobalt.gc.median.tsv            10_Purple/Amber_Cobalt_OldPurple

*.esvee.prep.disc_stats.tsv       08_ESVEE

*.redux.bqr.tsv                   06_REDux
*.redux.duplicate_freq.tsv        06_REDux
*.redux.ms_table.tsv.gz           06_REDux

*.bam_metric.summary.tsv          05_ASCAT_CN
*.bam_metric.coverage.tsv         05_ASCAT_CN
*.bam_metric.flag_counts.tsv      05_ASCAT_CN
*.bam_metric.frag_length.tsv      05_ASCAT_CN
*.bam_metric.gene_coverage.tsv    05_ASCAT_CN (currently absent)
```

# 16. Complete final R script

The following is the final production-style script corresponding to the completed QSEE run.

```r
library(dplyr)

main <- function(){

options(warn=1)

# ============================================================
# 1. PATHS
# ============================================================

BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"

QSEE_BASE <- file.path(BASE,"14_QSEE")
TOOL_DIR <- file.path(QSEE_BASE,"QSEE_tools")
REF_DIR <- file.path(QSEE_BASE,"QSEE_reference")
INPUT_DIR <- file.path(QSEE_BASE,"QSEE_input")
OUTPUT_DIR <- file.path(QSEE_BASE,"QSEE_output")
LOG_DIR <- file.path(QSEE_BASE,"logs")
COMMAND_DIR <- file.path(QSEE_BASE,"commands")

PURPLE_ROOT <- file.path(
  BASE,"10_Purple","Purple_rerun"
)

AMBER_COBALT_ROOT <- file.path(
  BASE,"10_Purple","Amber_Cobalt_OldPurple"
)

ESVEE_ROOT <- file.path(BASE,"08_ESVEE")
REDUX_ROOT <- file.path(BASE,"06_REDux")
BAMMETRICS_ROOT <- file.path(BASE,"05_ASCAT_CN")

HMF_REF <- "/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"

DRIVER_GENE_PANEL <- file.path(
  HMF_REF,
  "common",
  "DriverGenePanel.38.tsv"
)

JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"

QSEE_JAR <- file.path(
  TOOL_DIR,
  "qsee_v1.0.jar"
)

for(x in c(
  QSEE_BASE,
  TOOL_DIR,
  REF_DIR,
  INPUT_DIR,
  OUTPUT_DIR,
  LOG_DIR,
  COMMAND_DIR
)){
  dir.create(
    x,
    recursive=TRUE,
    showWarnings=FALSE
  )
}

# ============================================================
# 2. HELPERS
# ============================================================

valid_file <- function(x){
  length(x)==1 &&
    !is.na(x) &&
    file.exists(x) &&
    !is.na(file.info(x)$size) &&
    file.info(x)$size > 0
}

sha256_file <- function(x){
  if(!valid_file(x)) return(NA_character_)
  z <- suppressWarnings(
    system2(
      "sha256sum",
      x,
      stdout=TRUE,
      stderr=TRUE
    )
  )
  if(length(z)==0) return(NA_character_)
  strsplit(z[1],"\\s+")[[1]][1]
}

command_string <- function(exe,args){
  paste(
    shQuote(exe),
    paste(
      shQuote(args),
      collapse=" "
    )
  )
}

build_index <- function(root){
  cat("Indexing:",root,"\n")
  if(!dir.exists(root)) return(character())
  list.files(
    root,
    recursive=TRUE,
    full.names=TRUE
  )
}

find_latest_indexed <- function(
  sample,
  suffix,
  index
){
  wanted <- paste0(sample,suffix)
  files <- index[
    basename(index)==wanted
  ]
  if(length(files)==0){
    return(NA_character_)
  }
  files[
    which.max(
      file.info(files)$mtime
    )
  ]
}

# ============================================================
# 3. CONSOLE LOG
# ============================================================

console_log <- file.path(
  LOG_DIR,
  paste0(
    "QSEE_console_",
    format(Sys.time(),"%Y%m%d_%H%M%S"),
    ".log"
  )
)

sink(
  console_log,
  split=TRUE
)

run_status <- "INTERRUPTED_OR_FAILED"

on.exit({
  cat("\n====================================================\n")
  cat("QSEE RUN END\n")
  cat("STATUS:",run_status,"\n")
  cat(
    "TIME:",
    format(Sys.time(),"%Y-%m-%d %H:%M:%S %Z"),
    "\n"
  )
  cat("Console log:",console_log,"\n")
  cat("====================================================\n")

  while(sink.number(type="output")>0){
    sink(type="output")
  }
},add=TRUE)

cat("====================================================\n")
cat("QSEE v1.0 FINAL TUMOR-ONLY MULTI-SAMPLE RUN\n")
cat(
  "START:",
  format(Sys.time(),"%Y-%m-%d %H:%M:%S %Z"),
  "\n"
)
cat("====================================================\n")

# ============================================================
# 4. PREFLIGHT
# ============================================================

required_files <- c(
  JAVA,
  QSEE_JAR,
  DRIVER_GENE_PANEL
)

required_dirs <- c(
  PURPLE_ROOT,
  AMBER_COBALT_ROOT,
  ESVEE_ROOT,
  REDUX_ROOT,
  BAMMETRICS_ROOT
)

file_check <- data.frame(
  Path=required_files,
  Exists=file.exists(required_files),
  Size_bytes=ifelse(
    file.exists(required_files),
    file.info(required_files)$size,
    NA_real_
  )
)

dir_check <- data.frame(
  Path=required_dirs,
  Exists=dir.exists(required_dirs)
)

print(file_check,row.names=FALSE)
print(dir_check,row.names=FALSE)

if(
  !all(file_check$Exists) ||
  !all(dir_check$Exists)
){
  stop(
    "Missing required QSEE tool/reference/input directory."
  )
}

QSEE_SHA256 <- sha256_file(QSEE_JAR)

cat("\nQSEE JAR:",QSEE_JAR,"\n")
cat("QSEE SHA256:",QSEE_SHA256,"\n")
cat("Driver panel:",DRIVER_GENE_PANEL,"\n")

cat("\nJava version:\n")

java_version <- system2(
  JAVA,
  "-version",
  stdout=TRUE,
  stderr=TRUE
)

cat(
  paste(java_version,collapse="\n"),
  "\n"
)

# ============================================================
# 5. FINAL SAMPLE LIST FROM PURPLE
# ============================================================

purple_qc_files <- list.files(
  PURPLE_ROOT,
  pattern="\\.purple\\.qc$",
  recursive=TRUE,
  full.names=TRUE
)

if(length(purple_qc_files)==0){
  stop(
    "No *.purple.qc files found under final Purple_rerun."
  )
}

samples <- sort(
  unique(
    sub(
      "\\.purple\\.qc$",
      "",
      basename(purple_qc_files)
    )
  )
)

EXPECTED_N <- 23L

cat(
  "\nFound",
  length(samples),
  "samples from",
  PURPLE_ROOT,
  "\n"
)

print(samples)

if(length(samples)!=EXPECTED_N){
  stop(
    "Expected 23 samples but found ",
    length(samples)
  )
}

# ============================================================
# 6. INDEX INPUT ROOTS
# ============================================================

purple_index <- build_index(PURPLE_ROOT)
cobalt_index <- build_index(AMBER_COBALT_ROOT)
esvee_index <- build_index(ESVEE_ROOT)
redux_index <- build_index(REDUX_ROOT)
bammetrics_index <- build_index(BAMMETRICS_ROOT)

# ============================================================
# 7. INPUT DEFINITIONS
# ============================================================

input_def <- data.frame(
  category=c(
    "PURPLE","PURPLE","COBALT","ESVEE",
    "REDUX","REDUX","REDUX",
    "BAMMETRICS","BAMMETRICS","BAMMETRICS",
    "BAMMETRICS","BAMMETRICS"
  ),
  suffix=c(
    ".purple.purity.tsv",
    ".purple.qc",
    ".cobalt.gc.median.tsv",
    ".esvee.prep.disc_stats.tsv",
    ".redux.bqr.tsv",
    ".redux.duplicate_freq.tsv",
    ".redux.ms_table.tsv.gz",
    ".bam_metric.summary.tsv",
    ".bam_metric.coverage.tsv",
    ".bam_metric.flag_counts.tsv",
    ".bam_metric.frag_length.tsv",
    ".bam_metric.gene_coverage.tsv"
  ),
  stringsAsFactors=FALSE
)

index_list <- list(
  purple_index,
  purple_index,
  cobalt_index,
  esvee_index,
  redux_index,
  redux_index,
  redux_index,
  bammetrics_index,
  bammetrics_index,
  bammetrics_index,
  bammetrics_index,
  bammetrics_index
)

source_roots <- c(
  PURPLE_ROOT,
  PURPLE_ROOT,
  AMBER_COBALT_ROOT,
  ESVEE_ROOT,
  REDUX_ROOT,
  REDUX_ROOT,
  REDUX_ROOT,
  BAMMETRICS_ROOT,
  BAMMETRICS_ROOT,
  BAMMETRICS_ROOT,
  BAMMETRICS_ROOT,
  BAMMETRICS_ROOT
)

# ============================================================
# 8. CLEAN STAGING INPUT DIR
# ============================================================

existing <- list.files(
  INPUT_DIR,
  full.names=TRUE,
  all.files=TRUE,
  no..=TRUE
)

if(length(existing)>0){
  unlink(existing,recursive=TRUE)
}

# ============================================================
# 9. CREATE INPUT SYMLINKS + MANIFEST
# ============================================================

manifest <- list()
k <- 1L

for(sample in samples){

  cat(
    "\n====================================================\n",
    "SAMPLE: ",sample,"\n",
    "====================================================\n",
    sep=""
  )

  for(i in seq_len(nrow(input_def))){

    category <- input_def$category[i]
    suffix <- input_def$suffix[i]
    index <- index_list[[i]]
    source_root <- source_roots[i]

    source <- find_latest_indexed(
      sample,
      suffix,
      index
    )

    target <- file.path(
      INPUT_DIR,
      paste0(sample,suffix)
    )

    if(!is.na(source)){

      if(
        file.exists(target) ||
        nzchar(Sys.readlink(target))
      ){
        unlink(target)
      }

      ok <- file.symlink(
        source,
        target
      )

      if(
        isTRUE(ok) &&
        file.exists(target)
      ){
        status <- "FOUND"
      } else {
        status <- "LINK_FAILED"
      }

    } else {

      status <- "MISSING"
    }

    cat(
      sprintf(
        "%-12s %-36s %s\n",
        category,
        suffix,
        status
      )
    )

    manifest[[k]] <- data.frame(
      Sample=sample,
      Category=category,
      Suffix=suffix,
      Status=status,
      Source_Root=source_root,
      Source=ifelse(
        is.na(source),
        "",
        source
      ),
      QSEE_Input=target,
      Source_Size=ifelse(
        !is.na(source) &&
        file.exists(source),
        file.info(source)$size,
        NA_real_
      ),
      Source_Modified=ifelse(
        !is.na(source) &&
        file.exists(source),
        as.character(
          file.info(source)$mtime
        ),
        ""
      ),
      stringsAsFactors=FALSE
    )

    k <- k+1L
  }
}

manifest <- bind_rows(manifest)

manifest_file <- file.path(
  QSEE_BASE,
  "QSEE_input_manifest.tsv"
)

write.table(
  manifest,
  manifest_file,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

# ============================================================
# 10. INPUT SUMMARY + MANDATORY CHECK
# ============================================================

print(
  manifest %>%
    count(Category,Status),
  row.names=FALSE
)

missing <- manifest %>%
  filter(Status!="FOUND")

if(nrow(missing)>0){
  cat("\nMissing optional/required files:\n")
  print(
    missing %>%
      select(
        Sample,
        Category,
        Suffix,
        Status
      ),
    row.names=FALSE
  )
}

mandatory_suffixes <- c(
  ".purple.purity.tsv",
  ".purple.qc",
  ".bam_metric.summary.tsv",
  ".bam_metric.coverage.tsv",
  ".bam_metric.flag_counts.tsv"
)

mandatory_missing <- manifest %>%
  filter(
    Suffix %in% mandatory_suffixes,
    Status!="FOUND"
  )

if(nrow(mandatory_missing)>0){
  print(mandatory_missing,row.names=FALSE)
  stop(
    "QSEE NOT STARTED because mandatory inputs are missing."
  )
}

cat("\nALL MANDATORY QSEE INPUTS FOUND\n")

# ============================================================
# 11. PROVENANCE SOURCE CHECKS
# ============================================================

wrong_purple <- manifest %>%
  filter(
    Category=="PURPLE",
    Status=="FOUND",
    !startsWith(
      Source,
      paste0(PURPLE_ROOT,"/")
    )
  )

if(nrow(wrong_purple)>0){
  stop(
    "A PURPLE input came from the wrong directory."
  )
}

wrong_cobalt <- manifest %>%
  filter(
    Category=="COBALT",
    Status=="FOUND",
    !startsWith(
      Source,
      paste0(AMBER_COBALT_ROOT,"/")
    )
  )

if(nrow(wrong_cobalt)>0){
  stop(
    "A COBALT input came from the wrong directory."
  )
}

cat("\nPURPLE provenance check: PASS\n")
cat("COBALT provenance check: PASS\n")

# ============================================================
# 12. SAMPLE ID FILE
# ============================================================

sample_id_file <- file.path(
  QSEE_BASE,
  "qsee_sample_ids.tsv"
)

write.table(
  data.frame(TumorId=samples),
  sample_id_file,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

THREADS <- min(
  23L,
  length(samples)
)

# ============================================================
# 13. EXPECTED MULTI-SAMPLE OUTPUTS
# ============================================================

EXPECTED_STATUS <- file.path(
  OUTPUT_DIR,
  "multisample.qsee.status.tsv.gz"
)

EXPECTED_VIS_DATA <- file.path(
  OUTPUT_DIR,
  "multisample.qsee.vis.data.tsv.gz"
)

EXPECTED_REPORT <- file.path(
  OUTPUT_DIR,
  "multisample.qsee.vis.report.pdf"
)

FORCE_RERUN <- FALSE

existing_complete <- all(
  vapply(
    c(
      EXPECTED_STATUS,
      EXPECTED_VIS_DATA,
      EXPECTED_REPORT
    ),
    valid_file,
    logical(1)
  )
)

# ============================================================
# 14. RUN OR SKIP
# ============================================================

if(
  existing_complete &&
  !FORCE_RERUN
){

  cat(
    "\nQSEE MULTI-SAMPLE OUTPUT ALREADY COMPLETE\n",
    "QSEE execution will be SKIPPED.\n",
    sep=""
  )

  qsee_exit <- 0L
  qsee_runtime_min <- 0
  qsee_log <- NA_character_
  command_file <- NA_character_

} else {

  timestamp <- format(
    Sys.time(),
    "%Y%m%d_%H%M%S"
  )

  qsee_log <- file.path(
    LOG_DIR,
    paste0(
      "QSEE_all_samples_",
      timestamp,
      ".log"
    )
  )

  command_file <- file.path(
    COMMAND_DIR,
    paste0(
      "QSEE_all_samples_",
      timestamp,
      ".command.txt"
    )
  )

  args <- c(
    "-Xmx32G",
    "-jar",QSEE_JAR,
    "-sample_id_file",sample_id_file,
    "-sample_data_dir",INPUT_DIR,
    "-driver_gene_panel",DRIVER_GENE_PANEL,
    "-output_dir",OUTPUT_DIR,
    "-sequencing_type","ILLUMINA",
    "-threads",as.character(THREADS),
    "-merge_plots",
    "-allow_missing_input"
  )

  exact_command <- command_string(
    JAVA,
    args
  )

  writeLines(
    c(
      paste0(
        "DATE=",
        format(
          Sys.time(),
          "%Y-%m-%d %H:%M:%S %Z"
        )
      ),
      "TOOL=QSEE v1.0",
      "MODE=MULTI_SAMPLE_TUMOR_ONLY",
      paste0("QSEE_JAR=",QSEE_JAR),
      paste0(
        "QSEE_JAR_SHA256=",
        QSEE_SHA256
      ),
      paste0(
        "SAMPLES=",
        length(samples)
      ),
      paste0(
        "PURPLE_ROOT=",
        PURPLE_ROOT
      ),
      paste0(
        "COBALT_ROOT=",
        AMBER_COBALT_ROOT
      ),
      paste0(
        "ESVEE_ROOT=",
        ESVEE_ROOT
      ),
      paste0(
        "REDUX_ROOT=",
        REDUX_ROOT
      ),
      paste0(
        "BAMMETRICS_ROOT=",
        BAMMETRICS_ROOT
      ),
      paste0(
        "INPUT_MANIFEST=",
        manifest_file
      ),
      "",
      "ACTUAL_COMMAND:",
      exact_command
    ),
    command_file
  )

  cat(
    "\nACTUAL COMMAND:\n",
    exact_command,
    "\n",
    sep=""
  )

  start_time <- Sys.time()

  qsee_exit <- system2(
    JAVA,
    args=args,
    stdout=qsee_log,
    stderr=qsee_log
  )

  end_time <- Sys.time()

  qsee_runtime_min <- round(
    as.numeric(
      difftime(
        end_time,
        start_time,
        units="mins"
      )
    ),
    2
  )

  cat(
    "\nQSEE exit status:",
    qsee_exit,
    "\n"
  )

  cat(
    "Runtime:",
    qsee_runtime_min,
    "minutes\n"
  )

  if(qsee_exit!=0){

    if(file.exists(qsee_log)){

      z <- readLines(
        qsee_log,
        warn=FALSE
      )

      cat(
        tail(z,100),
        sep="\n"
      )
    }

    stop(
      "QSEE failed with exit status ",
      qsee_exit
    )
  }
}

# ============================================================
# 15. OUTPUT VALIDATION
# ============================================================

expected_outputs <- c(
  EXPECTED_STATUS,
  EXPECTED_VIS_DATA,
  EXPECTED_REPORT
)

output_validation <- data.frame(
  File=basename(expected_outputs),
  Path=expected_outputs,
  Exists=file.exists(expected_outputs),
  Size_bytes=ifelse(
    file.exists(expected_outputs),
    file.info(expected_outputs)$size,
    NA_real_
  ),
  Non_empty=vapply(
    expected_outputs,
    valid_file,
    logical(1)
  ),
  stringsAsFactors=FALSE
)

validation_file <- file.path(
  QSEE_BASE,
  "QSEE_multisample_output_validation.tsv"
)

write.table(
  output_validation,
  validation_file,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

print(
  output_validation,
  row.names=FALSE
)

# ============================================================
# 16. VERIFY 23 SAMPLE IDs IN STATUS TABLE
# ============================================================

if(valid_file(EXPECTED_STATUS)){

  qsee_status <- read.delim(
    gzfile(EXPECTED_STATUS),
    check.names=FALSE,
    stringsAsFactors=FALSE
  )

  character_columns <- names(qsee_status)[
    vapply(
      qsee_status,
      function(x){
        is.character(x) ||
        is.factor(x)
      },
      logical(1)
    )
  ]

  present <- vapply(
    samples,
    function(sample){

      any(
        vapply(
          character_columns,
          function(col){
            any(
              as.character(
                qsee_status[[col]]
              )==sample,
              na.rm=TRUE
            )
          },
          logical(1)
        )
      )
    },
    logical(1)
  )

  status_sample_check <- data.frame(
    Sample=samples,
    Present_in_status=present,
    stringsAsFactors=FALSE
  )

  write.table(
    status_sample_check,
    file.path(
      QSEE_BASE,
      "QSEE_multisample_sample_validation.tsv"
    ),
    sep="\t",
    quote=FALSE,
    row.names=FALSE
  )

  cat(
    "\nSamples explicitly found in status table:",
    sum(present),
    "/",
    length(samples),
    "\n"
  )
}

# ============================================================
# 17. FINAL STATUS
# ============================================================

all_outputs_complete <- all(
  output_validation$Non_empty
)

if(
  qsee_exit==0 &&
  all_outputs_complete
){
  run_status <- "COMPLETED"
} else {
  run_status <- "COMPLETED_WITH_OUTPUT_WARNING"
}

session_file <- file.path(
  QSEE_BASE,
  paste0(
    "QSEE_sessionInfo_",
    format(
      Sys.time(),
      "%Y%m%d_%H%M%S"
    ),
    ".txt"
  )
)

writeLines(
  capture.output(
    sessionInfo()
  ),
  session_file
)

cat("\n====================================================\n")
cat("QSEE FINAL SUMMARY\n")
cat("====================================================\n")
cat("Exit status      :",qsee_exit,"\n")
cat("Runtime          :",qsee_runtime_min,"minutes\n")
cat("Samples          :",length(samples),"\n")
cat("Mode             : MULTI-SAMPLE TUMOR-ONLY\n")
cat(
  "Expected outputs:",
  sum(output_validation$Non_empty),
  "/",
  nrow(output_validation),
  "\n"
)
cat("Final status     :",run_status,"\n")
cat("====================================================\n")

} # END main()

main()
```

# 17. Exact production QSEE command

The successful production configuration resolves conceptually to:

```bash
/home/zzr123/.conda/envs/purple/bin/java \
-Xmx32G \
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

The generated `.command.txt` should be treated as the authoritative record of the exact command actually executed on a given run.

# 18. Multi-sample output behavior

This point caused an earlier validation mistake and should be remembered.

In multi-sample mode with:

```text
-sample_id_file
-merge_plots
```

QSEE generates combined outputs:

```text
multisample.qsee.status.tsv.gz
multisample.qsee.vis.data.tsv.gz
multisample.qsee.vis.report.pdf
```

It does **not** necessarily generate one:

```text
<SAMPLE>.qsee.status.tsv
```

for each tumor.

Therefore the validator must check the three `multisample.*` files.

# 19. Current validated final outputs

Current output directory:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/QSEE_output
```

Validated:

```text
multisample.qsee.status.tsv.gz
    exists: YES
    non-empty: YES

multisample.qsee.vis.data.tsv.gz
    exists: YES
    non-empty: YES

multisample.qsee.vis.report.pdf
    exists: YES
    non-empty: YES
```

The status table contains all 23 tumor IDs.

# 20. Status table structure

Current `multisample.qsee.status.tsv.gz` contains columns:

```text
SampleId
SampleType
SourceTool
FeatureType
FeatureName
FeatureValue
QcStatus
FailCondition
```

This is the preferred file for downstream QC summary and comparison with DRAGEN QC.

Example R import:

```r
qsee_status <- read.delim(
  gzfile(
    "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/QSEE_output/multisample.qsee.status.tsv.gz"
  ),
  check.names=FALSE,
  stringsAsFactors=FALSE
)

table(
  qsee_status$QcStatus,
  useNA="ifany"
)
```

# 21. Extract WARN/FAIL calls

```r
library(dplyr)

qsee_problem <- qsee_status %>%
  filter(
    QcStatus %in% c(
      "WARN",
      "FAIL"
    )
  ) %>%
  arrange(
    SampleId,
    QcStatus,
    SourceTool,
    FeatureType,
    FeatureName
  )

print(
  qsee_problem,
  row.names=FALSE
)

write.table(
  qsee_problem,
  "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/14_QSEE/QSEE_WARN_FAIL.tsv",
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)
```

# 22. QSEE interpretation

QSEE flags metrics using:

```text
PASS
WARN
FAIL
```

Overall behavior:

```text
Any FAIL  -> overall FAIL
No FAIL but any WARN -> overall WARN
Otherwise -> PASS
```

Examples of visual/QC categories include:

```text
coverage
mapping summary
fragment length
duplicate frequency
GC bias
discordant fragment frequency
BQR
microsatellite indel error
copy number / PURPLE-derived QC
```

# 23. Comparing QSEE with DRAGEN QC

The next recommended analysis is not to compare only the overall QSEE status. Instead, map each WARN/FAIL to the corresponding raw DRAGEN metric.

Example logic:

```text
QSEE Mean coverage WARN/FAIL
    ↔ DRAGEN WGS average coverage

QSEE coverage >20x / >30x / etc.
    ↔ DRAGEN coverage distribution

QSEE duplicate-related abnormality
    ↔ DRAGEN duplicate marked reads %

QSEE mapping-related abnormality
    ↔ DRAGEN mapping metrics

QSEE purity warning
    ↔ PURPLE purity and, where appropriate, DRAGEN tumor metrics
```

This comparison should be performed feature-by-feature rather than assuming QSEE and DRAGEN use identical thresholds.

# 24. Troubleshooting record

## 24.1 Old PURPLE path ambiguity

Problem:

```text
PURPLE_ROOT = 10_Purple
```

was too broad after several PURPLE runs existed.

Fix:

```text
PURPLE_ROOT =
10_Purple/Purple_rerun
```

and sample IDs are derived only from this directory.

## 24.2 COBALT path after PURPLE rename

Current COBALT remains under:

```text
10_Purple/Amber_Cobalt_OldPurple
```

The final QSEE script explicitly verifies that any COBALT symlink comes from this root.

## 24.3 Gene coverage missing

Observed:

```text
23 / 23 bam_metric.gene_coverage.tsv missing
```

This is optional and does not prevent QSEE from running with `-allow_missing_input`.

## 24.4 False "0 / 23 status files" warning

An earlier validator incorrectly searched for:

```text
<SAMPLE>.qsee.status.tsv
```

The actual multi-sample output is:

```text
multisample.qsee.status.tsv.gz
```

Fix:

```text
validate the three multisample outputs
and confirm all 23 SampleId values are present in the status table
```

# 25. Provenance requirements

For every future QSEE production run, keep:

```text
1. QSEE version/tag
2. exact QSEE JAR path
3. QSEE JAR SHA256
4. Java version
5. DriverGenePanel path
6. DriverGenePanel/resource bundle version
7. exact sample list
8. exact source roots
9. QSEE_input_manifest.tsv
10. exact resolved command
11. .command.txt
12. program stdout/stderr log
13. full console log
14. exit code
15. runtime
16. output validation
17. sample-ID validation
18. R sessionInfo()
19. executed Rmd/notebook
```

A tool exit code of `0` is necessary but should not be the only completion criterion.

# 26. Final project status

Final validated state:

```text
QSEE tool preflight               PASS
Final PURPLE source               PASS
COBALT source                     PASS
Sample count                      23/23
Mandatory inputs                  PASS
Optional gene coverage            missing 23/23 (accepted)
QSEE process                      exit 0
Multi-sample output files         3/3
Samples found in status table     23/23
Final status                      COMPLETED
```

# 27. Exporting this notebook

This document is an R Markdown notebook.

Because all bioinformatics code chunks are:

```text
eval=FALSE
```

exporting does not execute QSEE.

## HTML

```r
rmarkdown::render(
  "QSEE_v1.0_UCaTS_tumor_only_complete_workflow.Rmd",
  output_format="html_document"
)
```

## PDF

```r
rmarkdown::render(
  "QSEE_v1.0_UCaTS_tumor_only_complete_workflow.Rmd",
  output_format="pdf_document"
)
```

PDF output requires a working LaTeX installation.

## Plain Markdown

A `.md` copy can be stored with the lab repository for direct browsing in GitHub or VS Code.

# 28. Reference links

QSEE:

```text
https://github.com/hartwigmedical/hmftools/tree/master/qsee
```

HMFtools releases:

```text
https://github.com/hartwigmedical/hmftools/releases
```

HMFtools resource documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/pipeline/README_RESOURCES.md
```

Oncoanalyser 3.0.0 reference-data documentation:

```text
https://nf-co.re/oncoanalyser/3.0.0/docs/usage/
```

BamMetrics:

```text
https://github.com/hartwigmedical/hmftools/tree/master/bam-tools
```

PURPLE:

```text
https://github.com/hartwigmedical/hmftools/tree/master/purple
```

REDUX:

```text
https://github.com/hartwigmedical/hmftools/tree/master/redux
```

ESVEE:

```text
https://github.com/hartwigmedical/hmftools/tree/master/esvee
```
