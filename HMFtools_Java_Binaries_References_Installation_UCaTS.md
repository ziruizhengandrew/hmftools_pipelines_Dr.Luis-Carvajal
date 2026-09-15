# HMFtools Java + Binary/JAR + Reference Installation Guide
## UCaTS Organoids WGS — HIVE

> This document explains **where the `bin` files come from, how to install Java, how to download the LINX and ORANGE JARs, how to preserve/install CHORD, and how to download the shared HMFtools GRCh38 reference bundle**.
>
> Current project tools covered here:
>
> ```text
> LINX   v2.3.1
> CHORD  v2.1.2  (existing tested project JAR)
> ORANGE v5.0.1
> Java   OpenJDK 21
> ```
>
> The commands are written for the current HIVE/Linux environment.

---

# 1. First: what is the "bin file"?

For these HMFtools programs there are two different things:

```text
Java executable
    ↓
.../bin/java

HMFtools program
    ↓
*.jar
```

For example:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

is the **Java binary**.

But:

```text
linx_v2.3.1.jar
chord_v2.1.2.jar
orange_v5.0.1.jar
```

are the actual **HMFtools application binaries/packages**.

You normally run them like:

```bash
/path/to/java -jar tool.jar ...
```

or for some HMFtools classes:

```bash
/path/to/java -cp tool.jar com.hartwig.hmftools.SomeClass ...
```

So for LINX / CHORD / ORANGE, you do **not** normally download a separate `bin/linx` or `bin/orange` executable.

---

# 2. Recommended directory structure

Current project structure:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/

11_LINX/
└── LINX_tools/
    └── linx_v2.3.1.jar

13_CHORD/
└── CHORD_tools/
    └── chord_v2.1.2.jar

16_Orange/
└── Orange_tools/
    └── orange_v5.0.1.jar
```

Shared references:

```text
/quobyte/luisccgrp/REFERENCE_DATA/

├── Illumina/Dragen/
│   ├── hg38.fa
│   ├── hg38.fa.fai
│   └── hg38.dict   # or compatible dictionary name
│
└── hmftools/
    └── hmf_pipeline_resources.38_v3.0.0--8/
```

---

# 3. Java requirement

HMFtools is primarily Java-based. Current HMFtools modules commonly require **Java 17+**.

For this project we have already successfully used:

```text
OpenJDK 21
```

Current Java executable:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Check:

```bash
/home/zzr123/.conda/envs/purple/bin/java -version
```

Expected style:

```text
openjdk version "21..."
OpenJDK Runtime Environment ...
OpenJDK 64-Bit Server VM ...
```

---

# 4. Recommended method: install Java 21 with Conda

This is the easiest method on HIVE because it does not require root/sudo.

## 4.1 Create a dedicated Java environment

```bash
conda create \
  -n hmftools_java \
  -c conda-forge \
  openjdk=21 \
  -y
```

Activate:

```bash
conda activate hmftools_java
```

Check:

```bash
which java
java -version
```

The executable will normally look similar to:

```text
/home/zzr123/.conda/envs/hmftools_java/bin/java
```

You can therefore use:

```bash
JAVA="/home/zzr123/.conda/envs/hmftools_java/bin/java"
"$JAVA" -version
```

## 4.2 Use Java without `conda activate`

For reproducible pipeline scripts, using the absolute path is safer than depending on the current shell environment.

For example:

```r
JAVA <- "/home/zzr123/.conda/envs/hmftools_java/bin/java"
```

or continue using the already-working project Java:

```r
JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"
```

## 4.3 See which Java packages exist in the environment

```bash
conda list | grep -i openjdk
```

Export environment information:

```bash
conda env export -n hmftools_java \
> hmftools_java_environment.yml
```

---

# 5. Alternative: download Java 21 directly from Eclipse Temurin

If Conda is unavailable, Eclipse Adoptium provides official Temurin OpenJDK binaries.

Official API format:

```text
https://api.adoptium.net/v3/binary/latest/{version}/ga/{os}/{arch}/{image_type}/hotspot/normal/eclipse
```

For Java 21 / Linux x64 / JDK:

```text
https://api.adoptium.net/v3/binary/latest/21/ga/linux/x64/jdk/hotspot/normal/eclipse
```

## 5.1 Check HIVE architecture

```bash
uname -m
```

Typical result:

```text
x86_64
```

which corresponds to:

```text
x64
```

## 5.2 Install Temurin 21 under your home directory

No sudo is required:

```bash
set -euo pipefail

