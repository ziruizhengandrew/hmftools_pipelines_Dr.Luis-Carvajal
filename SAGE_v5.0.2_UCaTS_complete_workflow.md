# SAGE v5.0.2 Complete Workflow — UCaTS Organoids

> **Project:** UCaTS Organoid WGS  
> **Pipeline stage:** `REDUX -> SAGE -> PAVE -> PURPLE`  
> **Input:** REDUX BAM only  
> **Genome:** GRCh38 / hg38  
> **Sequencing:** Illumina WGS, tumor-only  
> **SAGE version:** `v5.0.2`  
> **HMF resource bundle:** `hmf_pipeline_resources.38_v3.0.0--8`

## 1. Purpose

SAGE is the HMFtools small-variant caller used here to call tumor SNVs and small INDELs from REDUX-processed BAMs.

The project workflow is:

```text
Original tumor BAM
        |
        v
      REDUX
        |
        v
<SAMPLE>.redux.bam
        |
        v
     SAGE 5.0.2
        |
        v
<SAMPLE>.sage.somatic.vcf.gz
        |
        v
      PAVE
        |
        v
     PURPLE
```

A key rule for this project is:

```text
SAGE input = REDUX BAM only
```

Do **not** use the original BAMs from:

```text
05_ASCAT_CN/BAMs/
```

Official SAGE documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/sage/README.md
```

---

## 2. Current project directories

Project root:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids
```

REDUX input:

```text
06_REDux/Redux_BAMs/
```

SAGE project:

```text
07_SAGE/
├── SAGE_tools/
├── SAGE_reference/
└── SAGE_output/
```

Current tool:

```text
07_SAGE/SAGE_tools/sage_v5.0.2.jar
```

Current output:

```text
07_SAGE/SAGE_output/<SAMPLE>/<SAMPLE>.sage.somatic.vcf.gz
```

---

## 3. Java

Current Java:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Check:

```bash
/home/zzr123/.conda/envs/purple/bin/java -version
```

If a separate Java environment is ever needed:

```bash
conda create   -n sage_java   -c conda-forge   openjdk=21   -y

conda activate sage_java

java -version
```

For the actual project workflow, keep using the explicit working Java path:

```r
JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"
```

---

## 4. Download SAGE v5.0.2

Create the tool directory:

```bash
BASE="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"

mkdir -p "$BASE/07_SAGE/SAGE_tools" "$BASE/07_SAGE/SAGE_reference" "$BASE/07_SAGE/SAGE_output"
```

The production code expects:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/
07_SAGE/SAGE_tools/sage_v5.0.2.jar
```

A reproducible way to resolve the JAR from the fixed GitHub release tag is:

```bash
set -euo pipefail

TOOL_DIR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/07_SAGE/SAGE_tools"
TAG="sage-v5.0.2"

mkdir -p "$TOOL_DIR"

ASSET_URL="$(
python3 - "$TAG" <<'PY'
import json
import sys
import urllib.request

tag = sys.argv[1]

api = (
    "https://api.github.com/repos/"
    "hartwigmedical/hmftools/releases/tags/"
    + tag
)

with urllib.request.urlopen(api) as r:
    release = json.load(r)

jars = [
    a["browser_download_url"]
    for a in release.get("assets", [])
    if a["name"].lower().endswith(".jar")
    and "sage" in a["name"].lower()
    and "source" not in a["name"].lower()
    and "javadoc" not in a["name"].lower()
]

if len(jars) != 1:
    raise SystemExit(
        f"Expected one SAGE JAR for {tag}, found: {jars}"
    )

print(jars[0])
PY
)"

echo "$ASSET_URL"

curl   -L   --fail   "$ASSET_URL"   -o "$TOOL_DIR/sage_v5.0.2.jar"

ls -lh "$TOOL_DIR/sage_v5.0.2.jar"
sha256sum "$TOOL_DIR/sage_v5.0.2.jar"
```

Before production use, record the JAR checksum:

```bash
sha256sum /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/07_SAGE/SAGE_tools/sage_v5.0.2.jar
```

---

## 5. HMF reference bundle

This workflow uses:

```text
hmf_pipeline_resources.38_v3.0.0--8
```

The code searches recursively inside:

```text
07_SAGE/SAGE_reference/
```

for:

```text
KnownHotspots.somatic.38.vcf.gz
DriverGenePanel.38.tsv
HG001_GRCh38_GIAB_highconf*.bed.gz
ensembl_gene_data.csv
```

The `ensembl_gene_data.csv` location is used to identify the corresponding:

```text
ensembl_data/
```

directory.

---

## 6. Download HMF resources 3.0.0--8

Create the reference directory:

```bash
REF_DIR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/07_SAGE/SAGE_reference"

