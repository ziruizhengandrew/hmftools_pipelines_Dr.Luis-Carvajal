# LINX v2.3.1 Complete Workflow — UCaTS Organoids

> Complete lab-facing documentation from software/reference setup through the final production R code.  
> Current formal input: `10_Purple/Purple_rerun`  
> Current formal output: `11_LINX/LINX_rerun`

## 1. Purpose

LINX annotates, clusters, chains, and interprets somatic structural variants, including fusion and disruption calling. For this project it consumes the latest PURPLE tumor-only WGS results for 23 UCaTS Organoid samples.

```text
PURPLE_rerun
   ├── <SAMPLE>.purple.sv.vcf.gz
   ├── purity / CNV data
   └── other PURPLE outputs
             |
             v
         LINX v2.3.1
             |
             ├── structural-variant interpretation
             ├── fusion calls
             ├── driver/disruption annotation
             └── downstream ORANGE inputs
```

Official LINX documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/linx/README.md
```

Official HMFtools releases:

```text
https://github.com/hartwigmedical/hmftools/releases
```

## 2. Software

Current project JAR:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/11_LINX/LINX_tools/linx_v2.3.1.jar
```

Current validated SHA256:

```text
d2dc8659dfcd1d4cc84267a6574953045bed75910191eac28345557d5d001c7d
```

Java:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Check:

```bash
JAVA="/home/zzr123/.conda/envs/purple/bin/java"
JAR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/11_LINX/LINX_tools/linx_v2.3.1.jar"

"$JAVA" -version
ls -lh "$JAR"
sha256sum "$JAR"
```

## 3. Fresh LINX installation

Use the explicit release tag `linx-v2.3.1` rather than an unspecified "latest" version.

```bash
set -euo pipefail

TOOL_DIR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/11_LINX/LINX_tools"
TAG="linx-v2.3.1"

mkdir -p "$TOOL_DIR"
cd "$TOOL_DIR"

ASSET_URL="$(
python3 - "$TAG" <<'PY'
import json, sys, urllib.request
tag = sys.argv[1]
url = f"https://api.github.com/repos/hartwigmedical/hmftools/releases/tags/{tag}"

with urllib.request.urlopen(url) as r:
    release = json.load(r)

jars = [
    a["browser_download_url"]
    for a in release["assets"]
    if a["name"].endswith(".jar") and "linx" in a["name"].lower()
]

if len(jars) != 1:
    raise SystemExit(f"Expected exactly one LINX JAR asset, found: {jars}")

print(jars[0])
PY
)"

echo "$ASSET_URL"

curl -L --fail \
"$ASSET_URL" \
-o "$TOOL_DIR/linx_v2.3.1.jar"

sha256sum "$TOOL_DIR/linx_v2.3.1.jar"
```

## 4. Reference bundle

LINX uses the shared HMFtools GRCh38 resource bundle:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
```

Required LINX resources:

```text
common/DriverGenePanel.38.tsv
common/ensembl_data/
dna/sv/known_fusion_data.38.csv
```

The shared bundle can be staged with:

```bash
set -euo pipefail

REF_BASE="/quobyte/luisccgrp/REFERENCE_DATA/hmftools"
BUNDLE="hmf_pipeline_resources.38_v3.0.0--8.tar.gz"

mkdir -p "$REF_BASE"
cd "$REF_BASE"

wget -c \
"https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/$BUNDLE"

tar -xzvf "$BUNDLE"
```

Validate the files:

```bash
ROOT="/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"

ls -lh "$ROOT/common/DriverGenePanel.38.tsv"
ls -lh "$ROOT/dna/sv/known_fusion_data.38.csv"
ls -lh "$ROOT/common/ensembl_data/ensembl_gene_data.csv"
ls -lh "$ROOT/common/ensembl_data/ensembl_trans_exon_data.csv"
ls -lh "$ROOT/common/ensembl_data/ensembl_trans_splice_data.csv"
ls -lh "$ROOT/common/ensembl_data/ensembl_protein_features.csv"
```

Do not create a second private copy of this bundle under `11_LINX`; the final workflow uses the shared reference directly.

## 5. Current project directories

```text
Input PURPLE:
10_Purple/Purple_rerun/<SAMPLE>/purple/

Main LINX input:
<SAMPLE>.purple.sv.vcf.gz

Output:
11_LINX/LINX_rerun/<SAMPLE>/

Logs:
11_LINX/logs/<SAMPLE>/

Exact command records:
11_LINX/commands/<SAMPLE>/

