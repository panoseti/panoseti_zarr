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


## Pipeline Architecture (Stream-Based)

The pipeline implements **stream-based processing** where multiple PFF files with the same data product (`dp`) and module ID are consolidated into single Zarr files. This preserves temporal continuity across sequence numbers (`seqno`) and enables efficient batch processing.

### Pipeline Workflow

```
Observation Directory (*.pff files)
    │
    ├─> Step 1: Group by (dp, module), sort by seqno
    │       └─> Convert to consolidated L0 Zarr files
    │
    └─> Step 2: Process all L0 Zarr files
            └─> Apply baseline subtraction → L1 Zarr files
```

### Key Concepts

**Data Streams**: Files sharing the same `dp_[data_product]` and `module_[id]` belong to the same data stream. The pipeline groups these files and concatenates them along the time dimension based on `seqno` ordering.

**Example Stream Grouping**:
```
Input: 9 PFF files (seqno 0-8)
  - start_2024-07-25T04:34:46Z.dp_img16.bpp_2.module_1.seqno_0.pff
  - start_2024-07-25T05:01:09Z.dp_img16.bpp_2.module_1.seqno_1.pff
  - ... (seqno 2-7)
  - start_2024-07-25T08:04:52Z.dp_img16.bpp_2.module_1.seqno_8.pff

Output: 1 consolidated Zarr file
  - obs_Lick.start_2024-07-25T04:34:06Z.runtype_sci-data.pffd.dp_img16.module_1.zarr
```

### Step 0: Dask Cluster Setup (Optional)

```bash
python step0_setup_cluster.py config.toml /tmp/scheduler.txt
```

- Initializes Dask cluster based on `[cluster]` config
- Writes scheduler address to file
- Runs persistently until interrupted
- Can be replaced with any cluster management solution
- **Optional**: Disable by setting `use_cluster = false` in config

### Step 1: PFF to Zarr Conversion (Stream-Based)

```bash
python step1_pff_to_zarr.py <observation_dir> <output_L0_dir> [config.toml] [scheduler_address]
```

**What it does:**
- Scans observation directory for all `*.pff` files
- Groups files by `(data_product, module)` key
- Sorts each group by `seqno` (sequence number)
- Concatenates frames along time dimension into single Zarr per stream
- Preserves temporal continuity across file boundaries

**Supported data products:**
- `img16`, `img8`: 32×32 image data
- `ph256`: 16×16 photon histogram data
- `ph1024`: 32×32 photon histogram data

**Arguments:**
- `observation_dir`: Directory containing PFF files (e.g., `obs_*.pffd/`)
- `output_L0_dir`: Output directory for L0 Zarr files
- `config.toml`: Configuration file (optional)
- `scheduler_address`: Dask scheduler (e.g., `tcp://10.0.1.2:8786`)

**Processing modes:**
- **Distributed (Dask)**: Workers process file chunks in parallel
- **Local**: ProcessPoolExecutor with multiprocessing

### Step 2: Baseline Subtraction (Batch Processing)

```bash
python step2_dask_baseline.py <input_L0.zarr> <output_L1.zarr> --config config.toml [--dask-scheduler address]
```

**What it does:**
- Applies baseline/median subtraction to L0 Zarr files
- Creates L1 (science-ready) data products
- Processes each L0 Zarr file independently

**Processing algorithms:**
- **Photon data** (`ph*`): Pedestal subtraction with 5-sigma thresholding
- **Image data** (`img*`): 8×8 block median + temporal median subtraction

**Arguments:**
- `input_L0.zarr`: Input L0 Zarr file
- `output_L1.zarr`: Output L1 Zarr file
- `--config`: Configuration file
- `--dask-scheduler`: Dask scheduler address (optional)

### Orchestration Script

```bash
./run.sh <observation_directory> <output_l0_dir> <output_l1_dir>
```

**What it does:**
1. Validates observation directory and required files
2. Discovers and displays data stream structure
3. Starts Dask cluster (if enabled in config)
4. **Step 1**: Converts all PFF files to L0 Zarr (grouped by stream)
5. **Step 2**: Processes all L0 Zarr files to L1 (batch processing)
6. Displays comprehensive timing and size statistics
7. Guarantees cleanup on exit with `trap` command


## Using the `run.sh` Script

### Basic Usage

The `run.sh` script orchestrates the entire L0→L1 processing pipeline with automatic cluster management and stream-based processing.

