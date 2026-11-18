# PathSnap : Distributed Map-Matching & Route Reconstruction

**ML Hackathon 2026**

PathSnap reconstructs vehicle routes from noisy GPS observations by matching each observation to candidate road-network arcs, scoring the candidates with machine-learning models, and decoding a topologically coherent sequence with an HMM and the Viterbi algorithm.

The project includes data exploration and preprocessing, spatial candidate generation, feature engineering, five-model experimentation, Optuna hyperparameter optimisation, HMM route reconstruction, Folium visualisation, and a two-laptop FastAPI evaluator demo.

---

## Contents

- [System overview](#system-overview)
- [Repository layout](#repository-layout)
- [Team and division of work](#team-and-division-of-work)
- [Dataset setup](#dataset-setup)
- [Installation](#installation)
- [Run the pipeline](#run-the-pipeline)
- [Two-laptop evaluator demo](#two-laptop-evaluator-demo)
- [API reference](#api-reference)
- [Experiment outputs](#experiment-outputs)
- [Results at a glance](#results-at-a-glance)
- [Reproducibility and known limitations](#reproducibility-and-known-limitations)
- [AI-use disclosure](#ai-use-disclosure)

## System overview

```text
GPS track + directed road network
              |
              v
     Coordinate projection
              |
              v
   Spatial candidate generation
      (KD-tree indexing)
              |
              v
      12-feature extraction
              |
              v
    ML candidate probabilities
              |
              v
       HMM + Viterbi decoding
              |
              v
    Reconstructed road-arc sequence
              |
              v
     Metrics + Folium map output
```

**Two-laptop deployment**

- **Laptop 1 — backend/server:** loads the trained model, reads the selected trip, generates candidates and features, performs ML inference and HMM decoding, and calculates evaluation metrics.
- **Laptop 2 — evaluator/client:** opens the browser dashboard and sends the selected trip ID to Laptop 1 over the same local network.

The API does not send reference-route data to the browser client. Note, however, that the current public repository contains the evaluator trip `.route` files. They are therefore publicly downloadable from GitHub; the server-side boundary applies to the live API response, not to repository access.

## Repository layout

The following is the **target layout after applying the cleanup script** included with this kit. The reusable Python package remains in `src/`; runnable entry points are grouped by task; experiment outputs remain under `artifacts/`.

```text
PathSnap/
├── src/                                  # Reusable project code
│   ├── dataset.py                        # Trip loading, projection, route alignment,
│   │                                     # spatial indexing and candidate generation
│   ├── features.py                       # 12 features per GPS-observation/candidate pair
│   ├── models.py                         # Five model definitions and registry
│   ├── hmm_decoder.py                    # HMM emissions, transitions and Viterbi decoder
│   ├── metrics.py                        # Observation-level and route-level metrics
│   ├── visualize.py                      # Folium map-generation functions
│   └── server.py                         # FastAPI backend
│
├── scripts/
│   ├── eda/run_eda.py
│   ├── training/train_and_evaluate_models.py
│   ├── optimization/run_bayesian_optimization_pipeline.py
│   ├── evaluation/run_hmm_evaluation.py
│   ├── evaluation/run_evaluator_test_demo.py
│   ├── visualization/generate_folium_maps.py
│   ├── visualization/generate_all_evaluator_maps.py
│   ├── deployment/run_ui.py
│   └── reporting/generate_report.py
│
├── templates/
│   └── index.html                        # Browser evaluator dashboard
│
├── evaluator_test/
│   └── evaluator_test/<trip_id>/         # Ten evaluator demo trips
│       ├── <trip_id>.track
│       ├── <trip_id>.nodes
│       ├── <trip_id>.arcs
│       └── <trip_id>.route
│
├── artifacts/
│   ├── models/                           # Saved model artifacts (.joblib)
│   ├── results/                          # CSV/JSON experiment summaries
│   ├── plots/
│   │   ├── eda/
│   │   ├── models/
│   │   ├── bayesian_opt/
│   │   ├── hmm/
│   │   └── maps/
│   └── maps/                             # Generated interactive HTML maps (local output)
│
├── docs/
│   └── PathSnap_Workflow_Report.md       # Architecture, deployment and viva notes
│
├── requirements.txt
├── .gitignore
└── README.md
```

**Why the `scripts/` folder?** These files are executable pipeline entry points, not reusable library modules. The code that implements the pipeline remains in `src/`. The cleanup script inserts a small repository-root bootstrap into each moved script so imports such as `from src.dataset import ...` continue to work and relative paths resolve from the repository root.

## Team and Division of Work

All members collaborated on integration, debugging, validation, documentation and the final demonstration.

| Team member | Primary ownership | Main responsibilities |
|---|---|---|
| **Harshith R** | System Integration | Feature engineering, model training and comparison, Bayesian hyperparameter optimisation, HMM/Viterbi route reconstruction, FastAPI backend and deployment. |
| **Harshitha B.T.** | Core Pipeline & ML Development | Raw data preprocessing, GPS coordinate projection, ground-truth route alignment, road-network processing, spatial indexing, candidate generation and candidate-recall analysis. |
| **Janavi Agarwal** | Frontend Development and Visualisation | Evaluator dashboard UI, frontend implementation, interface design, API integration on the client side and presentation of route-reconstruction results. |
| **Hemanth** | Supporting Development and Testing | Assisting with testing, validation, system integration and other supporting project activities. |

**Shared responsibilities:** dataset exploration, debugging, experiment verification, report preparation, presentation and integration testing were collaborative activities.

## Dataset setup

### Trip file formats

| File | Meaning |
|---|---|
| `.track` | GPS observations containing longitude, latitude and timestamp. |
| `.nodes` | Road-network node coordinates. Node IDs are based on row order in the file. |
| `.arcs` | Directed road segments represented by start-node and end-node indices. Arc IDs are based on row order. |
| `.route` | Reference sequence of road-arc IDs, where available. |

### Training and student-test data (local only)

The training and student-test trip folders are **not committed** to this repository. The current scripts expect the extracted dataset in this local folder layout:

```text
Student folder_extracted/
└── Student folder/
    ├── train/
    │   └── <trip_id>/
    └── student_test/
        └── <trip_id>/
```

Place the extracted training/student-test dataset at that location before running EDA, model training, Bayesian optimisation, HMM student-test evaluation or the selected student-test map generator. The folder is ignored by Git. Do not commit the raw archive or extracted training data unless the dataset licence and event rules explicitly permit it.

### Evaluator-test data

The ten evaluator demo trips are currently committed under `evaluator_test/evaluator_test/`. They are used by the API, evaluator demo script and evaluator-map generation script. The `.route` files are present in the public repository, so they should not be described as confidential or inaccessible to repository visitors.

## Installation

### 1. Create and activate a virtual environment

**Windows PowerShell**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

**macOS / Linux**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 2. Install dependencies

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Run all commands below from the **repository root**.

## Run the pipeline

### 1. Exploratory data analysis

Requires the local training and student-test data described above.

```bash
python scripts/eda/run_eda.py
```

Writes dataset summaries to `artifacts/results/` and EDA plots to `artifacts/plots/eda/`.

### 2. Train and compare the five models

```bash
python scripts/training/train_and_evaluate_models.py
```

The current training script chooses 25 representative training trips using `train_dirs[::3][:25]` and evaluates on all ten discovered student-test trips. It uses 3-fold trip-grouped `GroupKFold` cross-validation. The script saves model files, metrics and comparison plots under `artifacts/`.

### 3. Bayesian hyperparameter optimisation

```bash
python scripts/optimization/run_bayesian_optimization_pipeline.py
```

Runs the Optuna TPE search for HistGradientBoosting and writes trial history, tuned model, tuned-versus-untuned comparison and plots to `artifacts/`.

### 4. HMM/Viterbi evaluation

```bash
python scripts/evaluation/run_hmm_evaluation.py
```

Compares classifier-only predictions against HMM/Viterbi-decoded predictions on the student-test partition.

### 5. Generate Folium maps

For the selected student-test trips:

```bash
python scripts/visualization/generate_folium_maps.py
```

For all ten evaluator-test trips:

```bash
python scripts/visualization/generate_all_evaluator_maps.py
```

Interactive HTML maps are written to `artifacts/maps/`. Static map snapshots are saved under `artifacts/plots/maps/`.

### 6. Run evaluator-test inference without the live API

```bash
python scripts/evaluation/run_evaluator_test_demo.py
```

Writes evaluator-test metrics to `artifacts/results/evaluator_test_results.csv` and `.json`.

### 7. Generate the technical report (optional)

```bash
python scripts/reporting/generate_report.py
```

Writes the PDF to `reports/`. The generated PDF is ignored by Git and should be submitted separately if required by the event.

## Two-laptop evaluator demo

Both laptops must be connected to the same Wi-Fi network or hotspot.

### Laptop 1 — FastAPI server

From the repository root:

```bash
uvicorn src.server:app --host 0.0.0.0 --port 8000
```

Find Laptop 1's local IPv4 address:

```powershell
ipconfig
```

Check the API on Laptop 1 at:

```text
http://localhost:8000/health
```

If Laptop 2 cannot connect, check that both laptops are on the same subnet, that client isolation is disabled on the hotspot/router, and that Windows Firewall allows inbound TCP traffic on port 8000.

### Laptop 2 — evaluator dashboard

Open this URL in a browser, replacing the placeholder with Laptop 1's actual IPv4 address:

```text
http://<LAPTOP_1_IP>:8000/ui
```

Alternatively, run the standalone dashboard server on Laptop 2:

```bash
python scripts/deployment/run_ui.py
```

Open the local URL printed by the script. The standalone UI server uses port **8501**; enter Laptop 1's API base URL in the dashboard.

## API reference

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/` | Service information and available endpoints. |
| GET | `/health` | Server and loaded-model health. |
| GET | `/trips` | Lists available evaluator-test trip IDs. |
| GET / POST | `/predict/{trip_id}` | Runs candidate generation, feature extraction, classifier inference, HMM decoding and server-side metric calculation. |
| GET | `/map/{trip_id}` | Returns an interactive Folium map, generating it on demand when required. |
| GET | `/ui` | Serves the evaluator dashboard. |

Interactive API documentation is available at `http://<LAPTOP_1_IP>:8000/docs` while the server is running.

## Experiment outputs

| Output | Location |
|---|---|
| Saved model artifacts | `artifacts/models/` |
| Dataset inventory and EDA summary | `artifacts/results/dataset_inventory.csv`, `eda_summary.csv` |
| Five-model comparison | `artifacts/results/model_comparison.csv` |
| Bayesian optimisation trials | `artifacts/results/bayesian_trials.csv` |
| Tuned vs. untuned comparison | `artifacts/results/tuned_vs_untuned_comparison.csv` |
| Student-test classifier vs. HMM comparison | `artifacts/results/hmm_vs_classifier_comparison.csv`, `hmm_summary_metrics.csv` |
| Evaluator-test demo results | `artifacts/results/evaluator_test_results.csv` and `.json` |
| EDA/model/Optuna/HMM/map plots | `artifacts/plots/` |
| Interactive Folium HTML maps | `artifacts/maps/` (generated locally; ignored by Git) |

## Results at a glance

The following figures are read from the committed result CSVs. Keep student-test and evaluator-test results separate when discussing them.

**Five-model baseline comparison on the student-test partition**

| Model | Grouped CV Macro F1 | Student-test Top-1 | Student-test Macro F1 | Log Loss |
|---|---:|---:|---:|---:|
| Logistic Regression | 0.8700 | 81.04% | 0.8745 | 0.1161 |
| K-Nearest Neighbours | 0.8664 | 79.51% | 0.8761 | 0.3166 |
| Decision Tree | 0.8706 | 80.16% | 0.8790 | 0.2255 |
| Random Forest | 0.8837 | 81.90% | 0.8883 | 0.0920 |
| HistGradientBoosting | **0.8849** | **82.03%** | 0.8897 | 0.0924 |

**Bayesian tuning of HistGradientBoosting**

| Metric | Untuned | Tuned |
|---|---:|---:|
| Grouped CV Macro F1 | 0.8849 | **0.8857** |
| Student-test Top-1 Accuracy | **82.03%** | 81.94% |
| Student-test Macro F1 | 0.8897 | **0.8900** |
| Log Loss | 0.0924 | **0.0919** |

**Student-test HMM comparison**

| Metric | Classifier only | Classifier + HMM |
|---|---:|---:|
| Pointwise Top-1 Accuracy | **81.94%** | 81.40% |
| Route Continuity | 89.40% | **99.72%** |

The HMM substantially improves route continuity in the student-test results, while pointwise accuracy decreases slightly. This is a continuity-versus-pointwise-accuracy trade-off, not an improvement in every metric.

For the ten-trip evaluator-test demo, refer to `artifacts/results/evaluator_test_results.csv`; do not substitute those numbers for the student-test results above.

## Reproducibility and known limitations

- The EDA inventory covers 80 training trips, but the current model-training and Bayesian-optimisation scripts select 25 representative trips for their training runs. State this distinction clearly in reports and presentations.
- Cross-validation is implemented with `GroupKFold` grouped by trip; it is not a stratified GroupKFold implementation.
- The API loads `artifacts/models/gradient_boosting_tuned.joblib` when available and falls back to `gradient_boosting.joblib`.
- The current GitHub tree contains Logistic Regression, Decision Tree, baseline HistGradientBoosting and tuned HistGradientBoosting model files. KNN and Random Forest model files are excluded by the current `.gitignore`; the training script can recreate them. Confirm whether the final submission requires those model binaries to be committed, or whether the size constraint justifies leaving them out.
- The interactive Folium HTML files are generated outputs and are not committed because `artifacts/maps/` is ignored. The map-generation scripts are committed; generate the maps on the demo machine before evaluation.
- All runner scripts should be launched from the repository root. The cleanup script preserves this behaviour after moving them into `scripts/`.
- Verify that saved models, result CSVs, maps and the final PDF come from the same final experiment run before submission.

## AI-use disclosure

AI assistance was used for selected programming boilerplate and documentation support. Dataset processing, experiment execution, model evaluation, interpretation and final technical claims was verified by the team against the code and respective generated outputs. 

---

**Repository:** https://github.com/Harshith493/PathSnap

---
