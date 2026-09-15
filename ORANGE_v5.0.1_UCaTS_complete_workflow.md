# ORANGE v5.0.1 Complete Workflow — UCaTS Organoids

> Complete lab-facing documentation from tool/reference setup through the final production R code.  
> Current formal PURPLE input: `10_Purple/Purple_rerun`  
> Current formal LINX input: `11_LINX/LINX_rerun`  
> Current ORANGE output: `16_Orange/Orange_rerun`

## 1. Purpose

ORANGE integrates upstream HMFtools outputs into a static PDF report and a JSON file.

For this project the run is **tumor-only WGS**:

```text
Formal PURPLE data  ----\
Verified PURPLE plots ----> ORANGE v5.0.1 --> PDF + JSON
Latest LINX data     ----/
Optional QSEE/Virus -/
```

The workflow deliberately does not use a matched normal, CUPPA, or CHORD inside this tumor-only ORANGE configuration.

Official ORANGE documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/orange/README.md
```

Official HMFtools releases:

```text
https://github.com/hartwigmedical/hmftools/releases
```

## 2. ORANGE software

Current project JAR:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/16_Orange/Orange_tools/orange_v5.0.1.jar
```

Current validated SHA256:

```text
2f64189b8680c62bec6e7c72cecc3cda51a9eb56950808f349cee7c6de666987
```

Java:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Check:

```bash
JAVA="/home/zzr123/.conda/envs/purple/bin/java"
JAR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/16_Orange/Orange_tools/orange_v5.0.1.jar"

"$JAVA" -version
ls -lh "$JAR"
sha256sum "$JAR"
```

## 3. Fresh ORANGE installation

Use the explicit release tag:

```text
orange-v5.0.1
```

A reproducible install method:

```bash
set -euo pipefail

TOOL_DIR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/16_Orange/Orange_tools"
TAG="orange-v5.0.1"

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
    if a["name"].endswith(".jar") and "orange" in a["name"].lower()
]

if len(jars) != 1:
    raise SystemExit(f"Expected exactly one ORANGE JAR asset, found: {jars}")

print(jars[0])
PY
)"

echo "$ASSET_URL"

curl -L --fail \
"$ASSET_URL" \
-o "$TOOL_DIR/orange_v5.0.1.jar"

sha256sum "$TOOL_DIR/orange_v5.0.1.jar"
```

## 4. ORANGE reference/resources

The current tumor-only ORANGE command does not directly require a reference FASTA. Instead, its main dependencies are the outputs from upstream tools.

HMFtools also defines optional ORANGE-specific resources in the shared resource bundle:

```text
cohort_percentiles.tsv
cohort_mapping.tsv
doid.json
```

These can enrich cohort/cancer-type reporting, but they are **not passed by the current project command**, so they are not required to reproduce the present run.

Shared HMF resource root:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
```

If that shared bundle is absent:

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

Official HMFtools resource documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/pipeline/README_RESOURCES.md
```

## 5. Current upstream directories

Formal biological PURPLE input:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/
10_Purple/Purple_rerun/<SAMPLE>/purple/
```

Latest LINX input:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/
11_LINX/LINX_rerun/<SAMPLE>/
```