```bash
./run.sh <observation_directory> <output_l0_dir> <output_l1_dir>
```


### Arguments

- **`observation_directory`**: Directory containing PFF files (e.g., `obs_*.pffd/`)
- **`output_l0_dir`**: Directory for L0 Zarr output (intermediate data)
- **`output_l1_dir`**: Directory for L1 Zarr output (science-ready data)


### Examples

Process an observation directory:

```bash
./run.sh /mnt/beegfs/data/L0/obs_Lick.start_2024-07-25T04:34:06Z.runtype_sci-data.pffd \
         /mnt/beegfs/zarr/L0 \
         /mnt/beegfs/zarr/L1
```

Use a custom configuration file:

```bash
CONFIG_FILE=custom.toml ./run.sh /path/to/obs.pffd /path/to/L0 /path/to/L1
```

Process with local computing (no cluster):

```bash
# Set use_cluster = false in config.toml
./run.sh /path/to/obs.pffd /path/to/L0 /path/to/L1
```


### What the Script Does

1. **Validates inputs**: Checks observation directory and required Python scripts
2. **Discovers streams**: Identifies data streams by grouping `(dp, module)` keys
3. **Starts Dask cluster**: Launches persistent cluster (if `use_cluster = true`)
4. **Step 1 - PFF to Zarr**:
    - Processes all PFF files simultaneously
    - Groups by stream and concatenates frames
    - Creates consolidated L0 Zarr files
5. **Step 2 - Baseline Subtraction**:
    - Batch processes all L0 Zarr files
    - Applies algorithm-specific baseline correction
    - Outputs L1 (science-ready) Zarr files
6. **Reports statistics**: Shows timing, throughput, compression ratios
7. **Cleanup**: Automatically shuts down cluster on completion or error


### Stream Discovery Output

When you run the script, it displays the detected data streams:

```
Discovering PFF data streams...
Found 18 PFF files

Discovered 2 data streams:

Stream: dp=img16, module=1
  Files: 9
    - start_2024-07-25T04:34:46Z.dp_img16.bpp_2.module_1.seqno_0.pff
    - start_2024-07-25T05:01:09Z.dp_img16.bpp_2.module_1.seqno_1.pff
    - start_2024-07-25T05:27:22Z.dp_img16.bpp_2.module_1.seqno_2.pff
    ... and 6 more

Stream: dp=img16, module=2
  Files: 9
    - start_2024-07-25T04:35:12Z.dp_img16.bpp_2.module_2.seqno_0.pff
    - start_2024-07-25T05:01:35Z.dp_img16.bpp_2.module_2.seqno_1.pff
    ... and 7 more

Total streams: 2
```


## Configuration with `config.toml`

The pipeline behavior is controlled by a TOML configuration file (default: `config.toml`).

### Configuration Sections

#### `[cluster]` - Dask Cluster Settings

Controls distributed computing behavior:

```toml
[cluster]
type = "local"          # Cluster type: "local" or "ssh"
use_cluster = false     # Enable/disable distributed computing

[cluster.ssh]
hosts = ["localhost", "panoseti-dfs0", "panoseti-dfs1", "panoseti-dfs2"]
workers_per_host = 1
threads_per_worker = 16
memory_per_worker = "16GB"
scheduler_port = 0      # 0 = auto-assign
dashboard_port = 8797
connect_timeout = 60

[cluster.local]
n_workers = 4
threads_per_worker = 2
memory_per_worker = "4GB"
dashboard_port = 8787
```

**Parameters:**

- **`type`**: Cluster backend (`"local"` or `"ssh"`)
- **`use_cluster`**: Enable (`true`) or disable (`false`) Dask
- **`hosts`**: List of SSH hostnames/IPs for distributed workers
- **`workers_per_host`**: Number of worker processes per host
- **`threads_per_worker`**: CPU threads per worker
- **`memory_per_worker`**: Memory limit per worker


#### `[pff_to_zarr]` - Step 1 Settings

Controls PFF-to-Zarr conversion:

```toml
[pff_to_zarr]
# Compression
codec = "blosc-lz4"  # Options: blosc-lz4, zstd, gzip, none
level = 1            # Compression level (1-9)
time_chunk = 32768   # Chunk size along time dimension

# Concurrency
max_concurrent_writes = 12  # Parallel write operations
num_workers = 8             # Local mode workers
blosc_threads = 8           # Blosc compression threads
chunk_size_mb = 50          # PFF read chunk size
```

