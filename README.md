# AlphaFold Insulin Structure Prediction

> End to end deployment of DeepMind's AlphaFold2 on Google Cloud Platform to predict the 3D structure of human insulin.....covering infrastructure setup, database downloads, Docker containerization, NVIDIA driver debugging and final structure prediction.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Infrastructure Setup](#infrastructure-setup)
- [Installation & Configuration](#installation--configuration)
- [Database Downloads](#database-downloads)
- [Running the Prediction](#running-the-prediction)
- [Output Files](#output-files)
- [Results & Interpretation](#results--interpretation)
- [Troubleshooting Log](#troubleshooting-log)
- [Commands Reference](#commands-reference)
- [Citation](#citation)

---

## Project Overview

This project deploys [DeepMind's AlphaFold2](https://github.com/google-deepmind/alphafold) inference pipeline on a GCP VM to predict the 3D folded structure of human insulin from its amino acid sequence.

**Why insulin?**  
Insulin is a small protein (54 amino acids in the B-chain sequence used here) with a well-known experimental structure (PDB: [2HIU](https://www.rcsb.org/structure/2HIU)),which makes it an ideal test case for validating the full pipeline end to end.

**What AlphaFold does:**  
AlphaFold2 takes an amino acid sequence (FASTA format) as input and predicts the 3D coordinates of every atom in the folded protein. It searches for evolutionary homologs via Multiple Sequence Alignment (MSA), finds structural templates from the PDB and runs a deep learning model to output per-residue 3D coordinates alongside pLDDT confidence scores (0–100).

**This is inference, not training.**  
The published model weights (~5.3 GB) and reference databases (~2.5 TB) are downloaded and used directly. There is no model training done here.

---

## Infrastructure Setup

### GCP VM Specifications

| Component | Specification |
|-----------|--------------|
| GPU | NVIDIA T4 (16 GB VRAM) |
| vCPUs | 8 |
| RAM | 30 GB |
| Boot Disk | 30 GB (Debian Bookworm) |
| Persistent Disk | 3 TB balanced SSD attached separately |
| CUDA | 12.2.2 |
| Docker | Latest (root directory redirected to persistent disk) |

### Why a Separate Persistent Disk?

AlphaFold's full database suite requires ~2.5 TB. The boot disk (30 GB) is far too small for it. So a separate 3 TB persistent disk was attached and mounted at `/mnt/alphafold-data` to hold all databases, docker images and output files.

**Key setup commands:**

```bash
# Format and mount the disk (first use only)
sudo mkfs.ext4 -F /dev/sdb
sudo mkdir -p /mnt/alphafold-data
sudo mount /dev/sdb /mnt/alphafold-data

# Verify mount and available space
df -h /mnt/alphafold-data

# Redirect Docker storage to persistent disk (critical — Docker images are large)
sudo mkdir -p /mnt/alphafold-data/docker
sudo mv /var/lib/docker /var/lib/docker.bak
sudo ln -s /mnt/alphafold-data/docker /var/lib/docker
sudo systemctl start docker
```

> **Note:** The disk must be manually remounted every time the VM restarts:
> ```bash
> sudo mount /dev/sdb /mnt/alphafold-data
> ```

---

## Installation & Configuration

### 1. Install System Dependencies

```bash
sudo apt update && sudo apt upgrade -y

# Install using persistent disk cache if boot disk is full
sudo mkdir -p /mnt/alphafold-data/apt-cache
sudo apt -o dir::cache::archives="/mnt/alphafold-data/apt-cache" install -y \
  git wget curl aria2 rsync
```

### 2. Clone AlphaFold

```bash
cd /mnt/alphafold-data
sudo chown -R $USER:$USER /mnt/alphafold-data
git clone https://github.com/google-deepmind/alphafold.git
```

### 3. Install NVIDIA Container Toolkit

The NVIDIA Container Toolkit allows Docker containers to access your host GPU.

```bash
# Add NVIDIA package repository (use stable/deb path)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Configure Docker to use the NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify GPU is accessible inside Docker
docker run --rm --gpus all nvidia/cuda:11.2.2-base nvidia-smi
```

### 4. Build the AlphaFold Docker Image

```bash
cd /mnt/alphafold-data/alphafold
docker build -f docker/Dockerfile -t alphafold .
```

This step can take 10–20 minutes and it installs all Python dependencies (JAX, TensorFlow, HHblits, Jackhmmer) inside the container.

### 5. Create the Insulin FASTA File

```bash
cat > /mnt/alphafold-data/insulin.fasta << 'EOF'
>insulin
MALWMRLLPLLALLALWGPDPAAAFVNQHLCGSHLVEALYLVCGERGFFYTPKT
EOF
```

---

## Database Downloads

AlphaFold requires multiple reference databases for MSA search and structural template lookup. Total size: ~2.5 TB. The downloads are resumable via `aria2c`.

You can run the full download script in the background using:

```bash
cd /mnt/alphafold-data/alphafold/scripts
nohup bash download_all_data.sh /mnt/alphafold-data full_dbs \
  > /mnt/alphafold-data/download.log 2>&1 &

# Monitor progress
tail -f /mnt/alphafold-data/download.log
```

### Database Summary

| Database | Purpose | Approx. Size |
|----------|---------|-------------|
| **params** | AlphaFold model weights (5 models) | ~5.3 GB |
| **uniref90** | Evolutionary sequences for MSA via Jackhmmer | ~90 GB |
| **mgnify** | Metagenomic sequences for MSA via Jackhmmer | ~120 GB |
| **bfd** | Large sequence clusters for MSA via HHblits | ~272 GB compressed / ~1.7 TB extracted |
| **uniref30** | Sequence clusters for HHblits co-search | ~90 GB |
| **pdb70** | Structural template search | ~56 GB |
| **pdb_mmcif** | Full PDB structure templates | ~38 GB |

### Why Should We Download These If AlphaFold Is Already Trained?

AlphaFold was trained on these databases, but they are not baked into the model weights. At inference time, AlphaFold searches them in real time to build a Multiple Sequence Alignment (MSA) for the specific query sequence. The co-evolutionary signals from the MSA which residue positions co-vary across thousands of species. They are a primary input to the neural network and they drive accurate 3D predictions.

### Checking Download Status

```bash
du -sh /mnt/alphafold-data/bfd/*
du -sh /mnt/alphafold-data/uniref90/
du -sh /mnt/alphafold-data/mgnify/
du -sh /mnt/alphafold-data/pdb70/
du -sh /mnt/alphafold-data/params/

# Check remaining disk space
df -h /mnt/alphafold-data
```

---

## Running the Prediction

Once all databases are downloaded and extracted:

```bash
mkdir -p /mnt/alphafold-data/output

sudo docker run --rm --gpus all \
  -v /mnt/alphafold-data:/mnt/alphafold-data \
  -v /mnt/alphafold-data/alphafold:/app/alphafold \
  alphafold \
  --fasta_paths=/mnt/alphafold-data/insulin.fasta \
  --output_dir=/mnt/alphafold-data/output \
  --model_preset=monomer \
  --db_preset=full_dbs \
  --data_dir=/mnt/alphafold-data \
  --uniref90_database_path=/mnt/alphafold-data/uniref90/uniref90.fasta \
  --mgnify_database_path=/mnt/alphafold-data/mgnify/mgy_clusters_2022_05.fa \
  --template_mmcif_dir=/mnt/alphafold-data/pdb_mmcif/mmcif_files \
  --obsolete_pdbs_path=/mnt/alphafold-data/pdb_mmcif/obsolete.dat \
  --bfd_database_path=/mnt/alphafold-data/bfd/bfd_metaclust_clu_complete_id30_c90_final_seq.sorted_opt \
  --uniref30_database_path=/mnt/alphafold-data/uniref30/UniRef30_2021_03 \
  --pdb70_database_path=/mnt/alphafold-data/pdb70/pdb70 \
  --use_gpu_relax=False \
  --max_template_date=2023-01-01
```

**Estimated runtime on NVIDIA T4:** ~90–120 minutes total
- MSA search (Jackhmmer + HHblits): ~30–40 minutes
- Structure prediction (5 models × ~90 seconds each): ~10 minutes
- Amber relaxation: skipped (`--use_gpu_relax=False`) due to CUDA driver incompatibility (see Troubleshooting #6)

---

## Output Files

```
/mnt/alphafold-data/output/insulin/
├── unrelaxed_model_1_pred_0.pdb      # 3D structure, model 1
├── unrelaxed_model_2_pred_0.pdb      # 3D structure, model 2
├── unrelaxed_model_3_pred_0.pdb
├── unrelaxed_model_4_pred_0.pdb
├── unrelaxed_model_5_pred_0.pdb
├── unrelaxed_model_1_pred_0.cif      # mmCIF format (same structure)
├── confidence_model_1_pred_0.json    # Per-residue pLDDT confidence scores
├── result_model_1_pred_0.pkl         # Full prediction internals (distogram, PAE, logits)
├── features.pkl                      # MSA and template features used as input
└── msas/                             # Raw MSA files from Jackhmmer and HHblits
```

**Key files:**
- **`.pdb` / `.cif`** — Predicted 3D structure. Open in [Mol*](https://molstar.org/viewer/), PyMOL or ChimeraX.
- **`confidence_*.json`** — Per-residue pLDDT scores. >90 = very high confidence; 70–90 = confident; <70 = low confidence.
- **`result_*.pkl`** — Raw logits, distogram and predicted aligned error (PAE).

---

## Results & Interpretation

AlphaFold ran successfully and produced unrelaxed structures for all 5 models.

**MSA statistics (from run logs):**
- UniRef90 hits: 675 sequences
- BFD hits: 372 sequences
- MGnify hits: 9 sequences
- Final deduplicated MSA: 981 sequences
- Templates found: 20 (including exact matches `3w7y_B`, `2jzq_A`, `1dcs_A`, `6ins_E`)

**Why RMSD vs. the native structure would be high:**  
The FASTA input used only the insulin B-chain (54 amino acids), not the full proinsulin sequence. Insulin's native fold depends on inter-chain contacts between the A-chain and B-chain. Feeding only the B-chain forces AlphaFold to predict a partial structure without those stabilizing interactions. The MSA search confirmed this.... alignments only covered residues corresponding to the B-chain region. Running AlphaFold with the complete proinsulin sequence (or using `--model_preset=multimer` with both chains) would yield more accurate results.

---

## Troubleshooting Log

### 1. Boot Disk Full at 100%

**Problem:** The 10 GB boot disk filled up during Docker image pulls and package installations.

**Symptoms:** `errno: 28 No space left on device`, conda installation failures mid-build, `apt` refusing to run.

**Fix:** Redirect Docker's storage root to the persistent disk and use persistent disk as apt cache:
```bash
sudo mkdir -p /mnt/alphafold-data/docker
sudo mv /var/lib/docker /var/lib/docker.bak
sudo ln -s /mnt/alphafold-data/docker /var/lib/docker
sudo systemctl restart docker

# For apt installs
sudo apt -o dir::cache::archives="/mnt/alphafold-data/apt-cache" install -y <package>
```

---

### 2. NVIDIA Driver Not Loading (`nvidia-smi` Failing)

**Problem:** `nvidia-smi` reported `NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver` despite `nvidia-driver-535` appearing installed.

**Root cause:** The VM boots with a cloud-optimized kernel (`6.1.0-33-cloud-amd64`) that lacks standard kernel headers needed for the NVIDIA DKMS module to compile. The DKMS build failed with:
```
fatal error: stdarg.h: No such file or directory
error: #error dma_buf_export() conftest failed!
```

**Fix:** Boot into the standard kernel via GRUB:
```bash
# Check available kernels
sudo grep menuentry /boot/grub/grub.cfg

# Edit GRUB to boot into the standard amd64 kernel (not cloud-amd64)
sudo nano /etc/default/grub
# Set: GRUB_DEFAULT="Advanced options>...Linux 6.1.0-33-amd64"

sudo update-grub
sudo reboot

# Verify after reboot
uname -r        # Should show 6.1.0-33-amd64 (not cloud-amd64)
nvidia-smi      # Should now succeed
```

---

### 3. NVIDIA Container Toolkit Not Recognized by Docker

**Problem:** `docker: Error response from daemon: could not select device driver "" with capabilities: [[gpu]]`

**Root cause:** Wrong NVIDIA repository URL used (distro-specific path returned a 404/HTML page, corrupting the apt source file).

**Fix:** Use the stable generic .deb repository:
```bash
sudo rm /etc/apt/sources.list.d/nvidia-container-toolkit.list

curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

### 4. BFD Database Extraction Failure

**Problem:** `tar: Wrote only 512 of 10240 bytes` followed by `tar: Exiting with failure status` during BFD extraction.

**Root cause:** Disk ran out of space mid-extraction. The BFD tarball is 272 GB, but the extracted files total ~1.7 TB. Both must fit on the disk.

**Fix:** Request disk quota expansion through GCP console, then resize the filesystem without unmounting:
```bash
sudo resize2fs /dev/sda
df -h /mnt/alphafold-data   # Should now reflect expanded size

# Monitor extraction progress
watch -n 30 "du -sh /mnt/alphafold-data/bfd/*ffdata"
```

---

### 5. HHblits "Could Not Find Database" Error

**Problem:** `ValueError: Could not find HHBlits database /mnt/alphafold-data/bfd`

**Root cause:** Passing the directory path instead of the database filename prefix. AlphaFold expects the prefix of the `.ffdata`/`.ffindex` files, not the containing directory.

**Fix:**
```bash
# Wrong:
--bfd_database_path=/mnt/alphafold-data/bfd

# Correct:
--bfd_database_path=/mnt/alphafold-data/bfd/bfd_metaclust_clu_complete_id30_c90_final_seq.sorted_opt
```

---

### 6. Amber GPU Relaxation Failed

**Problem:** `ValueError: Minimization failed after 100 attempts.` with repeated `No compatible CUDA device is available` during the amber relaxation step.

**Root cause:** The OpenMM amber relaxation uses a different CUDA interface than JAX (which ran structure prediction successfully). The CUDA libraries inside the Docker container were incompatible with the host driver for the relaxation step.

**Fix:** Disable GPU relaxation. The unrelaxed structures are valid for visualization and analysis. Relaxation only corrects minor steric clashes.
```bash
--use_gpu_relax=False
```

---

### 7. SSH Session Dropping Mid-Download

**Problem:** Long-running downloads (multi hour) were killed when the SSH session timed out due to inactivity or network issues which interrupted downloads midway.

**Fix:** Use `nohup` to detach the download process from the SSH session:
```bash
nohup bash download_all_data.sh /mnt/alphafold-data full_dbs \
  > /mnt/alphafold-data/download.log 2>&1 &

# Monitor from any session (including after reconnecting)
tail -f /mnt/alphafold-data/download.log
ps aux | grep download_all_data
```
`aria2c` also resumes interrupted downloads automatically via the `-c` (continue) flag.

---

### 8. Disk Appears Empty After VM Restart

**Problem:** After stopping and restarting the VM, `/mnt/alphafold-data` showed as empty... all database files appeared gone.

**Root cause:** GCP does not automatically remount attached persistent disks after a VM restart.

**Fix:** Remount manually after every restart:
```bash
sudo mkdir -p /mnt/alphafold-data
sudo mount /dev/sdb /mnt/alphafold-data
ls /mnt/alphafold-data   # Files reappear
```

---

### 9. pdb_mmcif Missing `mmcif_files/` Subdirectory

**Problem:** `ValueError: Could not find CIFs in /mnt/alphafold-data/pdb_mmcif/mmcif_files`

**Root cause:** The PDB mmCIF download uses `rsync` which can be interrupted, leaving the `mmcif_files/` subdirectory incomplete or absent entirely.

**Fix:** Re-run the PDB mmCIF download inside the Docker container:
```bash
sudo docker run --rm \
  --entrypoint bash \
  -v /mnt/alphafold-data:/mnt/alphafold-data \
  -v /mnt/alphafold-data/alphafold:/app/alphafold \
  alphafold \
  /app/alphafold/scripts/download_pdb_mmcif.sh /mnt/alphafold-data
```

---

## Commands Reference

Complete command sequence, A to Z, for running this from a fresh VM with persistent disk attached.

```bash
# ─── 1. Mount Persistent Disk ─────────────────────────────────────────────────
sudo mkdir -p /mnt/alphafold-data
sudo mount /dev/sdb /mnt/alphafold-data
df -h /mnt/alphafold-data

# ─── 2. Install Dependencies ──────────────────────────────────────────────────
sudo apt update
sudo mkdir -p /mnt/alphafold-data/apt-cache
sudo apt -o dir::cache::archives="/mnt/alphafold-data/apt-cache" install -y \
  git wget curl aria2 rsync

# ─── 3. Move Docker to Persistent Disk ────────────────────────────────────────
sudo mkdir -p /mnt/alphafold-data/docker
sudo mv /var/lib/docker /var/lib/docker.bak
sudo ln -s /mnt/alphafold-data/docker /var/lib/docker
sudo systemctl start docker

# ─── 4. Clone AlphaFold ───────────────────────────────────────────────────────
cd /mnt/alphafold-data
sudo chown -R $USER:$USER /mnt/alphafold-data
git clone https://github.com/google-deepmind/alphafold.git

# ─── 5. Install NVIDIA Container Toolkit ──────────────────────────────────────
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# ─── 6. Fix Kernel for NVIDIA Driver (if nvidia-smi fails) ───────────────────
# Check current kernel
uname -r
# If output is "6.1.0-33-cloud-amd64", switch to standard kernel:
sudo grep menuentry /boot/grub/grub.cfg
sudo nano /etc/default/grub
# Set GRUB_DEFAULT to point to the "6.1.0-33-amd64" (non-cloud) entry
sudo update-grub && sudo reboot
# After reboot: uname -r should show 6.1.0-33-amd64

# ─── 7. Build Docker Image ────────────────────────────────────────────────────
cd /mnt/alphafold-data/alphafold
docker build -f docker/Dockerfile -t alphafold .

# ─── 8. Download Databases (~2.5 TB, runs for hours) ─────────────────────────
cd /mnt/alphafold-data/alphafold/scripts
nohup bash download_all_data.sh /mnt/alphafold-data full_dbs \
  > /mnt/alphafold-data/download.log 2>&1 &
tail -f /mnt/alphafold-data/download.log

# ─── 9. Create FASTA File ─────────────────────────────────────────────────────
cat > /mnt/alphafold-data/insulin.fasta << 'EOF'
>insulin
MALWMRLLPLLALLALWGPDPAAAFVNQHLCGSHLVEALYLVCGERGFFYTPKT
EOF

# ─── 10. Run Structure Prediction ─────────────────────────────────────────────
mkdir -p /mnt/alphafold-data/output
sudo docker run --rm --gpus all \
  -v /mnt/alphafold-data:/mnt/alphafold-data \
  -v /mnt/alphafold-data/alphafold:/app/alphafold \
  alphafold \
  --fasta_paths=/mnt/alphafold-data/insulin.fasta \
  --output_dir=/mnt/alphafold-data/output \
  --model_preset=monomer \
  --db_preset=full_dbs \
  --data_dir=/mnt/alphafold-data \
  --uniref90_database_path=/mnt/alphafold-data/uniref90/uniref90.fasta \
  --mgnify_database_path=/mnt/alphafold-data/mgnify/mgy_clusters_2022_05.fa \
  --template_mmcif_dir=/mnt/alphafold-data/pdb_mmcif/mmcif_files \
  --obsolete_pdbs_path=/mnt/alphafold-data/pdb_mmcif/obsolete.dat \
  --bfd_database_path=/mnt/alphafold-data/bfd/bfd_metaclust_clu_complete_id30_c90_final_seq.sorted_opt \
  --uniref30_database_path=/mnt/alphafold-data/uniref30/UniRef30_2021_03 \
  --pdb70_database_path=/mnt/alphafold-data/pdb70/pdb70 \
  --use_gpu_relax=False \
  --max_template_date=2023-01-01

# ─── 11. Verify Output ────────────────────────────────────────────────────────
ls -lh /mnt/alphafold-data/output/insulin/
```

---

## Citation

If you use AlphaFold in your work, please cite:

```bibtex
@Article{AlphaFold2021,
  author  = {Jumper, John and Evans, Richard and Pritzel, Alexander and Green, Tim
             and Figurnov, Michael and Ronneberger, Olaf and Tunyasuvunakool, Kathryn
             and Bates, Russ and Zidek, Augustin and Potapenko, Anna and Bridgland, Alex
             and Meyer, Clemens and Kohl, Simon A A and Ballard, Andrew J and Cowie, Andrew
             and Romera-Paredes, Bernardino and Nikolov, Stanislav and Jain, Rishub
             and Adler, Jonas and Back, Trevor and Petersen, Stig and Reiman, David
             and Clancy, Ellen and Zielinski, Michal and Steinegger, Martin
             and Pacholska, Michalina and Berghammer, Tamas and Bodenstein, Sebastian
             and Silver, David and Vinyals, Oriol and Senior, Andrew W
             and Kavukcuoglu, Koray and Kohli, Pushmeet and Hassabis, Demis},
  journal = {Nature},
  title   = {Highly accurate protein structure prediction with {AlphaFold}},
  year    = {2021},
  volume  = {596},
  number  = {7873},
  pages   = {583--589},
  doi     = {10.1038/s41586-021-03819-2}
}
```

This project uses the [AlphaFold source code](https://github.com/google-deepmind/alphafold) under the Apache 2.0 License. Model weights are provided under the [CC BY 4.0 License](https://creativecommons.org/licenses/by/4.0/).