mkdir -p "$REF_DIR"
cd "$REF_DIR"
```

Download:

```bash
wget -c https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/hmf_pipeline_resources.38_v3.0.0--8.tar.gz
```

Extract:

```bash
tar -xzf hmf_pipeline_resources.38_v3.0.0--8.tar.gz
```

Check the required SAGE resources:

```bash
find "$REF_DIR" -type f | grep -E 'KnownHotspots\.somatic\.38\.vcf\.gz$|DriverGenePanel\.38\.tsv$|HG001_GRCh38_GIAB_highconf.*\.bed\.gz$|ensembl_gene_data\.csv$'
```

The lab also has the shared bundle:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
```

For reproducibility, do not mix files from different HMF resource versions in the same run.

---

## 7. Reference FASTA

The code uses:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

Required:

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
samtools faidx /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

Do not replace this FASTA with an arbitrary GRCh38 reference just because it has the same assembly label.

The REDUX BAM and SAGE FASTA need to be compatible.

---

## 8. REDUX BAM input

The code searches recursively under:

```text
06_REDux/Redux_BAMs
```

for files matching:

```text
*.redux.bam
```

Example:

```text
06_REDux/Redux_BAMs/I_26166_S_29146/
I_26166_S_29146.redux.bam
```

The sample ID is derived from the BAM filename:

```r
samples <- sub("\\.redux\\.bam$", "", basename(redux_bams))
samples <- sub("_tumor$", "", samples)
```

The safety check:

```r
if(any(!grepl("\\.redux\\.bam$", redux_bams)))
  stop("ERROR: Non-REDUX BAM detected.")
```

prevents the original BAMs from entering the SAGE workflow.

---

## 9. Tumor-only configuration

The command uses:

```text
-tumor <SAMPLE>
-tumor_bam <REDUX_BAM>
-ref_sample_count 0
```

No matched normal is supplied.

This is therefore a tumor-only SAGE run.

---

## 10. SAGE resources used by the command

The final command supplies:

```text
-hotspots
KnownHotspots.somatic.38.vcf.gz

-high_confidence_bed
HG001_GRCh38_GIAB_highconf*.bed.gz

-ensembl_data_dir
<directory containing ensembl_gene_data.csv>

-driver_gene_panel
DriverGenePanel.38.tsv
```

Genome settings:

```text
-ref_genome_version 38
-ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

Compute settings:

```text
-Xmx32G
-threads 16
```

---

## 11. Output behavior

Each sample gets:

```text
07_SAGE/SAGE_output/<SAMPLE>/
```

with:

```text
<SAMPLE>.sage.somatic.vcf.gz
<SAMPLE>.sage.log
<SAMPLE>.sage.err
```

Project-level summary:

```text
07_SAGE/SAGE_output/SAGE_run_summary.tsv
```

---

## 12. Resume / skip logic

The code considers a SAGE sample complete when:

```text
<SAMPLE>.sage.somatic.vcf.gz
```

exists and is larger than:

```text
1000 bytes
```

Completed samples are labeled:

```text
SKIPPED_COMPLETED
```

and are not rerun.

---

## 13. BAM index behavior

The code checks for:

```text
<SAMPLE>.redux.bam.bai
```

If absent, it prints a warning:

```text
WARNING: BAM index missing
```

The current supplied code does **not** create the missing BAM index automatically.

Therefore BAM indices should be prepared before starting SAGE.

Example:

```bash
samtools index /path/to/<SAMPLE>.redux.bam
```

---

## 14. Logging

Per sample:

```text
stdout:
<SAMPLE>.sage.log

stderr:
<SAMPLE>.sage.err
```

The return status from Java is captured by:

```r
status <- system2(...)
```

A successful sample requires:

```text
status == 0
AND
output VCF exists
AND
output VCF > 1000 bytes
```

---

## 15. Final complete R code

The code below is kept consistent with the supplied SAGE v5.0.2 workflow.

```r
# =========================
# SAGE ALL SAMPLES
# Tumor-only WGS
# ONLY use REDUX BAM
# SAGE 5.0.2 + HMF resources 3.0.0--8
# =========================

ROOT <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"
REDUX_DIR <- file.path(ROOT, "06_REDux/Redux_BAMs")
SAGE_BASE <- file.path(ROOT, "07_SAGE")
TOOL_DIR <- file.path(SAGE_BASE, "SAGE_tools")
REF_DIR <- file.path(SAGE_BASE, "SAGE_reference")
OUT_BASE <- file.path(SAGE_BASE, "SAGE_output")
dir.create(OUT_BASE, recursive = TRUE, showWarnings = FALSE)

JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"
SAGE_JAR <- file.path(TOOL_DIR, "sage_v5.0.2.jar")
FASTA <- "/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"

