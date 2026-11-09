# Cloud Computing Final Project — Graph Deanonymization

Iterative graph deanonymization on Databricks Spark. Given two graphs (original and anonymized), a partial seed mapping, and ground-truth mappings, the notebook progressively identifies which anonymized node corresponds to which original node using similarity-based scoring.

Based on the original shared-neighbor/eccentricity formula by Arvind Narayanan & Vitaly Shmatikov (2009).

## Prerequisites

- **Databricks workspace** on AWS with access to a compute cluster (see Cell Guide for per-cell requirements)
- **Unity Catalog Volume** at `/Volumes/custom_compute/default/deanon_project/` (used by Cells 10–11 for lineage-breaking parquet temp writes)
- **Python packages:** `networkx` and `graphframes` (installed by Cells 1–2)

## Import & Setup

The notebook is included as a `.ipynb` file (Jupyter Notebook format). To run it in Databricks:

1. **Extract** the submission zip to a local folder.
2. **Import the notebook:**
   - In your Databricks workspace, navigate to the target directory (e.g., your user folder).
   - Click the **⋮** menu (or right-click) → **Import**.
   - Select `CloudComputing Project Notebook.ipynb` from the extracted files.
   - Databricks will import it as a multi-cell notebook.
3. **Upload the data files:**
   - In the same workspace directory, create a `Data/` folder.
   - Upload the contents of `Data/Smaller Set/` and `Data/6mil edges set/` into their respective subfolders.
   - Alternatively, upload the data to a Unity Catalog Volume and adjust the parameter paths accordingly.
4. **Create the UC Volume** (required for Cells 10–11):
   - Navigate to **Catalog** → your catalog/schema → **Create Volume**.
   - The expected path is `/Volumes/custom_compute/default/deanon_project/`. If using a different path, update `SEEDS_TEMP_PATH` in Cells 10–11.
5. **Update parameter paths:** After importing, set the six query parameter widgets at the top of the notebook to point to the locations where you uploaded the data files (see Configure Parameters below).

## Directory Structure

```
Cloud Computing Final Project/
├── CloudComputing Project Notebook.ipynb               # Main notebook (all algorithms)
├── README.md                                           # This file
├── log.txt                                             # Execution log output
├── Data/
│   ├── Smaller Set/                                    # Small graph (7,126 nodes)
│   │   ├── nonAnon Edgelist.txt                        #   Original graph edges
│   │   ├── anonSmaller.txt                             #   Anonymized graph edges
│   │   ├── smallerMappings SOME.txt                    #   Partial seed mappings (input)
│   │   ├── smallerMappings ALL.txt                     #   Ground-truth mappings (validation)
│   │   ├── smallerSet - aa deanon seeds.txt            #   AA algorithm output seeds
│   │   └── smallerSet - cloud spark.txt                #   Spark eccentricity output
│   └── 6mil edges set/                                 # Large graph (168K nodes, ~6M edges)
│       ├── largeSet edgelist.txt                       #   Original graph edges
│       ├── anonLargerSet.txt                           #   Anonymized graph edges
│       ├── largerSet mappings SOME.txt                 #   Partial seed mappings (input)
│       ├── largerSet mappings ALL.txt                  #   Ground-truth mappings (validation)
│       └── largerSet - aa deanon seeds.txt             #   AA algorithm output seeds
```

## Cell Guide

| Cell | Title | Required Compute | Notes |
| ---- | ----- | ---------------- | ----- |
| 1 | *(pip install networkx)* | Any | Run once per cluster session |
| 2 | Install GraphFrames | Any | Run once per cluster session |
| 3 | Original Python Multithreading | **Will NOT run on Databricks** | For reference to original algorithm only. Uses `multiprocessing.Pool`, `Manager`, `Lock` |
| 4 | Local Eccentricity Benchmarks | Any (Serverless or Classic) | matplotlib charts for Cell 3's performance |
| 5 | Spark, Serverless Compatible | Serverless or Classic | Does not complete execution. OOMs after ~1 iteration. |
| 6 | Spark v2: Hybrid Approach | **Classic single-node** (i3.xlarge) | Uses `localCheckpoint`, heavy caching. NOT serverless-compatible |
| 7 | **Shared Helper Functions** | Any (Serverless or Classic) | **Run before any cells below (Cells 8–11)** |
| 8 | Jaccard Similarity | **Classic single-node** (i3.xlarge) | Depends on Cell 7 helpers |
| 9 | Adamic-Adar (single node) | **Classic single-node** (i3.xlarge) | Practical only for small graph |
| 10 | Adamic-Adar (multi node) | **Classic multi-node** (r5d.xlarge) | Switches to UC Volume for lineage-breaking parquet writes |
| 11 | **AA, multinode + large graph optimizations** | **Classic multi-node** (r5d.xlarge) | Final version of algorithm. Broadcast seeds, hub cap, dynamic partitions, Spark-native loading |

## How to Run

### 1. Configure Parameters

Set the six query parameters (widgets) at the top of the notebook:

| Parameter | Description |
| --------- | ----------- |
| `g1fp` | Original graph edgelist |
| `g2fp` | Anonymized graph edgelist |
| `givenseedsfp` | Partial seed mappings (input) |
| `trueseedsfp` | Ground-truth mappings (for accuracy check) |
| `outputfp` | Where to write discovered seed mappings |
| `logfp` | Log file path |

Point each parameter at the corresponding file under `Data/Smaller Set/` or `Data/6mil edges set/`.

### 2. Run

**Adamic-Adar (large graph):** Attach a classic multi-node cluster (r5d.xlarge, 2+ workers with Photon)

```
Cell 1 → Cell 2 → Cell 7 → Cell 11
```

**Small graph exploration:** a classic single-node cluster (i3.xlarge) is sufficient

```
Cell 1 → Cell 2 → Cell 7 → Cell 9 (or Cell 10)
```

Results (accuracy, matches, runtime) are printed at the end of each algorithm cell and written to `log.txt`. Output seed mappings are written to the `outputfp` path.
For detailed explanation of evolution of algorithm across cells and results, see full report.