**Parameters:**

- **`codec`**: Compression algorithm (trade-off between speed and ratio)
  - `blosc-lz4`: Fast compression, good for real-time (recommended)
  - `zstd`: Better compression, slower
  - `gzip`: Standard compression
  - `none`: No compression
- **`level`**: Compression level (1=fast, 9=best compression)
- **`time_chunk`**: Frames per Zarr chunk (affects I/O patterns)
- **`max_concurrent_writes`**: TensorStore parallel writes
- **`num_workers`**: ProcessPoolExecutor workers (local mode)


#### `[baseline_subtract]` - Step 2 Settings

Controls baseline subtraction processing:

```toml
[baseline_subtract]
baseline_window = 100       # Frames to average for baseline
codec = "blosc-lz4"         # Output compression
level = 5                   # Output compression level
compute_chunk_size = 8192   # Dask computation rechunk size
```

**Parameters:**

- **`baseline_window`**: Number of initial frames for baseline calculation
- **`codec`**: Output Zarr compression algorithm
- **`level`**: Output compression level
- **`compute_chunk_size`**: Rechunk size for Dask arrays


### Performance Tuning

**For BeeGFS/HDD clusters (sequential I/O):**

```toml
[pff_to_zarr]
codec = "blosc-lz4"
level = 1
time_chunk = 8192              # Large chunks for sequential reads
max_concurrent_writes = 12
num_workers = 8

[cluster.ssh]
threads_per_worker = 16         # Saturate CPU during compression
memory_per_worker = "16GB"
```

**For NVMe/SSD storage (parallel I/O):**

```toml
[pff_to_zarr]
codec = "zstd"
level = 3
time_chunk = 32768              # Smaller chunks for parallelism
max_concurrent_writes = 32      # More parallel writes
num_workers = 16

[cluster.ssh]
threads_per_worker = 8
memory_per_worker = "8GB"
```

**For limited RAM:**

```toml
[pff_to_zarr]
time_chunk = 16384              # Reduce memory footprint
max_concurrent_writes = 6
num_workers = 4

[cluster.ssh]
threads_per_worker = 4
memory_per_worker = "4GB"

[baseline_subtract]
compute_chunk_size = 4096       # Smaller chunks
```

**For local processing (no cluster):**

```toml
[cluster]
use_cluster = false

[pff_to_zarr]
num_workers = 8                 # Use all CPU cores
blosc_threads = 8
max_concurrent_writes = 16
```


## Running Individual Steps

Process steps can be run independently for debugging or custom workflows:

### Step 1: PFF to Zarr (Stream-Based)

```bash
# With Dask cluster
python3 step1_pff_to_zarr.py /path/to/obs.pffd /output/L0 config.toml tcp://scheduler:8786

# Local mode
python3 step1_pff_to_zarr.py /path/to/obs.pffd /output/L0 config.toml
```

**Input**: Observation directory containing `*.pff` files  
**Output**: One Zarr file per `(dp, module)` stream


### Step 2: Baseline Subtraction

```bash
# With Dask cluster
python3 step2_dask_baseline.py input_L0.zarr output_L1.zarr \
    --config config.toml \
    --dask-scheduler tcp://scheduler:8786

# Local mode
python3 step2_dask_baseline.py input_L0.zarr output_L1.zarr \
    --config config.toml
```

Command-line arguments override config file settings:

```bash
python3 step2_dask_baseline.py input.zarr output.zarr \
    --baseline-window 200 \
    --codec zstd \
    --level 5
```


## Output Structure

### L0 (Intermediate Data)

After Step 1, L0 Zarr files contain raw data grouped by stream:

```
output_L0/
├── obs_Lick.start_2024-07-25T04:34:06Z.runtype_sci-data.pffd.dp_img16.module_1.zarr/
│   ├── images/          # Raw image data (time, y, x)
│   ├── timestamps/      # Frame timestamps
│   ├── headers/         # Metadata from PFF headers
│   │   ├── quabo_0/
│   │   ├── quabo_1/
│   │   ├── quabo_2/
│   │   └── quabo_3/
│   └── zarr.json        # Zarr v3 metadata
│
└── obs_Lick.start_2024-07-25T04:34:06Z.runtype_sci-data.pffd.dp_img16.module_2.zarr/
    └── ...
```