JAVA_VERSION="21"
INSTALL_BASE="$HOME/software/java"
INSTALL_DIR="$INSTALL_BASE/temurin-${JAVA_VERSION}"

ARCH_RAW="$(uname -m)"

case "$ARCH_RAW" in
  x86_64)
    ARCH="x64"
    ;;
  aarch64|arm64)
    ARCH="aarch64"
    ;;
  *)
    echo "Unsupported architecture: $ARCH_RAW"
    exit 1
    ;;
esac

API_URL="https://api.adoptium.net/v3/binary/latest/${JAVA_VERSION}/ga/linux/${ARCH}/jdk/hotspot/normal/eclipse"

mkdir -p "$INSTALL_DIR"

echo "Downloading Java from:"
echo "$API_URL"

curl \
  -L \
  --fail \
  "$API_URL" \
  -o "$HOME/temurin-${JAVA_VERSION}.tar.gz"

tar \
  -xzf "$HOME/temurin-${JAVA_VERSION}.tar.gz" \
  -C "$INSTALL_DIR" \
  --strip-components=1

rm -f "$HOME/temurin-${JAVA_VERSION}.tar.gz"

"$INSTALL_DIR/bin/java" -version
```

After installation:

```text
$HOME/software/java/temurin-21/bin/java
```

is the Java binary.

## 5.3 Set JAVA_HOME

For the current shell:

```bash
export JAVA_HOME="$HOME/software/java/temurin-21"
export PATH="$JAVA_HOME/bin:$PATH"
```

Check:

```bash
which java
java -version
echo "$JAVA_HOME"
```

To persist it:

```bash
cat >> ~/.bashrc <<'EOF'

# HMFtools Java
export JAVA_HOME="$HOME/software/java/temurin-21"
export PATH="$JAVA_HOME/bin:$PATH"
EOF
```

Then:

```bash
source ~/.bashrc
```

---

# 6. Which Java should we use for the current project?

The current project has already successfully used:

```text
/home/zzr123/.conda/envs/purple/bin/java
```

Therefore **there is no need to replace it** unless we intentionally want a dedicated environment.

For maximum reproducibility, keep this explicit path in the actual production scripts:

```r
JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"
```

and save:

```bash
/home/zzr123/.conda/envs/purple/bin/java -version
```

in the run logs.

---

# 7. Downloading HMFtools JAR files

The official HMFtools release page is:

```text
https://github.com/hartwigmedical/hmftools/releases
```

A release generally has:

```text
release tag
    ↓
GitHub release assets
    ↓
tool JAR
```

It is better to use a **fixed version tag** than "latest".

---

# 8. Generic GitHub release JAR downloader

The following helper avoids guessing the exact GitHub asset URL.

Create:

```bash
mkdir -p "$HOME/bin"
```

Create the script:

```bash
cat > "$HOME/bin/download_hmftools_jar.sh" <<'EOF'
#!/usr/bin/env bash

set -euo pipefail

if [[ "$#" -ne 4 ]]; then
    echo "Usage:"
    echo "  $0 TOOL_NAME RELEASE_TAG OUTPUT_DIR OUTPUT_FILENAME"
    echo
    echo "Example:"
    echo "  $0 linx linx-v2.3.1 /path/LINX_tools linx_v2.3.1.jar"
    exit 1
fi

