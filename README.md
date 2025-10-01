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


## Pipeline Overview

This basic pipeline processes PANOSETI File Format (PFF) files in two stages:

1. **Step 1 (L0):** Convert PFF binary files to Zarr format with compression
2. **Step 2 (L1):** Apply baseline subtraction to produce science-ready data

The pipeline supports both **distributed computing** via Dask clusters and **local multiprocessing** modes.

## Using the `run.sh` Script

### Basic Usage

The `run.sh` script orchestrates the entire L0→L1 processing pipeline with automatic Dask cluster management.

```bash
./run.sh <input_pff_file1> [input_pff_file2 ...] <output_l1_dir>
```


### Arguments

- **`input_pff_file`**: One or more PFF files to process (e.g., `data_img16_*.pff`)
- **`output_l1_dir`**: Directory where L1 Zarr products will be written


### Examples

Process a single PFF file:

```bash
./run.sh /path/to/L0/<file name>.pff /scratch/output/L1
```

Process multiple PFF files:

```bash
./run.sh file1.pff file2.pff file3.pff /scratch/output/L1
```

Use a custom configuration file:

```bash
CONFIG_FILE=custom.toml ./run.sh data.pff /scratch/output/L1
```


### What the Script Does

1. **Validates inputs**: Checks that all PFF files exist and required Python scripts are present
2. **Creates directories**: Sets up L0 temporary storage and L1 output directories
3. **Starts Dask cluster**: Launches a persistent distributed cluster (if `use_dask = true` in config)
4. **Processes each file**:
    - Step 1: Converts PFF → Zarr (L0) with compression
    - Step 2: Applies baseline subtraction (L0 → L1)
    - Cleans up intermediate L0 files
5. **Cleanup**: Automatically shuts down the Dask cluster on completion or error

### Cluster Management

The script uses `cluster_lifecycle_manager.py` to create a **persistent Dask cluster** that is reused across all input files.

## Configuration with `config.toml`

The pipeline behavior is controlled by a TOML configuration file (default: `config.toml`).

### Configuration Sections

#### `[cluster]` - Dask Cluster Settings

Controls distributed computing behavior:

```toml
[cluster]
use_dask = true  # Enable distributed processing (false = local multiprocessing)

# SSH-based multi-node cluster configuration
ssh_hosts = ["localhost", "panoseti-dfs0", "panoseti-dfs1", "panoseti-dfs2"]
ssh_workers_per_host = 1
ssh_threads_per_worker = 16
ssh_memory_per_worker = "16GB"
```

**Parameters:**

- **`use_dask`**: Enable (`true`) or disable (`false`) distributed computing
- **`ssh_hosts`**: List of hostnames/IPs for Dask workers
- **`ssh_workers_per_host`**: Number of worker processes per host
- **`ssh_threads_per_worker`**: CPU threads per worker
- **`ssh_memory_per_worker`**: Memory limit per worker


#### `[pff_to_zarr]` - Step 1 Settings

Controls PFF-to-Zarr conversion:

```toml
[pff_to_zarr]
# Compression
codec = "blosc-lz4"  # Options: blosc-lz4, zstd, gzip, none
level = 5            # Compression level (1-9, higher = better compression, slower)
time_chunk = 65536   # Chunk size along time dimension

# Local multiprocessing (when use_dask = false)
num_workers = 10
blosc_threads = 8
max_concurrent_writes = 16
chunk_size_mb = 150
```

**Parameters:**

- **`codec`**: Compression algorithm (`blosc-lz4`, `zstd`, `gzip`, or `none`)
- **`level`**: Compression level (1=fast, 9=best compression)
- **`time_chunk`**: Number of frames per Zarr chunk (affects I/O performance)
- **`num_workers`**: Worker processes for local mode
- **`blosc_threads`**: Blosc compression threads
- **`max_concurrent_writes`**: Maximum parallel write operations
- **`chunk_size_mb`**: Target chunk size for reading PFF files


#### `[baseline_subtract]` - Step 2 Settings

Controls baseline subtraction processing:

```toml
[baseline_subtract]
baseline_window = 100  # Number of initial frames to average for baseline
codec = "blosc-lz4"
level = 5
compute_chunk_size = 8192  # Rechunk size for computation
```

**Parameters:**

- **`baseline_window`**: Number of first frames to average for baseline calculation
- **`codec`**: Output compression algorithm
- **`level`**: Output compression level
- **`compute_chunk_size`**: Rechunk size for Dask computation


### Performance Tuning

**For BeeGFS/HDD clusters:**

- Use `codec = "blosc-lz4"` with `level = 5` for balanced speed/compression
- Set `time_chunk = 65536` for large sequential reads
- Use `ssh_threads_per_worker = 16` to saturate CPU during compression

**For NVMe/SSD storage:**

- Increase `max_concurrent_writes = 32`
- Use smaller `time_chunk = 32768` for better parallelism
- Consider `codec = "zstd"` for better compression ratios

**For limited RAM:**

- Reduce `ssh_memory_per_worker = "8GB"`
- Decrease `compute_chunk_size = 4096`
- Lower `ssh_threads_per_worker = 8`


## Running Individual Steps

Process steps can be run independently for debugging or custom workflows:

### Step 1: PFF to Zarr (L0)

```bash
# With Dask cluster
python3 step1_pff_to_zarr.py input.pff output_L0.zarr config.toml tcp://scheduler:8786

# Local mode
python3 step1_pff_to_zarr.py input.pff output_L0.zarr config.toml
```


### Step 2: Baseline Subtraction (L1)

```bash
# With Dask cluster
python3 step2_dask_baseline.py input_L0.zarr output_L1.zarr --config config.toml --dask-scheduler tcp://scheduler:8786

# Local mode
python3 step2_dask_baseline.py input_L0.zarr output_L1.zarr --config config.toml
```

Command-line arguments override config file settings:

```bash
python3 step2_dask_baseline.py input.zarr output.zarr --baseline-window 200
```


## Output Structure

After processing, the L1 directory contains Zarr datasets:

```
output_L1/
├── obs_20250101_img16_L1.zarr/
│   ├── images/          # Baseline-subtracted image data
│   ├── timestamps/      # Frame timestamps
│   ├── headers/         # Metadata from PFF headers
│   └── .zarray, .zattrs, etc.
└── obs_20250102_img16_L1.zarr/
    └── ...
```

Each L1 Zarr dataset contains:

- **`images`**: 3D array `(time, y, x)` with baseline-subtracted pixel values
- **`timestamps`**: Unix timestamps for each frame
- **`headers`**: Extracted metadata (packet numbers, TAI times, etc.)
