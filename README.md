# Structure-Aware Raster-Vector Integration for Post-Landslide Road-Network Assessment and Safety-Routing

**DR-Carys** is a geospatial decision-support prototype for post-landslide road-network assessment. It combines raster landslide evidence with a vector road graph, identifies road segments exposed to predicted hazards, and returns a safety-oriented route through the remaining network.

The included web application demonstrates the workflow for the Chimanimani, Zimbabwe study area following the March 2019 event.

## Overview

Satellite-derived landslide probability alone does not describe which roads are affected or whether an alternative route remains available. DR-Carys links that raster evidence to individual road segments and their network topology:

1. A post-event raster is accepted as either a georeferenced landslide-probability GeoTIFF or a supported multi-temporal Sentinel-2 NetCDF scene.
2. Hazard pixels above a conservative threshold are converted into polygons and buffered.
3. The buffered hazard footprint is intersected with an intersection-aware road graph.
4. Raster evidence and vector/topological attributes are fused by a weakly supervised GMU + GAT model.
5. Exposed segments are withheld from the routing graph and a safe route is calculated where a connected alternative exists.

## Repository contents

```
DR-carys/
├── app/drcarys-v3/                 Browser dashboard and runtime assets
│   ├── dist/                       Static interface, map layers, demo scenes
│   └── runtime/                    Locally generated upload/session outputs
├── code/
│   ├── drcarys_v3_server.py        Flask server and safety-routing workflow
│   ├── build_weak_road_edge_dataset.py
│   ├── train_weak_gmu_gat.py       GMU + GAT training implementation
│   ├── fuse_landslide_road_confidence.py
│   └── roadseg/                    Segmentation model utilities
├── results/
│   ├── chimanimani_full_post_event_coverage/
│   ├── chimanimani_study_graph_v2_local_buffer_update/
│   ├── sen12_temporal_chimanimani_finetuned/
│   └── weak_gmu_gat_chimanimani/
└── README.md
```

Only source code, application assets, and generated results are versioned. Raw imagery, training labels, and other datasets are intentionally excluded. To reproduce model training, place the required inputs in a local `dataset/` directory; it is ignored by Git.

## Included outputs

The `results/` directory contains the published project artifacts, including:

- Post-event landslide-probability mosaics for Chimanimani.
- A noded road graph (`road_edges.geojson`, `road_nodes.geojson`, and `road_graph.graphml`).
- Hazard-buffer, exposed-road, and routing-evaluation layers.
- Temporal segmentation checkpoints and evaluation summaries.
- The weak GMU + GAT checkpoint, training history, and validation visualizations.

## Running the dashboard

### Prerequisites

- Python 3.11 or later
- A Python environment with `flask`, `torch`, `numpy`, `geopandas`, `rasterio`, `shapely`, `pyproj`, `xarray`, and their geospatial dependencies

Install the required packages in an environment appropriate for your platform. CUDA is optional; the included server loads the shipped checkpoints on CPU.

### Start

From the repository root:

```powershell
python code/drcarys_v3_server.py
```

Then open `http://127.0.0.1:8766` in a browser.

The dashboard lets you select demonstration scenes or upload a georeferenced probability GeoTIFF (`.tif` / `.tiff`) or supported Sentinel-2 NetCDF scene (`.nc`). Uploaded GeoTIFF values must represent probabilities in the range 0–1 and use the same projected CRS as the study graph.

## Training the weak GMU + GAT model

The project provides a reproducible weak-supervision workflow for research and demonstration:

```powershell
# Build road-edge features and weak exposure labels from local datasets.
python code/build_weak_road_edge_dataset.py

# Train the gated multimodal unit and graph-attention model.
python code/train_weak_gmu_gat.py
```

The generated label is named `weak_exposed`: a road piece is positive when it intersects a buffer around a ground-truth landslide mask. It describes **hazard exposure**, not independently observed road damage.

## Safety and interpretation

This is a research prototype, not an operational road-closure system.

- Predicted landslide footprints and road exposure are model outputs and require expert review.
- A route returned by the application is a network-based safety recommendation under the available raster and road-graph evidence; it is not a guarantee of road accessibility.
- The weakly supervised labels must not be reported as independently verified road-condition labels.
- The workflow is intentionally conservative: potential hazard exposure is treated as a reason to avoid a segment during routing.

## Citation

If you use this repository, please cite it as:

> *Structure-Aware Raster-Vector Integration for Post-Landslide Road-Network Assessment and Safety-Routing* (DR-Carys project).

## License

No license has been specified for this repository. Contact the project maintainers before redistributing or using the code and results outside the intended academic/research context.
