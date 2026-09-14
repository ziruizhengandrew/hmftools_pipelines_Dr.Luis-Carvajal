---
title: "BamMetrics v1.6.2 Workflow — UCaTS Organoids Tumor WGS"
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

> **Scope.** This notebook documents the BamMetrics step used for the UCaTS Organoids WGS cohort, including software installation, reference requirements, exact project paths, the original analysis code, expected outputs, QC interpretation, downstream QSEE use, troubleshooting, and export.
>
> **Important.** The BamMetrics analysis code in this document is intentionally kept **exactly as used**. It uses the original tumor BAMs in `05_ASCAT_CN/BAMs`, not REDUX BAMs. No parameters or paths inside the analysis code have been changed.
>
> **Export safety.** All executable chunks are `eval=FALSE`, so knitting/exporting this notebook will not rerun BamMetrics.

# 1. What BamMetrics does

HMFtools BamMetrics is part of the `bam-tools` package. It computes QC statistics directly from a BAM file, including:

```text
overall BAM/read statistics
coverage summary
coverage distribution
flag counts
fragment-length distribution
partition-level read statistics
gene coverage / median exon depth when the optional gene configuration is supplied
```

For this project, BamMetrics provides QC files later consumed by QSEE and compared against DRAGEN QC.

# 2. Software version used

This project uses:

```text
HMFtools bam-tools v1.6.2
```

JAR path:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BamMetrics_tools/bam-tools_v1.6.2.jar
```

Java:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

The official HMFtools GitHub releases include the `bam-tools-v1.6.2` release tag.

Official repository:

```text
https://github.com/hartwigmedical/hmftools/tree/master/bam-tools
```

Official releases:

```text
https://github.com/hartwigmedical/hmftools/releases
```

# 3. Project directory structure

Current BamMetrics area:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/
├── BAMs/
│   ├── <SAMPLE>_tumor.bam
│   └── <SAMPLE>_tumor.bam.bai or <SAMPLE>_tumor.bai
├── BamMetrics_tools/
│   └── bam-tools_v1.6.2.jar
└── BamMetrics_output/
    ├── <SAMPLE>/
    │   ├── <SAMPLE>.bam_metric.summary.tsv
    │   ├── <SAMPLE>.bam_metric.coverage.tsv
    │   ├── <SAMPLE>.bam_metric.flag_counts.tsv
    │   ├── <SAMPLE>.bam_metric.frag_length.tsv
    │   └── <SAMPLE>.bammetrics.log
    └── BamMetrics_run_status.tsv
```

The project code uses the **original tumor BAMs**:

```text
05_ASCAT_CN/BAMs/<SAMPLE>_tumor.bam
```

This is intentional in the documented workflow and is not changed here.

# 4. Reference genome requirement

The BamMetrics documentation describes `-ref_genome` as the reference genome used to create the BAM.

Therefore, the safest rule is:

```text
Use the exact same genome build/reference sequence that was used to align the BAM.
```

For this cohort the shared reference is:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

and the command specifies:

```text
-ref_genome_version V38
```

## 4.1 Current reference path

```bash
REF="/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"

ls -lh "$REF"
ls -lh "$REF.fai"
```

The `.fai` FASTA index is recommended and is also used by multiple other pipeline components.

If the index is missing:

```bash
samtools faidx /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

## 4.2 Do not replace this with an arbitrary hg38 FASTA

For an existing aligned cohort, do **not** download a random GRCh38/hg38 FASTA and substitute it only because it has the same genome-build label.

Different GRCh38 distributions may differ in:

```text
contig naming
decoy/alternate contigs
masking
sequence content
reference dictionary
```

BamMetrics should use the reference compatible with the BAM alignment.

For this project, that is the shared DRAGEN reference above.

# 5. Fresh setup: obtaining the reference

For the current HIVE project, the reference already exists centrally and should be reused:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

That is preferable to making another large copy.

If rebuilding the entire analysis environment from scratch, obtain the **same GRCh38 reference package used by the alignment/DRAGEN workflow**, place the FASTA at a stable shared path, and index it with:

```bash
samtools faidx hg38.fa
```

Then verify:

```bash
ls -lh hg38.fa
ls -lh hg38.fa.fai
```

For reproducibility, record:

```text
reference source/package
reference version
FASTA filename
FASTA SHA256
FAI SHA256
```

Example:

```bash
sha256sum hg38.fa
sha256sum hg38.fa.fai
```

# 6. Shared HMFtools resource bundle

The current BamMetrics code does **not** use the HMFtools bundle directly.

However, the lab's WGS pipeline uses the shared GRCh38 HMFtools resources:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8/
```

