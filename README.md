# AlphaFold2 – Insulin Structure Prediction

This repository documents my hands-on experience running DeepMind's AlphaFold2 on Google CLoud using Docker to predict the 3D structure of human insulin.

##  Objective
To predict the folded structure of human insulin from its amino acid FASTA sequence using the full database mode.

##  Stack
- AlphaFold2 (Docker)
- NVIDIA GPU (CUDA)
- HHblits, Jackhmmer, PDB70, UniRef90, MGnify, BFD
- Debian-based VM on cloud

## Outputs
Includes PDB and CIF files for each unrelaxed predicted model (1–5). These files can be visualized using PyMOL or Mol*.

---

##  Citation


```bibtex
@article{jumper2021highly,
  title={Highly accurate protein structure prediction with AlphaFold},
  author={Jumper, John and Evans, Richard and Pritzel, Alexander and Green, Tim and Figurnov, Michael and Ronneberger, Olaf and Tunyasuvunakool, Kathryn and Bates, Russ and {\v{Z}}{\'\i}dek, Augustin and Potapenko, Anna and others},
  journal={Nature},
  volume={596},
  number={7873},
  pages={583--589},
  year={2021},
  publisher={Nature Publishing Group}
}

