# REDUX v2.0.4 Complete Workflow + MSI Prediction — UCaTS Organoids

> **Project:** UCaTS Organoid WGS  
> **Tool:** REDUX v2.0.4  
> **Mode:** tumor-only WGS preprocessing  
> **Genome:** GRCh38 / hg38  
> **Sequencing:** Illumina  
> **Threads:** 24  
> **Java heap:** 40G  
> **Input:** original tumor BAMs  
> **Output:** REDUX-processed BAMs, BQR/jitter/MS-table sidecars, and a separate MSI-only prediction rerun

## 1. Purpose

This workflow runs HMFtools REDUX on all tumor BAMs in the UCaTS Organoid WGS dataset.

The project now has **two REDUX stages**:

```text
STAGE 1 — main REDUX preprocessing

Original tumor BAM
        |
        v
      REDUX
        |
        +--> <SAMPLE>.redux.bam
        +--> BAM index
        +--> <SAMPLE>.redux.bqr.tsv
        +--> <SAMPLE>.redux.jitter_params.tsv
        +--> <SAMPLE>.redux.ms_table.tsv.gz


STAGE 2 — MSI-only rerun on the existing REDUX BAM

<SAMPLE>.redux.bam
        |
        +--> msi_jitter_sites.38.tsv.gz
        +--> ms_model_coefficients.hmf_wgs.tsv
        +--> ms_model_error_rates.tso500.37.tsv
        |
        v
REDUX -bqr_jitter_msi_only
        |
        v
<SAMPLE>.redux.msi_prediction.tsv
```

The second stage does **not** regenerate the REDUX BAM. It only attempts to generate the MSI prediction from the already completed REDUX BAM.

The script also:

```text
checks required references
loads samtools
finds all *_tumor.bam files
indexes input BAMs if needed
skips completed REDUX samples
runs one sample at a time
stops immediately on REDUX failure
writes Redux_run_summary.csv
```

---

## 2. Current project-layout note

The original REDUX script was first written under `05_ASCAT_CN`, but the project was later reorganized.

The **current canonical REDUX location** is:

```text
06_REDux/
├── Redux_tools/
├── Redux_reference/
└── Redux_BAMs/
```

The original tumor BAMs remain under:

```text
05_ASCAT_CN/BAMs/
```

Therefore the current workflow is:

```text
05_ASCAT_CN/BAMs/<SAMPLE>_tumor.bam
            |
            v
         REDUX
            |
            v
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam
```

All current downstream SAGE and ESVEE workflows should use the REDUX BAMs under `06_REDux/Redux_BAMs`.

---

## 3. Current paths used by this code

Project root:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids
```

Input BAM directory:

```text
05_ASCAT_CN/BAMs
```

Tool directory:

```text
06_REDux/Redux_tools
```

Reference directory:

```text
06_REDux/Redux_reference
```

Current REDUX BAM output:

```text
06_REDux/Redux_BAMs
```

Reference FASTA:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

Java:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

REDUX JAR:

```text
06_REDux/Redux_tools/redux_v2.0.4.jar
```

---

## 4. Java

The workflow uses:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Check:

```bash
/home/zzr123/.conda/envs/purple/bin/java -version
```

If Java needs to be installed in a separate Conda environment:

```bash
conda create   -n hmftools_java   -c conda-forge   openjdk=21   -y

conda activate hmftools_java

java -version
```

For this specific workflow, the R code continues to use the existing absolute Java path.

---

## 5. REDUX v2.0.4 JAR

The code expects:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/
06_REDux/Redux_tools/redux_v2.0.4.jar
```

Create the directory if necessary:

```bash
mkdir -p /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_tools
```

Check the JAR:

```bash
JAR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_tools/redux_v2.0.4.jar"

ls -lh "$JAR"
sha256sum "$JAR"
```

For reproducibility, record the checksum of the exact JAR actually used.

---

## 6. REDUX reference files

The script searches recursively inside:

```text
06_REDux/Redux_reference
```

for:

```text
unmap_regions.38.tsv
msi_jitter_sites.38.tsv.gz
```

The code resolves them with:

```r
unmap_regions <- list.files(
  ref_dir,
  pattern="^unmap_regions\.38\.tsv$",
  recursive=TRUE,
  full.names=TRUE
)[1]

msi_file <- list.files(
  ref_dir,
  pattern="^msi_jitter_sites\.38\.tsv\.gz$",
  recursive=TRUE,
  full.names=TRUE
)[1]
```

These are then supplied to REDUX as:

```text
-unmap_regions
-ref_genome_msi_file
```

---

## 7. HMF reference bundle

These resources are normally provided by the HMFtools GRCh38 pipeline resource bundle.

Project resource version used elsewhere in this workflow:

```text
hmf_pipeline_resources.38_v3.0.0--8
```

A reproducible download pattern is:

```bash
set -euo pipefail

REF_DIR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/06_REDux/Redux_reference"
BUNDLE="hmf_pipeline_resources.38_v3.0.0--8.tar.gz"

mkdir -p "$REF_DIR"
cd "$REF_DIR"

wget -c "https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/$BUNDLE"

tar -xzf "$BUNDLE"
```

Check the two resources used by this code:

```bash
find "$REF_DIR" -type f | grep -E 'unmap_regions\.38\.tsv$|msi_jitter_sites\.38\.tsv\.gz$'
```

---


## 8. MSI prediction model resources

The MSI-only rerun uses three resources from the shared HMF bundle:

```text
dna/variants/msi_jitter_sites.38.tsv.gz
dna/variants/ms_model_coefficients.hmf_wgs.tsv
dna/variants/ms_model_error_rates.tso500.37.tsv
```

Current shared root:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
```

Exact project paths:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
dna/variants/msi_jitter_sites.38.tsv.gz

/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
dna/variants/ms_model_coefficients.hmf_wgs.tsv

/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
dna/variants/ms_model_error_rates.tso500.37.tsv
```

Check:

```bash
HMF_REF="/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"

ls -lh "$HMF_REF/dna/variants/msi_jitter_sites.38.tsv.gz"
ls -lh "$HMF_REF/dna/variants/ms_model_coefficients.hmf_wgs.tsv"
ls -lh "$HMF_REF/dna/variants/ms_model_error_rates.tso500.37.tsv"
```

### Important experimental caveat

The coefficient model is explicitly:

```text
hmf_wgs
```

but the only available error-rate file used in this project is:

```text
tso500.37
```

This is **not being treated as a standard matched WGS/GRCh38 model pair**. The file is included in this workflow only because it was specifically tested at the mentor's suggestion.

The MSI output should therefore be documented as:

```text
experimental REDUX MSI prediction using:
HMF WGS coefficients
+
TSO500.37 error rates
```

and should not silently be presented as a fully standard WGS MSI configuration.

---

## 9. Reference FASTA

The workflow uses:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

The code also requires:

```text
hg38.fa.fai
```

Check:

```bash
REF="/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"

ls -lh "$REF"
ls -lh "$REF.fai"

sha256sum "$REF"
```

If the `.fai` is absent:

```bash
samtools faidx "$REF"
```

The FASTA should remain the same reference family used to align the input tumor BAMs.

---

## 10. samtools

The supplied code does not assume `samtools` is already on `PATH`.

Instead it runs:

```bash
module load samtools
```

inside a login shell and then resolves:

```bash
command -v samtools
```

The R expression is:

```r
samtools_path <- system(
  "bash -lc 'module load samtools >/dev/null 2>&1; command -v samtools'",
  intern=TRUE
)[1]
```

If no executable is found, the script stops.

To check manually:

```bash
module load samtools
command -v samtools
samtools --version
```

---

## 11. Input BAM discovery

The workflow selects only files matching:

```text
*_tumor.bam
```

inside:

```text
05_ASCAT_CN/BAMs
```

R code:

```r
bam_files <- list.files(
  bam_dir,
  pattern="_tumor\.bam$",
  full.names=TRUE
)
```

Sample IDs are created by removing:

```text
_tumor.bam
```

from each filename.

Example:

```text
I_26166_S_29146_tumor.bam
```

becomes:

```text
I_26166_S_29146
```

---

## 12. Input BAM index behavior

Before REDUX starts on a sample, the script looks for either:

```text
<SAMPLE>_tumor.bam.bai
```

or:

```text
<SAMPLE>_tumor.bai
```

If neither exists, it automatically runs:

```bash
samtools index -@ 24 <BAM>
```

If indexing fails, the script stops immediately.

---

## 13. Main REDUX command used

The command is equivalent to:

```bash
java -Xmx40G   -jar redux_v2.0.4.jar   -sample SAMPLE   -input_bam SAMPLE_tumor.bam   -ref_genome hg38.fa   -ref_genome_version V38   -sequencing_type ILLUMINA   -unmap_regions unmap_regions.38.tsv   -ref_genome_msi_file msi_jitter_sites.38.tsv.gz   -form_consensus   -bamtool /path/to/samtools   -output_dir SAMPLE_OUTPUT   -log_level INFO   -threads 24
```