Bundle:

```text
hmf_pipeline_resources.38_v3.0.0--8.tar.gz
```

If the shared bundle is not already installed, the project previously staged it from:

```text
https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/hmf_pipeline_resources.38_v3.0.0--8.tar.gz
```

Example:

```bash
set -euo pipefail

REF_BASE="/quobyte/luisccgrp/REFERENCE_DATA/hmftools"
BUNDLE="hmf_pipeline_resources.38_v3.0.0--8.tar.gz"

mkdir -p "$REF_BASE"
cd "$REF_BASE"

wget -c "https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/$BUNDLE"

tar -xzvf "$BUNDLE"

ls -lh "$REF_BASE/hmf_pipeline_resources.38_v3.0.0--8/common/DriverGenePanel.38.tsv"
```

This bundle is shared by PURPLE, QSEE, and other HMFtools components.

# 7. Optional gene-coverage resources

The current BamMetrics script does **not** request gene coverage and should not be changed for this documented run.

That is why QSEE later reported:

```text
<SAMPLE>.bam_metric.gene_coverage.tsv
```

as missing for all 23 samples.

This did **not** invalidate QSEE, because gene coverage was optional in the QSEE workflow.

The HMFtools BamMetrics documentation states that gene coverage / median exon depths can be produced when the appropriate driver-gene and Ensembl resources are configured.

The relevant shared resources are already available in the HMFtools bundle:

```text
Driver gene panel:
.../common/DriverGenePanel.38.tsv

Ensembl data:
.../common/ensembl_data
```

For this project, these resources are documented here for completeness only. They were **not added to the BamMetrics command**, because the user requested that the existing BamMetrics code remain unchanged.

# 8. Downloading bam-tools v1.6.2

For reproducibility, use an explicit release tag rather than downloading an unspecified "latest" JAR.

Release tag:

```text
bam-tools-v1.6.2
```

A robust fresh-install method is to resolve the JAR asset from the GitHub release API:

```bash
set -euo pipefail

BASE="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN"
TOOL_DIR="$BASE/BamMetrics_tools"
TAG="bam-tools-v1.6.2"

mkdir -p "$TOOL_DIR"
cd "$TOOL_DIR"

ASSET_URL="$(
python3 - "$TAG" <<'PY'
import json
import sys
import urllib.request

tag = sys.argv[1]
api = f"https://api.github.com/repos/hartwigmedical/hmftools/releases/tags/{tag}"

with urllib.request.urlopen(api) as response:
    release = json.load(response)

jars = [
    asset["browser_download_url"]
    for asset in release["assets"]
    if asset["name"].endswith(".jar")
]

if len(jars) != 1:
    raise SystemExit(
        f"Expected exactly one JAR asset for {tag}, found {len(jars)}: {jars}"
    )

print(jars[0])
PY
)"

echo "Resolved asset:"
echo "$ASSET_URL"

curl -L --fail "$ASSET_URL" -o "$TOOL_DIR/bam-tools_v1.6.2.jar"

ls -lh "$TOOL_DIR/bam-tools_v1.6.2.jar"

sha256sum "$TOOL_DIR/bam-tools_v1.6.2.jar"
```

After installation, the expected project path is:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BamMetrics_tools/bam-tools_v1.6.2.jar
```

# 9. Check the installed tool

```bash
JAVA="/home/zzr123/.conda/envs/purple/bin/java"
JAR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BamMetrics_tools/bam-tools_v1.6.2.jar"

"$JAVA" -version

ls -lh "$JAR"

sha256sum "$JAR"
```

The exact checksum from the production copy should be retained with the executed notebook/logs.

# 10. BAM and BAM-index requirements

The script searches for:

```text
*_tumor.bam
```

under:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BAMs
```

For each BAM it accepts either index naming convention:

```text
<SAMPLE>_tumor.bam.bai
```

or:

```text
<SAMPLE>_tumor.bai
```

If an index is absent, the sample is skipped with:

```text
SKIPPED_NO_BAI
```

