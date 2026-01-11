# Kraken2 – Custom Database Build (Linux Server, ONT, Bracken)

## Overview

This repository documents the construction and use of a **custom Kraken2 database** for metagenomic analyses, with a focus on:

- Environmental and wastewater samples  
- Oxford Nanopore Technologies (ONT) long reads (mean read length ~2500–2700 bp)  
- Downstream abundance estimation using **Bracken**  
- A **dedicated Linux server** with **restricted network access** (FTP/rsync blocked)

This README is both:
- a quick reference for day-to-day use,
- a troubleshooting log based on real issues encountered during installation and database build,
- a reproducible guide for future reuse (thesis, workflow, collaborators).

---

## System Requirements

### Operating System
- Linux (dedicated server)

### Hardware (recommended)
- CPU: ≥16 cores
- RAM: ≥64 GB (depends on database size)
- Disk space:
  - ≥600 GB during database construction (before cleaning)
  - ~100 GB after cleaning (final database)

### Software Dependencies
- Kraken2
- kraken2-build
- Perl
- curl / wget (for downloads)
- Bracken (optional but recommended)

### Useful sanity checks
```bash
nproc
free -h
df -h
```

### Official Documentation
- https://ccb.jhu.edu/software/kraken2/
- https://github.com/DerrickWood/kraken2/wiki/Manual
- https://github.com/jenniferlu717/Bracken

---

## Context: Network Restrictions (Server)

The standard Kraken2 database download procedure failed due to server network restrictions:

- FTP access (port 21) blocked
- rsync access (port 873) blocked
- Typical errors observed:
```text
OSError: [Errno 101] Network is unreachable
nc -vz ftp.ncbi.nlm.nih.gov 21  -> connection timed out
```

Note: HTTPS access to NCBI was possible (e.g., `curl -I https://ftp.ncbi.nlm.nih.gov/` returned 200 OK), but the standard Kraken2/NCBI workflow relies on FTP/rsync for some steps.

---

## Installation

### Recommended: Conda
```bash
conda create -n kraken2 -c bioconda kraken2 bracken
conda activate kraken2
```

### Verification
```bash
kraken2 --version
kraken2-build --help
bracken-build -h
```

---

## Real-World Workflow (as performed)

### 1) Install Kraken2 from source to use `k2`

Following the official manual, Kraken2 was cloned and installed to obtain the `k2` helper script:

```bash
git clone https://github.com/DerrickWood/kraken2.git
cd kraken2
./install_kraken2.sh $KRAKEN2_DIR
```

### 2) Start download + standard DB setup with `k2 build`

From the Kraken2 install directory:

```bash
k2 build --db ../mydb --standard --threads 40
```

This successfully initiated downloads and created:
- `taxonomy/`
- `library/` with multiple libraries (Archaea, Bacteria, Viral, etc.)

### 3) Failure on plasmid library download (FTP)

The process failed on plasmid download with an FTP connection error, consistent with blocked port 21:

```text
OSError: [Errno 101] Network is unreachable
```

At this point:
- taxonomy and several libraries were already present
- `library/plasmid/` remained empty

### 4) Workaround for plasmid: download locally + transfer to server

Because FTP/rsync were blocked on the server:
1. plasmid sequences were downloaded locally using an academic VPN
2. transferred to the server via FileZilla
3. placed into:
```bash
mydb/library/plasmid/
```

---

## Adding plasmid sequences to the database

### Notes on format and decompression
If plasmid files are already compressed as `*.gz`, do **not** run `gunzip *.genomic.fna.gz` unless you explicitly want uncompressed FASTA on disk. Kraken2 can work with FASTA files either way depending on your workflow, but be consistent (and mindful of disk usage).

### Why `wget -r` may not fetch plasmid FASTA files
Recursive `wget -r` on NCBI directory listings may download only HTML index pages (and respect robots), which can result in “downloaded” files like `index.html.tmp` but no `.fna.gz` files. Prefer an explicit file list or controlled mirroring method.

---

## Masking and `--add-to-library` performance

Adding large FASTA files using `--add-to-library` can be slow because Kraken2 may run low-complexity masking (`k2mask` / `segmasker`).

Typical message:
```text
Masking low-complexity regions of new file..
```

To confirm progress:
```bash
ps -ef | grep -E "kraken2-build|k2mask|segmasker" | grep -v grep
```

If masking is too slow for your use case, consider skipping masking when appropriate:
```bash
kraken2-build --db mydb --add-to-library plasmid_all.fna --no-masking
```

---