Compute settings:

```text
Java heap = 40G
Threads   = 24
```

Genome:

```text
V38
```

Sequencing type:

```text
ILLUMINA
```

---

## 14. `-form_consensus`

The supplied workflow explicitly enables:

```text
-form_consensus
```

This is part of the REDUX processing used in this project.

The Markdown preserves this parameter exactly because it is part of the actual code being documented.

---

## 15. Main REDUX output files

For each sample, the code expects:

```text
<SAMPLE>.redux.bam
<SAMPLE>.redux.bam.bai
or
<SAMPLE>.redux.bai

<SAMPLE>.redux.bqr.tsv
<SAMPLE>.redux.jitter_params.tsv
<SAMPLE>.redux.ms_table.tsv.gz
<SAMPLE>.redux.log
```

The sample output directory is:

```text
06_REDux/Redux_BAMs/<SAMPLE>/
```

Example:

```text
06_REDux/Redux_BAMs/I_26166_S_29146/
├── I_26166_S_29146.redux.bam
├── I_26166_S_29146.redux.bam.bai
├── I_26166_S_29146.redux.bqr.tsv
├── I_26166_S_29146.redux.jitter_params.tsv
├── I_26166_S_29146.redux.ms_table.tsv.gz
└── I_26166_S_29146.redux.log
```

---

## 16. Main REDUX completion rule

A sample is treated as complete only when all of the following exist:

```text
REDUX BAM
BAM index
BQR table
jitter parameter table
MS table
```

The code is:

```r
completed <- file.exists(output_bam) &&
  (file.exists(output_bai1) || file.exists(output_bai2)) &&
  file.exists(bqr_file) &&
  file.exists(jitter_file) &&
  file.exists(ms_file)
```

This check is performed:

```text
before REDUX
and
after REDUX
```

---

## 17. Main REDUX resume / skip logic

If all expected REDUX outputs already exist, the sample is skipped:

```text
SKIP: <SAMPLE> - already completed
```

and its summary status becomes:

```text
Already completed
```

This allows an interrupted cohort run to resume without repeating completed samples.

---

## 18. Main REDUX failure behavior

REDUX is considered successful only when:

```text
Java exit status = 0
AND
all expected output files exist
```

If either condition fails:

```text
Status = Failed
```

The script then:

```text
writes Redux_run_summary.csv
prints the log path
prints the last 80 lines of the REDUX log
stops immediately
does not start remaining samples
```

This avoids wasting compute time after a failed sample.

---

## 19. Main REDUX project-level summary

The run summary is written to:

```text
06_REDux/Redux_BAMs/Redux_run_summary.csv
```

Columns:

```text
Sample
Status
```

Example statuses:

```text
Completed
Already completed
Failed
```

The file is updated after every sample.

---

## 20. Expected directory structure

```text
UCaTS_Organoids/
│
└── 05_ASCAT_CN/
    │
    ├── BAMs/
    │   ├── I_26166_S_29146_tumor.bam
    │   ├── I_26166_S_29146_tumor.bam.bai
    │   └── ...
    │
    ├── Redux_tools/
    │   └── redux_v2.0.4.jar
    │
    ├── Redux_reference/
    │   └── hmf_pipeline_resources.38_v3.0.0--8/
    │       ├── .../unmap_regions.38.tsv
    │       └── .../msi_jitter_sites.38.tsv.gz
    │
    └── Redux_output/
        ├── Redux_run_summary.csv
        │
        ├── I_26166_S_29146/
        │   ├── I_26166_S_29146.redux.bam
        │   ├── I_26166_S_29146.redux.bam.bai
        │   ├── I_26166_S_29146.redux.bqr.tsv
        │   ├── I_26166_S_29146.redux.jitter_params.tsv
        │   ├── I_26166_S_29146.redux.ms_table.tsv.gz
        │   ├── I_26166_S_29146.redux.msi_prediction.tsv  # after MSI-only success
        │   └── I_26166_S_29146.redux.log
        │
        └── ...
    │
    └── MSI_only/
        ├── REDUX_MSI_23_samples_summary.tsv
        ├── I_26166_S_29146/
        │   ├── I_26166_S_29146.redux.msi_prediction.tsv
        │   ├── I_26166_S_29146.redux.msi.log
        │   └── I_26166_S_29146.redux.msi.command.txt
        └── ...
```

