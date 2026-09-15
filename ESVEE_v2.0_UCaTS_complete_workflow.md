# ESVEE v2.0 Complete Workflow — UCaTS Organoids

> **Project:** UCaTS Organoid WGS  
> **Pipeline stage:** `REDUX -> ESVEE -> PURPLE -> LINX`  
> **Input:** REDUX-processed tumor BAMs  
> **Genome:** GRCh38 / hg38  
> **Sequencing:** Illumina WGS, tumor-only  
> **ESVEE version:** `v2.0`  
> **Samples:** 23

## 1. Purpose

ESVEE is the HMFtools structural-variant caller. In the project workflow it consumes the **REDUX BAM**, not the original tumor BAM, and performs the four ESVEE stages:

```text
REDUX BAM
   |
   v
1. Prep
   |
   v
2. Assembly + Alignment
   |
   v
3. Reference Depth Annotation
   |
   v
4. Caller / Filtering
   |
   v
<SAMPLE>.esvee.somatic.vcf.gz
   |
   v
PURPLE
   |
   v
LINX
```

The official ESVEE documentation describes the same four stages and supports running them together in a single command.

Official documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/esvee/README.md
```

HMFtools repository:

```text
https://github.com/hartwigmedical/hmftools
```

## 2. Current project directories

```text
Original BAM:
05_ASCAT_CN/BAMs/

REDUX input:
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam

REDUX reference:
06_REDux/Redux_reference/

ESVEE:
08_ESVEE/

Tool:
08_ESVEE/ESVEE_tools/esvee_v2.0.jar

Reference resources:
08_ESVEE/ESVEE_reference/

Output:
08_ESVEE/ESVEE_output/<SAMPLE>/
```

The final structural-variant VCF is:

```text
08_ESVEE/ESVEE_output/<SAMPLE>/<SAMPLE>.esvee.somatic.vcf.gz
```

## 3. Java and samtools

Current Java used by the project:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Check:

```bash
/home/zzr123/.conda/envs/purple/bin/java -version
samtools --version
```

If a separate environment is ever required:

```bash
conda create   -n esvee_env   -c conda-forge   openjdk=21   samtools   -y

conda activate esvee_env

java -version
samtools --version
```

For reproducibility, the actual R workflow continues to use the explicit Java path:

```r
java <- "/home/zzr123/.conda/envs/purple/bin/java"
```

## 4. Install / download ESVEE v2.0

Create directories:

```bash
BASE="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"

mkdir -p "$BASE/08_ESVEE/ESVEE_tools" "$BASE/08_ESVEE/ESVEE_reference" "$BASE/08_ESVEE/ESVEE_output"
```

Because GitHub release asset names can change independently from the release page, the safest installation method is to resolve the JAR from the fixed `esvee-v2.0` release tag instead of manually guessing the asset URL.

```bash
set -euo pipefail

TOOL_DIR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/08_ESVEE/ESVEE_tools"
TAG="esvee-v2.0"

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
    and "esvee" in a["name"].lower()
    and "source" not in a["name"].lower()
    and "javadoc" not in a["name"].lower()
]

if len(jars) != 1:
    raise SystemExit(
        f"Expected one ESVEE JAR for {tag}, found: {jars}"
    )

print(jars[0])
PY
)"

echo "$ASSET_URL"

curl   -L   --fail   "$ASSET_URL"   -o "$TOOL_DIR/esvee_v2.0.jar"

ls -lh "$TOOL_DIR/esvee_v2.0.jar"
sha256sum "$TOOL_DIR/esvee_v2.0.jar"
```

If the JAR already exists and has already been used successfully, do not silently replace it. Record its checksum first:

```bash
sha256sum /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/08_ESVEE/ESVEE_tools/esvee_v2.0.jar
```

## 5. ESVEE reference resources

The code requires these GRCh38 ESVEE resources:

```text
known_fusions.38.bedpe
sgl_pon.38.bed.gz
sv_pon.38.bedpe.gz
repeat_mask_data.38.fa.gz
unmap_regions.38.tsv
```

The official HMFtools resource documentation describes:

```text
known_fusions.38.bedpe
    known hotspot SV/fusion regions used by ESVEE