**Naming convention**: `{obs_dir_name}.dp_{data_product}.module_{module_id}.zarr`

### L1 (Science-Ready Data)

After Step 2, L1 Zarr files contain baseline-subtracted data:

```
output_L1/
├── obs_Lick.start_2024-07-25T04:34:06Z.runtype_sci-data.pffd.dp_img16.module_1_L1.zarr/
│   ├── images/          # Baseline-subtracted image data
│   ├── timestamps/      # Frame timestamps
│   ├── headers/         # Preserved metadata
│   └── zarr.json
│
└── obs_Lick.start_2024-07-25T04:34:06Z.runtype_sci-data.pffd.dp_img16.module_2_L1.zarr/
    └── ...
```

### Accessing Zarr Data

**Using xarray** (recommended):

```python
import xarray as xr

# Open L1 Zarr file
ds = xr.open_zarr('output_L1/obs_*.dp_img16.module_1_L1.zarr', 
                   consolidated=False)

# Access arrays
images = ds['images']        # (time, y, x)
timestamps = ds['timestamps'] # (time,)

# Slice data
subset = images[1000:2000, :, :]  # 1000 frames
```

**Using TensorStore** (high-performance):

```python
import tensorstore as ts

# Open specific array
images = await ts.open({
    'driver': 'zarr3',
    'kvstore': {'driver': 'file', 'path': 'output_L1/obs_*.zarr'},
    'path': 'images'
})

# Efficient slicing
chunk = images[10000:20000, :, :].read().result()
```



## Troubleshooting

### "No valid PFF files found"

Check that:
- Observation directory path is correct
- Directory contains `*.pff` files
- Filenames follow naming convention: `*.dp_*.module_*.seqno_*.pff`

### "Frames written < expected"

Possible causes:
- Corrupted PFF files (check with `pff.img_info()`)
- Incorrect frame structure parameters
- Review Step 1 conversion logs

### "Step 2 failed" / Baseline subtraction errors

Check:
- All L0 Zarr files created successfully in Step 1
- Sufficient disk space for L1 output
- Dask cluster connectivity (if enabled)
- Memory limits in config (reduce if necessary)

### Dask cluster connection timeout

Solutions:
- Verify SSH connectivity: `ssh hostname`
- Check `connect_timeout` in config (increase to 120s)
- Ensure firewall allows scheduler port
- Set `use_cluster = false` to use local processing

### Performance Issues

**Slow compression:**
- Use `codec = "blosc-lz4"` with `level = 1`
- Increase `blosc_threads` and `threads_per_worker`

**High memory usage:**
- Reduce `time_chunk` size
- Lower `compute_chunk_size`
- Decrease `memory_per_worker`
- Reduce `num_workers` in local mode

**Slow I/O on HDD:**
- Increase `time_chunk` for sequential reads
- Use `codec = "blosc-lz4"` (fast compression)
- Reduce `max_concurrent_writes` to avoid thrashing


## Advanced Usage

### Processing Specific Streams

To process only certain data products:

```bash
# Extract specific streams manually
python3 -c "
from pathlib import Path
import pff

obs_dir = Path('/path/to/obs.pffd')
for pff_file in obs_dir.glob('*dp_img16*module_1*.pff'):
    print(pff_file)
"
```

Then copy those files to a temporary directory and process:

```bash
./run.sh /tmp/filtered_obs/ /output/L0 /output/L1
```


## Development

### Running Tests

```bash
# Test PFF parsing
python3 -c "import pff; print(pff.parse_name('test.dp_img16.module_1.seqno_0.pff'))"

# Validate stream grouping
python3 step1_pff_to_zarr.py --help

# Check Zarr structure
python3 -c "import zarr; print(zarr.open('output_L0/test.zarr', mode='r').tree())"
```

### Contributing

Contributions welcome! Please:
1. Follow existing code style (Black formatting)
2. Add docstrings to new functions
3. Update README for user-facing changes
4. Test with both local and distributed modes


## References

- [Zarr v3 Specification](https://zarr.dev/zeps/accepted/ZEP0001.html)
- [TensorStore Documentation](https://google.github.io/tensorstore/)
- [Dask Distributed](https://distributed.dask.org/)
- [PANOSETI Project](https://panoseti.org/)


## License

See LICENSE file for details.