ORANGE output:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/
16_Orange/Orange_rerun/<SAMPLE>/
```

Historical directories are deliberately not reused:

```text
10_Purple/Purple_output
11_LINX/LINX_output
16_Orange/Orange_output
```

## 6. PURPLE plot requirement

ORANGE v5.0.1 expects the following eight PURPLE plot files:

```text
<SAMPLE>.input.png
<SAMPLE>.circos.png
<SAMPLE>.copynumber.png
<SAMPLE>.somatic.clonality.png
<SAMPLE>.purity.range.png
<SAMPLE>.map.png
<SAMPLE>.somatic.png
<SAMPLE>.somatic.rainfall.png
```

The formal biological PURPLE result remains:

```text
10_Purple/Purple_rerun
```

Some formal PURPLE directories initially contained only five of the eight plots.

A critical detail is that:

```text
copyNumberPlots.R
```

does not generate the three somatic plots:

```text
somatic.clonality.png
somatic.png
somatic.rainfall.png
```

Those belong to PURPLE's somatic-variant charting workflow.

Therefore the current ORANGE workflow uses the already-created isolated chart-enabled PURPLE run:

```text
10_Purple/Purple_plot_regen_with_charts
```

as a **plot donor only**.

It is never used as the biological PURPLE input.

Before any plot is copied, the script verifies SHA256 equality of:

```text
<SAMPLE>.purple.purity.tsv
<SAMPLE>.purple.segment.tsv
<SAMPLE>.purple.cnv.somatic.tsv
```

between the formal and chart-regeneration PURPLE outputs.

Only if all three match are missing PNGs allowed to be copied into the formal PURPLE `plot/` directory.

## 7. LINX files required by ORANGE

The current ORANGE preflight explicitly checks:

```text
<SAMPLE>.linx.svs.tsv
<SAMPLE>.linx.breakend.tsv
<SAMPLE>.linx.fusion.tsv
<SAMPLE>.linx.drivers.tsv
<SAMPLE>.linx.driver.catalog.tsv
```

under:

```text
11_LINX/LINX_rerun/<SAMPLE>/
```

This is stricter and safer than locating LINX only from a single `linx.svs.tsv`.

## 8. QSEE behavior

The current QSEE workflow is multi-sample and writes:

```text
multisample.qsee.status.tsv.gz
multisample.qsee.vis.data.tsv.gz
multisample.qsee.vis.report.pdf
```

ORANGE is not forced to consume that multi-sample PDF.

The code only passes `-qsee_dir` if it finds a genuine per-sample:

```text
<SAMPLE>.qsee.vis.report.png
```

Otherwise QSEE remains an independent QC product.

## 9. VirusInterpreter behavior

VirusInterpreter is optional.

If the code finds:

```text
<SAMPLE>.virus.annotated.tsv
```

then its directory is passed via:

```text
-virus_dir
```

Otherwise VirusInterpreter is omitted.

## 10. CHORD and CUPPA

In this tumor-only ORANGE configuration the final command does not pass:

```text
-reference
-chord_dir
-cuppa_dir
```

CHORD remains an independent downstream HRD product:

```text
13_CHORD/CHORD_rerun
```

and is not required for the current tumor-only ORANGE report.

## 11. Final outputs

Per sample:

```text
<SAMPLE>.orange.pdf
<SAMPLE>.orange.json
```

Project-level:

```text
Orange_23_samples_status.csv
Orange_23_samples_output_validation.tsv
ORANGE_Purple_8_plots_validation.tsv
```

A sample is complete only when both PDF and JSON exist and are non-empty.

## 12. Final complete R code

```r
# ============================================================
# ORANGE v5.0.1 | FINAL TUMOR-ONLY RERUN
# UCaTS Organoids WGS | 23 tumor samples
# ============================================================