TOOL="$1"
TAG="$2"
OUT_DIR="$3"
OUT_NAME="$4"

mkdir -p "$OUT_DIR"

ASSET_URL="$(
python3 - "$TOOL" "$TAG" <<'PY'
import json
import sys
import urllib.request

tool = sys.argv[1].lower()
tag = sys.argv[2]

api = (
    "https://api.github.com/repos/"
    "hartwigmedical/hmftools/releases/tags/"
    + tag
)

with urllib.request.urlopen(api) as response:
    release = json.load(response)

jars = []

for asset in release.get("assets", []):
    name = asset["name"].lower()

    if not name.endswith(".jar"):
        continue

    if tool not in name:
        continue

    if any(x in name for x in ["source", "sources", "javadoc", "tests"]):
        continue

    jars.append(
        (
            asset["name"],
            asset["browser_download_url"]
        )
    )

if len(jars) != 1:
    raise SystemExit(
        f"Expected exactly one {tool} JAR for {tag}; found: {jars}"
    )

print(jars[0][1])
PY
)"

echo "Resolved URL:"
echo "$ASSET_URL"

curl \
  -L \
  --fail \
  "$ASSET_URL" \
  -o "$OUT_DIR/$OUT_NAME"

echo
echo "Downloaded:"
ls -lh "$OUT_DIR/$OUT_NAME"

echo
echo "SHA256:"
sha256sum "$OUT_DIR/$OUT_NAME"
EOF
```

Make executable:

```bash
chmod +x "$HOME/bin/download_hmftools_jar.sh"
```

Now it can download fixed HMFtools releases reproducibly.

---

# 9. Download LINX v2.3.1

Target directory:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/11_LINX/LINX_tools
```

Run:

```bash
"$HOME/bin/download_hmftools_jar.sh" \
  linx \
  linx-v2.3.1 \
  /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/11_LINX/LINX_tools \
  linx_v2.3.1.jar
```

Verify:

```bash
LINX_JAR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/11_LINX/LINX_tools/linx_v2.3.1.jar"

ls -lh "$LINX_JAR"
sha256sum "$LINX_JAR"
```

Current project SHA256:

```text
d2dc8659dfcd1d4cc84267a6574953045bed75910191eac28345557d5d001c7d
```

If a newly downloaded JAR does not have the expected checksum, **do not silently replace the production JAR**. Investigate the asset/version first.

---

# 10. Download ORANGE v5.0.1

Target:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/16_Orange/Orange_tools
```

Run:

```bash
"$HOME/bin/download_hmftools_jar.sh" \
  orange \
  orange-v5.0.1 \
  /quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/16_Orange/Orange_tools \
  orange_v5.0.1.jar
```

Verify:

```bash
ORANGE_JAR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/16_Orange/Orange_tools/orange_v5.0.1.jar"

ls -lh "$ORANGE_JAR"
sha256sum "$ORANGE_JAR"
```

Current project SHA256:

```text
2f64189b8680c62bec6e7c72cecc3cda51a9eb56950808f349cee7c6de666987
```

---

# 11. CHORD binary/JAR: important exception

Current project CHORD:

```text
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/13_CHORD/CHORD_tools/chord_v2.1.2.jar
```

The HMFtools project page currently labels CHORD as:

```text
2.2
```

However, during this project:

```text
https://api.github.com/repos/hartwigmedical/hmftools/releases/tags/chord-v2.2
```

returned:

```text
HTTP 404
```

Therefore **do not write a fake CHORD v2.2 download URL into a reproducible workflow**.

For this project the correct rule is:

```text
Use the exact existing tested chord_v2.1.2.jar
+ record its SHA256
+ archive that JAR with the workflow.
```

## 11.1 Verify existing CHORD

```bash
CHORD_JAR="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/13_CHORD/CHORD_tools/chord_v2.1.2.jar"

ls -lh "$CHORD_JAR"
sha256sum "$CHORD_JAR"