sgl_pon.38.bed.gz
    panel-of-normals for single breakends

sv_pon.38.bedpe.gz
    panel-of-normals for structural variants

repeat_mask_data.38.fa.gz
    repeat-mask annotation for inserted/unmappable sequence
```

`unmap_regions.38.tsv` is inherited from the REDUX reference setup in this project.

## 6. Download the HMF GRCh38 resource bundle

Current project resource version:

```text
hmf_pipeline_resources.38_v3.0.0--8
```

Download into `08_ESVEE/ESVEE_reference`:

```bash
set -euo pipefail

REF_DIR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/08_ESVEE/ESVEE_reference"
BUNDLE="hmf_pipeline_resources.38_v3.0.0--8.tar.gz"

mkdir -p "$REF_DIR"
cd "$REF_DIR"

wget -c "https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/$BUNDLE"

tar -xzf "$BUNDLE"
```

Check the ESVEE resources:

```bash
find "$REF_DIR" -type f | grep -E 'known_fusions\.38\.bedpe$|sgl_pon\.38\.bed\.gz$|sv_pon\.38\.bedpe\.gz$|repeat_mask_data\.38\.fa\.gz$|unmap_regions\.38\.tsv$'
```

The code intentionally prefers the exact `unmap_regions.38.tsv` that was used during REDUX:

```text
06_REDux/Redux_reference
```

and only falls back to the ESVEE reference directory.

## 7. Reference FASTA

The code uses:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

This is intentional.

ESVEE must use a FASTA compatible with the REDUX BAM alignment reference. Do **not** switch to another GRCh38 FASTA merely because it already has an index image.

Required files checked by the script:

```text
hg38.fa
hg38.fa.fai
hg38.fa.img
```

Check:

```bash
REF="/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"

ls -lh "$REF"
ls -lh "$REF.fai"
ls -lh "$REF.img"
```

The official HMFtools resource documentation lists the BWA-MEM image as:

```text
<reference>.fasta.img
```

for the reference genome.

## 8. BWA image requirement

The ESVEE Assembly stage realigns assembled structural-variant sequence with BWA. The workflow therefore explicitly requires:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa.img
```

The R code will stop immediately if this file is absent:

```r
ref_img <- paste0(ref_fasta,".img")
```

Do not solve a missing `.img` by changing the FASTA to a different genome reference.

Instead, create or obtain the BWA index image corresponding to the **same exact FASTA**.

The current project should preserve:

```text
hg38.fa
hg38.fa.fai
hg38.fa.img
```

as a matched reference set.

## 9. Optional native BWA library

The script searches the ESVEE tool directory for:

```text
libbwwwa.*.so
```

using:

```r
pattern="^libbwwwa\..*\.so$"
```

If found, it is passed to ESVEE as:

```text
-bwa_lib
```

If absent, the script does not force that parameter.

This behavior is preserved exactly from the supplied working code.

## 10. Tumor-only mode

The supplied workflow is tumor-only.

It passes:

```text
-tumor
-tumor_bam
```

and intentionally does **not** pass:

```text
-reference
-reference_bam
```

The official ESVEE documentation allows the reference sample to be omitted in tumor-only operation for the relevant stages.

## 11. ESVEE single-command mode

The final code uses:

```bash
java -jar esvee_v2.0.jar ...
```

rather than manually calling each Java class.

This invokes the complete workflow:

```text
Prep
  ↓
Assembly
  ↓
Reference Depth
  ↓
Caller
```

The official documentation also describes ESVEE as runnable either as one command or as the four individual stages.

## 12. Main parameters used

The supplied code runs:

```text
-tumor
-tumor_bam

-ref_genome
-ref_genome_version 38

-known_hotspot_file
-pon_sgl_file
-pon_sv_file
-repeat_mask_file
-unmap_regions

-bamtool
-output_dir
-threads 24
```

Optional when available:

```text
-bwa_lib
```

Java heap:

```text
-Xmx32G
```

The script intentionally does not override:

```text
-write_types
```

and therefore uses the ESVEE v2.0 defaults.

## 13. BAM / FASTA compatibility check