Input manifest:
11_LINX/LINX_rerun_input_manifest.tsv
```

Do not use:

```text
10_Purple/Purple_output
11_LINX/LINX_output
```

Those are historical outputs.

## 6. Important output-naming note

Depending on how the LINX output is surfaced, the core result names may appear as pipeline-style:

```text
<SAMPLE>.linx.svs.tsv
<SAMPLE>.linx.clusters.tsv
<SAMPLE>.linx.fusion.tsv
```

or standalone-style:

```text
<SAMPLE>.svs.tsv
<SAMPLE>.clusters.tsv
<SAMPLE>.fusions.tsv
```

The final validator below accepts both naming styles.

## 7. Final complete R code

```r
# ============================================================
# LINX v2.3.1 | FINAL RERUN
# UCaTS Organoids WGS | GRCh38 | 23 tumor samples
# ============================================================

main <- function(){

options(warn=1)

BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"
LINX_BASE <- file.path(BASE,"11_LINX")
TOOL_DIR <- file.path(LINX_BASE,"LINX_tools")
OUTPUT_BASE <- file.path(LINX_BASE,"LINX_rerun")
LOG_ROOT <- file.path(LINX_BASE,"logs")
COMMAND_ROOT <- file.path(LINX_BASE,"commands")
PURPLE_BASE <- file.path(BASE,"10_Purple","Purple_rerun")
HMF_REF <- "/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"
JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"
LINX_VERSION <- "2.3.1"
LINX_JAR <- file.path(TOOL_DIR,"linx_v2.3.1.jar")

for(x in c(LINX_BASE,TOOL_DIR,OUTPUT_BASE,LOG_ROOT,COMMAND_ROOT)){
  dir.create(x,recursive=TRUE,showWarnings=FALSE)
}

valid_file <- function(x){
  length(x)==1 && !is.na(x) && nzchar(x) &&
    file.exists(x) && !is.na(file.info(x)$size) &&
    file.info(x)$size>0
}

sha256_file <- function(x){
  if(!valid_file(x)) return(NA_character_)
  y <- suppressWarnings(system2("sha256sum",x,stdout=TRUE,stderr=TRUE))
  if(length(y)==0) return(NA_character_)
  strsplit(y[1],"\\s+")[[1]][1]
}

command_string <- function(exe,args){
  paste(shQuote(exe),paste(shQuote(args),collapse=" "))
}

find_reference <- function(root,pattern){
  x <- list.files(root,pattern=pattern,recursive=TRUE,full.names=TRUE)
  if(length(x)==0) stop("Reference not found: ",pattern,"\nRoot: ",root)
  if(length(x)>1){
    print(x)
    stop("Reference match is ambiguous: ",pattern)
  }
  x[1]
}

resolve_output <- function(sample,out_dir,type){
  if(!dir.exists(out_dir)) return(NA_character_)

  all_files <- list.files(
    out_dir,
    recursive=TRUE,
    full.names=TRUE
  )

  if(type=="SVS"){
    candidates <- c(
      paste0(sample,".linx.svs.tsv"),
      paste0(sample,".svs.tsv")
    )
  } else if(type=="CLUSTERS"){
    candidates <- c(
      paste0(sample,".linx.clusters.tsv"),
      paste0(sample,".clusters.tsv")
    )
  } else if(type=="FUSIONS"){
    candidates <- c(
      paste0(sample,".linx.fusion.tsv"),
      paste0(sample,".linx.fusions.tsv"),
      paste0(sample,".fusion.tsv"),
      paste0(sample,".fusions.tsv")
    )
  } else {
    stop("Unknown LINX output type: ",type)
  }

  hit <- all_files[
    basename(all_files) %in% candidates
  ]

  if(length(hit)==0) return(NA_character_)

  if(length(hit)>1){
    hit <- hit[
      order(
        file.info(hit)$mtime,
        decreasing=TRUE
      )
    ]
  }

  hit[1]
}

resolve_linx_outputs <- function(sample,out_dir){

  files <- c(
    SVS=resolve_output(sample,out_dir,"SVS"),
    CLUSTERS=resolve_output(sample,out_dir,"CLUSTERS"),
    FUSIONS=resolve_output(sample,out_dir,"FUSIONS")
  )

  valid <- vapply(
    files,
    valid_file,
    logical(1)
  )

  size <- vapply(
    files,
    function(x){
      if(valid_file(x)){
        file.info(x)$size
      } else {
        NA_real_
      }
    },
    numeric(1)
  )

  data.frame(
    Type=names(files),
    File=unname(files),
    Exists=!is.na(files) &
      file.exists(
        ifelse(is.na(files),"",files)
      ),
    Size_bytes=size,
    Valid=valid,
    stringsAsFactors=FALSE
  )
}

console_log <- file.path(
  LOG_ROOT,
  paste0(
    "LINX_console_",
    format(Sys.time(),"%Y%m%d_%H%M%S"),
    ".log"
  )
)

sink(console_log,split=TRUE)
run_status <- "INTERRUPTED_OR_FAILED"

on.exit({
  cat("\n====================================================\n")
  cat("LINX RUN END\n")
  cat("STATUS:",run_status,"\n")
  cat("TIME:",format(Sys.time(),"%Y-%m-%d %H:%M:%S %Z"),"\n")
  cat("Console log:",console_log,"\n")
  cat("====================================================\n")

  while(sink.number(type="output")>0){
    sink(type="output")
  }
},add=TRUE)

if(!valid_file(JAVA)) stop("Java not found: ",JAVA)
if(!valid_file(LINX_JAR)) stop("LINX JAR not found: ",LINX_JAR)
if(!dir.exists(HMF_REF)) stop("HMF resource directory not found: ",HMF_REF)
if(!dir.exists(PURPLE_BASE)) stop("PURPLE directory not found: ",PURPLE_BASE)

LINX_SHA256 <- sha256_file(LINX_JAR)

cat("\n================ LINX TOOL ==========================\n")
cat("LINX version:",LINX_VERSION,"\n")
cat("LINX JAR:",LINX_JAR,"\n")
cat("LINX SHA256:",LINX_SHA256,"\n")

DRIVER_GENE_PANEL <- find_reference(
  HMF_REF,
  "^DriverGenePanel\\.38\\.tsv$"
)

KNOWN_FUSION_FILE <- find_reference(
  HMF_REF,
  "^known_fusion_data\\.38\\.csv$"
)

ENSEMBL_DATA_DIR <- file.path(
  HMF_REF,
  "common",
  "ensembl_data"
)

required_ensembl <- c(
  "ensembl_gene_data.csv",
  "ensembl_trans_exon_data.csv",
  "ensembl_trans_splice_data.csv",
  "ensembl_protein_features.csv"
)

reference_check <- data.frame(
  Resource=c(
    "LINX JAR",
    "DriverGenePanel",
    "KnownFusion",
    required_ensembl
  ),
  Path=c(
    LINX_JAR,
    DRIVER_GENE_PANEL,
    KNOWN_FUSION_FILE,
    file.path(
      ENSEMBL_DATA_DIR,
      required_ensembl
    )
  ),
  stringsAsFactors=FALSE
)

reference_check$Exists <- file.exists(
  reference_check$Path
)

print(reference_check,row.names=FALSE)

if(!all(reference_check$Exists)){
  stop("LINX reference preflight failed.")
}

purity_files <- list.files(
  PURPLE_BASE,
  pattern="\\.purple\\.purity\\.tsv$",
  recursive=TRUE,
  full.names=TRUE
)

if(length(purity_files)==0){
  stop("No PURPLE purity files found.")
}

sample_table <- data.frame(
  sample=sub(
    "\\.purple\\.purity\\.tsv$",
    "",
    basename(purity_files)
  ),
  purple_dir=dirname(purity_files),
  stringsAsFactors=FALSE
)

sample_table <- sample_table[
  !duplicated(sample_table$sample),
  ,
  drop=FALSE
]

sample_table <- sample_table[
  order(sample_table$sample),
  ,
  drop=FALSE
]

if(nrow(sample_table)!=23L){
  stop(
    "Expected 23 samples but found ",
    nrow(sample_table)
  )
}

sample_table$purity_file <- mapply(
  function(sample,purple_dir){
    file.path(
      purple_dir,
      paste0(sample,".purple.purity.tsv")
    )
  },
  sample_table$sample,
  sample_table$purple_dir,
  USE.NAMES=FALSE
)

sample_table$sv_vcf <- mapply(
  function(sample,purple_dir){
    file.path(
      purple_dir,
      paste0(sample,".purple.sv.vcf.gz")
    )
  },
  sample_table$sample,
  sample_table$purple_dir,
  USE.NAMES=FALSE
)

sample_table$cnv_file <- mapply(
  function(sample,purple_dir){
    file.path(
      purple_dir,
      paste0(sample,".purple.cnv.somatic.tsv")
    )
  },
  sample_table$sample,
  sample_table$purple_dir,
  USE.NAMES=FALSE
)

sample_table$purity_ok <- vapply(
  sample_table$purity_file,
  valid_file,
  logical(1)
)

sample_table$sv_vcf_ok <- vapply(
  sample_table$sv_vcf,
  valid_file,
  logical(1)
)

sample_table$cnv_ok <- vapply(
  sample_table$cnv_file,
  valid_file,
  logical(1)
)

if(!all(
  sample_table$purity_ok &
  sample_table$sv_vcf_ok &
  sample_table$cnv_ok
)){
  stop("At least one final PURPLE input is incomplete.")
}

wrong_source <- !startsWith(
  sample_table$purple_dir,
  paste0(PURPLE_BASE,"/")
)

if(any(wrong_source)){
  stop("A LINX input came from outside Purple_rerun.")
}

manifest_file <- file.path(
  LINX_BASE,
  "LINX_rerun_input_manifest.tsv"
)

write.table(
  sample_table,
  manifest_file,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

run_linx <- function(sample,purple_dir,sv_vcf){

  cat("\n====================================================\n")
  cat("LINX SAMPLE:",sample,"\n")
  cat("====================================================\n")

  out_dir <- file.path(OUTPUT_BASE,sample)
  log_dir <- file.path(LOG_ROOT,sample)
  command_dir <- file.path(COMMAND_ROOT,sample)

  dir.create(out_dir,recursive=TRUE,showWarnings=FALSE)
  dir.create(log_dir,recursive=TRUE,showWarnings=FALSE)
  dir.create(command_dir,recursive=TRUE,showWarnings=FALSE)

  validation_file <- file.path(
    out_dir,
    paste0(sample,".LINX.output_validation.tsv")
  )

  existing_validation <- resolve_linx_outputs(
    sample,
    out_dir
  )

  write.table(
    existing_validation,
    validation_file,
    sep="\t",
    quote=FALSE,
    row.names=FALSE
  )

  if(all(existing_validation$Valid)){

    cat("[SKIP] LINX outputs already complete.\n")

    return(
      data.frame(
        Sample=sample,
        Status="SKIPPED_COMPLETED",
        Exit_code=0L,
        Runtime_min=0,
        Log=NA_character_,
        Command=NA_character_,
        stringsAsFactors=FALSE
      )
    )
  }

  success_marker <- file.path(
    out_dir,
    ".LINX_SUCCESS"
  )

  if(file.exists(success_marker)){
    unlink(success_marker)
  }

  args <- c(
    "-Xmx16G",
    "-jar",LINX_JAR,
    "-sample",sample,
    "-ref_genome_version","38",
    "-sv_vcf",sv_vcf,
    "-purple_dir",purple_dir,
    "-output_dir",out_dir,
    "-ensembl_data_dir",ENSEMBL_DATA_DIR,
    "-known_fusion_file",KNOWN_FUSION_FILE,
    "-driver_gene_panel",DRIVER_GENE_PANEL
  )

  exact_command <- command_string(
    JAVA,
    args
  )

  timestamp <- format(
    Sys.time(),
    "%Y%m%d_%H%M%S"
  )

  command_file <- file.path(
    command_dir,
    paste0(
      sample,
      ".LINX.",
      timestamp,
      ".command.txt"
    )
  )

  log_file <- file.path(
    log_dir,
    paste0(
      sample,
      ".LINX.",
      timestamp,
      ".log"
    )
  )

  writeLines(
    c(
      paste0(
        "DATE=",
        format(Sys.time(),"%Y-%m-%d %H:%M:%S %Z")
      ),
      paste0("SAMPLE=",sample),
      paste0("TOOL=LINX v",LINX_VERSION),
      paste0("LINX_JAR=",LINX_JAR),
      paste0("LINX_JAR_SHA256=",LINX_SHA256),
      paste0("PURPLE_DIR=",purple_dir),
      paste0("SV_VCF=",sv_vcf),
      paste0("HMF_RESOURCE_ROOT=",HMF_REF),
      paste0("DRIVER_GENE_PANEL=",DRIVER_GENE_PANEL),
      paste0("KNOWN_FUSION_FILE=",KNOWN_FUSION_FILE),
      paste0("ENSEMBL_DATA_DIR=",ENSEMBL_DATA_DIR),
      "",
      "ACTUAL_COMMAND:",
      exact_command
    ),
    command_file
  )

  cat("\nACTUAL COMMAND:\n")
  cat(exact_command,"\n")

  start_time <- Sys.time()

  exit_code <- system2(
    JAVA,
    args=args,
    stdout=log_file,
    stderr=log_file
  )

  runtime_min <- round(
    as.numeric(
      difftime(
        Sys.time(),
        start_time,
        units="mins"
      )
    ),
    2
  )

  validation <- resolve_linx_outputs(
    sample,
    out_dir
  )

  write.table(
    validation,
    validation_file,
    sep="\t",
    quote=FALSE,
    row.names=FALSE
  )

  if(exit_code==0 && all(validation$Valid)){

    writeLines(
      c(
        paste0("Sample=",sample),
        paste0(
          "Completed=",
          format(Sys.time(),"%Y-%m-%d %H:%M:%S %Z")
        ),
        paste0("LINX_version=",LINX_VERSION),
        paste0("LINX_JAR_SHA256=",LINX_SHA256),
        paste0("PURPLE_DIR=",purple_dir),
        paste0("SV_VCF=",sv_vcf)
      ),
      success_marker
    )

    final_status <- "COMPLETED"

  } else {

    final_status <- "FAILED"

    if(file.exists(log_file)){
      z <- readLines(
        log_file,
        warn=FALSE
      )
      cat(
        tail(z,80),
        sep="\n"
      )
    }
  }

  data.frame(
    Sample=sample,
    Status=final_status,
    Exit_code=exit_code,
    Runtime_min=runtime_min,
    Log=log_file,
    Command=command_file,
    stringsAsFactors=FALSE
  )
}

test_row <- sample_table[
  1,
  ,
  drop=FALSE
]

test_result <- run_linx(
  sample=test_row$sample,
  purple_dir=test_row$purple_dir,
  sv_vcf=test_row$sv_vcf
)

if(
  !test_result$Status %in%
  c("COMPLETED","SKIPPED_COMPLETED")
){
  stop(
    "FIRST SAMPLE LINX TEST FAILED. ",
    "Remaining samples were NOT started."
  )
}

results <- vector(
  "list",
  nrow(sample_table)
)

for(i in seq_len(nrow(sample_table))){

  row <- sample_table[
    i,
    ,
    drop=FALSE
  ]

  results[[i]] <- run_linx(
    sample=row$sample,
    purple_dir=row$purple_dir,
    sv_vcf=row$sv_vcf
  )

  if(
    !results[[i]]$Status %in%
    c("COMPLETED","SKIPPED_COMPLETED")
  ){
    stop(
      "LINX failed for ",
      row$sample,
      ". Remaining samples were NOT started."
    )
  }
}

run_summary <- do.call(
  rbind,
  results
)

summary_file <- file.path(
  OUTPUT_BASE,
  "LINX_rerun_summary.tsv"
)

write.table(
  run_summary,
  summary_file,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

final_validation <- do.call(
  rbind,
  lapply(
    sample_table$sample,
    function(sample){

      out_dir <- file.path(
        OUTPUT_BASE,
        sample
      )

      x <- resolve_linx_outputs(
        sample,
        out_dir
      )

      x$Sample <- sample

      x[
        ,
        c(
          "Sample",
          "Type",
          "File",
          "Exists",
          "Size_bytes",
          "Valid"
        )
      ]
    }
  )
)

final_validation_file <- file.path(
  OUTPUT_BASE,
  "LINX_23_samples_output_validation.tsv"
)

write.table(
  final_validation,
  final_validation_file,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

sample_validation <- aggregate(
  Valid ~ Sample,
  data=final_validation,
  FUN=all
)

if(!all(sample_validation$Valid)){
  stop(
    "At least one sample failed final LINX validation."
  )
}

session_file <- file.path(
  LINX_BASE,
  paste0(
    "LINX_sessionInfo_",
    format(Sys.time(),"%Y%m%d_%H%M%S"),
    ".txt"
  )
)

writeLines(
  capture.output(sessionInfo()),
  session_file
)

run_status <- "COMPLETED"

cat("\n====================================================\n")
cat("LINX FINAL SUMMARY\n")
cat("LINX version     :",LINX_VERSION,"\n")
cat("LINX JAR         :",LINX_JAR,"\n")
cat("LINX SHA256      :",LINX_SHA256,"\n")
cat("PURPLE source    :",PURPLE_BASE,"\n")
cat("Samples complete :",sum(sample_validation$Valid),"/23\n")
cat("Output           :",OUTPUT_BASE,"\n")
cat("FINAL STATUS     :",run_status,"\n")
cat("====================================================\n")
}

main()
```

## 8. Downstream note

The latest downstream ORANGE workflow should use:

```text
11_LINX/LINX_rerun
```

not historical `11_LINX/LINX_output`.
