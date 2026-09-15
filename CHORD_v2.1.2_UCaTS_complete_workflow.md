# CHORD v2.1.2 Complete Workflow — UCaTS Organoids

> Complete lab-facing documentation from reference setup through the final production R code.  
> Current formal input: `10_Purple/Purple_rerun`  
> Current formal output: `13_CHORD/CHORD_rerun`

## 1. Purpose

CHORD predicts homologous recombination deficiency (HRD) from somatic SNV/INDEL and structural-variant mutation contexts.

```text
PURPLE_rerun
   ├── <SAMPLE>.purple.somatic.vcf.gz
   └── <SAMPLE>.purple.sv.vcf.gz
               |
               v
          CHORD v2.1.2
               |
               ├── <SAMPLE>.chord.mutation_contexts.tsv
               └── <SAMPLE>.chord.prediction.tsv
```

The prediction file contains:

```text
p_BRCA1
p_BRCA2
p_hrd
hr_status
hrd_type
```

Official CHORD documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/chord/README.md
```

## 2. Version decision

The HMFtools landing page currently labels CHORD as `2.2`, but the project encountered a `404` for the GitHub release tag:

```text
chord-v2.2
```

Therefore the reproducible project workflow does **not** invent or auto-download a `2.2` JAR.

The final workflow pins the existing, previously tested:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/13_CHORD/CHORD_tools/chord_v2.1.2.jar
```

The exact JAR SHA256 is recorded at runtime.

## 3. Java and R requirements

Java:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Check:

```bash
/home/zzr123/.conda/envs/purple/bin/java -version
```

CHORD feature extraction is Java, but the random-forest prediction step requires R with `randomForest`.

Check:

```bash
Rscript -e "stopifnot(requireNamespace('randomForest', quietly=TRUE))"
```

If needed:

```r
install.packages("randomForest")
```

## 4. Reference genome

CHORD requires the same reference genome used to generate the input VCFs.

For this project:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

CHORD also requires the reference index and sequence dictionary.

Expected:

```text
hg38.fa
hg38.fa.fai
hg38.dict
```

or another compatible dictionary naming convention that resolves to the same FASTA.

Check:

```bash
REF="/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"

ls -lh "$REF"
ls -lh "$REF.fai"
ls -lh "${REF%.fa}.dict" 2>/dev/null || true
ls -lh "$REF.dict" 2>/dev/null || true

sha256sum "$REF"
```

If `.fai` is missing:

```bash
samtools faidx "$REF"
```

Do **not** replace this FASTA with a random hg38/GRCh38 download. The reference should match the one used by the upstream variant-calling workflow.

## 5. CHORD JAR validation

```bash
JAR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/13_CHORD/CHORD_tools/chord_v2.1.2.jar"
JAVA="/home/zzr123/.conda/envs/purple/bin/java"

ls -lh "$JAR"
sha256sum "$JAR"

"$JAVA" -jar "$JAR" -help
```

## 6. Current project directories

```text
Input PURPLE:
10_Purple/Purple_rerun/

SNV/INDEL:
<SAMPLE>.purple.somatic.vcf.gz

SV:
<SAMPLE>.purple.sv.vcf.gz

Output:
13_CHORD/CHORD_rerun/<SAMPLE>/

Logs:
13_CHORD/logs/<SAMPLE>/

Exact command records:
13_CHORD/commands/<SAMPLE>/

Input manifest:
13_CHORD/CHORD_rerun_input_manifest.tsv

Combined predictions:
13_CHORD/CHORD_rerun/CHORD_all_samples_predictions.tsv
13_CHORD/CHORD_rerun/CHORD_all_samples_predictions.csv
```

Do not use:

```text
10_Purple/Purple_output
13_CHORD/CHORD_output
```

Those are historical outputs.

## 7. Output interpretation

CHORD writes two main output files:

```text
<SAMPLE>.chord.mutation_contexts.tsv
<SAMPLE>.chord.prediction.tsv
```

The current official mutation-context format uses:

```text
sample_id
```