Before the 23-sample run starts, the supplied code checks the first REDUX BAM header:

```text
@SQ SN + LN
```

against:

```text
hg38.fa.fai
```

It stops if:

```text
a BAM contig is absent from FASTA
or
a contig length differs
```

This protects against mistakes such as:

```text
chr1 vs 1
different ALT/decoy composition
different reference builds
different contig lengths
```

## 14. BAM input rule

Every sample must use:

```text
06_REDux/Redux_BAMs/<SAMPLE>/<SAMPLE>.redux.bam
```

The code does not use the original:

```text
05_ASCAT_CN/BAMs/<SAMPLE>_tumor.bam
```

If the REDUX BAM index is absent, the script creates it using:

```bash
samtools index -@ 24 <SAMPLE>.redux.bam
```

## 15. Resume behavior

The completion file used by the script is the final somatic ESVEE VCF:

```text
<SAMPLE>.esvee.somatic.vcf.gz
```

If that file already exists and has non-zero size, the sample is labeled:

```text
COMPLETE_ALREADY
```

and skipped.

If any sample fails, the script stops instead of spending resources on the remaining samples.

## 16. Final output

Per sample:

```text
08_ESVEE/ESVEE_output/<SAMPLE>/
├── <SAMPLE>.esvee.somatic.vcf.gz
├── <SAMPLE>.esvee.log
└── additional ESVEE intermediate/default outputs
```

Project status table:

```text
08_ESVEE/ESVEE_output/ESVEE_23_samples_status.csv
```

The formal structural-variant input later used by PURPLE is:

```text
<SAMPLE>.esvee.somatic.vcf.gz
```

## 17. Final complete R code

The following code is preserved from the supplied ESVEE workflow.