/home/zzr123/.conda/envs/purple/bin/java \
  -jar "$CHORD_JAR" \
  -help
```

## 11.2 Back up the exact CHORD binary

Because a clean public release URL for this exact binary is not being assumed, keep a project-level backup.

For example:

```bash
BACKUP="$HOME/software/hmftools_archive/chord_v2.1.2"

mkdir -p "$BACKUP"

cp \
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/13_CHORD/CHORD_tools/chord_v2.1.2.jar \
"$BACKUP/"

sha256sum \
"$BACKUP/chord_v2.1.2.jar" \
> "$BACKUP/chord_v2.1.2.jar.sha256"
```

To restore:

```bash
mkdir -p \
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/13_CHORD/CHORD_tools

cp \
"$HOME/software/hmftools_archive/chord_v2.1.2/chord_v2.1.2.jar" \
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/13_CHORD/CHORD_tools/
```

Then verify:

```bash
sha256sum \
/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/13_CHORD/CHORD_tools/chord_v2.1.2.jar
```

---

# 12. CHORD also needs R + randomForest

Check Rscript:

```bash
which Rscript
Rscript --version
```

Check `randomForest`:

```bash
Rscript -e \
"quit(status=ifelse(requireNamespace('randomForest', quietly=TRUE),0,1))"
```

If missing:

```bash
Rscript -e \
"install.packages('randomForest', repos='https://cloud.r-project.org')"
```

Then verify again.

---

# 13. Download the shared HMFtools GRCh38 resources

Current shared bundle:

```text
hmf_pipeline_resources.38_v3.0.0--8
```

Target:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
```

Download:

```bash
set -euo pipefail

REF_BASE="/quobyte/luisccgrp/REFERENCE_DATA/hmftools"
BUNDLE="hmf_pipeline_resources.38_v3.0.0--8.tar.gz"

mkdir -p "$REF_BASE"
cd "$REF_BASE"

wget \
  -c \
  "https://data.oncoanalyser.com/r2/reference/dist/v1/hartwig/pipeline_resources/$BUNDLE"
```

Check the archive:

```bash
ls -lh "$BUNDLE"
```

Extract:

```bash
tar \
  -xzvf \
  "$BUNDLE"
```

Expected:

```text
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/
```

---

# 14. LINX reference files inside the HMF bundle

LINX requires:

```text
common/DriverGenePanel.38.tsv

dna/sv/known_fusion_data.38.csv

common/ensembl_data/
```

Check:

```bash
HMF_REF="/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"

ls -lh \
"$HMF_REF/common/DriverGenePanel.38.tsv"

ls -lh \
"$HMF_REF/dna/sv/known_fusion_data.38.csv"

ls -lh \
"$HMF_REF/common/ensembl_data/ensembl_gene_data.csv"

ls -lh \
"$HMF_REF/common/ensembl_data/ensembl_trans_exon_data.csv"

ls -lh \
"$HMF_REF/common/ensembl_data/ensembl_trans_splice_data.csv"

ls -lh \
"$HMF_REF/common/ensembl_data/ensembl_protein_features.csv"
```

Do not create a separate duplicate of this resource bundle in every tool directory.

Use one shared reference root.

---

# 15. CHORD reference genome

CHORD uses the same genome reference as the upstream VCF workflow.

Current project FASTA:

```text
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

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
samtools faidx \
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

Do **not** download an arbitrary new GRCh38 FASTA just because the assembly is called hg38.

The safest rule is:

```text
CHORD reference FASTA
=
the exact reference used for upstream alignment / variant calling
```

---

# 16. One-shot installation block for the current project

This sets up directories, Java 21, LINX, and ORANGE.

It deliberately does **not** replace CHORD automatically.

```bash
set -euo pipefail

BASE="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"

# ============================================================
# 1. JAVA 21
# ============================================================

if ! conda env list | awk '{print $1}' | grep -qx "hmftools_java"; then

  conda create \
    -n hmftools_java \
    -c conda-forge \
    openjdk=21 \
    -y