Shared reference:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/
├── hg38.fa
└── hg38.fa.fai
```

---

## 21. Downstream use

The REDUX BAM produced here is intended for downstream HMFtools analyses such as:

```text
SAGE
ESVEE
```

Conceptually:

```text
Original tumor BAM
        |
        v
      REDUX
       /  \
      v    v
    SAGE  ESVEE
```

The important methodological rule is:

```text
downstream HMFtools variant calling should use the REDUX BAM,
not the original tumor BAM.
```

---

## 22. Separate REDUX MSI-only prediction rerun

The base REDUX preprocessing stage does **not** use the MSI prediction file as its completion criterion.

The project later added a separate MSI-only REDUX rerun to attempt to generate:

```text
<SAMPLE>.redux.msi_prediction.tsv
```

from the already completed REDUX BAM.

### 22.1 Input

Current input:

```text
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam.bai
```

The REDUX BAM is **not regenerated**.

### 22.2 MSI-only output workspace

Intermediate MSI-only output is written to:

```text
06_REDux/MSI_only/<SAMPLE>/
```

Per sample:

```text
<SAMPLE>.redux.msi_prediction.tsv
<SAMPLE>.redux.msi.log
<SAMPLE>.redux.msi.command.txt
```

Cohort summary:

```text
06_REDux/MSI_only/REDUX_MSI_23_samples_summary.tsv
```

### 22.3 Final MSI prediction location

After a successful MSI-only run, the code copies **only** the prediction file beside the existing REDUX BAM:

```text
06_REDux/Redux_BAMs/<SAMPLE>/
├── <SAMPLE>.redux.bam
├── <SAMPLE>.redux.bam.bai
├── <SAMPLE>.redux.bqr.tsv
├── <SAMPLE>.redux.jitter_params.tsv
├── <SAMPLE>.redux.ms_table.tsv.gz
└── <SAMPLE>.redux.msi_prediction.tsv
```

Existing REDUX BAM/BQR/jitter outputs are not replaced.

### 22.4 MSI-only command

The essential command is:

```text
REDUX v2.0.4

-input_bam <existing REDUX BAM>
-ref_genome hg38.fa
-ref_genome_version V38
-sequencing_type ILLUMINA
-ref_genome_msi_file msi_jitter_sites.38.tsv.gz
-msi_model_coefficients ms_model_coefficients.hmf_wgs.tsv
-msi_model_error_rates ms_model_error_rates.tso500.37.tsv
-bqr_jitter_msi_only
-threads 24
```

Java heap:

```text
-Xmx40G
```

### 22.5 First-sample gate

The workflow deliberately tests:

```text
I_26166_S_29146
```

first.

The remaining 22 samples are started only if the first sample produces a non-empty:

```text
I_26166_S_29146.redux.msi_prediction.tsv
```

If the test sample fails, the cohort run stops.

### 22.6 Completion / skip rule

The MSI-only stage skips a sample only when the actual prediction exists and is non-empty:

```r
file.exists(msi_prediction) &&
file.info(msi_prediction)$size > 0
```

A historical REDUX success marker, BQR file, jitter table, or BAM by itself does **not** count as MSI-prediction completion.

### 22.7 Provenance

For every MSI sample the code stores the exact command in:

```text
<SAMPLE>.redux.msi.command.txt
```

and the complete REDUX MSI log in:

```text
<SAMPLE>.redux.msi.log
```

The command record explicitly stores:

```text
sample
REDUX BAM
MSI jitter-site resource
MSI coefficient model
MSI error-rate model
note about the experimental TSO500.37 error-rate file
exact executed command
```

### 22.8 Experimental interpretation warning

The output should be labelled clearly:

```text
Experimental REDUX MSI prediction
HMF WGS coefficient model
+
TSO500.37 error-rate model
tested per mentor instruction
```

This experiment does **not** change the core REDUX BAM used by SAGE/ESVEE.

### 22.9 Complete MSI-only R code

```r
# ============================================================
# REDUX v2.0.4
# MSI-only rerun on existing REDUX BAMs
#
# TEST FIRST SAMPLE -> only continue if MSI prediction succeeds
#
# Important:
# - Existing REDUX BAM is NOT regenerated
# - Uses HMF WGS coefficient model
# - Experimental use of TSO500.37 error-rate file per mentor
# - Exact command printed + saved for every sample
# ============================================================

BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"

REDUX_BASE <- file.path(BASE, "06_REDux")
REDUX_DIR <- file.path(REDUX_BASE, "Redux_BAMs")
TOOL_DIR <- file.path(REDUX_BASE, "Redux_tools")