```r
# ============================================================
# ESVEE v2.0 | Tumor-only WGS | 23 UCATS samples
# Input : REDUX BAM
# Run   : Prep -> Assembly -> Ref Depth -> Caller
# Output: SAMPLE.esvee.somatic.vcf.gz
# ============================================================

# -------------------- 1. PATHS --------------------

BASE <- "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"

REDUX_DIR <- file.path(BASE,"06_REDux","Redux_BAMs")
REDUX_REF <- file.path(BASE,"06_REDux","Redux_reference")

ESVEE_DIR <- file.path(BASE,"08_ESVEE")
TOOL_DIR <- file.path(ESVEE_DIR,"ESVEE_tools")
REF_DIR <- file.path(ESVEE_DIR,"ESVEE_reference")
OUT_DIR <- file.path(ESVEE_DIR,"ESVEE_output")

dir.create(OUT_DIR,recursive=TRUE,showWarnings=FALSE)

# ESVEE v2.0
esvee_jar <- file.path(TOOL_DIR,"esvee_v2.0.jar")

# Java
java <- "/home/zzr123/.conda/envs/purple/bin/java"

# IMPORTANT:
# Must be the FASTA compatible with the BAM reference.
ref_fasta <- "/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"

# Resources
threads <- 24L
java_mem <- "32G"


# -------------------- 2. ALL 23 SAMPLES --------------------

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


# -------------------- 3. HELPER FUNCTIONS --------------------

find_file <- function(dirs,name){
  for(d in dirs){
    if(!dir.exists(d)) next
    x <- list.files(d,recursive=TRUE,full.names=TRUE)
    x <- x[basename(x)==name]
    if(length(x)>0) return(x[1])
  }
  stop("Cannot find required file: ",name)
}

valid_file <- function(x){
  file.exists(x) && !is.na(file.info(x)$size) && file.info(x)$size>0
}


# -------------------- 4. ESVEE REFERENCES --------------------

known_fusions <- find_file(c(REF_DIR),"known_fusions.38.bedpe")
pon_sgl <- find_file(c(REF_DIR),"sgl_pon.38.bed.gz")
pon_sv <- find_file(c(REF_DIR),"sv_pon.38.bedpe.gz")
repeat_mask <- find_file(c(REF_DIR),"repeat_mask_data.38.fa.gz")

# Prefer the exact unmap_regions resource used by REDUX
unmap_regions <- find_file(
  c(REDUX_REF,REF_DIR),
  "unmap_regions.38.tsv"
)


# -------------------- 5. SOFTWARE CHECK --------------------

samtools <- Sys.which("samtools")

if(samtools=="")
  stop("samtools not found. Load samtools before running R.")

required <- c(
  esvee_jar,
  java,
  ref_fasta,
  paste0(ref_fasta,".fai"),
  known_fusions,
  pon_sgl,
  pon_sv,
  repeat_mask,
  unmap_regions
)

missing <- required[!file.exists(required)]

if(length(missing)>0)
  stop("Missing required files:\n",paste(missing,collapse="\n"))


# -------------------- 6. CHECK BWA IMAGE --------------------
# ESVEE Assembly realigns assembled SV sequences using BWA.
# The image is expected beside the reference FASTA.

ref_img <- paste0(ref_fasta,".img")

if(!file.exists(ref_img)){
  stop(
    "\nMissing BWA image:\n",ref_img,
    "\n\nESVEE Assembly requires the .img corresponding to THIS SAME FASTA.",
    "\nDo not switch to a different FASTA just to obtain an .img."
  )
}


# -------------------- 7. OPTIONAL BWA NATIVE LIBRARY --------------------

bwa_lib <- list.files(
  TOOL_DIR,
  pattern="^libbwwwa\\..*\\.so$",
  full.names=TRUE
)

if(length(bwa_lib)>0){
  bwa_lib <- bwa_lib[1]
  cat("BWA native library:",bwa_lib,"\n")
} else {
  bwa_lib <- NA_character_
  cat("BWA native library not explicitly found in TOOL_DIR.\n")
}


# -------------------- 8. CHECK SETTINGS --------------------

cat("\n================ ESVEE SETTINGS ================\n")
cat("ESVEE jar      :",esvee_jar,"\n")
cat("REDUX BAMs     :",REDUX_DIR,"\n")
cat("ESVEE output   :",OUT_DIR,"\n")
cat("Reference      :",ref_fasta,"\n")
cat("Reference IMG  :",ref_img,"\n")
cat("Known fusions  :",known_fusions,"\n")
cat("SGL PON        :",pon_sgl,"\n")
cat("SV PON         :",pon_sv,"\n")
cat("Repeat mask    :",repeat_mask,"\n")
cat("Unmap regions  :",unmap_regions,"\n")
cat("Samtools       :",samtools,"\n")
cat("Threads        :",threads,"\n")
cat("Samples        :",length(samples),"\n")
cat("================================================\n")


# -------------------- 9. CHECK BAM / FASTA COMPATIBILITY --------------------

test_sample <- samples[1]
test_bam <- file.path(
  REDUX_DIR,
  test_sample,
  paste0(test_sample,".redux.bam")
)

if(!file.exists(test_bam))
  stop("First REDUX BAM not found: ",test_bam)

bam_header <- system2(
  samtools,
  c("view","-H",test_bam),
  stdout=TRUE
)

sq <- bam_header[grepl("^@SQ",bam_header)]

bam_chr <- sub(".*SN:([^[:space:]]+).*","\\1",sq)
bam_len <- as.numeric(sub(".*LN:([0-9]+).*","\\1",sq))

fai <- read.delim(
  paste0(ref_fasta,".fai"),
  header=FALSE,
  stringsAsFactors=FALSE
)

idx <- match(bam_chr,fai$V1)

bad <- is.na(idx) | bam_len != fai$V2[idx]

if(any(bad)){
  cat("\nReference mismatch examples:\n")
  print(head(data.frame(
    BAM_chr=bam_chr[bad],
    BAM_length=bam_len[bad],
    FASTA_length=ifelse(is.na(idx[bad]),NA,fai$V2[idx[bad]])
  ),20))
  stop("REDUX BAM and ref_fasta are not compatible.")
}

cat("\nBAM/reference contig check: PASS\n")


# -------------------- 10. RESULT TABLE --------------------

status_file <- file.path(
  OUT_DIR,
  "ESVEE_23_samples_status.csv"
)

results <- data.frame(
  Sample=samples,
  Status="NOT_STARTED",
  Runtime_hours=NA_real_,
  stringsAsFactors=FALSE
)


# -------------------- 11. RUN ALL 23 SAMPLES --------------------

for(i in seq_along(samples)){

  sample <- samples[i]

  cat("\n============================================================\n")
  cat("[",i,"/",length(samples),"] ",sample,"\n",sep="")
  cat("============================================================\n")

  # REDUX BAM structure confirmed from your folder
  bam <- file.path(
    REDUX_DIR,
    sample,
    paste0(sample,".redux.bam")
  )

  if(!file.exists(bam)){
    results$Status[i] <- "BAM_NOT_FOUND"
    write.csv(results,status_file,row.names=FALSE)
    stop("REDUX BAM not found: ",bam)
  }

  cat("Redux BAM:",bam,"\n")

  # Each sample gets its own ESVEE output directory
  sample_out <- file.path(OUT_DIR,sample)
  dir.create(sample_out,recursive=TRUE,showWarnings=FALSE)

  final_vcf <- file.path(
    sample_out,
    paste0(sample,".esvee.somatic.vcf.gz")
  )

  # Resume support: skip already completed samples
  if(valid_file(final_vcf)){
    cat("[SKIP] Already complete\n")
    results$Status[i] <- "COMPLETE_ALREADY"
    write.csv(results,status_file,row.names=FALSE)
    next
  }

  # ---------------- BAM INDEX ----------------

  bai1 <- paste0(bam,".bai")
  bai2 <- sub("\\.bam$",".bai",bam)

  if(!file.exists(bai1) && !file.exists(bai2)){

    cat("[INDEX] Creating BAM index...\n")

    index_status <- system2(
      samtools,
      c("index","-@",as.character(threads),bam)
    )

    if(index_status!=0){
      results$Status[i] <- "BAM_INDEX_FAILED"
      write.csv(results,status_file,row.names=FALSE)
      stop("BAM indexing failed: ",sample)
    }
  }

  # ---------------- LOG ----------------

  log_file <- file.path(
    sample_out,
    paste0(sample,".esvee.log")
  )

  # ---------------- ESVEE ARGUMENTS ----------------
  #
  # Single-command mode automatically runs:
  # Prep -> Assembly -> Reference Depth -> Caller
  #
  # Tumor-only: no -reference / -reference_bam
  #
  # We intentionally do NOT specify write_types because
  # ESVEE v2.0 defaults already contain the required files.

  args <- c(
    paste0("-Xmx",java_mem),
    "-jar",esvee_jar,

    "-tumor",sample,
    "-tumor_bam",bam,

    "-ref_genome",ref_fasta,
    "-ref_genome_version","38",

    "-known_hotspot_file",known_fusions,
    "-pon_sgl_file",pon_sgl,
    "-pon_sv_file",pon_sv,
    "-repeat_mask_file",repeat_mask,

    "-unmap_regions",unmap_regions,

    "-bamtool",samtools,
    "-output_dir",sample_out,
    "-threads",as.character(threads)
  )

  # Supply native BWA library if present
  if(!is.na(bwa_lib))
    args <- c(args,"-bwa_lib",bwa_lib)

  cat("[RUN] Starting ESVEE...\n")

  start_time <- Sys.time()

  exit_code <- system2(
    java,
    args=args,
    stdout=log_file,
    stderr=log_file
  )

  end_time <- Sys.time()

  runtime <- as.numeric(
    difftime(end_time,start_time,units="hours")
  )

  # ---------------- SUCCESS ----------------

  if(exit_code==0 && valid_file(final_vcf)){

    results$Status[i] <- "COMPLETE"
    results$Runtime_hours[i] <- round(runtime,2)

    cat("[DONE]",sample,"\n")
    cat("Runtime:",round(runtime,2),"hours\n")
    cat("VCF:",final_vcf,"\n")

    write.csv(results,status_file,row.names=FALSE)

  # ---------------- FAILURE ----------------

  } else {

    results$Status[i] <- "FAILED"
    results$Runtime_hours[i] <- round(runtime,2)

    write.csv(results,status_file,row.names=FALSE)

    cat("\n[FAILED]",sample,"\n")
    cat("Log:",log_file,"\n")

    if(file.exists(log_file)){
      log_text <- readLines(log_file,warn=FALSE)

      cat("\n================ LAST 80 LOG LINES ================\n")
      cat(tail(log_text,80),sep="\n")
      cat("\n===================================================\n")
    }

    stop(
      "\nESVEE failed on ",sample,
      ". The script has stopped so the remaining samples are not wasted."
    )
  }
}


# -------------------- 12. FINAL SUMMARY --------------------

cat("\n\n================ ESVEE FINAL SUMMARY ================\n")

print(
  results[,c("Sample","Status","Runtime_hours")],
  row.names=FALSE
)

completed <- sum(
  results$Status %in% c("COMPLETE","COMPLETE_ALREADY")
)

cat("\nCompleted:",completed,"/",length(samples),"\n")
cat("Status table:",status_file,"\n")

write.csv(
  results,
  status_file,
  row.names=FALSE
)

```