HOTSPOTS <- list.files(REF_DIR, pattern="KnownHotspots\\.somatic\\.38\\.vcf\\.gz$", recursive=TRUE, full.names=TRUE)[1]
DRIVER <- list.files(REF_DIR, pattern="DriverGenePanel\\.38\\.tsv$", recursive=TRUE, full.names=TRUE)[1]
HIGHCONF <- list.files(REF_DIR, pattern="HG001_GRCh38_GIAB_highconf.*\\.bed\\.gz$", recursive=TRUE, full.names=TRUE)[1]
ENSEMBL_FILE <- list.files(REF_DIR, pattern="ensembl_gene_data\\.csv$", recursive=TRUE, full.names=TRUE)[1]
ENSEMBL_DIR <- dirname(ENSEMBL_FILE)

cat("===== REFERENCE CHECK =====\n")
cat("JAVA:      ", JAVA, "\n")
cat("SAGE:      ", SAGE_JAR, "\n")
cat("FASTA:     ", FASTA, "\n")
cat("HOTSPOTS:  ", HOTSPOTS, "\n")
cat("DRIVER:    ", DRIVER, "\n")
cat("HIGHCONF:  ", HIGHCONF, "\n")
cat("ENSEMBL:   ", ENSEMBL_DIR, "\n\n")

stopifnot(
  file.exists(JAVA),
  file.exists(SAGE_JAR),
  file.exists(FASTA),
  file.exists(paste0(FASTA, ".fai")),
  file.exists(HOTSPOTS),
  file.exists(DRIVER),
  file.exists(HIGHCONF),
  dir.exists(ENSEMBL_DIR)
)

# Find ONLY REDUX BAMs
redux_bams <- list.files(
  REDUX_DIR,
  pattern="\\.redux\\.bam$",
  recursive=TRUE,
  full.names=TRUE
)

if(length(redux_bams) == 0) stop("No REDUX BAMs found.")

samples <- sub("\\.redux\\.bam$", "", basename(redux_bams))
samples <- sub("_tumor$", "", samples)

cat("Found", length(redux_bams), "REDUX BAMs\n")
print(data.frame(Sample=samples, BAM=redux_bams))

# Safety check: absolutely no original BAM
if(any(!grepl("\\.redux\\.bam$", redux_bams))) stop("ERROR: Non-REDUX BAM detected.")

results <- data.frame(
  Sample=character(),
  Status=character(),
  Runtime_min=numeric(),
  Output=character(),
  stringsAsFactors=FALSE
)

for(i in seq_along(redux_bams)) {

  BAM <- redux_bams[i]
  SAMPLE <- samples[i]
  OUT_DIR <- file.path(OUT_BASE, SAMPLE)
  dir.create(OUT_DIR, recursive=TRUE, showWarnings=FALSE)

  OUTPUT_VCF <- file.path(OUT_DIR, paste0(SAMPLE, ".sage.somatic.vcf.gz"))
  LOG_FILE <- file.path(OUT_DIR, paste0(SAMPLE, ".sage.log"))
  ERR_FILE <- file.path(OUT_DIR, paste0(SAMPLE, ".sage.err"))

  cat("\n========================================\n")
  cat("[", i, "/", length(redux_bams), "] ", SAMPLE, "\n", sep="")
  cat("BAM: ", BAM, "\n", sep="")

  # Skip completed samples
  if(file.exists(OUTPUT_VCF) && file.info(OUTPUT_VCF)$size > 1000) {
    cat("SKIP: SAGE output already exists\n")
    results <- rbind(results, data.frame(
      Sample=SAMPLE,
      Status="SKIPPED_COMPLETED",
      Runtime_min=0,
      Output=OUTPUT_VCF
    ))
    next
  }

  # Confirm REDUX BAM and index
  if(!file.exists(BAM)) {
    cat("FAILED: BAM missing\n")
    results <- rbind(results, data.frame(
      Sample=SAMPLE,
      Status="FAILED_BAM_MISSING",
      Runtime_min=NA,
      Output=OUTPUT_VCF
    ))
    next
  }

  if(!grepl("\\.redux\\.bam$", BAM)) stop("ERROR: Original BAM detected: ", BAM)

  if(!file.exists(paste0(BAM, ".bai"))) {
    cat("WARNING: BAM index missing: ", paste0(BAM, ".bai"), "\n")
  }

  cat("Starting SAGE...\n")

  args <- c(
    "-Xmx32G",
    "-jar", SAGE_JAR,
    "-tumor", SAMPLE,
    "-tumor_bam", BAM,
    "-ref_sample_count", "0",
    "-ref_genome_version", "38",
    "-ref_genome", FASTA,
    "-hotspots", HOTSPOTS,
    "-high_confidence_bed", HIGHCONF,
    "-ensembl_data_dir", ENSEMBL_DIR,
    "-driver_gene_panel", DRIVER,
    "-output_vcf", OUTPUT_VCF,
    "-threads", "16"
  )

  start <- Sys.time()

  status <- tryCatch(
    system2(
      command=JAVA,
      args=args,
      stdout=LOG_FILE,
      stderr=ERR_FILE
    ),
    error=function(e) {
      cat("ERROR:", conditionMessage(e), "\n")
      return(999)
    }
  )

  runtime <- round(as.numeric(difftime(Sys.time(), start, units="mins")), 1)

  if(status == 0 && file.exists(OUTPUT_VCF) && file.info(OUTPUT_VCF)$size > 1000) {
    cat("SUCCESS:", SAMPLE, "\n")
    cat("Runtime:", runtime, "minutes\n")
    result_status <- "SUCCESS"
  } else {
    cat("FAILED:", SAMPLE, "\n")
    cat("Check:", ERR_FILE, "\n")
    result_status <- paste0("FAILED_", status)
  }

  results <- rbind(results, data.frame(
    Sample=SAMPLE,
    Status=result_status,
    Runtime_min=runtime,
    Output=OUTPUT_VCF
  ))

  write.table(
    results,
    file=file.path(OUT_BASE, "SAGE_run_summary.tsv"),
    sep="\t",
    quote=FALSE,
    row.names=FALSE
  )
}

