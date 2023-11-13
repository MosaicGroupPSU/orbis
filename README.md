<h1 align="center">ORBIS</h1>

[![arXiv](https://img.shields.io/badge/arXiv-2302.14231-blue)](http://arxiv.org/abs/2311.06179)
[![Requires Python 3.9+](https://img.shields.io/badge/Python-3.9+-blue.svg?logo=python&logoColor=white)](https://python.org/downloads)

**O**ptimally **R**escaled **B**ayesian **I**terative **S**ampling (ORBIS) is an efficient method for construction of cluster expansions for complex, nonperiodic, or low-dimensional systems with a high number of components.
<p align="center">
<img src="https://github.com/MosaicGroupPSU/orbis/blob/d76a063ab02f368249088687301ccc6790bea2db/orbis_logo.png" width="60%">
</p>

## Example Notebook

[Open in Google Colab]: https://colab.research.google.com/assets/colab-badge.svg

An example for Pt:Ni system:       [![Open in Google Colab]](https://colab.research.google.com/drive/1SMQreJ8h0Jd1biTIxc6N6Pca0jlBfXxK?usp=sharing)

## Usage
### Generating structures
The first step is to generate training structures. [ICET](https://icet.materialsmodeling.org/advanced_topics/structure_enumeration.html) is one of the packages that can be used. 
Then you should calculate the energy of all the structures using empirical potentials.
The last step is to construct a cluster expansion for each empirical potential and to calculate the energy of each structure using each cluster expansion.
Note that you need to export the matrix $\Pi$ from your cluster expansion to the Bayesian structure selection code. 
You can find the tutorial for constructing a cluster expansion using ICET [here](https://icet.materialsmodeling.org/tutorial/index.html#building-a-cluster-expansion).


To tailor the example code to your specific project you should change the following: 
### Datasets
You should start by importing all the energies that you have from empirical potentials and the matrix $\Pi$ which is attained from cluster expansion.

```python
import numpy as np
import pandas as pd

EP = pd.read_excel('location of your dataset')

Pi = np.load('location of your matrix')
```

### Variable Initialization
Here the starting variables should be changed based on your data:

```python
# Define the number of optimization loops and the total number of structures
number_of_loops = 74
n_structures = 370

# Extract and scale energy values from empirical potentials DataFrame
# for different interatomic potentials.
# *0.001 converts the energy values from meV/f.u. to eV/f.u..
f_comb = EP['Cluster_COMB3 (meV/f.u.)']*0.001
f_reaxff = EP['Cluster_ReaxxFF (meV/f.u.)']*0.001
f_meam = EP['Cluster_MEAM (meV/f.u.)']*0.001
f_eam = EP['Cluster_EAM (meV/f.u.)']*0.001
f_CHGNet = EP['Cluster_CHGNet (meV/f.u.)']*0.001
f_M3GNet = EP['Cluster_M3GNet (meV/f.u.)']*0.001
```

### Data Processing
Here you can change `ep` and `X` based on the energies that you imported.
`K` is constructed by introducing a slight diagonal perturbation to the original covariance matrix k. This perturbation is essential to guarantee that the prior distribution adequately encompasses the energies obtained from DFT. To account for potentially large uncertainties in the empirically estimated energies, we assign a greater weight (i.e., a higher confidence) to DFT energies using `w`.
Note that `K` and `w` can be changed based on your system. More information on that can be found in our [paper](http://arxiv.org/abs/2311.06179).

```python
# Create a DataFrame 'ep' by stacking energy arrays for various interatomic potentials
ep = pd.DataFrame([f_comb, f_reaxff, f_meam, f_eam, f_CHGNet, f_M3GNet])

# Calculate the mean of energy values across different interatomic potentials
mu = np.mean(ep, axis=0)

# Stack energy arrays into matrix 'X' for further calculations
X = np.stack((f_comb, f_reaxff, f_eam, f_meam, f_CHGNet, f_M3GNet))

# Calculate the covariance matrix 'k' for the stacked matrix 'X'
k = np.cov(X.T)

# Add a small diagonal perturbation to the covariance matrix 'K'
K = k + np.identity(n_structures) * 0.001

# Initialize a square matrix 'w' with zeros and set diagonal elements to 0.10 to assign a higher weight to DFT calculations.
w = np.zeros((n_structures, n_structures))
np.fill_diagonal(w, 0.10)
```
 ### Iterative Optimization Loop
This part of the code mainly remains the same except for the scaling part. You should change the following lines based on your variables:

```python
# Calculate scaling factors for various potential energy contributions
    scaling_factor_meam = np.dot(f_sampled, f_meam[index_sampled]) / np.dot(f_meam[index_sampled], f_meam[index_sampled])
    scaling_factor_eam = np.dot(f_sampled, f_eam[index_sampled]) / np.dot(f_eam[index_sampled], f_eam[index_sampled])
    scaling_factor_reaxff = np.dot(f_sampled, f_reaxff[index_sampled]) / np.dot(f_reaxff[index_sampled], f_reaxff[index_sampled])
    scaling_factor_comb = np.dot(f_sampled, f_comb[index_sampled]) / np.dot(f_comb[index_sampled], f_comb[index_sampled])
    scaling_factor_CHGNet = np.dot(f_sampled, f_CHGNet[index_sampled]) / np.dot(f_CHGNet[index_sampled], f_CHGNet[index_sampled])
    scaling_factor_M3GNet = np.dot(f_sampled, f_M3GNet[index_sampled]) / np.dot(f_M3GNet[index_sampled], f_M3GNet[index_sampled])

    # Rescale potential energy contributions for interatomic potentials
    f_comb_rescaled = f_comb * scaling_factor_comb
    f_reaxff_rescaled = f_reaxff * scaling_factor_reaxff
    f_meam_rescaled = f_meam * scaling_factor_meam
    f_eam_rescaled = f_eam * scaling_factor_eam
    f_CHGNet_rescaled = f_CHGNet * scaling_factor_CHGNet
    f_M3GNet_rescaled = f_M3GNet * scaling_factor_M3GNet

    # Create a DataFrame with rescaled potential energy contributions
    ep_rescaled = pd.DataFrame([f_comb_rescaled, f_reaxff_rescaled, f_meam_rescaled, f_eam_rescaled, f_CHGNet_rescaled, f_M3GNet_rescaled])

    # Calculate the mean of rescaled contributions and their covariance matrix
    X_rescaled = np.stack((f_comb_rescaled, f_reaxff_rescaled, f_meam_rescaled, f_eam_rescaled, f_CHGNet_rescaled, f_M3GNet_rescaled))
```
### Results
You should get an array that shows the index of the structures that need to be sampled as an output for each iteration:

`index of sampled structures: [369, 354, 353, 274, 258, 351, 350, 312, 25, 20]`

You should calculate the energy of these structures using DFT and re-iterate. 

## Reference

If you use ORBIS, please cite [this paper](http://arxiv.org/abs/2311.06179):

```bib
@misc{dana2023cluster,
      title={Cluster Expansion by Transfer Learning from Empirical Potentials}, 
      author={A. Dana and L. Mu and S. Gelin and S. B. Sinnott and I. Dabo},
      year={2023},
      eprint={2311.06179},
      archivePrefix={arXiv},
      primaryClass={cond-mat.mtrl-sci}
}
```