## Building the database (critical step)

### Important: threads and OpenMP behavior

Observed behavior:
- requesting too many threads could lead to instability or crashes
- build tool output included messages such as:
```text
build_db: OMP only wants you to use 24 threads
BrokenPipeError: [Errno 32] Broken pipe
```

Successful strategy:
- match `--threads` to the real number of available CPU cores
- when in doubt, set OpenMP limits explicitly

Example (stable):
```bash
export OMP_NUM_THREADS=24
export OMP_THREAD_LIMIT=24

kraken2-build --build --db mydb --threads 24 2>&1 | tee build.log
```

### Monitoring build status

Check that the main build process exists:
```bash
pgrep -af build_db
```

Monitor CPU and memory usage:
```bash
ps -o pid,etime,%cpu,%mem -p <PID>
```

During heavy build phases, CPU usage can be very high (e.g., >2000% on multi-core systems). Some build sub-steps may become less parallel and temporarily show lower CPU usage; this can be normal.

Check temporary output files:
```bash
ls -lh *.k2d* *.tmp 2>/dev/null | head -n 50
```

Expected build milestones in logs:
- “Taxonomy parsed and converted.”
- “CHT created …”
- “Completed processing … bp”
- “Writing data to disk… complete.”
- “Database construction complete.”

Perl locale warnings may appear:
```text
perl: warning: Setting locale failed.
```
These were benign and did not affect the build.

---

## Database validation

Inspect the database:
```bash
kraken2-inspect --db mydb | head -n 40
```

Example output includes:
- k and l parameters (e.g., k=35, l=31)
- taxonomy nodes
- table size/capacity
- top taxonomic ranks under root

---

## Cleaning the database (disk optimization)

After successful validation:
```bash
kraken2-build --db mydb --clean
```

Observed disk usage in this project:
- Before cleaning: ~587 GB
- After cleaning: ~92 GB

### Critical consequence
`--clean` removed `library/`. This makes the database still usable for **classification**, but prevents building Bracken distributions later.

---

## Bracken (abundance estimation for ONT)

### Requirement
Bracken build requires `library/`. If cleaned, you will see:
```text
ERROR: Database library ./library does not exist
```

### Correct sequence (recommended)
1. Build Kraken2 DB (with library present)
2. Run `bracken-build` for your read length
3. Optionally clean the database

Example:
```bash
bracken-build -d mydb -t 24 -k 35 -l 2500
```

Then for a sample report:
```bash
bracken -d mydb -i sample.report -o sample.bracken -r 2500 -l S
```

---

## Classification example

```bash
kraken2   --db mydb   --threads 16   --report sample.report   sample.fastq.gz > sample.kraken
```

Useful options:
- `--confidence` (helps reduce noisy assignments, especially for ONT)
- `--minimum-hit-groups`
- `--use-names`
- `--gzip-compressed`

---

## Common Errors and Fixes (project log)

| Error / Symptom | Likely cause | Fix / Workaround |
|---|---|---|
| `OSError: [Errno 101] Network is unreachable` during download | FTP blocked on server | Download locally (VPN) and transfer to server; use HTTPS-only approaches if possible |
| `nc -vz ftp.ncbi.nlm.nih.gov 21` times out | Port 21 blocked | Same as above |
| `wget -r` downloads only HTML index | robots / directory listing behavior | Use explicit file lists or alternative mirroring methods |
| `Masking low-complexity...` takes very long | `k2mask` on large FASTA | Wait; monitor processes; or use `--no-masking` |
| `build_db: OMP only wants you to use 24 threads` | Thread limit on server/OpenMP | Reduce `--threads` to allowed CPU; export `OMP_NUM_THREADS` |
| `BrokenPipeError` during build | Mismatch threads/resources or interrupted pipe | Re-run with appropriate threads; log with `tee` |
| Bracken: `Database library ./library does not exist` | Database cleaned too early | Keep a dev DB with `library/`; run Bracken before clean |

---

## Best Practices (validated)

- Keep two database versions:
  - `mydb_dev/` (includes `library/` for Bracken and future updates)
  - `mydb_run/` (cleaned, compact, for routine classification)
- Always capture logs:
```bash
kraken2-build --build --db mydb --threads 24 2>&1 | tee build.log
```
- Run Bracken **before** `--clean`
- Check disk space before building:
```bash
df -h mydb
```
- Use `tmux` for long runs and avoid losing sessions.

---

## License / Usage

This documentation is provided as a reproducible technical guide and may be reused or adapted for academic and research purposes.