cat("\n\n========== SAGE COMPLETE ==========\n")
print(results)

cat("\nSummary:\n")
print(table(results$Status))

cat("\nResults saved to:\n")
cat(file.path(OUT_BASE, "SAGE_run_summary.tsv"), "\n")

```

---

## 16. Expected directory structure

```text
UCaTS_Organoids/
│
├── 06_REDux/
│   └── Redux_BAMs/
│       ├── I_26166_S_29146/
│       │   ├── I_26166_S_29146.redux.bam
│       │   └── I_26166_S_29146.redux.bam.bai
│       └── ...
│
├── 07_SAGE/
│   ├── SAGE_tools/
│   │   └── sage_v5.0.2.jar
│   │
│   ├── SAGE_reference/
│   │   └── hmf_pipeline_resources.38_v3.0.0--8/
│   │       ├── .../KnownHotspots.somatic.38.vcf.gz
│   │       ├── .../DriverGenePanel.38.tsv
│   │       ├── .../HG001_GRCh38_GIAB_highconf*.bed.gz
│   │       └── .../ensembl_data/
│   │
│   └── SAGE_output/
│       ├── SAGE_run_summary.tsv
│       ├── I_26166_S_29146/
│       │   ├── I_26166_S_29146.sage.somatic.vcf.gz
│       │   ├── I_26166_S_29146.sage.log
│       │   └── I_26166_S_29146.sage.err
│       └── ...
│
└── reference/
    └── hg38.fa
        hg38.fa.fai
```

---

## 17. Downstream use

The current downstream chain is:

```text
SAGE
  |
  v
<SAMPLE>.sage.somatic.vcf.gz
  |
  v
PAVE
  |
  v
<SAMPLE>.sage.somatic.pave.vcf.gz
  |
  v
PURPLE
```

Do not feed an unannotated or wrong SAGE VCF into the later PURPLE workflow when the current pipeline is designed around the PAVE-annotated result.

---

## 18. Reproducibility checklist

```text
[ ] Java path recorded
[ ] Java version recorded

[ ] SAGE v5.0.2 JAR exists
[ ] SAGE JAR SHA256 recorded

[ ] HMF resources = 38_v3.0.0--8
[ ] KnownHotspots.somatic.38.vcf.gz exists
[ ] DriverGenePanel.38.tsv exists
[ ] HG001_GRCh38_GIAB_highconf*.bed.gz exists
[ ] Ensembl data directory exists

[ ] hg38.fa exists
[ ] hg38.fa.fai exists
[ ] FASTA is compatible with REDUX BAM

[ ] input BAMs all end in .redux.bam
[ ] no original BAM enters SAGE
[ ] BAM indices are present

[ ] tumor-only uses ref_sample_count 0
[ ] Java heap = 32G
[ ] threads = 16

[ ] per-sample stdout log saved
[ ] per-sample stderr log saved
[ ] SAGE_run_summary.tsv saved
```

---

## 19. Project-specific summary

```text
Tool:
SAGE v5.0.2

Input:
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam

Reference:
Illumina/Dragen/hg38.fa

Resource bundle:
hmf_pipeline_resources.38_v3.0.0--8

Output:
07_SAGE/SAGE_output/<SAMPLE>/<SAMPLE>.sage.somatic.vcf.gz

Mode:
tumor-only

Matched normal:
none

Threads:
16

Java heap:
32G
```