If an index has to be created:

```bash
samtools index /path/to/<SAMPLE>_tumor.bam
```

# 11. Exact BamMetrics analysis code used

The following code is reproduced **unchanged** from the project workflow.

```r
base <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN"
bam_dir <- file.path(base, "BAMs")
output_base <- file.path(base, "BamMetrics_output")
bamtools_jar <- file.path(base, "BamMetrics_tools", "bam-tools_v1.6.2.jar")
java_bin <- "/home/zzr123/.conda/envs/purple/bin/java"
ref_fasta <- "/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"
threads <- 8
dir.create(output_base, recursive = TRUE, showWarnings = FALSE)

if (!file.exists(bamtools_jar)) stop("BamMetrics JAR not found")
if (!file.exists(java_bin)) stop("Java not found")
if (!file.exists(ref_fasta)) stop("hg38 reference not found")

bam_files <- list.files(bam_dir, pattern = "_tumor\\.bam$", full.names = TRUE)
sample_ids <- sub("_tumor\\.bam$", "", basename(bam_files))
cat("Found", length(bam_files), "tumor BAM files\n")

results <- data.frame(Sample=sample_ids, Status=NA_character_)

for (i in seq_along(bam_files)) {
  sample_id <- sample_ids[i]
  bam_file <- bam_files[i]
  output_dir <- file.path(output_base, sample_id)
  complete_file <- file.path(output_dir, paste0(sample_id, ".bam_metric.flag_counts.tsv"))
  bam_index1 <- paste0(bam_file, ".bai")
  bam_index2 <- sub("\\.bam$", ".bai", bam_file)

  if (file.exists(complete_file) && file.info(complete_file)$size > 0) {
    cat("[", i, "/", length(bam_files), "] SKIP:", sample_id, "- already completed\n")
    results$Status[i] <- "SKIPPED_COMPLETED"
    next
  }

  if (!file.exists(bam_index1) && !file.exists(bam_index2)) {
    cat("[", i, "/", length(bam_files), "] SKIP:", sample_id, "- BAM index missing\n")
    results$Status[i] <- "SKIPPED_NO_BAI"
    next
  }

  dir.create(output_dir, recursive = TRUE, showWarnings = FALSE)
  log_file <- file.path(output_dir, paste0(sample_id, ".bammetrics.log"))
  cat("[", i, "/", length(bam_files), "] RUN:", sample_id, "\n")

  args <- c("-Xmx16G", "-cp", bamtools_jar, "com.hartwig.hmftools.bamtools.metrics.BamMetrics", "-sample", sample_id, "-bam_file", bam_file, "-ref_genome", ref_fasta, "-ref_genome_version", "V38", "-output_dir", output_dir, "-threads", as.character(threads))
  status <- system2(java_bin, args=args, stdout=log_file, stderr=log_file)

  if (status == 0 && file.exists(complete_file) && file.info(complete_file)$size > 0) {
    cat("DONE:", sample_id, "\n")
    results$Status[i] <- "COMPLETED"
  } else {
    cat("FAILED:", sample_id, "- check", log_file, "\n")
    results$Status[i] <- "FAILED"
  }
}

write.table(results, file.path(output_base, "BamMetrics_run_status.tsv"), sep="\t", row.names=FALSE, quote=FALSE)
print(table(results$Status))
```

# 12. What the script does

The code performs the following sequence:

```text
1. Defines the BAM, output, JAR, Java, and reference paths.
2. Checks that the BamMetrics JAR exists.
3. Checks that Java exists.
4. Checks that hg38.fa exists.
5. Discovers all *_tumor.bam files.
6. Derives sample IDs from BAM filenames.
7. Iterates through samples sequentially.
8. Checks for an existing non-empty flag-count output.
9. Skips samples already completed.
10. Checks for a BAM index.
11. Skips a sample when no BAM index exists.
12. Creates a per-sample output directory.
13. Runs BamMetrics using 16 GB Java heap and 8 threads.
14. Captures stdout/stderr into a per-sample log.
15. Requires exit status 0 plus a non-empty flag-count TSV.
16. Writes the final run-status table.
```

# 13. Exact command generated for one sample

Conceptually, each sample resolves to:

```bash
/home/zzr123/.conda/envs/purple/bin/java -Xmx16G -cp /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BamMetrics_tools/bam-tools_v1.6.2.jar com.hartwig.hmftools.bamtools.metrics.BamMetrics -sample <SAMPLE> -bam_file /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BAMs/<SAMPLE>_tumor.bam -ref_genome /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa -ref_genome_version V38 -output_dir /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BamMetrics_output/<SAMPLE> -threads 8
```

The R code uses `system2()` to build and execute this command.

# 14. Memory and CPU configuration

The code uses:

```text
Java heap:
-Xmx16G

BamMetrics threads:
8
```

This means the code runs one sample at a time, and each sample can use up to eight BamMetrics threads.

The current script is sequential; it does not launch 23 samples simultaneously.

# 15. Completion / skip logic

The script considers a sample complete when:

```text
<SAMPLE>.bam_metric.flag_counts.tsv
```

exists and has size greater than zero.

Therefore:

```text
existing non-empty flag_counts
    -> SKIPPED_COMPLETED

missing BAM index
    -> SKIPPED_NO_BAI

exit status 0 + non-empty flag_counts
    -> COMPLETED

otherwise
    -> FAILED
```

This is the completion rule used in the existing analysis code and is intentionally not changed here.

# 16. Expected BamMetrics outputs

Typical files used in this project include:

```text
<SAMPLE>.bam_metric.summary.tsv
<SAMPLE>.bam_metric.coverage.tsv
<SAMPLE>.bam_metric.flag_counts.tsv
<SAMPLE>.bam_metric.frag_length.tsv
```

The current run does not produce:

```text
<SAMPLE>.bam_metric.gene_coverage.tsv
```

because the gene-coverage configuration was not supplied.

Per-sample log:

```text
<SAMPLE>.bammetrics.log
```

Global status table:

```text
BamMetrics_run_status.tsv
```

# 17. BamMetrics outputs used by QSEE

The current QSEE workflow searches for:

```text
<SAMPLE>.bam_metric.summary.tsv
<SAMPLE>.bam_metric.coverage.tsv
<SAMPLE>.bam_metric.flag_counts.tsv
<SAMPLE>.bam_metric.frag_length.tsv
<SAMPLE>.bam_metric.gene_coverage.tsv
```

In the validated 23-sample QSEE run:

```text
summary          23/23 found
coverage         23/23 found
flag_counts      23/23 found
frag_length      23/23 found
gene_coverage     0/23 found
```

QSEE's mandatory tumor-only BamMetrics inputs were present, and the missing gene-coverage files were accepted as optional.

# 18. Important BamMetrics QC metrics

## 18.1 Summary

The summary file provides high-level BAM QC measurements.

Examples include overall sequencing/read metrics and coverage-related summaries.

## 18.2 Coverage

```text
<SAMPLE>.bam_metric.coverage.tsv
```

describes the genome-wide coverage distribution.

This is useful when comparing HMF/QSEE results with DRAGEN WGS coverage QC.

## 18.3 Flag counts

```text
<SAMPLE>.bam_metric.flag_counts.tsv
```

summarizes read counts by SAM/BAM flag category.

This is also the file used by the existing script as the completion sentinel.

## 18.4 Fragment length

```text
<SAMPLE>.bam_metric.frag_length.tsv
```

contains the fragment-length distribution.

## 18.5 Gene coverage

```text
<SAMPLE>.bam_metric.gene_coverage.tsv
```

is optional in the current pipeline and is absent in this run.

# 19. Relationship to DRAGEN QC

BamMetrics and DRAGEN calculate overlapping but not always identically defined QC measurements.

Useful comparisons include:

```text
BamMetrics coverage
    ↔ DRAGEN WGS average coverage / coverage distribution

BamMetrics flag counts
    ↔ DRAGEN mapping/read-count metrics

BamMetrics fragment length
    ↔ DRAGEN insert-length metrics

QSEE warnings derived from BamMetrics
    ↔ corresponding DRAGEN QC metrics
```

Do not assume the tools use exactly the same thresholds or definitions. Compare both the raw metric definition and its value.

# 20. Relationship to REDUX

This point is important because other parts of the HMF pipeline use REDUX BAMs.

For SAGE and later HMF processing, the project used REDUX-processed BAMs.

However, **this documented BamMetrics script uses the original tumor BAMs**:

```text
05_ASCAT_CN/BAMs/<SAMPLE>_tumor.bam
```

