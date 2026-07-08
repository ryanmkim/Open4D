# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Open4D is an open research platform for representation, compression, processing, evaluation, and streaming of time-varying 4D geometry data. This is Ryan's fork. Target domains: XR, robotics, teleoperation, digital twins, autonomous systems.

## Installation

```bash
pip install -e .
```

Only installs the `open4d` Python package (numpy dependency). The research modules under `open4d/core/` are **git submodules** with their own environments—see their individual READMEs.

## Repository Structure

```
Open4D/
├── open4d/             # Installable Python package
│   ├── io/             # .o4d format readers/writers
│   ├── player/         # PyQt6/pyqtgraph visualization
│   ├── tools/          # Batch conversion scripts
│   ├── core/           # Research modules (git submodules + standalone)
│   │   ├── N4MC/       # Neural 4D Mesh Compression (submodule)
│   │   ├── tsmc/       # Time-varying 4D Scene Mesh Compression (submodule)
│   │   └── tvmc/       # Time-Varying Mesh Compression (standalone)
│   ├── modules/        # Placeholder for future algorithm integrations
│   └── metrics/        # Placeholder for quality/distortion metrics
├── examples/           # Usage examples for players
├── benchmarks/         # Placeholder for reproducible evaluations
├── apps/               # Placeholder for end-to-end pipelines
└── docker/             # Placeholder for reproducible environments
```

## Core IO: The `.o4d` Binary Format

Central to the project is a custom chunk-based binary container (little-endian). Every chunk is `4-byte type | u64 payload_size | payload`.

- **HEAD** — Magic `O4D1`, version (`1`), geometry type, codec type, JSON metadata
- **KFRM** — One keyframe of geometry data
- **INDX** — Frame index at end of file for O(1) random access

Geometry/codec combinations:

| Reader/Writer class | Geom | Codec | Data per frame |
|---|---|---|---|
| `O4DMeshWriter/Reader` | `GEOM_MESH=1` | `CODEC_RAW_MESH=1` | float32 (N,3) verts + uint32 (M,3) faces |
| `O4DPointCloudWriter/Reader` | `GEOM_POINTCLOUD=2` | `CODEC_RAW_POINTS=1` | float32 (N,3) XYZ + optional uint8 (N,3) RGB |
| `O4DDracoPointCloudWriter/Reader` | `GEOM_POINTCLOUD=2` | `CODEC_DRACO_POINTS=3` | Draco-compressed blob (requires `DracoPy`) |

All IO classes are in `open4d/io/` and support context managers (`with ... as w/r`).

```python
from open4d.io.o4d_mesh_io import O4DMeshWriter, O4DMeshReader

with O4DMeshWriter("out.o4d") as w:
    w.write_keyframe(vertices, faces, timestamp=t, frame_index=i)

with O4DMeshReader("out.o4d") as r:
    verts, faces, ts = r.get_frame(0)
```

## Research Modules Under `open4d/core/`

These are self-contained research codebases, each with their own conda environments and dependencies. They are **not** part of the installable `open4d` package.

### N4MC (git submodule)
Neural 4D Mesh Compression. Uses TSDF-Def tensors + auto-decoder + interpolation transformer.
- Environment: `conda env create -f n4mc_source/environment.yml` (Python 3.10, PyTorch, Open3D 0.19.0, CUDA)
- Requires .NET 5.0 + 7.0 (for ARAP volume tracking)
- Pipeline: volume tracking → scale meshes → TSDF-Def tensors → train auto-decoder → train interpolation → evaluate

### TSMC (git submodule)
Time-varying 4D Scene Mesh Compression (SIGGRAPH 2026). Extends TVMC with SAM3-based scene decomposition into static/dynamic parts.
- Environment: `conda env create -f environment.yml` (Python 3.12)
- Requires .NET 5.0 + 7.0
- Pipeline: SAM3 segmentation → volume tracking → reference centers → transformations → mesh deformation → reference mesh extraction → displacements → compression → evaluation
- Contains its own bundled copy of `arap-volume-tracking` and `draco`

### TVMC (not a submodule, lives directly in repo)
Time-Varying Mesh Compression (ACM MMSys 2025). The foundational method that N4MC and TSMC build upon.
- Environment: Python 3.8, `open3d==0.18.0`, `scikit-learn`, `scipy`, `trimesh==4.1.0`
- Requires .NET 5.0 + 7.0
- Docker: `docker build -t tvmc-linux .` from `open4d/core/tvmc/`
- Pipeline: ARAP volume tracking → MDS reference centers → dual quaternion transforms → tvm-editing mesh deformation → reference mesh → displacements → Draco compression → evaluation
- Sub-components: `arap-volume-tracking/` (C#/.NET), `tvm-editing/` (C#/.NET), `TVMC/` (Python scripts)

### Common Dependency: ARAP Volume Tracking
All three modules rely on ARAP volume tracking (C#/.NET 7.0). The `arap-volume-tracking/` directory appears in both `tvmc/` and `tsmc/`. Key commands:
```bash
cd arap-volume-tracking
dotnet build -c release
dotnet ./bin/Client.dll ./config/max/<config.xml>
```

## Player / Visualization

`open4d/player/` provides PyQt6+pyqtgraph viewers. Requires: `PyQt6`, `pyqtgraph`.

```bash
python examples/play_mesh_o4d.py           # mesh_file.o4d
python examples/play_poincloud_o4d.py       # pointcloud .o4d
python examples/play_draco_pointcloud_o4d.py
```

## Tools

`open4d/tools/` has batch conversion scripts (folder of frames → single `.o4d`). These use **relative imports** from `open4d/io/`, so they need to be run from within `open4d/io/` or with `sys.path` adjusted. They depend on `trimesh` for loading `.obj`/`.ply` files.

## Git Submodules

```bash
git clone --recursive <repo>
# or after clone:
git submodule update --init --recursive
```

Registered submodules (`.gitmodules`):
- `open4d/core/tsmc` → `https://github.com/SINRG-Lab/TSMC.git`
- `open4d/core/N4MC` → `https://github.com/frozzzen3/N4MC.git`