# Separate output directory for MSI-only rerun
MSI_OUT <- file.path(REDUX_BASE, "MSI_only")
dir.create(MSI_OUT, recursive = TRUE, showWarnings = FALSE)

JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"
REDUX_JAR <- file.path(TOOL_DIR, "redux_v2.0.4.jar")

REF_FASTA <- "/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"

HMF_REF <- "/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"

MSI_SITES <- file.path(
  HMF_REF,
  "dna/variants/msi_jitter_sites.38.tsv.gz"
)

MSI_COEFFICIENTS <- file.path(
  HMF_REF,
  "dna/variants/ms_model_coefficients.hmf_wgs.tsv"
)

MSI_ERROR_RATES <- file.path(
  HMF_REF,
  "dna/variants/ms_model_error_rates.tso500.37.tsv"
)

SAMTOOLS <- system(
  "bash -lc 'module load samtools >/dev/null 2>&1; command -v samtools'",
  intern = TRUE
)[1]

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

# ============================================================
# 1. PREFLIGHT
# ============================================================

required <- c(
  JAVA,
  REDUX_JAR,
  REF_FASTA,
  paste0(REF_FASTA, ".fai"),
  MSI_SITES,
  MSI_COEFFICIENTS,
  MSI_ERROR_RATES
)

check <- data.frame(
  Resource = c(
    "Java",
    "REDUX jar",
    "Reference FASTA",
    "FASTA index",
    "MSI jitter sites",
    "MSI coefficients",
    "MSI error rates"
  ),
  Path = required,
  Exists = file.exists(required),
  stringsAsFactors = FALSE
)

print(check, row.names = FALSE)

if (!all(check$Exists)) {
  stop("One or more required files are missing.")
}

if (
  length(SAMTOOLS) == 0 ||
  is.na(SAMTOOLS) ||
  SAMTOOLS == ""
) {
  stop("samtools not found.")
}

cat("\n====================================================\n")
cat("MSI MODEL CONFIGURATION\n")
cat("====================================================\n")
cat("Redux:        v2.0.4\n")
cat("MSI sites:    ", MSI_SITES, "\n", sep = "")
cat("Coefficients: ", MSI_COEFFICIENTS, "\n", sep = "")
cat("Error rates:  ", MSI_ERROR_RATES, "\n", sep = "")
cat("\nNOTE: TSO500.37 error rates are being tested\n")
cat("      specifically at mentor's suggestion.\n")
cat("====================================================\n")

# ============================================================
# 2. RUN ONE SAMPLE
# ============================================================

