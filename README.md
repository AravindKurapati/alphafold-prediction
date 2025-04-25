# AlphaFold2 – Insulin Structure Prediction

This repository documents my hands-on experience running DeepMind's AlphaFold2 on Google CLoud using Docker to predict the 3D structure of human insulin.

##  Objective
To predict the folded structure of human insulin from its amino acid FASTA sequence using the full database mode.

##  Stack
- AlphaFold2 (Docker)
- NVIDIA GPU (CUDA)
- HHblits, Jackhmmer, PDB70, UniRef90, MGnify, BFD
- Debian-based VM on cloud


## Insulin Output Files

This folder (`alphafold_insulin_output/insulin/`) contains:
- `unrelaxed_model_X_pred_0.pdb`: Raw structure predictions
- `unrelaxed_model_X_pred_0.cif`: Crystallographic Information Format for models
- `result_model_X_pred_0.pkl`: Python dictionary containing per-residue confidence scores, distograms, etc.
- `confidence_model_X_pred_0.json`: Confidence metrics
- `features.pkl`: Input features fed to the model

---

## Predicted Insulin Structure 

Below is the visualized result from the `unrelaxed_model_1_pred_0.pdb` output.

![Insulin 3D Structure](images/insulin_structure.png)

The structure was rendered using [Mol* Viewer](https://molstar.org/viewer/).


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

