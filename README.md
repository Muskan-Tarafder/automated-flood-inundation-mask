# SAR Flood Detection — Proof of Concept

One-line summary
- End-to-end PoC pipeline that produces a binary flood/inundation mask from a single SAR (or multispectral) scene. Intended as a reproducible starting point: ingest → preprocess → deterministic mask (index + morphology) → small segmentation baseline (U‑Net in the notebook) → evaluation.

Problem (what this actually solves)
- Given a large georeferenced scene, produce a spatially-aligned binary mask highlighting potential inundated/flooded areas so that analysts or downstream systems can review, correct, and use results for alerting or training a supervised model. The hardest constraints solved here are reproducibility, explainability, and a simple path from deterministic methods to a trained segmentation model.

Quick status
- Primary artifact: `POC_Model.ipynb` — a Colab/Jupyter notebook implementing:
  - synthetic SAR data generation (for offline testing),
  - Lee speckle filter,
  - tile extraction,
  - a U-Net model definition and Dice loss,
  - simple IoU/Dice evaluation and visualization.
- This repo is a PoC — not production-ready. Use it to validate methods and to generate labelled examples for model training.

Design / Approach
- Deterministic baseline (recommended first step)
  - Compute an index (e.g., NDWI where applicable) or simple threshold on SAR polarization values.
  - Morphological cleanup (closing, remove small objects) to create a reviewable mask.
  - Pros: deterministic, explainable, usable immediately to create labels.
- Preprocessing
  - Lee speckle filter applied to SAR bands to reduce high-frequency speckle while preserving edges.
  - Tile extraction with overlap to feed a segmentation model and avoid memory issues with large rasters.
- Segmentation baseline
  - U-Net defined in the notebook (two-channel input supporting VV & VH polarizations).
  - DiceLoss + IoU metrics for training/validation.
- Evaluation & human-in-the-loop
  - Compute IoU/Dice; save per-scene metrics.
  - Use manual review to create corrected masks and bootstrap supervised training.

Configuration (values used in the notebook)
- Tile size: 256 × 256
- Tile stride: 128
- Example scene size (synthetic): 3000 × 3000
- Training: epochs = 20, batch_size = 16, lr = 1e-3
Adjust these for your environment and GPU memory.

How to run (shortest reproducible path)
1. Clone the repo:
   git clone https://github.com/Muskan-Tarafder/automated-flood-inundation-mask.git
   cd automated-flood-inundation-mask

2. Create environment and install dependencies:
   python -m venv .venv
   source .venv/bin/activate
   pip install --upgrade pip
   pip install rasterio numpy scipy matplotlib tqdm torch scikit-learn flask pyngrok nest-asyncio werkzeug scikit-image

   (If you only want to run deterministic parts without training, `torch` is optional.)

3. Open the notebook:
   - Locally: jupyter notebook POC_Model.ipynb
   - Or upload to Colab (the notebook contains a cell that installs dependencies when executed in Colab).

4. Minimal smoke test using synthetic data (in the notebook)
   - Run the synthetic data cells, run the filtering, tile extraction and visualize outputs.
   - Confirm you see VV/VH and a produced flood ground-truth visualization.

File overview
- POC_Model.ipynb — the complete notebook with:
  - dependency install and imports
  - configuration block
  - synthetic SAR data generator
  - Lee speckle filter implementation
  - tile extraction function
  - U-Net architecture definition
  - DiceLoss and IoU calculation
  - FloodDataset wrapper
  - visualization and example runs
- (No CLI or python modules included in repo — recommended next step: extract stable functions to a small module like `poc_inference.py` and add unit tests.)

Reproducibility & verification
- Unit tests to add (recommended):
  - test_compute_ndwi / test_lee_filter: deterministic inputs → expected outputs.
  - test_extract_tiles: small synthetic image → expected tile count and shapes.
- Integration smoke test:
  - Run the notebook end-to-end on a sample scene. Confirm `output_mask.tif` is generated and has the same transform/CRS as the input.
- Metrics:
  - If ground truth is available, compute IoU, Dice, precision, recall, and save to results/metrics.json.
  - Suggested acceptance threshold for this PoC: IoU ≥ 0.45 (adjust by use case).

Limitations & risks
- Band-order and data type variability: different sensors and providers use different band orders and scales — explicit band-index configuration is required.
- Threshold brittleness: a single global threshold is not robust across sensors/seasons; consider adaptive thresholding (Otsu) or supervised learning.
- Clouds, shadows, and vegetation change can cause false positives; use QA/cloud masks or a supervised model trained with such samples.
- Large GeoTIFFs can exhaust memory if processed naïvely; tile and stream reads are required.
- Ground truth misalignment/CRS mismatch can produce misleading metrics — always reproject to a common CRS and validate alignment before evaluation.

Next steps (recommended short roadmap)
1. Extract stable functions from the notebook into a small module (`poc_inference.py`) and add a CLI.
2. Add a `requirements.txt`, small `Makefile` targets (`make setup`, `make test`, `make run`).
3. Add unit tests under `tests/` and a CI job that runs the smoke tests on CPU (fast).
4. Collect a small set of real scenes + human-reviewed masks (10–50 scenes) to bootstrap training.
5. Train a small U‑Net on tiles (with augmentation), validate on held-out scenes, and compare deterministic baseline vs model.
6. Add inference code that mosaics tile predictions back to full-scene georeferenced masks and preserves CRS/transform.

How to contribute
- Create a branch, add modules / tests, and open a PR with:
  - code refactor (functions extracted from the notebook)
  - tests that cover preprocessing and tile extraction
  - short README update if you add new usage patterns

Contact / author
- Repo owner: Muskan-Tarafder
- For integration or productionization questions (e.g., running at scale, monitoring, model registry), tag the team and include a small sample scene and expected outputs.

License
- Add an appropriate license for your project. This PoC does not include a license by default.

Acknowledgements / references
- Lee speckle filter — standard SAR preprocessing
- U-Net — segmentation baseline