run_msi <- function(sample) {

  cat("\n\n====================================================\n")
  cat("SAMPLE: ", sample, "\n", sep = "")
  cat("====================================================\n")

  redux_bam <- file.path(
    REDUX_DIR,
    sample,
    paste0(sample, ".redux.bam")
  )

  redux_bai <- paste0(redux_bam, ".bai")

  if (!file.exists(redux_bam)) {
    cat("FAILED: REDUX BAM missing\n")
    return(
      data.frame(
        Sample = sample,
        Status = "REDUX_BAM_MISSING",
        MSI_prediction = NA_character_,
        Runtime_min = NA_real_
      )
    )
  }

  if (!file.exists(redux_bai)) {
    cat("FAILED: REDUX BAM index missing\n")
    return(
      data.frame(
        Sample = sample,
        Status = "REDUX_BAI_MISSING",
        MSI_prediction = NA_character_,
        Runtime_min = NA_real_
      )
    )
  }

  sample_out <- file.path(
    MSI_OUT,
    sample
  )

  dir.create(
    sample_out,
    recursive = TRUE,
    showWarnings = FALSE
  )

  msi_prediction <- file.path(
    sample_out,
    paste0(sample, ".redux.msi_prediction.tsv")
  )

  # Skip only if actual MSI prediction exists
  if (
    file.exists(msi_prediction) &&
    file.info(msi_prediction)$size > 0
  ) {

    cat("MSI prediction already exists -> SKIP\n")

    return(
      data.frame(
        Sample = sample,
        Status = "Already completed",
        MSI_prediction = msi_prediction,
        Runtime_min = 0
      )
    )
  }

  log_file <- file.path(
    sample_out,
    paste0(sample, ".redux.msi.log")
  )

  command_file <- file.path(
    sample_out,
    paste0(sample, ".redux.msi.command.txt")
  )

  # Actual REDUX MSI-only command
  args <- c(
    "-Xmx40G",
    "-jar", REDUX_JAR,

    "-sample", sample,

    "-input_bam", redux_bam,

    "-ref_genome", REF_FASTA,
    "-ref_genome_version", "V38",

    "-sequencing_type", "ILLUMINA",

    "-ref_genome_msi_file", MSI_SITES,

    "-msi_model_coefficients",
    MSI_COEFFICIENTS,

    "-msi_model_error_rates",
    MSI_ERROR_RATES,

    "-bamtool", SAMTOOLS,

    "-bqr_jitter_msi_only",

    "-output_dir", sample_out,

    "-threads", "24",

    "-log_level", "INFO"
  )

  # Save exact command for reproducibility
  command_string <- paste(
    shQuote(JAVA),
    paste(
      shQuote(args),
      collapse = " "
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

      paste0(
        "SAMPLE=",
        sample
      ),

      "TOOL=REDUX v2.0.4",

      paste0(
        "REDUX_BAM=",
        redux_bam
      ),

      paste0(
        "MSI_SITES=",
        MSI_SITES
      ),

      paste0(
        "MSI_COEFFICIENTS=",
        MSI_COEFFICIENTS
      ),

      paste0(
        "MSI_ERROR_RATES=",
        MSI_ERROR_RATES
      ),

      "NOTE=TSO500.37 error-rate file tested per mentor instruction",

      "",

      command_string
    ),
    command_file
  )

  cat("\nACTUAL COMMAND:\n")
  cat(command_string, "\n\n")

  cat(
    "Command saved:",
    command_file,
    "\n"
  )

  cat(
    "Log:",
    log_file,
    "\n\n"
  )

  # RUN
  start_time <- Sys.time()

  status <- system2(
    JAVA,
    args = args,
    stdout = log_file,
    stderr = log_file
  )

  runtime_min <- round(
    as.numeric(
      difftime(
        Sys.time(),
        start_time,
        units = "mins"
      )
    ),
    2
  )

  cat(
    "\nExit status:",
    status,
    "\n"
  )

  cat(
    "Runtime:",
    runtime_min,
    "min\n"
  )

  # Validate MSI prediction
  success <-
    status == 0 &&
    file.exists(msi_prediction) &&
    file.info(msi_prediction)$size > 0

  if (success) {

    cat("\n***************************************\n")
    cat("SUCCESS: MSI prediction generated\n")
    cat("***************************************\n")

    cat(
      "MSI prediction:",
      msi_prediction,
      "\n\n"
    )

    print(
      read.delim(
        msi_prediction,
        check.names = FALSE
      )
    )

    # Copy ONLY final MSI prediction beside REDUX BAM.
    # Existing BQR/jitter files are NOT replaced.
    final_msi <- file.path(
      dirname(redux_bam),
      paste0(
        sample,
        ".redux.msi_prediction.tsv"
      )
    )

    copied <- file.copy(
      msi_prediction,
      final_msi,
      overwrite = TRUE
    )

    if (!copied) {
      warning(
        "MSI generated but could not copy to REDUX directory: ",
        final_msi
      )
    } else {
      cat(
        "\nCopied MSI prediction to REDUX directory:\n",
        final_msi,
        "\n",
        sep = ""
      )
    }

    return(
      data.frame(
        Sample = sample,
        Status = "Completed",
        MSI_prediction = final_msi,
        Runtime_min = runtime_min
      )
    )

  } else {

    cat("\n***************************************\n")
    cat("MSI prediction NOT generated\n")
    cat("***************************************\n")

    log <- readLines(
      log_file,
      warn = FALSE
    )

    cat("\nMSI-related log lines:\n")

    x <- grep(
      "msi|microsatellite|jitter|model|error",
      log,
      ignore.case = TRUE,
      value = TRUE
    )

    cat(
      paste(
        x,
        collapse = "\n"
      )
    )

    cat("\n\nLast 80 log lines:\n")

    cat(
      paste(
        tail(log, 80),
        collapse = "\n"
      )
    )

    return(
      data.frame(
        Sample = sample,
        Status = "Failed",
        MSI_prediction = NA_character_,
        Runtime_min = runtime_min
      )
    )
  }
}

# ============================================================
# 3. TEST FIRST SAMPLE
# ============================================================

test_sample <- samples[1]

cat("\n\n############################################\n")
cat("FIRST TEST SAMPLE:", test_sample, "\n")
cat("############################################\n")

test_result <- run_msi(
  test_sample
)

print(
  test_result,
  row.names = FALSE
)

# ============================================================
# 4. ONLY CONTINUE IF TEST SUCCESSFUL
# ============================================================