The code is preserved unchanged as requested.

Therefore, when interpreting QSEE's BamMetrics-derived QC, remember that those BamMetrics values correspond to these original tumor BAMs.

# 21. Recommended preflight checks before a future rerun

Without changing the existing analysis code, these shell checks can be run separately before execution:

```bash
BASE="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN"
BAM_DIR="$BASE/BAMs"
JAR="$BASE/BamMetrics_tools/bam-tools_v1.6.2.jar"
REF="/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"
JAVA="/home/zzr123/.conda/envs/purple/bin/java"

echo "===== JAVA ====="
"$JAVA" -version

echo
echo "===== JAR ====="
ls -lh "$JAR"
sha256sum "$JAR"

echo
echo "===== REFERENCE ====="
ls -lh "$REF"
ls -lh "$REF.fai" 2>/dev/null || true
sha256sum "$REF"

echo
echo "===== BAM COUNT ====="
find "$BAM_DIR" -maxdepth 1 -type f -name '*_tumor.bam' | wc -l

echo
echo "===== BAM INDEX CHECK ====="
for bam in "$BAM_DIR"/*_tumor.bam; do
    bai1="${bam}.bai"
    bai2="${bam%.bam}.bai"

    if [[ -s "$bai1" || -s "$bai2" ]]; then
        echo "PASS $(basename "$bam")"
    else
        echo "MISSING_BAI $(basename "$bam")"
    fi
done
```

This does not alter the BamMetrics R code.

# 22. Provenance recommendations

The original BamMetrics script predates the stricter provenance format later adopted for PURPLE/QSEE.

For future runs, preserve the original code but also save, externally if necessary:

```text
1. bam-tools exact version
2. JAR path
3. JAR SHA256
4. Java version
5. BAM directory
6. BAM filenames
7. BAM index status
8. reference FASTA path
9. FASTA SHA256
10. exact R script/Rmd used
11. per-sample BamMetrics logs
12. final BamMetrics_run_status.tsv
13. run date/time
14. R sessionInfo()
```

Historical runs should not be described as having provenance files that were not actually captured at the time.

# 23. Troubleshooting

## BamMetrics JAR not found

Check:

```bash
ls -lh /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/05_ASCAT_CN/BamMetrics_tools/
```

Expected:

```text
bam-tools_v1.6.2.jar
```

## Reference not found

Check:

```bash
ls -lh /quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

## BAM index missing

Create it with:

```bash
samtools index SAMPLE_tumor.bam
```

## A sample says FAILED

Open:

```text
05_ASCAT_CN/BamMetrics_output/<SAMPLE>/<SAMPLE>.bammetrics.log
```

and inspect the last lines:

```bash
tail -100 /path/to/<SAMPLE>.bammetrics.log
```

## Gene coverage is missing

This is expected for the current code. The existing BamMetrics command does not configure driver-gene/Ensembl gene-coverage inputs.

Do not interpret this as a failed BamMetrics run.

# 24. Final project interpretation

For the current project:

```text
Tool                         BamMetrics / bam-tools v1.6.2
Input                        original tumor BAM
Genome build                 GRCh38 / V38
Reference                    shared DRAGEN hg38.fa
Threads per sample           8
Java heap                    16 GB
Execution                    sequential
Completion sentinel          bam_metric.flag_counts.tsv
Gene coverage                not generated
QSEE mandatory inputs        available
```

# 25. Exporting this notebook

All analysis chunks use:

```text
eval=FALSE
```

so export does not rerun BamMetrics.

## HTML

```r
rmarkdown::render(
  "BamMetrics_v1.6.2_UCaTS_complete_workflow.Rmd",
  output_format="html_document"
)
```

## PDF

```r
rmarkdown::render(
  "BamMetrics_v1.6.2_UCaTS_complete_workflow.Rmd",
  output_format="pdf_document"
)
```

PDF export requires a LaTeX installation.

# 26. Official references

BamTools / BamMetrics:

```text
https://github.com/hartwigmedical/hmftools/tree/master/bam-tools
```

HMFtools releases:

```text
https://github.com/hartwigmedical/hmftools/releases
```

HMFtools repository:

```text
https://github.com/hartwigmedical/hmftools
```

Oncoanalyser/HMF resource staging:

```text
https://nf-co.re/oncoanalyser/
```
