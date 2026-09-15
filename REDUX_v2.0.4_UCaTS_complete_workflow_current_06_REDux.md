# REDUX v2.0.4 Complete Workflow — UCaTS Organoids (Current 06_REDux Layout)

> **Project:** UCaTS Organoid WGS  
> **Tool:** REDUX v2.0.4  
> **Mode:** tumor-only WGS preprocessing  
> **Genome:** GRCh38 / hg38  
> **Sequencing:** Illumina  
> **Threads:** 24  
> **Java heap:** 40G  
> **Input:** original tumor BAMs  
> **Output:** REDUX-processed BAMs and sidecar files

## 1. Purpose

This workflow runs HMFtools REDUX on all tumor BAMs in the UCaTS Organoid WGS dataset.

The code follows this structure:

```text
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
        |
        v
 downstream HMFtools analysis
```

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

## 8. Reference FASTA

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

## 9. samtools

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

## 10. Input BAM discovery

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

## 11. Input BAM index behavior

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

## 12. REDUX command used

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

## 13. `-form_consensus`

The supplied workflow explicitly enables:

```text
-form_consensus
```

This is part of the REDUX processing used in this project.

The Markdown preserves this parameter exactly because it is part of the actual code being documented.

---

## 14. REDUX output files

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

## 15. Completion rule

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

## 16. Resume / skip logic

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

## 17. Failure behavior

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

## 18. Project-level summary

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

## 19. Expected directory structure

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
        │   └── I_26166_S_29146.redux.log
        │
        └── ...
```

Shared reference:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/
├── hg38.fa
└── hg38.fa.fai
```

---

## 20. Downstream use

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

## 21. Reproducibility checklist

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

[ ] per-sample REDUX log written
[ ] Redux_run_summary.csv written
```

---

## 22. Final complete R code

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

## 23. Final project summary

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
```