if (
  !test_result$Status %in%
  c("Completed", "Already completed")
) {

  stop(
    "\nMSI prediction was NOT generated for the test sample.\n",
    "Remaining 22 samples were NOT started."
  )
}

# ============================================================
# 5. RUN REMAINING 22 SAMPLES
# ============================================================

cat("\n\n============================================\n")
cat("TEST SUCCESSFUL\n")
cat("Starting remaining 22 samples\n")
cat("============================================\n")

remaining_samples <- setdiff(
  samples,
  test_sample
)

results_list <- lapply(
  remaining_samples,
  run_msi
)

results <- do.call(
  rbind,
  c(
    list(test_result),
    results_list
  )
)

rownames(results) <- NULL

# ============================================================
# 6. SAVE SUMMARY
# ============================================================

summary_file <- file.path(
  MSI_OUT,
  "REDUX_MSI_23_samples_summary.tsv"
)

write.table(
  results,
  summary_file,
  sep = "\t",
  quote = FALSE,
  row.names = FALSE
)

cat("\n\n============================================\n")
cat("FINAL MSI SUMMARY\n")
cat("============================================\n")

print(
  results,
  row.names = FALSE
)

cat(
  "\nSummary saved:\n",
  summary_file,
  "\n",
  sep = ""
)

```

---

## 23. Reproducibility checklist

```text
[ ] REDUX v2.0.4 JAR exists
[ ] REDUX JAR SHA256 recorded
[ ] Java version recorded

[ ] hg38.fa exists
[ ] hg38.fa.fai exists

[ ] unmap_regions.38.tsv exists
[ ] msi_jitter_sites.38.tsv.gz exists

[ ] samtools module loads successfully
[ ] samtools executable path resolved

[ ] input BAMs match *_tumor.bam
[ ] input BAM index exists or is created

[ ] Java heap = 40G
[ ] threads = 24
[ ] genome version = V38
[ ] sequencing type = ILLUMINA
[ ] form_consensus enabled

[ ] REDUX BAM exists
[ ] REDUX BAM index exists
[ ] BQR exists
[ ] jitter parameters exist
[ ] MS table exists

MSI-only extension:
[ ] ms_model_coefficients.hmf_wgs.tsv exists
[ ] ms_model_error_rates.tso500.37.tsv exists
[ ] experimental model mismatch documented
[ ] first MSI test sample succeeds before cohort continuation
[ ] per-sample MSI exact command saved
[ ] per-sample MSI log saved
[ ] MSI prediction non-empty before marking complete
[ ] final MSI prediction copied beside REDUX BAM
[ ] REDUX_MSI_23_samples_summary.tsv written

[ ] per-sample REDUX log written
[ ] Redux_run_summary.csv written
```

---

## 24. Main REDUX complete R code

The code below is preserved from the supplied REDUX workflow.

```r
ROOT <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"

bam_dir <- file.path(
  ROOT,
  "05_ASCAT_CN",
  "BAMs"
)

tool_dir <- file.path(
  ROOT,
  "06_REDux",
  "Redux_tools"
)

ref_dir <- file.path(
  ROOT,
  "06_REDux",
  "Redux_reference"
)

output_base <- file.path(
  ROOT,
  "06_REDux",
  "Redux_BAMs"
)
ref_fasta <- "/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"
redux_jar <- file.path(tool_dir, "redux_v2.0.4.jar")
java_bin <- "/home/zzr123/.conda/envs/purple/bin/java"

dir.create(output_base, recursive = TRUE, showWarnings = FALSE)

# References
unmap_regions <- list.files(ref_dir, pattern = "^unmap_regions\\.38\\.tsv$", recursive = TRUE, full.names = TRUE)[1]
msi_file <- list.files(ref_dir, pattern = "^msi_jitter_sites\\.38\\.tsv\\.gz$", recursive = TRUE, full.names = TRUE)[1]

# Samtools
samtools_path <- system("bash -lc 'module load samtools >/dev/null 2>&1; command -v samtools'", intern = TRUE)[1]

# Check required files
required <- c(ref_fasta, paste0(ref_fasta, ".fai"), redux_jar, unmap_regions, msi_file)
if (!all(file.exists(required))) {
  print(data.frame(File = required, Exists = file.exists(required)))
  stop("Required file missing.")
}
if (is.na(samtools_path) || samtools_path == "") stop("samtools not found")

# Find all tumor BAMs
bam_files <- list.files(bam_dir, pattern = "_tumor\\.bam$", full.names = TRUE)
samples <- sub("_tumor\\.bam$", "", basename(bam_files))
cat("Found", length(samples), "tumor samples\n")

