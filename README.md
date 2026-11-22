# Structure-Aware Raster-Vector Routing

End-to-end pipeline that extracts **roads** from three imagery sources
(Sentinel-2 satellite, DJI drone, Aeroscapes aerial) and fuses the deep-learning
raster output with **OSM vector** data — including a **graph-attention fusion**
(GCN + relaxation labeling) for topology-consistent refinement.

---

## 1. What this project does

| Task | Imagery | Method | Key result |
|---|---|---|---|
| Road segmentation | Aeroscapes (drone) | U-Net ResNet34 | test IoU **0.939** |
| Road segmentation | Sentinel-2 (satellite) | multi-spectral U-Net (RGB+NIR+NDVI+NIR-Red) + OSM weak supervision | held-out IoU **0.82** |
| Road segmentation | DJI drone photo | Aeroscapes model | 8.3 % road coverage |
| Raster+vector fusion | Sentinel-2 | alpha-overlay + **graph-attention (GRAFF)** + relaxation labeling | see results |
| Risk-aware routing | OSM + road/landslide confidence | intersection-noded graph + cost-aware A* | `results/risk_aware_routing/` |

## 2. Folder structure

This release includes the source code, application, and generated results only.
Raw/derived datasets are deliberately not tracked; provide your own data under
`dataset/` when reproducing the pipeline.

```
FINAL/
├── README.md
├── code/            all source code
│   ├── roadseg/                 segmentation package (UNet/DLinkNet/DeepLabV3+/SegFormer)
│   ├── sentinel_seg/            multi-spectral Sentinel-2 pipeline
│   ├── Raster/                  satellite download + super-resolution + vector rasterize
│   ├── run.py                   Sentinel-2 end-to-end entrypoint
│   ├── train_aeroscapes_road.py Aeroscapes training
│   ├── roboflow_landslide_inference.py hosted SegFormer landslide inference
│   ├── train_sentinel2_road.py  independently supervised Sentinel-2 road training
│   ├── fuse_landslide_road_confidence.py per-road confidence fusion on MaxScenes
│   ├── evaluate_road_confidence_proxy.py proxy evaluation against MaxScenes event area
│   ├── graph_fusion.py          GRAFF: graph-attention raster-vector fusion
│   ├── road_network_update.py   conservative vector-update candidate generation
│   ├── drone_pipeline.py        DJI drone segmentation + vector fusion
│   ├── leave_roads_out.py       honest non-circular evaluation
│   └── spectral_analysis.py     spectral separability diagnostic
├── dataset/
│   ├── aeroscapes/              drone dataset (3269 images + masks + splits)
│   ├── Drone/                   DJI_0275.JPG + get_location.py
│   ├── Vector/                  OSM roads + buildings (Geofabrik)
│   └── sentinel2_stac_downloads/ Sentinel-2 bands + SR + rasterized vectors
├── results/
│   ├── aerospace_segmentation/  Aeroscapes road model + predictions
│   ├── sentinel_seg_outputs/    Sentinel-2 multispectral model + zoom in/out
│   ├── graph_fusion/            GRAFF graph fusion outputs
│   ├── drone_segmentation/      drone segmentation + comparison + zoom
│   └── raster_vector_fusion/    organized fusion/segmentation/comparison/zoom
└── docs/
    ├── <paper>.pdf              the reference paper (DL + GCN + relaxation labeling)
    └── MANIFEST.txt             complete file listing
```

## 3. Honest results (read this before trusting any number)

* **Aeroscapes roads** — model trained *and tested* on drone imagery: **test IoU 0.94**
  (strong, and a genuine match of training/testing domain).
* **Sentinel-2 roads** — the model was trained on **OSM road labels** and scored
  **IoU 0.90** against those same labels. A **leave-roads-out test** (train on 70 % of
  road segments, test on the hidden 30 %) revealed the truth:
  * recall on **known** roads = **0.84** (memorised)
  * recall on **hidden** roads = **0.02** (approx random)
  => the 0.90 was ~90 % **memorisation**, not detection. Real Sentinel-2 road
  detection needs **many scenes** (multi-tile training), not one.

## 4. How to reproduce