while the prediction table uses:

```text
sample
```

The final validator handles this difference.

CHORD can return:

```text
cannot_be_determined
```

for biological/QC reasons such as MSI or insufficient indel/SV counts. This is not the same thing as a program crash.

## 8. Resource / memory choice

The previous workflow used `48G` Java heap and one sample previously hit an out-of-memory condition. The rerun uses:

```text
-Xmx64G
```

The workflow runs one sample at a time with:

```text
-threads 1
```

This is intentional because CHORD threads parallelize samples rather than substantially accelerating one sample.

## 9. Final complete R code

```r
# ============================================================
# CHORD v2.1.2 | FINAL RERUN
# UCaTS Organoids WGS | GRCh38 | 23 tumor samples
# ============================================================

main <- function(){

options(warn=1)

BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"
CHORD_BASE <- file.path(BASE,"13_CHORD")
TOOL_DIR <- file.path(CHORD_BASE,"CHORD_tools")
OUTPUT_BASE <- file.path(CHORD_BASE,"CHORD_rerun")
LOG_ROOT <- file.path(CHORD_BASE,"logs")
COMMAND_ROOT <- file.path(CHORD_BASE,"commands")

PURPLE_BASE <- file.path(
  BASE,
  "10_Purple",
  "Purple_rerun"
)

REF_GENOME <- "/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"
JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"

CHORD_VERSION <- "2.1.2"

CHORD_JAR <- file.path(
  TOOL_DIR,
  "chord_v2.1.2.jar"
)

JAVA_MEMORY <- "64G"

for(x in c(
  CHORD_BASE,
  TOOL_DIR,
  OUTPUT_BASE,
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

command_string <- function(exe,args){
  paste(
    shQuote(exe),
    paste(
      shQuote(args),
      collapse=" "
    )
  )
}

console_log <- file.path(
  LOG_ROOT,
  paste0(
    "CHORD_console_",
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
  cat("CHORD RUN END\n")
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
cat("CHORD v2.1.2 FINAL RERUN\n")
cat(
  "START:",
  format(Sys.time(),"%Y-%m-%d %H:%M:%S %Z"),
  "\n"
)
cat("PURPLE:",PURPLE_BASE,"\n")
cat("OUTPUT:",OUTPUT_BASE,"\n")
cat("====================================================\n")

if(!valid_file(JAVA)){
  stop("Java not found: ",JAVA)
}

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

if(!valid_file(CHORD_JAR)){

  cat(
    "\nExpected CHORD JAR not found:\n",
    CHORD_JAR,
    "\n",
    sep=""
  )

  existing_chord <- list.files(
    TOOL_DIR,
    pattern="chord.*\\.jar$",
    full.names=TRUE,
    ignore.case=TRUE
  )

  if(length(existing_chord)>0){
    cat("\nOther CHORD JARs found:\n")
    print(existing_chord)
  }

  stop(
    "CHORD v2.1.2 JAR is required. ",
    "No automatic v2.2 download will be attempted."
  )
}

CHORD_SHA256 <- sha256_file(
  CHORD_JAR
)

cat("\n================ CHORD TOOL =========================\n")
cat("CHORD version:",CHORD_VERSION,"\n")
cat("CHORD JAR:",CHORD_JAR,"\n")
cat("CHORD SHA256:",CHORD_SHA256,"\n")

jar_help <- system2(
  JAVA,
  args=c(
    "-jar",
    CHORD_JAR,
    "-help"
  ),
  stdout=TRUE,
  stderr=TRUE
)

cat(
  "\nCHORD JAR startup check completed.\n"
)

if(!valid_file(REF_GENOME)){
  stop(
    "Reference FASTA not found: ",
    REF_GENOME
  )
}

REF_FAI <- paste0(
  REF_GENOME,
  ".fai"
)

if(!valid_file(REF_FAI)){
  stop(
    "Reference FASTA index not found: ",
    REF_FAI
  )
}

dict_candidates <- unique(
  c(
    sub(
      "\\.(fa|fasta)$",
      ".dict",
      REF_GENOME
    ),
    paste0(
      REF_GENOME,
      ".dict"
    )
  )
)

dict_existing <- dict_candidates[
  vapply(
    dict_candidates,
    valid_file,
    logical(1)
  )
]

if(length(dict_existing)==0){
  stop(
    "Reference sequence dictionary not found.\nChecked:\n",
    paste(
      dict_candidates,
      collapse="\n"
    )
  )
}

REF_DICT <- dict_existing[1]

cat("\n================ REFERENCE ==========================\n")
cat("FASTA:",REF_GENOME,"\n")
cat("FAI  :",REF_FAI,"\n")
cat("DICT :",REF_DICT,"\n")
cat(
  "FASTA SHA256:",
  sha256_file(REF_GENOME),
  "\n"
)

RSCRIPT <- Sys.which(
  "Rscript"
)

if(!nzchar(RSCRIPT)){
  stop(
    "Rscript is not available in PATH."
  )
}

rf_status <- system2(
  RSCRIPT,
  args=c(
    "-e",
    shQuote(
      "quit(status=ifelse(requireNamespace('randomForest', quietly=TRUE),0,1))"
    )
  )
)

if(rf_status!=0){
  stop(
    "R package 'randomForest' is not available to:\n",
    RSCRIPT
  )
}

cat("\n================ R CHECK ============================\n")
cat("Rscript:",RSCRIPT,"\n")
cat("randomForest: PASS\n")

if(!dir.exists(PURPLE_BASE)){
  stop(
    "Final PURPLE directory not found:\n",
    PURPLE_BASE
  )
}

SNV_FILES <- list.files(
  PURPLE_BASE,
  pattern="\\.purple\\.somatic\\.vcf\\.gz$",
  recursive=TRUE,
  full.names=TRUE
)

SV_FILES <- list.files(
  PURPLE_BASE,
  pattern="\\.purple\\.sv\\.vcf\\.gz$",
  recursive=TRUE,
  full.names=TRUE
)

if(length(SNV_FILES)==0){
  stop(
    "No *.purple.somatic.vcf.gz files found."
  )
}

if(length(SV_FILES)==0){
  stop(
    "No *.purple.sv.vcf.gz files found."
  )
}

SNV_SAMPLES <- sub(
  "\\.purple\\.somatic\\.vcf\\.gz$",
  "",
  basename(SNV_FILES)
)

SV_SAMPLES <- sub(
  "\\.purple\\.sv\\.vcf\\.gz$",
  "",
  basename(SV_FILES)
)

duplicate_snv <- unique(
  SNV_SAMPLES[
    duplicated(SNV_SAMPLES)
  ]
)

duplicate_sv <- unique(
  SV_SAMPLES[
    duplicated(SV_SAMPLES)
  ]
)

if(length(duplicate_snv)>0){
  stop(
    "Duplicate PURPLE somatic VCFs:\n",
    paste(
      duplicate_snv,
      collapse="\n"
    )
  )
}

if(length(duplicate_sv)>0){
  stop(
    "Duplicate PURPLE SV VCFs:\n",
    paste(
      duplicate_sv,
      collapse="\n"
    )
  )
}

SNV_MAP <- setNames(
  SNV_FILES,
  SNV_SAMPLES
)

SV_MAP <- setNames(
  SV_FILES,
  SV_SAMPLES
)

missing_sv <- setdiff(
  names(SNV_MAP),
  names(SV_MAP)
)

missing_snv <- setdiff(
  names(SV_MAP),
  names(SNV_MAP)
)

if(length(missing_sv)>0){
  stop(
    "Samples missing SV VCF:\n",
    paste(
      missing_sv,
      collapse="\n"
    )
  )
}

if(length(missing_snv)>0){
  stop(
    "Samples missing SNV/INDEL VCF:\n",
    paste(
      missing_snv,
      collapse="\n"
    )
  )
}

SAMPLES <- sort(
  intersect(
    names(SNV_MAP),
    names(SV_MAP)
  )
)

EXPECTED_N <- 23L

cat("\n================ PURPLE INPUT SUMMARY ===============\n")
cat("Somatic VCFs:",length(SNV_MAP),"\n")
cat("SV VCFs     :",length(SV_MAP),"\n")
cat("Samples      :",length(SAMPLES),"\n")

if(length(SAMPLES)!=EXPECTED_N){
  stop(
    "Expected 23 samples but found ",
    length(SAMPLES)
  )
}

cat(
  "Sample count check: PASS (23/23)\n"
)

INPUT_MANIFEST <- data.frame(
  Sample=SAMPLES,
  SNV_INDEL_VCF=unname(
    SNV_MAP[SAMPLES]
  ),
  SV_VCF=unname(
    SV_MAP[SAMPLES]
  ),
  stringsAsFactors=FALSE
)

INPUT_MANIFEST$SNV_OK <- vapply(
  INPUT_MANIFEST$SNV_INDEL_VCF,
  valid_file,
  logical(1)
)

INPUT_MANIFEST$SV_OK <- vapply(
  INPUT_MANIFEST$SV_VCF,
  valid_file,
  logical(1)
)

INPUT_MANIFEST$From_Purple_rerun <-
  startsWith(
    INPUT_MANIFEST$SNV_INDEL_VCF,
    paste0(PURPLE_BASE,"/")
  ) &
  startsWith(
    INPUT_MANIFEST$SV_VCF,
    paste0(PURPLE_BASE,"/")
  )

cat("\n================ INPUT MANIFEST =====================\n")
print(
  INPUT_MANIFEST,
  row.names=FALSE
)

if(!all(
  INPUT_MANIFEST$SNV_OK &
  INPUT_MANIFEST$SV_OK &
  INPUT_MANIFEST$From_Purple_rerun
)){
  stop(
    "CHORD input manifest validation failed."
  )
}

cat(
  "\nPURPLE provenance check: PASS\n"
)

MANIFEST_FILE <- file.path(
  CHORD_BASE,
  "CHORD_rerun_input_manifest.tsv"
)

write.table(
  INPUT_MANIFEST,
  MANIFEST_FILE,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

valid_prediction <- function(
  pred_file,
  sample
){

  if(!valid_file(pred_file)){
    return(FALSE)
  }

  x <- tryCatch(
    read.delim(
      pred_file,
      stringsAsFactors=FALSE,
      check.names=FALSE
    ),
    error=function(e){
      NULL
    }
  )

  if(is.null(x)){
    return(FALSE)
  }

  required_cols <- c(
    "sample",
    "p_BRCA1",
    "p_BRCA2",
    "p_hrd",
    "hr_status",
    "hrd_type"
  )

  if(!all(
    required_cols %in%
    colnames(x)
  )){
    return(FALSE)
  }

  sample %in%
    as.character(
      x$sample
    )
}

valid_context <- function(
  context_file,
  sample
){

  if(!valid_file(context_file)){
    return(FALSE)
  }

  x <- tryCatch(
    read.delim(
      context_file,
      stringsAsFactors=FALSE,
      check.names=FALSE
    ),
    error=function(e){
      NULL
    }
  )

  if(
    is.null(x) ||
    nrow(x)==0
  ){
    return(FALSE)
  }

  if(
    "sample_id" %in%
    colnames(x)
  ){
    return(
      sample %in%
      as.character(
        x$sample_id
      )
    )
  }

  if(
    "sample" %in%
    colnames(x)
  ){
    return(
      sample %in%
      as.character(
        x$sample
      )
    )
  }

  FALSE
}

chord_complete <- function(
  sample,
  sample_out
){

  pred <- file.path(
    sample_out,
    paste0(
      sample,
      ".chord.prediction.tsv"
    )
  )

  context <- file.path(
    sample_out,
    paste0(
      sample,
      ".chord.mutation_contexts.tsv"
    )
  )

  valid_prediction(
    pred,
    sample
  ) &&
    valid_context(
      context,
      sample
    )
}

run_chord <- function(
  sample,
  snv_vcf,
  sv_vcf
){

  cat("\n====================================================\n")
  cat("CHORD SAMPLE:",sample,"\n")
  cat("====================================================\n")

  SAMPLE_OUT <- file.path(
    OUTPUT_BASE,
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

  PREDICTION_FILE <- file.path(
    SAMPLE_OUT,
    paste0(
      sample,
      ".chord.prediction.tsv"
    )
  )

  CONTEXT_FILE <- file.path(
    SAMPLE_OUT,
    paste0(
      sample,
      ".chord.mutation_contexts.tsv"
    )
  )

  SUCCESS_FILE <- file.path(
    SAMPLE_OUT,
    ".CHORD_SUCCESS"
  )

  if(
    chord_complete(
      sample,
      SAMPLE_OUT
    )
  ){

    cat(
      "[SKIP] Valid CHORD outputs already complete.\n"
    )

    return(
      data.frame(
        Sample=sample,
        Status="SKIPPED_COMPLETED",
        Exit_code=0L,
        Runtime_min=0,
        Prediction=PREDICTION_FILE,
        Mutation_context=CONTEXT_FILE,
        Log=NA_character_,
        Command=NA_character_,
        stringsAsFactors=FALSE
      )
    )
  }

  if(file.exists(SUCCESS_FILE)){
    unlink(
      SUCCESS_FILE
    )
  }

  args <- c(
    paste0(
      "-Xmx",
      JAVA_MEMORY
    ),
    "-jar",CHORD_JAR,
    "-sample",sample,
    "-snv_indel_vcf_file",snv_vcf,
    "-sv_vcf_file",sv_vcf,
    "-output_dir",SAMPLE_OUT,
    "-ref_genome",REF_GENOME,
    "-threads","1",
    "-log_level","INFO"
  )

  exact_command <- command_string(
    JAVA,
    args
  )

  timestamp <- format(
    Sys.time(),
    "%Y%m%d_%H%M%S"
  )

  COMMAND_FILE <- file.path(
    SAMPLE_COMMAND_DIR,
    paste0(
      sample,
      ".CHORD.",
      timestamp,
      ".command.txt"
    )
  )

  LOG_FILE <- file.path(
    SAMPLE_LOG_DIR,
    paste0(
      sample,
      ".CHORD.",
      timestamp,
      ".log"
    )
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
      paste0("TOOL=CHORD v",CHORD_VERSION),
      paste0("CHORD_JAR=",CHORD_JAR),
      paste0("CHORD_JAR_SHA256=",CHORD_SHA256),
      paste0("JAVA_MEMORY=",JAVA_MEMORY),
      paste0("PURPLE_ROOT=",PURPLE_BASE),
      paste0("SNV_INDEL_VCF=",snv_vcf),
      paste0("SV_VCF=",sv_vcf),
      paste0("REFERENCE_FASTA=",REF_GENOME),
      paste0(
        "REFERENCE_FASTA_SHA256=",
        sha256_file(REF_GENOME)
      ),
      "",
      "ACTUAL_COMMAND:",
      exact_command
    ),
    COMMAND_FILE
  )

  cat("\nACTUAL COMMAND:\n")
  cat(exact_command,"\n")

  START_TIME <- Sys.time()

  EXIT_CODE <- system2(
    JAVA,
    args=args,
    stdout=LOG_FILE,
    stderr=LOG_FILE
  )

  RUNTIME_MIN <- round(
    as.numeric(
      difftime(
        Sys.time(),
        START_TIME,
        units="mins"
      )
    ),
    2
  )

  PREDICTION_OK <- valid_prediction(
    PREDICTION_FILE,
    sample
  )

  CONTEXT_OK <- valid_context(
    CONTEXT_FILE,
    sample
  )

  validation <- data.frame(
    File=c(
      basename(PREDICTION_FILE),
      basename(CONTEXT_FILE)
    ),
    Exists=c(
      file.exists(PREDICTION_FILE),
      file.exists(CONTEXT_FILE)
    ),
    Size_bytes=c(
      ifelse(
        file.exists(PREDICTION_FILE),
        file.info(PREDICTION_FILE)$size,
        NA_real_
      ),
      ifelse(
        file.exists(CONTEXT_FILE),
        file.info(CONTEXT_FILE)$size,
        NA_real_
      )
    ),
    Valid=c(
      PREDICTION_OK,
      CONTEXT_OK
    ),
    stringsAsFactors=FALSE
  )

  VALIDATION_FILE <- file.path(
    SAMPLE_OUT,
    paste0(
      sample,
      ".CHORD.output_validation.tsv"
    )
  )

  write.table(
    validation,
    VALIDATION_FILE,
    sep="\t",
    quote=FALSE,
    row.names=FALSE
  )

  if(
    EXIT_CODE==0 &&
    PREDICTION_OK &&
    CONTEXT_OK
  ){

    writeLines(
      c(
        paste0("Sample=",sample),
        paste0(
          "Completed=",
          format(
            Sys.time(),
            "%Y-%m-%d %H:%M:%S %Z"
          )
        ),
        paste0("CHORD_version=",CHORD_VERSION),
        paste0("CHORD_JAR_SHA256=",CHORD_SHA256),
        paste0("PURPLE_ROOT=",PURPLE_BASE),
        paste0("SNV_INDEL_VCF=",snv_vcf),
        paste0("SV_VCF=",sv_vcf)
      ),
      SUCCESS_FILE
    )

    FINAL_STATUS <- "COMPLETED"

  } else {

    FINAL_STATUS <- "FAILED"

    if(file.exists(LOG_FILE)){
      z <- readLines(
        LOG_FILE,
        warn=FALSE
      )
      cat(
        tail(z,100),
        sep="\n"
      )
    }
  }

  data.frame(
    Sample=sample,
    Status=FINAL_STATUS,
    Exit_code=EXIT_CODE,
    Runtime_min=RUNTIME_MIN,
    Prediction=PREDICTION_FILE,
    Mutation_context=CONTEXT_FILE,
    Log=LOG_FILE,
    Command=COMMAND_FILE,
    stringsAsFactors=FALSE
  )
}

TEST_SAMPLE <- SAMPLES[1]

TEST_RESULT <- run_chord(
  sample=TEST_SAMPLE,
  snv_vcf=SNV_MAP[[TEST_SAMPLE]],
  sv_vcf=SV_MAP[[TEST_SAMPLE]]
)

if(
  !TEST_RESULT$Status %in%
  c("COMPLETED","SKIPPED_COMPLETED")
){
  stop(
    "FIRST SAMPLE CHORD TEST FAILED. ",
    "Remaining samples were NOT started."
  )
}

RESULTS <- vector(
  "list",
  length(SAMPLES)
)

for(i in seq_along(SAMPLES)){

  sample <- SAMPLES[i]

  RESULTS[[i]] <- run_chord(
    sample=sample,
    snv_vcf=SNV_MAP[[sample]],
    sv_vcf=SV_MAP[[sample]]
  )

  if(
    !RESULTS[[i]]$Status %in%
    c("COMPLETED","SKIPPED_COMPLETED")
  ){
    stop(
      "CHORD failed for ",
      sample,
      ". Remaining samples were NOT started."
    )
  }
}

RUN_SUMMARY <- do.call(
  rbind,
  RESULTS
)

SUMMARY_FILE <- file.path(
  OUTPUT_BASE,
  "CHORD_rerun_summary.tsv"
)

write.table(
  RUN_SUMMARY,
  SUMMARY_FILE,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

FINAL_VALIDATION <- do.call(
  rbind,
  lapply(
    SAMPLES,
    function(sample){

      sample_out <- file.path(
        OUTPUT_BASE,
        sample
      )

      pred <- file.path(
        sample_out,
        paste0(
          sample,
          ".chord.prediction.tsv"
        )
      )

      context <- file.path(
        sample_out,
        paste0(
          sample,
          ".chord.mutation_contexts.tsv"
        )
      )

      pred_ok <- valid_prediction(
        pred,
        sample
      )

      context_ok <- valid_context(
        context,
        sample
      )

      data.frame(
        Sample=sample,
        Prediction=pred,
        Prediction_valid=pred_ok,
        Mutation_context=context,
        Mutation_context_valid=context_ok,
        Complete=pred_ok && context_ok,
        stringsAsFactors=FALSE
      )
    }
  )
)

FINAL_VALIDATION_FILE <- file.path(
  OUTPUT_BASE,
  "CHORD_23_samples_output_validation.tsv"
)

write.table(
  FINAL_VALIDATION,
  FINAL_VALIDATION_FILE,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

if(!all(FINAL_VALIDATION$Complete)){
  stop(
    "At least one sample failed final CHORD validation."
  )
}

PREDICTION_LIST <- lapply(
  FINAL_VALIDATION$Prediction,
  function(f){
    read.delim(
      f,
      stringsAsFactors=FALSE,
      check.names=FALSE
    )
  }
)

ALL_PREDICTIONS <- do.call(
  rbind,
  PREDICTION_LIST
)

ALL_PREDICTIONS <- ALL_PREDICTIONS[
  !duplicated(
    ALL_PREDICTIONS$sample
  ),
  ,
  drop=FALSE
]

ALL_PREDICTIONS <- ALL_PREDICTIONS[
  order(
    ALL_PREDICTIONS$sample
  ),
  ,
  drop=FALSE
]

if(nrow(ALL_PREDICTIONS)!=EXPECTED_N){
  stop(
    "Expected 23 combined predictions but obtained ",
    nrow(ALL_PREDICTIONS)
  )
}

COMBINED_TSV <- file.path(
  OUTPUT_BASE,
  "CHORD_all_samples_predictions.tsv"
)

COMBINED_CSV <- file.path(
  OUTPUT_BASE,
  "CHORD_all_samples_predictions.csv"
)

write.table(
  ALL_PREDICTIONS,
  COMBINED_TSV,
  sep="\t",
  quote=FALSE,
  row.names=FALSE
)

write.csv(
  ALL_PREDICTIONS,
  COMBINED_CSV,
  row.names=FALSE
)

cat("\n================ HR STATUS ==========================\n")
print(
  table(
    ALL_PREDICTIONS$hr_status,
    useNA="ifany"
  )
)

cat("\n================ HRD TYPE ===========================\n")
print(
  table(
    ALL_PREDICTIONS$hrd_type,
    useNA="ifany"
  )
)

SESSION_FILE <- file.path(
  CHORD_BASE,
  paste0(
    "CHORD_sessionInfo_",
    format(Sys.time(),"%Y%m%d_%H%M%S"),
    ".txt"
  )
)

writeLines(
  capture.output(sessionInfo()),
  SESSION_FILE
)

run_status <- "COMPLETED"

cat("\n====================================================\n")
cat("CHORD FINAL SUMMARY\n")
cat("CHORD version     :",CHORD_VERSION,"\n")
cat("CHORD JAR         :",CHORD_JAR,"\n")
cat("CHORD SHA256      :",CHORD_SHA256,"\n")
cat("PURPLE source     :",PURPLE_BASE,"\n")
cat("Reference         :",REF_GENOME,"\n")
cat("Java memory       :",JAVA_MEMORY,"\n")
cat("Samples complete  :",sum(FINAL_VALIDATION$Complete),"/23\n")
cat("Output            :",OUTPUT_BASE,"\n")
cat("FINAL STATUS      :",run_status,"\n")
cat("====================================================\n")
}

main()
```

## 10. Final outputs

For each sample:

```text
<SAMPLE>.chord.prediction.tsv
<SAMPLE>.chord.mutation_contexts.tsv
```

Cohort-level:

```text
CHORD_all_samples_predictions.tsv
CHORD_all_samples_predictions.csv
CHORD_23_samples_output_validation.tsv
CHORD_rerun_summary.tsv
```
