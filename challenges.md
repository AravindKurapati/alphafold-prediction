# Troubleshooting AlphaFold

## Storage & Resource Issues
- Hit 100% disk capacity while extracting the 1.4TB BFD database.
- Resolved by requesting additional storage and monitoring space using `df -h`.
- Disk expansion didn’t reflect immediately; required a full VM reboot to remount and recognize added storage.

## Database & Extraction Issues
- Encountered error: `ValueError: Could not find HHBlits database`.
- Root cause: `bfd_metaclust_clu_complete_id30_c90_final_seq.sorted_opt.tar.gz` was partially extracted due to low disk space.
- Resolution:
  - Re-extracted the tarball using `tar -xvzf` after freeing up space.
  - Confirmed presence of all `.ffdata` and `.ffindex` files using `ls -lh`.

## NVIDIA Driver & CUDA Issues
- Running `nvidia-smi` initially failed due to broken NVIDIA driver (`nvidia-dkms` couldn't be configured).
- `amber_minimize.py` failed repeatedly with: No compatible CUDA device is available Minimization failed after 100 attempts.

- Solution: Skipped relaxation step by using the flag `--use_gpu_relax=False`.

## Docker Container & Mounting Issues
- AlphaFold couldn't detect databases due to incorrect volume mounts.
- Fixed by ensuring all paths were consistently mounted as: -v /mnt/alphafold-data:/mnt/alphafold-data -v /mnt/alphafold-data/alphafold:/app/alphafold
- Verified database paths inside the container using `ls` to ensure visibility.

## Connectivity & Lifecycle Interruptions
- The VM shut down mid-download due to bandwidth or inactivity limits.
- Large dataset downloads (like BFD, PDB, UniRef) were interrupted frequently.
- Solutions:
- Used `aria2c` and `rsync` to resume downloads reliably.
- Avoided `wget` on Kaggle by pre-cloning necessary files into mounted paths.

