
# drosophila_body_orientation_predictor
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.23738821.svg)](https://doi.org/10.5281/zenodo.23738821)

Repo for [Mangat et al. (2026)](https://www.biorxiv.org/content/10.64898/2026.03.30.715335v1.full). Predicts *Drosophila* body heading angle from flight trajectory data using a neural network trained on data from [van Breugel et al. (2014)](https://www.sciencedirect.com/science/article/pii/S0960982213015820).

## Repository structure

```
drosophila_body_orientation_predictor/
├── experimentaldata/           # Raw data files (download separately from Dryad)
│   ├── 30cms/                  # Training data — Parquet format (van Breugel et al. 2014)
│   ├── 40cms/
│   └── 60cms/
├── figures/                    # Figures written by the analysis notebooks (not the paper figures)
├── generate_figures/           # One notebook per paper figure, plus render_pdfs.py
├── heading_labeling/           # Hand-labelling of corrected headings (marimo tool + queue builder)
├── models/
│   ├── ann_model.keras   # model
│   ├── ann_lowo_heldout_{0.3,0.4,0.6}.keras          # leave-one-wind-speed-out models
│   └── ensemble/ann_seed_{42,7,13,99,123}.keras      # 5-seed ensemble (robustness + confidence signal)
├── notebooks/
│   ├── 01_data_correction_and_processing.ipynb
│   ├── 02_training_and_testing_the_ann.ipynb
│   ├── 03_ann_vs_linear_models.ipynb
│   ├── 04_generate_tables.ipynb
│   └── 05_ann_performance_examples.ipynb
├── publication_figures/        # Paper figures: figure_<n>.svg (layout) and figure_<n>.pdf
├── tables/                     # LaTex tables (.tex)
├── trainingdata/               # Intermediate outputs written by the notebook
│   ├── 01_merged/
│   ├── 02_augmented/
│   ├── 03_corrected/
│   └── 04_trainingdata/
├── utils/
│   ├── utils.py
│   └── fly_plot_lib_plot.py
├── CITATION.cff
├── LICENSE
├── README.md
└── requirements.txt
```


## Installation

### 1. Clone the repo

```bash
git clone https://github.com/vanbreugel-lab/drosophila_body_orientation_predictor.git
cd drosophila_body_orientation_predictor
```

### 2. Download the experimental data

The training data are archived on Dryad in Parquet format ([Mangat & van Breugel 2026](https://doi.org/10.5061/dryad.nzs7h455k); see [Data](#data)). Download and place the files so the directory tree matches:

```
experimentaldata/
├── 30cms/
│   ├── flight_trajectories_3d_HCS_odor_horizon_matched.parquet
│   ├── body_orientations_HCS_odor_horizon_matched.parquet
│   └── body_trajec_matches.parquet
├── 40cms/
│   └── ...
└── 60cms/
    └── ...
```

### 3. Create and activate a virtual environment

Requires **Python 3.12**.

```bash
python3.12 -m venv .venv
source .venv/bin/activate
```

### 4. Install dependencies

```bash
pip install -r requirements.txt
```

### 5. Launch JupyterLab (or Jupyter Notebook)

```bash
jupyter lab
```

## Workflow

Run the notebooks in this order. **`notebooks/01_data_correction_and_processing.ipynb`** turns the raw data into the training-ready dataset (`trainingdata/04_trainingdata/`); **`notebooks/02_training_and_testing_the_ann.ipynb`** then trains and evaluates the model, **`notebooks/03_ann_vs_linear_models.ipynb`** compares it with the baselines, and **`notebooks/04_generate_tables.ipynb`** writes the paper tables.


## Publication figures

The paper's figures are in **`publication_figures/`**: `figure_<n>.svg` is the figure and `figure_<n>.pdf` its rendering. The code that draws them is in **`generate_figures/`**, one Jupyter notebook per figure:

| Figure | Contents | Notebook | Run after |
|---|---|---|---|
| 1 | Heading-correction examples (measured vs. corrected heading) and the hand validation of the correction | `generate_figure_1.ipynb` | notebook 01 (also reads `heading_labeling/labels.csv`) |
| 2 | Diagram of the ANN | `generate_figure_2.ipynb` | notebook 02 |
| 3 | Heading vs. groundspeed, airspeed and thrust direction | `generate_figure_3.ipynb` | notebook 01 |
| 4 | ANN performance: example trajectories, true vs. predicted heading, error analysis | `generate_figure_4.ipynb` | notebook 02 |
| 5 | Headings inferred for the SplitFlowOptoTracking trajectories | `generate_figure_5.ipynb` | notebook 02 and `inferencedata/infer_splitflow_headings.py` (needs the SplitFlowOptoTracking data, see [Data](#data)) |
| 6 | ANN vs. the best linear, circular and Möbius baselines by flight regime | `generate_figure_6.ipynb` | notebook 02 |

`generate_figure_6_course.ipynb` draws the same Figure 6 with the flight regimes defined by course direction instead of heading, as in the tables (`publication_figures/figure_6_course.*`).

Each SVG is a [figurefirst](https://github.com/FlyRanch/figurefirst) layout: running a notebook redraws the panels in its SVG and leaves the rest (e.g. panel labels added in Inkscape) as it is. Run a notebook from the command line with

```bash
jupyter nbconvert --to notebook --execute --inplace --ExecutePreprocessor.timeout=-1 generate_figures/generate_figure_4.ipynb
```

and render the SVGs to PDF with [Inkscape](https://inkscape.org):

```bash
python generate_figures/render_pdfs.py            # all of publication_figures/; --crop to crop to the drawing
```

figurefirst and the package that draws Figure 2's network (ann_graphic_generator) are installed by hand; `requirements.txt` explains how.

The `figures/` directory holds a different set of figures: the ones the analysis notebooks in `notebooks/` save as they run, for checking results. They are not the paper figures.


## Example: predicting heading for a new trajectory

`example_inference/heading_inference.py` is an interactive [marimo](https://marimo.io) notebook that applies the trained models to a trajectory with no heading information. You pick a trajectory and set the wind, compute the ANN inputs, choose a model (or the 5-seed ensemble), and plot the predicted headings as wedges, with SVG export. It needs only the files in this repo, not the Dryad data:

```bash
marimo edit example_inference/heading_inference.py
```

Trajectories are CSV files in `example_inference/trajectories/` with columns `x, y, z, xvel, yvel, zvel, time, frame` (m, m/s, s). An optional `<name>.json` next to a CSV sets the default wind (`wind_speed` in m/s, `wind_direction_deg`: the direction the air blows toward, counter-clockwise from +x). The included examples are laminar-flow (0.4 m/s) trajectories from the SplitFlowOptoTracking data set, extracted by `example_inference/extract_splitflow_trajectory.py`: the one in Figure 5B of that manuscript, and five (`splitflow_laminar_casting_*`) that cast crosswind while progressing upwind after the opto flash.

To run every trajectory through the notebook with its default settings and tile the heading plots into one SVG (one Inkscape layer per trajectory), run:

```bash
python example_inference/make_trajectory_grid.py   # --help for the column count, spacing and labels
```

It writes `example_inference/figures/all_trajectories.svg`.

## Citing this code

The code is archived on Zenodo. Cite the version you used; v1.0.1 is the version for the manuscript.

Mangat, N., May, C., & van Breugel, F. (2026). Predicting Drosophila Body Orientation from a Translational Trajectory Using an Artificial Neural Network (v1.0.1) [Software]. Zenodo. https://doi.org/10.5281/zenodo.23738822

```bibtex
@software{mangat2026code,
  author    = {Mangat, Nehal and May, Christina and van Breugel, Floris},
  title     = {{Predicting \emph{Drosophila} Body Orientation from a Translational Trajectory Using an Artificial Neural Network}},
  year      = {2026},
  version   = {v1.0.1},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.23738822},
  url       = {https://doi.org/10.5281/zenodo.23738822}
}
```

https://doi.org/10.5281/zenodo.23738821 always resolves to the latest version.

## Data

The data are archived on Dryad. Both DOIs are reserved but not yet public, so the links will work once the data sets are published.

- **Training data** (`experimentaldata/`, read by notebook 01): free-flight trajectories and body orientations at 0.3, 0.4 and 0.6 m/s wind (van Breugel et al. 2014).
  Mangat, N., & van Breugel, F. (2026). *Drosophila* trajectory data for training an ANN to predict heading [Dataset]. Dryad. https://doi.org/10.5061/dryad.nzs7h455k
- **SplitFlowOptoTracking data** (read by `inferencedata/infer_splitflow_headings.py` and `example_inference/extract_splitflow_trajectory.py` for Figure 5 and the example trajectories): flies in laminar and variable wind.
  Houle, J., Lopez, A., Kumar, G., Stupski, D., Nair, A., & van Breugel, F. (2026). Data -- Olfactory Search Behavior Across Flow Regimes Supports a Unifying Algorithm [Dataset]. Dryad. https://doi.org/10.5061/dryad.x0k6djhwc

```bibtex
@misc{mangat2026data,
  author       = {Mangat, Nehal and van Breugel, Floris},
  title        = {{\emph{Drosophila} trajectory data for training an {ANN} to predict heading}},
  year         = {2026},
  note         = {Dataset, forthcoming},
  publisher    = {Dryad},
  howpublished = {Dryad},
  doi          = {10.5061/dryad.nzs7h455k},
  url          = {https://doi.org/10.5061/dryad.nzs7h455k}
}

@misc{houle2026data,
  author       = {Houle, Jaleesa and Lopez, Austin and Kumar, Gaurav and Stupski, David and Nair, Aditya and van Breugel, Floris},
  title        = {{Data -- Olfactory Search Behavior Across Flow Regimes Supports a Unifying Algorithm}},
  year         = {2026},
  note         = {Dataset, forthcoming},
  publisher    = {Dryad},
  howpublished = {Dryad},
  doi          = {10.5061/dryad.x0k6djhwc},
  url          = {https://doi.org/10.5061/dryad.x0k6djhwc}
}
```

If you have any questions regarding the data pipeline, please shoot me an email at nehalsinghmangat.software@gmail.com. If you encounter any issues with running the pipeline or reproducing results, please submit a Github issue.   