## 18. Expected directory structure

```text
UCaTS_Organoids/
│
├── 06_REDux/
│   ├── Redux_BAMs/
│   │   ├── I_26166_S_29146/
│   │   │   ├── I_26166_S_29146.redux.bam
│   │   │   └── I_26166_S_29146.redux.bam.bai
│   │   └── ...
│   │
│   └── Redux_reference/
│       └── .../unmap_regions.38.tsv
│
├── 08_ESVEE/
│   ├── ESVEE_tools/
│   │   ├── esvee_v2.0.jar
│   │   └── libbwwwa.*.so              # optional
│   │
│   ├── ESVEE_reference/
│   │   └── hmf_pipeline_resources.38_v3.0.0--8/
│   │       ├── .../known_fusions.38.bedpe
│   │       ├── .../sgl_pon.38.bed.gz
│   │       ├── .../sv_pon.38.bedpe.gz
│   │       └── .../repeat_mask_data.38.fa.gz
│   │
│   └── ESVEE_output/
│       ├── ESVEE_23_samples_status.csv
│       ├── I_26166_S_29146/
│       │   ├── I_26166_S_29146.esvee.somatic.vcf.gz
│       │   └── I_26166_S_29146.esvee.log
│       └── ...
│
└── reference/
    └── hg38.fa
        hg38.fa.fai
        hg38.fa.img
```

