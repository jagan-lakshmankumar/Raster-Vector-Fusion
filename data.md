# Data

## Study Area

**Chimanimani, Zimbabwe** was selected as the study area because it was strongly affected by landslides triggered by Cyclones.

## Raster Dataset

The landslide segmentation model uses the **Sen12Landslides** dataset.

- Dataset: Sen12Landslides
- Source: Sentinel-2 temporal satellite imagery
- Spatial resolution: 10 m
- Input bands: B02, B03, B04, and B08
- Temporal input: pre-event image, post-event image, and post-minus-pre band differences
- Model input: 12 channels
- Ground truth: annotated post-event landslide masks
- Study-region split: Chimanimani patches are used for the held-out evaluation

Dataset source:  [Sen12Landslides](https://huggingface.co/datasets/paulhoehn/Sen12Landslides)

## Road Network Data

Road-network vector data is obtained from **OpenStreetMap (OSM)** for the Chimanimani study area.

Road geometries are:

- Noded at road intersections  
- Split into approximately 50 m physical road pieces  
- Represented using road class, nominal speed, direction, length, and graph connectivity  

The OSM road data is used to construct the routing graph and provide vector features for GMU-GAT raster-vector fusion.

Data source:  [OpenStreetMap](https://www.openstreetmap.org/)
