# panoseti_zarr
Zarr utilities for the PANOSETI data reduction pipeline

# Environment Setup
Install `miniconda` ([link](https://www.anaconda.com/docs/getting-started/miniconda/install)), then follow these steps:
```bash
# 0. Clone this repo and go to the repo root 
git clone https://github.com/panoseti/panoseti_zarr.git
cd panoseti_zarr

# 1. Create the zarr-py313 conda environment
conda create -n zarr-py313 python=3.13
conda activate zarr-py313

# 2. Install package dependencies
pip install -r requirements.txt

```