## 19. Downstream use

The relevant workflow is:

```text
REDUX
  |
  +----> SAGE ----> PAVE ----  |                             +----> ESVEE ----------------> PURPLE
                                  |
                                  v
                                 LINX
```

For PURPLE, use the current ESVEE somatic SV VCF:

```text
08_ESVEE/ESVEE_output/<SAMPLE>/<SAMPLE>.esvee.somatic.vcf.gz
```

## 20. Reproducibility checklist

```text
[ ] ESVEE v2.0 JAR exists
[ ] JAR checksum recorded
[ ] Java version recorded
[ ] samtools version recorded
[ ] only REDUX BAMs are used
[ ] REDUX BAM index exists
[ ] hg38.fa is the BAM-compatible reference
[ ] hg38.fa.fai exists
[ ] hg38.fa.img exists
[ ] BAM/FASTA contig check passes
[ ] known_fusions.38.bedpe exists
[ ] sgl_pon.38.bed.gz exists
[ ] sv_pon.38.bedpe.gz exists
[ ] repeat_mask_data.38.fa.gz exists
[ ] REDUX unmap_regions.38.tsv is used
[ ] 23 expected samples are listed
[ ] completed VCFs are skipped
[ ] per-sample log is written
[ ] final status CSV is written
```

## 21. Notes for the lab

The most important methodological rules in this ESVEE setup are:

```text
1. ESVEE input = REDUX BAM, not original BAM.
2. Reference FASTA must match the BAM alignment reference.
3. The BWA .img must correspond to that exact FASTA.
4. ESVEE resources remain GRCh38.
5. Tumor-only means no matched normal/reference BAM.
6. The final somatic SV VCF feeds into PURPLE.
```

This document intentionally keeps the execution logic aligned with the supplied project code rather than silently redesigning the pipeline.