main <- function(){

options(warn=1)

BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"

ORANGE_BASE <- file.path(
  BASE,
  "16_Orange"
)

TOOL_DIR <- file.path(
  ORANGE_BASE,
  "Orange_tools"
)

OUT_ROOT <- file.path(
  ORANGE_BASE,
  "Orange_rerun"
)

LOG_ROOT <- file.path(
  ORANGE_BASE,
  "logs"
)

COMMAND_ROOT <- file.path(
  ORANGE_BASE,
  "commands"
)

PURPLE_ROOT <- file.path(
  BASE,
  "10_Purple",
  "Purple_rerun"
)

PURPLE_REGEN_ROOT <- file.path(
  BASE,
  "10_Purple",
  "Purple_plot_regen_with_charts"
)

LINX_ROOT <- file.path(
  BASE,
  "11_LINX",
  "LINX_rerun"
)

QSEE_ROOT <- file.path(
  BASE,
  "14_QSEE",
  "QSEE_output"
)

VIRUS_ROOT <- file.path(
  BASE,
  "15_Virus"
)

JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"

ORANGE_VERSION <- "5.0.1"

ORANGE_JAR <- file.path(
  TOOL_DIR,
  "orange_v5.0.1.jar"
)

for(x in c(
  ORANGE_BASE,
  TOOL_DIR,
  OUT_ROOT,
  LOG_ROOT,
  COMMAND_ROOT
)){
  dir.create(
    x,
    recursive=TRUE,
    showWarnings=FALSE
  )
}

valid_file <- function(x){
  length(x)==1 &&
    !is.na(x) &&
    nzchar(x) &&
    file.exists(x) &&
    !is.na(file.info(x)$size) &&
    file.info(x)$size>0
}

sha256_file <- function(x){
  if(!valid_file(x)){
    return(NA_character_)
  }

  z <- suppressWarnings(
    system2(
      "sha256sum",
      x,
      stdout=TRUE,
      stderr=TRUE
    )
  )

  if(length(z)==0){
    return(NA_character_)
  }

  strsplit(
    z[1],
    "\\s+"
  )[[1]][1]
}

command_string <- function(
  exe,
  args
){
  paste(
    shQuote(exe),
    paste(
      shQuote(args),
      collapse=" "
    )
  )
}

find_exact_recursive <- function(
  root,
  filename
){

  if(!dir.exists(root)){
    return(NA_character_)
  }

  x <- list.files(
    root,
    recursive=TRUE,
    full.names=TRUE
  )

  x <- x[
    basename(x)==filename
  ]

  if(length(x)==0){
    return(NA_character_)
  }

  if(length(x)>1){
    x <- x[
      order(
        file.info(x)$mtime,
        decreasing=TRUE
      )
    ]
  }

  x[1]
}

console_log <- file.path(
  LOG_ROOT,
  paste0(
    "ORANGE_console_",
    format(
      Sys.time(),
      "%Y%m%d_%H%M%S"
    ),
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
  cat("ORANGE RUN END\n")
  cat("STATUS:",run_status,"\n")
  cat(
    "TIME:",
    format(
      Sys.time(),
      "%Y-%m-%d %H:%M:%S %Z"
    ),
    "\n"
  )
  cat(
    "Console log:",
    console_log,
    "\n"
  )
  cat("====================================================\n")

  while(
    sink.number(type="output")>0
  ){
    sink(type="output")
  }
},add=TRUE)

cat("====================================================\n")
cat("ORANGE v5.0.1 FINAL TUMOR-ONLY RERUN\n")
cat(
  "START:",
  format(
    Sys.time(),
    "%Y-%m-%d %H:%M:%S %Z"
  ),
  "\n"
)
cat("PURPLE:",PURPLE_ROOT,"\n")
cat("PLOT DONOR:",PURPLE_REGEN_ROOT,"\n")
cat("LINX:",LINX_ROOT,"\n")
cat("OUTPUT:",OUT_ROOT,"\n")
cat("====================================================\n")

if(!valid_file(JAVA)){
  stop(
    "Java not found: ",
    JAVA
  )
}

if(!valid_file(ORANGE_JAR)){
  stop(
    "Orange JAR not found: ",
    ORANGE_JAR
  )
}

if(!dir.exists(PURPLE_ROOT)){
  stop(
    "Formal Purple_rerun not found: ",
    PURPLE_ROOT
  )
}

if(!dir.exists(PURPLE_REGEN_ROOT)){
  stop(
    "Purple chart-regeneration directory not found:\n",
    PURPLE_REGEN_ROOT
  )
}

if(!dir.exists(LINX_ROOT)){
  stop(
    "LINX_rerun not found:\n",
    LINX_ROOT,
    "\nRun/complete LINX_rerun first."
  )
}

ORANGE_SHA256 <- sha256_file(
  ORANGE_JAR
)

cat("\n================ ORANGE TOOL ========================\n")
cat("Orange version:",ORANGE_VERSION,"\n")
cat("Orange JAR:",ORANGE_JAR,"\n")
cat("Orange SHA256:",ORANGE_SHA256,"\n")

java_version <- system2(
  JAVA,
  "-version",
  stdout=TRUE,
  stderr=TRUE
)

cat("\n================ JAVA ===============================\n")
cat(
  paste(java_version,collapse="\n"),
  "\n"
)

purity_files <- list.files(
  PURPLE_ROOT,
  pattern="\\.purple\\.purity\\.tsv$",
  recursive=TRUE,
  full.names=TRUE
)

if(length(purity_files)==0){
  stop(
    "No final PURPLE purity files found."
  )
}

samples <- sort(
  unique(
    sub(
      "\\.purple\\.purity\\.tsv$",
      "",
      basename(purity_files)
    )
  )
)

if(length(samples)!=23L){
  stop(
    "Expected 23 samples but found ",
    length(samples)
  )
}

cat("\n================ SAMPLE LIST ========================\n")
cat("Samples:",length(samples),"\n")
print(samples)
cat(
  "Sample count check: PASS (23/23)\n"
)

required_purple_plots <- function(sample){
  paste0(
    sample,
    c(
      ".input.png",
      ".circos.png",
      ".copynumber.png",
      ".somatic.clonality.png",
      ".purity.range.png",
      ".map.png",
      ".somatic.png",
      ".somatic.rainfall.png"
    )
  )
}

check_purple_plots <- function(
  sample,
  plot_dir
){

  files <- file.path(
    plot_dir,
    required_purple_plots(sample)
  )

  data.frame(
    File=basename(files),
    Path=files,
    Exists=file.exists(files),
    Size_bytes=ifelse(
      file.exists(files),
      file.info(files)$size,
      NA_real_
    ),
    Valid=vapply(
      files,
      valid_file,
      logical(1)
    ),
    stringsAsFactors=FALSE
  )
}

check_regen_core_match <- function(sample){

  formal_dir <- file.path(
    PURPLE_ROOT,
    sample,
    "purple"
  )

  regen_dir <- file.path(
    PURPLE_REGEN_ROOT,
    sample,
    "purple"
  )

  suffixes <- c(
    ".purple.purity.tsv",
    ".purple.segment.tsv",
    ".purple.cnv.somatic.tsv"
  )

  formal <- file.path(
    formal_dir,
    paste0(
      sample,
      suffixes
    )
  )

  regen <- file.path(
    regen_dir,
    paste0(
      sample,
      suffixes
    )
  )

  result <- data.frame(
    Sample=sample,
    File=paste0(
      sample,
      suffixes
    ),
    Formal=formal,
    Regen=regen,
    Formal_SHA256=vapply(
      formal,
      sha256_file,
      character(1)
    ),
    Regen_SHA256=vapply(
      regen,
      sha256_file,
      character(1)
    ),
    stringsAsFactors=FALSE
  )

  result$Match <-
    !is.na(result$Formal_SHA256) &
    !is.na(result$Regen_SHA256) &
    result$Formal_SHA256==
      result$Regen_SHA256

  result
}

sync_purple_plots <- function(sample){

  formal_dir <- file.path(
    PURPLE_ROOT,
    sample,
    "purple"
  )

  formal_plot_dir <- file.path(
    formal_dir,
    "plot"
  )

  regen_dir <- file.path(
    PURPLE_REGEN_ROOT,
    sample,
    "purple"
  )

  regen_plot_dir <- file.path(
    regen_dir,
    "plot"
  )

  dir.create(
    formal_plot_dir,
    recursive=TRUE,
    showWarnings=FALSE
  )

  before <- check_purple_plots(
    sample,
    formal_plot_dir
  )

  cat(
    "\nFormal Purple plots BEFORE sync:",
    sum(before$Valid),
    "/8\n"
  )

  if(all(before$Valid)){
    return(before)
  }

  core_check <- check_regen_core_match(
    sample
  )

  cat(
    "\nPURPLE FORMAL vs REGEN CORE SHA256:\n"
  )

  print(
    core_check,
    row.names=FALSE
  )

  if(!all(core_check$Match)){

    cat(
      "\nERROR: Purple chart donor core does not match ",
      "formal Purple_rerun.\n",
      sep=""
    )

    return(before)
  }

  cat(
    "\nCore SHA256 match: PASS (3/3)\n"
  )

  missing_names <- before$File[
    !before$Valid
  ]

  for(filename in missing_names){

    source <- file.path(
      regen_plot_dir,
      filename
    )

    target <- file.path(
      formal_plot_dir,
      filename
    )

    if(valid_file(source)){

      ok <- file.copy(
        from=source,
        to=target,
        overwrite=TRUE
      )

      cat(
        ifelse(
          ok,
          "COPIED",
          "COPY_FAILED"
        ),
        ": ",
        filename,
        "\n",
        sep=""
      )

    } else {

      cat(
        "NOT FOUND IN REGEN: ",
        source,
        "\n",
        sep=""
      )
    }
  }

  after <- check_purple_plots(
    sample,
    formal_plot_dir
  )

  cat(
    "\nFormal Purple plots AFTER sync:",
    sum(after$Valid),
    "/8\n"
  )

  after
}

plot_validation_list <- vector(
  "list",
  length(samples)
)

for(i in seq_along(samples)){

  sample <- samples[i]

  cat(
    "\n####################################################\n",
    "PURPLE PLOT CHECK ",
    i,
    "/",
    length(samples),
    ": ",
    sample,
    "\n",
    "####################################################\n",
    sep=""
  )

  x <- sync_purple_plots(
    sample
  )

  x$Sample <- sample

  plot_validation_list[[i]] <- x[
    ,
    c(
      "Sample",
      "File",
      "Path",
      "Exists",
      "Size_bytes",
      "Valid"
    )
  ]
}

plot_validation <- do.call(
  rbind,
  plot_validation_list
)

plot_validation_file <- file.path(
  ORANGE_BASE,
  "ORANGE_Purple_8_plots_validation.tsv"
)

write.table(
  plot_validation,
  plot_validation_file,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

plot_sample_validation <- aggregate(
  Valid ~ Sample,
  data=plot_validation,
  FUN=all
)

cat(
  "\n================ PURPLE 8-PLOT VALIDATION ===========\n"
)

print(
  plot_sample_validation,
  row.names=FALSE
)

if(!all(plot_sample_validation$Valid)){

  bad <- plot_validation[
    !plot_validation$Valid,
    ,
    drop=FALSE
  ]

  cat(
    "\nPURPLE plots still missing:\n"
  )

  print(
    bad,
    row.names=FALSE
  )

  stop(
    "\nORANGE NOT STARTED.\n",
    "At least one sample does not have all 8 required PURPLE plots.\n",
    "Do NOT create placeholder PNGs."
  )
}

cat(
  "\nALL 23 SAMPLES HAVE 8/8 PURPLE PLOTS: PASS\n"
)

results <- data.frame(
  Sample=samples,
  Status=NA_character_,
  Purple_plots=NA_integer_,
  LINX=FALSE,
  QSEE=FALSE,
  Virus=FALSE,
  Runtime_min=NA_real_,
  PDF=NA_character_,
  JSON=NA_character_,
  Log=NA_character_,
  Command=NA_character_,
  stringsAsFactors=FALSE
)

run_orange <- function(sample){

  cat("\n====================================================\n")
  cat("ORANGE SAMPLE:",sample,"\n")
  cat("====================================================\n")

  PURPLE_DIR <- file.path(
    PURPLE_ROOT,
    sample,
    "purple"
  )

  PURPLE_PLOT_DIR <- file.path(
    PURPLE_DIR,
    "plot"
  )

  if(!dir.exists(PURPLE_DIR)){

    return(
      list(
        status="MISSING_PURPLE",
        plots=0L,
        linx=FALSE,
        qsee=FALSE,
        virus=FALSE,
        runtime=NA_real_,
        pdf=NA_character_,
        json=NA_character_,
        log=NA_character_,
        command=NA_character_
      )
    )
  }

  PURPLE_REQUIRED <- c(
    file.path(
      PURPLE_DIR,
      paste0(
        sample,
        ".purple.qc"
      )
    ),
    file.path(
      PURPLE_DIR,
      paste0(
        sample,
        ".purple.purity.tsv"
      )
    ),
    file.path(
      PURPLE_DIR,
      paste0(
        sample,
        ".purple.driver.catalog.somatic.tsv"
      )
    ),
    file.path(
      PURPLE_DIR,
      paste0(
        sample,
        ".purple.somatic.vcf.gz"
      )
    ),
    file.path(
      PURPLE_DIR,
      paste0(
        sample,
        ".purple.cnv.gene.tsv"
      )
    ),
    file.path(
      PURPLE_DIR,
      paste0(
        sample,
        ".purple.chromosome_arm.tsv"
      )
    )
  )

  purple_data_ok <- vapply(
    PURPLE_REQUIRED,
    valid_file,
    logical(1)
  )

  if(!all(purple_data_ok)){

    cat(
      "FAILED PRECHECK: Missing Purple biological data:\n"
    )

    cat(
      paste0(
        "  ",
        PURPLE_REQUIRED[
          !purple_data_ok
        ]
      ),
      sep="\n"
    )

    return(
      list(
        status="MISSING_PURPLE_DATA",
        plots=8L,
        linx=FALSE,
        qsee=FALSE,
        virus=FALSE,
        runtime=NA_real_,
        pdf=NA_character_,
        json=NA_character_,
        log=NA_character_,
        command=NA_character_
      )
    )
  }

  PLOT_CHECK <- check_purple_plots(
    sample,
    PURPLE_PLOT_DIR
  )

  if(!all(PLOT_CHECK$Valid)){

    return(
      list(
        status="MISSING_PURPLE_PLOTS",
        plots=sum(PLOT_CHECK$Valid),
        linx=FALSE,
        qsee=FALSE,
        virus=FALSE,
        runtime=NA_real_,
        pdf=NA_character_,
        json=NA_character_,
        log=NA_character_,
        command=NA_character_
      )
    )
  }

  cat(
    "PURPLE plots: 8/8 PASS\n"
  )

  LINX_DIR <- file.path(
    LINX_ROOT,
    sample
  )

  LINX_REQUIRED <- c(
    file.path(
      LINX_DIR,
      paste0(
        sample,
        ".linx.svs.tsv"
      )
    ),
    file.path(
      LINX_DIR,
      paste0(
        sample,
        ".linx.breakend.tsv"
      )
    ),
    file.path(
      LINX_DIR,
      paste0(
        sample,
        ".linx.fusion.tsv"
      )
    ),
    file.path(
      LINX_DIR,
      paste0(
        sample,
        ".linx.drivers.tsv"
      )
    ),
    file.path(
      LINX_DIR,
      paste0(
        sample,
        ".linx.driver.catalog.tsv"
      )
    )
  )

  linx_ok <- vapply(
    LINX_REQUIRED,
    valid_file,
    logical(1)
  )

  if(!all(linx_ok)){

    cat(
      "\nFAILED PRECHECK: Missing LINX_rerun data:\n"
    )

    cat(
      paste0(
        "  ",
        LINX_REQUIRED[
          !linx_ok
        ]
      ),
      sep="\n"
    )

    return(
      list(
        status="MISSING_LINX",
        plots=8L,
        linx=FALSE,
        qsee=FALSE,
        virus=FALSE,
        runtime=NA_real_,
        pdf=NA_character_,
        json=NA_character_,
        log=NA_character_,
        command=NA_character_
      )
    )
  }

  cat(
    "LINX data: PASS\n"
  )

  QSEE_PNG <- find_exact_recursive(
    QSEE_ROOT,
    paste0(
      sample,
      ".qsee.vis.report.png"
    )
  )

  QSEE_DIR <- if(
    valid_file(QSEE_PNG)
  ){
    dirname(
      QSEE_PNG
    )
  } else {
    NA_character_
  }

  VIRUS_FILE <- find_exact_recursive(
    VIRUS_ROOT,
    paste0(
      sample,
      ".virus.annotated.tsv"
    )
  )

  VIRUS_DIR <- if(
    valid_file(VIRUS_FILE)
  ){
    dirname(
      VIRUS_FILE
    )
  } else {
    NA_character_
  }

  cat(
    "QSEE:",
    ifelse(
      is.na(QSEE_DIR),
      "NOT INCLUDED",
      QSEE_DIR
    ),
    "\n"
  )

  cat(
    "Virus:",
    ifelse(
      is.na(VIRUS_DIR),
      "NOT INCLUDED",
      VIRUS_DIR
    ),
    "\n"
  )

  SAMPLE_OUT <- file.path(
    OUT_ROOT,
    sample
  )

  SAMPLE_LOG_DIR <- file.path(
    LOG_ROOT,
    sample
  )

  SAMPLE_COMMAND_DIR <- file.path(
    COMMAND_ROOT,
    sample
  )

  dir.create(
    SAMPLE_OUT,
    recursive=TRUE,
    showWarnings=FALSE
  )

  dir.create(
    SAMPLE_LOG_DIR,
    recursive=TRUE,
    showWarnings=FALSE
  )

  dir.create(
    SAMPLE_COMMAND_DIR,
    recursive=TRUE,
    showWarnings=FALSE
  )

  PDF_OUT <- file.path(
    SAMPLE_OUT,
    paste0(
      sample,
      ".orange.pdf"
    )
  )

  JSON_OUT <- file.path(
    SAMPLE_OUT,
    paste0(
      sample,
      ".orange.json"
    )
  )

  if(
    valid_file(PDF_OUT) &&
    valid_file(JSON_OUT)
  ){

    cat(
      "[SKIP] Orange already complete.\n"
    )

    return(
      list(
        status="SKIPPED_COMPLETE",
        plots=8L,
        linx=TRUE,
        qsee=!is.na(QSEE_DIR),
        virus=!is.na(VIRUS_DIR),
        runtime=0,
        pdf=PDF_OUT,
        json=JSON_OUT,
        log=NA_character_,
        command=NA_character_
      )
    )
  }

  args <- c(
    "-Xmx8G",
    "-jar",ORANGE_JAR,
    "-experiment_type","WGS",
    "-tumor",sample,
    "-ref_genome_version","38",
    "-sequencing_type","ILLUMINA",
    "-purple_dir",PURPLE_DIR,
    "-purple_plot_dir",PURPLE_PLOT_DIR,
    "-linx_dir",LINX_DIR,
    "-output_dir",SAMPLE_OUT,
    "-log_level","INFO",
    "-add_disclaimer"
  )

  if(!is.na(QSEE_DIR)){
    args <- c(
      args,
      "-qsee_dir",
      QSEE_DIR
    )
  }

  if(!is.na(VIRUS_DIR)){
    args <- c(
      args,
      "-virus_dir",
      VIRUS_DIR
    )
  }

  timestamp <- format(
    Sys.time(),
    "%Y%m%d_%H%M%S"
  )

  LOG_FILE <- file.path(
    SAMPLE_LOG_DIR,
    paste0(
      sample,
      ".ORANGE.",
      timestamp,
      ".log"
    )
  )

  COMMAND_FILE <- file.path(
    SAMPLE_COMMAND_DIR,
    paste0(
      sample,
      ".ORANGE.",
      timestamp,
      ".command.txt"
    )
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
      paste0("SAMPLE=",sample),
      paste0(
        "TOOL=ORANGE v",
        ORANGE_VERSION
      ),
      paste0(
        "ORANGE_JAR=",
        ORANGE_JAR
      ),
      paste0(
        "ORANGE_JAR_SHA256=",
        ORANGE_SHA256
      ),
      "MODE=TUMOR_ONLY_WGS",
      paste0(
        "PURPLE_DIR=",
        PURPLE_DIR
      ),
      paste0(
        "PURPLE_PLOT_DIR=",
        PURPLE_PLOT_DIR
      ),
      paste0(
        "LINX_DIR=",
        LINX_DIR
      ),
      paste0(
        "QSEE_DIR=",
        ifelse(
          is.na(QSEE_DIR),
          "NOT_INCLUDED",
          QSEE_DIR
        )
      ),
      paste0(
        "VIRUS_DIR=",
        ifelse(
          is.na(VIRUS_DIR),
          "NOT_INCLUDED",
          VIRUS_DIR
        )
      ),
      "CHORD_DIR=NOT_USED_TUMOR_ONLY",
      "CUPPA_DIR=NOT_USED_TUMOR_ONLY",
      "",
      "ACTUAL_COMMAND:",
      exact_command
    ),
    COMMAND_FILE
  )

  cat(
    "\nACTUAL COMMAND:\n",
    exact_command,
    "\n",
    sep=""
  )

  start_time <- Sys.time()

  exit_code <- system2(
    JAVA,
    args=args,
    stdout=LOG_FILE,
    stderr=LOG_FILE
  )

  runtime <- round(
    as.numeric(
      difftime(
        Sys.time(),
        start_time,
        units="mins"
      )
    ),
    2
  )

  pdf_ok <- valid_file(
    PDF_OUT
  )

  json_ok <- valid_file(
    JSON_OUT
  )

  validation <- data.frame(
    File=c(
      basename(PDF_OUT),
      basename(JSON_OUT)
    ),
    Exists=c(
      file.exists(PDF_OUT),
      file.exists(JSON_OUT)
    ),
    Size_bytes=c(
      ifelse(
        file.exists(PDF_OUT),
        file.info(PDF_OUT)$size,
        NA_real_
      ),
      ifelse(
        file.exists(JSON_OUT),
        file.info(JSON_OUT)$size,
        NA_real_
      )
    ),
    Valid=c(
      pdf_ok,
      json_ok
    ),
    stringsAsFactors=FALSE
  )

  write.table(
    validation,
    file.path(
      SAMPLE_OUT,
      paste0(
        sample,
        ".ORANGE.output_validation.tsv"
      )
    ),
    sep="\t",
    quote=FALSE,
    row.names=FALSE
  )

  if(
    exit_code==0 &&
    pdf_ok &&
    json_ok
  ){

    final_status <- "SUCCESS"

  } else {

    final_status <- "FAILED"

    if(file.exists(LOG_FILE)){
      log_text <- readLines(
        LOG_FILE,
        warn=FALSE
      )

      cat(
        tail(
          log_text,
          120
        ),
        sep="\n"
      )
    }
  }

  list(
    status=final_status,
    plots=8L,
    linx=TRUE,
    qsee=!is.na(QSEE_DIR),
    virus=!is.na(VIRUS_DIR),
    runtime=runtime,
    pdf=PDF_OUT,
    json=JSON_OUT,
    log=LOG_FILE,
    command=COMMAND_FILE
  )
}

update_result <- function(
  sample,
  x
){

  i <- which(
    results$Sample==sample
  )

  results$Status[i] <<- x$status
  results$Purple_plots[i] <<- x$plots
  results$LINX[i] <<- x$linx
  results$QSEE[i] <<- x$qsee
  results$Virus[i] <<- x$virus
  results$Runtime_min[i] <<- x$runtime
  results$PDF[i] <<- x$pdf
  results$JSON[i] <<- x$json
  results$Log[i] <<- x$log
  results$Command[i] <<- x$command
}

TEST_SAMPLE <- samples[1]

test_result <- run_orange(
  TEST_SAMPLE
)

update_result(
  TEST_SAMPLE,
  test_result
)

if(
  !test_result$status %in%
  c(
    "SUCCESS",
    "SKIPPED_COMPLETE"
  )
){
  stop(
    "FIRST SAMPLE ORANGE TEST FAILED. ",
    "Remaining samples were NOT started."
  )
}

remaining <- setdiff(
  samples,
  TEST_SAMPLE
)

for(i in seq_along(remaining)){

  sample <- remaining[i]

  x <- run_orange(
    sample
  )

  update_result(
    sample,
    x
  )

  write.csv(
    results,
    file.path(
      OUT_ROOT,
      "Orange_23_samples_status.csv"
    ),
    row.names=FALSE
  )

  if(
    !x$status %in%
    c(
      "SUCCESS",
      "SKIPPED_COMPLETE"
    )
  ){
    stop(
      "ORANGE failed for ",
      sample,
      ". Remaining samples were NOT started."
    )
  }
}

FINAL_VALIDATION <- do.call(
  rbind,
  lapply(
    samples,
    function(sample){

      sample_out <- file.path(
        OUT_ROOT,
        sample
      )

      pdf <- file.path(
        sample_out,
        paste0(
          sample,
          ".orange.pdf"
        )
      )

      json <- file.path(
        sample_out,
        paste0(
          sample,
          ".orange.json"
        )
      )

      data.frame(
        Sample=sample,
        PDF=pdf,
        PDF_valid=valid_file(pdf),
        JSON=json,
        JSON_valid=valid_file(json),
        Complete=
          valid_file(pdf) &&
          valid_file(json),
        stringsAsFactors=FALSE
      )
    }
  )
)

VALIDATION_FILE <- file.path(
  OUT_ROOT,
  "Orange_23_samples_output_validation.tsv"
)

write.table(
  FINAL_VALIDATION,
  VALIDATION_FILE,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

STATUS_FILE <- file.path(
  OUT_ROOT,
  "Orange_23_samples_status.csv"
)

write.csv(
  results,
  STATUS_FILE,
  row.names=FALSE
)

if(!all(FINAL_VALIDATION$Complete)){

  run_status <- "COMPLETED_WITH_FAILURES"

  stop(
    "At least one sample failed Orange PDF/JSON validation."
  )
}

SESSION_FILE <- file.path(
  ORANGE_BASE,
  paste0(
    "ORANGE_sessionInfo_",
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
  SESSION_FILE
)

run_status <- "COMPLETED"

cat("\n====================================================\n")
cat("ORANGE FINAL SUMMARY\n")
cat("Orange version     :",ORANGE_VERSION,"\n")
cat("Orange JAR         :",ORANGE_JAR,"\n")
cat("Orange SHA256      :",ORANGE_SHA256,"\n")
cat("Mode               : TUMOR-ONLY WGS\n")
cat("PURPLE data        :",PURPLE_ROOT,"\n")
cat("PURPLE plot donor  :",PURPLE_REGEN_ROOT,"\n")
cat("LINX               :",LINX_ROOT,"\n")
cat("CHORD              : NOT USED\n")
cat("CUPPA              : NOT USED\n")
cat("Samples            :",length(samples),"\n")
cat(
  "8 Purple plots    :",
  sum(plot_sample_validation$Valid),
  "/",
  length(samples),
  "\n"
)
cat(
  "PDF+JSON complete :",
  sum(FINAL_VALIDATION$Complete),
  "/",
  length(samples),
  "\n"
)
cat("Output             :",OUT_ROOT,"\n")
cat("FINAL STATUS       :",run_status,"\n")
cat("====================================================\n")
}

main()
```

## 13. Final directory map

```text
10_Purple/Purple_rerun
    formal biological PURPLE result

10_Purple/Purple_plot_regen_with_charts
    verified plot donor only

11_LINX/LINX_rerun
    latest LINX

13_CHORD/CHORD_rerun
    independent HRD result; not consumed by this tumor-only ORANGE command

14_QSEE/QSEE_output
    independent multi-sample QC; per-sample QSEE PNG used only if present

15_Virus
    optional VirusInterpreter

16_Orange/Orange_rerun
    latest ORANGE PDF + JSON
```