fi

JAVA="$HOME/.conda/envs/hmftools_java/bin/java"

"$JAVA" -version


# ============================================================
# 2. TOOL DIRECTORIES
# ============================================================

LINX_TOOL="$BASE/11_LINX/LINX_tools"
CHORD_TOOL="$BASE/13_CHORD/CHORD_tools"
ORANGE_TOOL="$BASE/16_Orange/Orange_tools"

mkdir -p \
"$LINX_TOOL" \
"$CHORD_TOOL" \
"$ORANGE_TOOL"


# ============================================================
# 3. GENERIC HMFTOOLS DOWNLOADER
# ============================================================

download_hmf_jar(){

  TOOL="$1"
  TAG="$2"
  OUT_DIR="$3"
  OUT_NAME="$4"

  ASSET_URL="$(
  python3 - "$TOOL" "$TAG" <<'PY'
import json
import sys
import urllib.request

tool = sys.argv[1].lower()
tag = sys.argv[2]

api = (
    "https://api.github.com/repos/"
    "hartwigmedical/hmftools/releases/tags/"
    + tag
)

with urllib.request.urlopen(api) as response:
    release = json.load(response)

jars = []

for asset in release.get("assets", []):
    name = asset["name"].lower()

    if not name.endswith(".jar"):
        continue

    if tool not in name:
        continue

    if any(x in name for x in ["source", "sources", "javadoc", "tests"]):
        continue

    jars.append(
        (
            asset["name"],
            asset["browser_download_url"]
        )
    )

if len(jars) != 1:
    raise SystemExit(
        f"Expected exactly one {tool} JAR for {tag}; found: {jars}"
    )

print(jars[0][1])
PY
  )"

  mkdir -p "$OUT_DIR"

  curl \
    -L \
    --fail \
    "$ASSET_URL" \
    -o "$OUT_DIR/$OUT_NAME"

  sha256sum \
    "$OUT_DIR/$OUT_NAME"
}


# ============================================================
# 4. LINX
# ============================================================

download_hmf_jar \
  linx \
  linx-v2.3.1 \
  "$LINX_TOOL" \
  linx_v2.3.1.jar


# ============================================================
# 5. ORANGE
# ============================================================

download_hmf_jar \
  orange \
  orange-v5.0.1 \
  "$ORANGE_TOOL" \
  orange_v5.0.1.jar


# ============================================================
# 6. CHORD
# ============================================================

CHORD_JAR="$CHORD_TOOL/chord_v2.1.2.jar"

if [[ -s "$CHORD_JAR" ]]; then

  echo
  echo "Existing tested CHORD JAR found:"
  ls -lh "$CHORD_JAR"

  echo
  echo "CHORD SHA256:"
  sha256sum "$CHORD_JAR"

else

  echo
  echo "CHORD v2.1.2 JAR is NOT present."
  echo
  echo "Do NOT automatically download chord-v2.2."
  echo "The public chord-v2.2 release tag was not available"
  echo "when this project workflow was validated."
  echo
  echo "Restore the exact archived chord_v2.1.2.jar instead."

fi


# ============================================================
# 7. FINAL TOOL CHECK
# ============================================================

echo
echo "================ JAVA ================"
"$JAVA" -version

echo
echo "================ LINX ================"
ls -lh \
"$LINX_TOOL/linx_v2.3.1.jar"

echo
echo "================ CHORD ================"
ls -lh \
"$CHORD_TOOL/chord_v2.1.2.jar" \
2>/dev/null || true