```powershell
# Aeroscapes road model
python train_aeroscapes_road.py

# Sentinel-2 multispectral pipeline (train + infer + fuse + evaluate)
python run.py all

# Zoom-in / zoom-out of the Sentinel-2 result
python -m sentinel_seg.visualize

# Graph-attention raster-vector fusion (GRAFF)
python graph_fusion.py

# Generate GIS change candidates: green=added, black=confirmed, amber=uncertain
# (red removals are opt-in because the Sentinel model needs stronger validation)
python road_network_update.py

# DJI drone segmentation + vector fusion
python drone_pipeline.py

# Run the hosted Roboflow SegFormer landslide model. The key must be supplied
# outside the project; never put it in source code or notebooks.
$env:ROBOFLOW_API_KEY = "<your key>"
python roboflow_landslide_inference.py

# Fuse the hosted landslide probability with road evidence.
python fuse_landslide_road_confidence.py

# Compare the legacy locally trained Bijie model against Roboflow on a scene.
python compare_landslide_models.py

# Train the independent Sentinel-2 road model (RGB tiles + road masks)
python train_sentinel2_road.py --epochs 30

# Produce continuous per-road confidence scores for the Shirur MaxScenes post-event scene
python fuse_landslide_road_confidence.py

# Proxy comparison only: uses the broad event-area KMZ, not a landslide footprint
python evaluate_road_confidence_proxy.py

# Build a routable, intersection-aware graph.  It carries road/landslide
# evidence to each edge and writes road_nodes.geojson, road_edges.geojson,
# and road_graph.graphml under results/risk_aware_routing/.
python code/routing/pipeline.py build

# Risk-aware route. Coordinates must use the projected CRS of road_edges.geojson.
python code/routing/pipeline.py route --start 348000,2012000 --end 351000,2015000

# Optional learned refinement: first create labelled atomic road edges with a
# risk_label column (0=safe, 1=dangerous), then train and route on its output.
python code/routing/pipeline.py train --labelled-edges results/risk_aware_routing/road_edges_labelled.geojson
python code/routing/pipeline.py route --edges results/risk_aware_routing/road_edges_gat.geojson --start 348000,2012000 --end 351000,2015000

# Route output is the safest route plus a risk map. Compare deterministic and
# GAT safest-route behaviour:
python code/routing/pipeline.py evaluate --start 348000,2012000 --end 351000,2015000

# No-data smoke test for graph topology, risk cost, and routing behavior.
python code/routing/smoke_test.py

# Demonstration-only GMU+GAT workflow using a broad event-area proxy. These are
# not manual road-condition labels and must not be used for real accuracy claims.
python code/routing/create_proxy_labels.py
python code/routing/pipeline.py train --labelled-edges results/risk_aware_routing/road_edges_proxy_labelled.geojson

# Honest (non-circular) evaluation
python leave_roads_out.py
```

## 5. Dependencies

Python 3.11; `torch`, `torchvision`, `segmentation-models-pytorch`, `inference-sdk`, `rasterio`,
`geopandas`, `shapely`, `networkx`, `opencv-python`, `numpy`, `matplotlib`,
`scipy`. GPU (CUDA) recommended.

## 6. Key design notes

1. **Multi-spectral input is the fix for Sentinel-2.** Visible RGB alone has a
   Fisher separability of ~0.05 for roads vs background; adding NIR and NDVI/NIR-Red
   raises it to ~0.16 (see `spectral_analysis.py`).
2. **Super-resolution** (10 m -> 5 m) is required: at 10 m roads are sub-pixel
   (99.7 % of road pixels are boundary/mixed pixels).
3. **GRAFF** (`graph_fusion.py`) fuses raster evidence onto the OSM road graph via a
   from-scratch Graph Attention Network + Rosenfeld-Hummel-Zucker relaxation labeling,
   then diffuses the refined confidence back onto the pixel grid.
4. **Road-network update candidates** (`road_network_update.py`) converts raster-only
   road evidence to centreline candidates, compares it against OSM, and exports a
   GeoJSON change layer and a green/red/black/amber review map. By default weak evidence
   is marked **uncertain**, not removed, because the Sentinel model is not sufficiently
   validated for autonomous map edits.
5. **Risk-aware routing** (`routing/`) splits vector roads at real intersections,
   samples raster evidence per directed segment, and applies A* to a transparent
   travel-time-plus-risk cost. `routing/gnn.py` supplies an optional GMU +
   line-graph GAT refinement model, but it is only trainable when time-aligned,
   road-segment safety labels are supplied; it does not make unsupported
   operational closure decisions from the current proxy outputs.