results <- data.frame(Sample = samples, Status = NA_character_, stringsAsFactors = FALSE)

for (i in seq_along(samples)) {
  sample <- samples[i]
  bam <- bam_files[i]
  sample_dir <- file.path(output_base, sample)
  dir.create(sample_dir, recursive = TRUE, showWarnings = FALSE)

  output_bam <- file.path(sample_dir, paste0(sample, ".redux.bam"))
  output_bai1 <- paste0(output_bam, ".bai")
  output_bai2 <- sub("\\.bam$", ".bai", output_bam)
  bqr_file <- file.path(sample_dir, paste0(sample, ".redux.bqr.tsv"))
  jitter_file <- file.path(sample_dir, paste0(sample, ".redux.jitter_params.tsv"))
  ms_file <- file.path(sample_dir, paste0(sample, ".redux.ms_table.tsv.gz"))
  log_file <- file.path(sample_dir, paste0(sample, ".redux.log"))

  completed <- file.exists(output_bam) &&
    (file.exists(output_bai1) || file.exists(output_bai2)) &&
    file.exists(bqr_file) &&
    file.exists(jitter_file) &&
    file.exists(ms_file)

  if (completed) {
    cat("\nSKIP:", sample, "- already completed\n")
    results$Status[i] <- "Already completed"
    next
  }

  # Check/index input BAM
  input_bai1 <- paste0(bam, ".bai")
  input_bai2 <- sub("\\.bam$", ".bai", bam)

  if (!file.exists(input_bai1) && !file.exists(input_bai2)) {
    cat("\nIndexing:", sample, "\n")
    index_status <- system2(samtools_path, c("index", "-@", "24", bam))
    if (index_status != 0) stop("BAM indexing failed: ", sample)
  }

  cat("\n========================================\n")
  cat("Running REDUX:", sample, "\n")
  cat("Genome: V38 | Threads: 24\n")
  cat("========================================\n")

  status <- system2(
    java_bin,
    args = c(
      "-Xmx40G",
      "-jar", redux_jar,
      "-sample", sample,
      "-input_bam", bam,
      "-ref_genome", ref_fasta,
      "-ref_genome_version", "V38",
      "-sequencing_type", "ILLUMINA",
      "-unmap_regions", unmap_regions,
      "-ref_genome_msi_file", msi_file,
      "-form_consensus",
      "-bamtool", samtools_path,
      "-output_dir", sample_dir,
      "-log_level", "INFO",
      "-threads", "24"
    ),
    stdout = log_file,
    stderr = log_file
  )

  completed <- file.exists(output_bam) &&
    (file.exists(output_bai1) || file.exists(output_bai2)) &&
    file.exists(bqr_file) &&
    file.exists(jitter_file) &&
    file.exists(ms_file)

  if (status == 0 && completed) {
    cat("COMPLETED:", sample, "\n")
    results$Status[i] <- "Completed"
  } else {
    results$Status[i] <- "Failed"
    write.csv(results, file.path(output_base, "Redux_run_summary.csv"), row.names = FALSE)
    cat("\nFAILED:", sample, "\n")
    cat("Log:", log_file, "\n\n")
    log <- readLines(log_file, warn = FALSE)
    cat(paste(tail(log, 80), collapse = "\n"))
    stop("\nREDUX failed. Remaining samples were NOT started.")
  }

  write.csv(results, file.path(output_base, "Redux_run_summary.csv"), row.names = FALSE)
}

write.csv(results, file.path(output_base, "Redux_run_summary.csv"), row.names = FALSE)

cat("\n==============================\n")
cat("ALL REDUX SAMPLES FINISHED\n")
cat("==============================\n")
print(results)
```

---

## 25. Final project summary

```text
Tool:
REDUX v2.0.4

Input:
05_ASCAT_CN/BAMs/<SAMPLE>_tumor.bam

Current output:
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam

Reference FASTA:
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa

REDUX references:
unmap_regions.38.tsv
msi_jitter_sites.38.tsv.gz

Genome:
V38

Sequencing:
ILLUMINA

Consensus:
enabled

Threads:
24

Java heap:
40G

Failure policy:
stop immediately and do not start remaining samples

MSI-only extension:
06_REDux/MSI_only/

MSI final file:
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.msi_prediction.tsv

MSI coefficient model:
ms_model_coefficients.hmf_wgs.tsv

MSI error-rate model:
ms_model_error_rates.tso500.37.tsv
(experimental use per mentor instruction)
```