echo
echo "================ ORANGE =============="
ls -lh \
"$ORANGE_TOOL/orange_v5.0.1.jar"
```

---

# 17. Recommended production paths after installation

If using the existing project Java:

```r
JAVA <- "/home/zzr123/.conda/envs/purple/bin/java"
```

If using the new dedicated Java environment:

```r
JAVA <- "/home/zzr123/.conda/envs/hmftools_java/bin/java"
```

LINX:

```r
LINX_JAR <- paste0(
  "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/",
  "11_LINX/LINX_tools/linx_v2.3.1.jar"
)
```

CHORD:

```r
CHORD_JAR <- paste0(
  "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/",
  "13_CHORD/CHORD_tools/chord_v2.1.2.jar"
)
```

ORANGE:

```r
ORANGE_JAR <- paste0(
  "/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids/",
  "16_Orange/Orange_tools/orange_v5.0.1.jar"
)
```

---

# 18. Verify everything before a production run

```bash
BASE="/quobyte/luisccgrp/SEQ_DATA/WGS/UCaTS_Organoids"
JAVA="/home/zzr123/.conda/envs/purple/bin/java"

LINX="$BASE/11_LINX/LINX_tools/linx_v2.3.1.jar"
CHORD="$BASE/13_CHORD/CHORD_tools/chord_v2.1.2.jar"
ORANGE="$BASE/16_Orange/Orange_tools/orange_v5.0.1.jar"

echo "================ JAVA ================="
"$JAVA" -version

echo
echo "================ LINX ================="
ls -lh "$LINX"
sha256sum "$LINX"

echo
echo "================ CHORD ================"
ls -lh "$CHORD"
sha256sum "$CHORD"

echo
echo "================ ORANGE ==============="
ls -lh "$ORANGE"
sha256sum "$ORANGE"

echo
echo "================ HMF REFERENCES ======="
HMF_REF="/quobyte/luisccgrp/REFERENCE_DATA/hmftools/hmf_pipeline_resources.38_v3.0.0--8"

ls -ld "$HMF_REF"
ls -lh "$HMF_REF/common/DriverGenePanel.38.tsv"
ls -lh "$HMF_REF/dna/sv/known_fusion_data.38.csv"

echo
echo "================ GENOME ==============="
REF="/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa"

ls -lh "$REF"
ls -lh "$REF.fai"
```

---

# 19. What should be archived for the lab?

For every tool, archive at minimum:

```text
tool name
exact version
exact JAR file
JAR SHA256
download source/tag
Java version
Java executable path
reference bundle version
reference paths
reference checksums where practical
exact executed command
console log
program log
sessionInfo()
```

For CHORD in particular, also archive the exact:

```text
chord_v2.1.2.jar
```

because this project is not relying on an assumed public `chord-v2.2` binary URL.

---

# 20. Current project installation summary

```text
JAVA
/home/zzr123/.conda/envs/purple/bin/java
OpenJDK 21

LINX
11_LINX/LINX_tools/linx_v2.3.1.jar

CHORD
13_CHORD/CHORD_tools/chord_v2.1.2.jar

ORANGE
16_Orange/Orange_tools/orange_v5.0.1.jar

HMF REFERENCES
/quobyte/luisccgrp/REFERENCE_DATA/hmftools/
hmf_pipeline_resources.38_v3.0.0--8/

GRCh38 FASTA
/quobyte/luisccgrp/REFERENCE_DATA/Illumina/Dragen/hg38.fa
```

---

# 21. Official sources

HMFtools repository:

```text
https://github.com/hartwigmedical/hmftools
```

HMFtools releases:

```text
https://github.com/hartwigmedical/hmftools/releases
```

HMFtools resource documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/pipeline/README_RESOURCES.md
```

LINX documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/linx/README.md
```

CHORD documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/chord/README.md
```

ORANGE documentation:

```text
https://github.com/hartwigmedical/hmftools/blob/master/orange/README.md
```

Eclipse Temurin / Adoptium Java downloads:

```text
https://adoptium.net/temurin/releases/
```

Adoptium API installation guide:

```text
https://adoptium.net/installation/ci-scripts
```

Conda-forge OpenJDK:

```text
https://anaconda.org/conda-forge/openjdk
```
