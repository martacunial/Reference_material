# MLflow Reference Guide

Personal backbone for MLflow experiment tracking in ML/anomaly-detection projects.

**What MLflow does:** it records every training run (hyperparameters, metrics, model weights, arbitrary files) in a structured store so you can compare runs, reproduce results, and deploy models — all from a local folder or a remote server.

**Core concepts:**
- **Experiment** — a named group of runs (one per project / model type).
- **Run** — a single execution: has params, metrics, artifacts, tags, and a unique `run_id`.
- **Artifact** — any file attached to a run (model weights, plots, JSON profiles, joblib objects).
- **Model Registry** — a versioned catalogue of registered models with lifecycle stages (Staging → Production → Archived).
- **Tracking URI** — where MLflow stores data. Defaults to `./mlruns/` (local). Can be a remote server.

---

## Install & Setup

```bash
pip install mlflow
# or with uv:
uv add mlflow
```

```python
import mlflow
import mlflow.pytorch
from mlflow.tracking import MlflowClient
```

---

## Tracking URI

By default MLflow writes to `./mlruns/` (local filesystem). Always set it **before** calling `set_experiment()` or `start_run()`, otherwise MLflow silently creates a new `mlruns/` directory in the current working directory — which is the most common source of "where did my run go?" confusion.

```python
# Read current URI
uri = mlflow.get_tracking_uri()

# Set custom path
mlflow.set_tracking_uri("file:///absolute/path/to/mlruns")
```

---

## Experiment

Experiments group related runs. Use one experiment per model architecture or task. If an experiment was previously deleted it still exists in the store with `lifecycle_stage='deleted'` — you must restore it before adding new runs; otherwise MLflow creates a duplicate with the same name.

```python
# Create or get experiment
experiment = mlflow.get_experiment_by_name("my_experiment")
if experiment is None:
    experiment_id = mlflow.create_experiment("my_experiment")
elif experiment.lifecycle_stage == "deleted":
    client = MlflowClient()
    client.restore_experiment(experiment.experiment_id)

mlflow.set_experiment("my_experiment")
```

---

## Run Lifecycle

A run tracks everything that happens during one training or evaluation session. Using the `with` context manager is preferred because it automatically calls `mlflow.end_run()` even if an exception is raised — avoiding orphaned runs that show as "RUNNING" forever in the UI.

```python
# Start run
with mlflow.start_run(run_name="run_20260505", tags={"model": "cnn_ae"}):
    mlflow.log_params({"lr": 0.001, "epochs": 50})
    mlflow.log_metric("val_loss", 0.023, step=10)
    mlflow.log_artifact("path/to/file.json", artifact_path="profiles")
    mlflow.pytorch.log_model(model, "model")

# Or manual open/close (useful in class-based trainers)
run = mlflow.start_run(run_name="my_run")
run_id = run.info.run_id
# ... training loop ...
mlflow.end_run()
```

---

## Log params, metrics, artifacts

**Params** are logged once per run and describe the configuration (hyperparameters, data preprocessing choices). **Metrics** can be logged multiple times with a `step` (e.g. epoch number) and are plotted as curves in the UI. **Artifacts** are files — save anything you want to keep alongside the run.

```python
# Params: string-valued, appear as columns in the UI
mlflow.log_params({"window_size": 120, "scaler": "minmax"})

# Metrics: numeric, tracked over steps/epochs
mlflow.log_metric("train_loss", 0.05, step=1)
mlflow.log_metric("val_loss", 0.07, step=1)

# Single file artifact
mlflow.log_artifact("outputs/profile.json", artifact_path="profiles")

# Via MlflowClient (outside active run)
client = MlflowClient()
client.log_artifact(run_id, "outputs/profile.json", "profiles")
```

---

## Log PyTorch model

The `signature` records the expected input/output schema (shapes and dtypes). The `input_example` stores a sample input that MLflow shows in the UI and uses when serving the model via REST. Both are optional but highly recommended — they make the artifact self-documenting.

```python
import mlflow.pytorch
from mlflow.models.signature import infer_signature

# Inside a run
input_example = train_sequences[:1].numpy()
signature = infer_signature(input_example, model(train_sequences[:1]).detach().numpy())
mlflow.pytorch.log_model(model, "model", signature=signature, input_example=input_example)
```

---

## Log custom object (e.g. embedder/scaler) with joblib

`mlflow.log_artifact()` requires a file on disk — it cannot log Python objects directly. The pattern below creates a temporary directory (automatically cleaned up), serialises the object with joblib, and uploads the file. This is the standard way to store preprocessing objects (scalers, embedders, label encoders) alongside the model so predictions can be reproduced exactly.

```python
import joblib, tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    path = Path(tmp) / "embedder.joblib"
    joblib.dump(embedder, path)
    mlflow.log_artifact(str(path), artifact_path="embedder")
```

---

## Search and load latest run

Always load the latest run programmatically rather than hardcoding a `run_id` — the ID changes every time you retrain. `order_by=["attribute.start_time DESC"]` ensures you get the most recent run first. Check `runs[0].info.status == "FINISHED"` if you want to skip crashed runs.

```python
client = MlflowClient()
experiment = client.get_experiment_by_name("anomaly_detection_ConvAE_1d")

runs = client.search_runs(
    experiment_ids=[experiment.experiment_id],
    order_by=["attribute.start_time DESC"],
    max_results=1,
)
run_id = runs[0].info.run_id
run_params = runs[0].data.params  # dict of logged params
```

---

## Load artifacts from a run

`download_artifacts` copies the file from the artifact store to a local temp directory and returns the local path. The `runs:/` URI scheme is the most portable: it works with local `mlruns/`, remote servers, and S3-backed stores without code changes.

```python
# Load PyTorch model
model = mlflow.pytorch.load_model(f"runs:/{run_id}/model")

# Download any artifact to local disk
from mlflow.artifacts import download_artifacts
local_path = download_artifacts(f"runs:/{run_id}/embedder/embedder.joblib")
embedder = joblib.load(local_path)

# Or via client
local_path = client.download_artifacts(run_id, "profiles/model_train_val_error_profile.json")
```

---

## List artifacts

```python
artifacts = client.list_artifacts(run_id, "profiles")
json_files = [a.path for a in artifacts if a.path.endswith(".json")]
```

---

## Start MLflow UI

```bash
# Auto-detect tracking URI
uv run mlflow ui --backend-store-uri 'file:///path/to/mlruns' --host 127.0.0.1 --port 5000

# Windows: add --workers 1 for stability
uv run mlflow ui --backend-store-uri 'file:///path/to/mlruns' --host 127.0.0.1 --port 5000 --workers 1
```

UI URL: `http://127.0.0.1:5000`

---

## Check if UI is reachable (Python)

```python
from urllib.request import urlopen

try:
    with urlopen("http://127.0.0.1:5000", timeout=1):
        ui_reachable = True
except Exception:
    ui_reachable = False
```

---

## Auto-start UI in background (subprocess)

```python
import subprocess, sys

cmd = [
    sys.executable, "-m", "mlflow", "ui",
    "--backend-store-uri", mlflow.get_tracking_uri(),
    "--host", "127.0.0.1",
    "--port", "5000",
]
subprocess.Popen(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
```

---

## Deep link patterns

```
Experiment: http://127.0.0.1:5000/#/experiments/{experiment_id}
Run:        http://127.0.0.1:5000/#/experiments/{experiment_id}/runs/{run_id}
```

---

## Key notes

- `log_params` values are always cast to string → retrieve with `runs[0].data.params["key"]`.
- `log_artifact` needs an **existing** local file path; use `tempfile.TemporaryDirectory()` for in-memory objects.
- Deleted experiments must be restored before use: `client.restore_experiment(experiment_id)`.
- On Windows force `--workers 1` to avoid Gunicorn multiprocess issues.

---

## Tags

Tags are free-form string key-value pairs. Unlike params they can be set **outside** an active run (via the client) and can be updated after the run finishes. Useful for marking runs as reviewed, flagging production candidates, or storing metadata that isn't a hyperparameter (e.g. dataset name, git commit hash).

```python
# Set tags on the active run
mlflow.set_tag("model_type", "cnn_ae")
mlflow.set_tags({"status": "production", "dataset": "DPC5CC01"})

# Set tag via client (any run)
client.set_tag(run_id, "reviewed", "true")

# Read tags from a run
tags = runs[0].data.tags   # dict
```

---

## Log Matplotlib figures

`mlflow.log_figure()` (Option 1) is cleaner — it serialises the figure in-memory without touching the filesystem. Option 2 (save → log) gives more control over DPI and format. Always call `plt.close(fig)` after logging to free memory, especially inside training loops.

```python
import matplotlib.pyplot as plt
import mlflow

fig, ax = plt.subplots()
ax.plot(losses)
ax.set_title("Training Loss")

# Option 1: log directly as figure (saved as PNG)
mlflow.log_figure(fig, "plots/train_loss.png")
plt.close(fig)

# Option 2: save locally first, then log
fig.savefig("/tmp/loss.png", dpi=150)
mlflow.log_artifact("/tmp/loss.png", artifact_path="plots")
```

---

## Nested runs (e.g. hyperparameter sweeps)

Nested runs appear as collapsible children under the parent in the UI, making it easy to compare all trials of a sweep in one view. The parent run typically logs the best metric across children; children log individual trial results. The `nested=True` flag is required or MLflow raises an error when a run is already active.

```python
with mlflow.start_run(run_name="sweep_parent") as parent_run:
    for lr in [1e-3, 1e-4]:
        with mlflow.start_run(run_name=f"lr={lr}", nested=True):
            mlflow.log_param("lr", lr)
            mlflow.log_metric("val_loss", train_model(lr))
```

---

## Model Registry

The Model Registry adds a versioning layer on top of run artifacts. Each time you call `register_model` for the same name, MLflow increments the version number automatically. Lifecycle stages (Staging → Production → Archived) are just labels — the actual promotion logic is up to you. Loading via `models:/Name/Production` always fetches the latest model in that stage, so your inference code never needs to change when you promote a new version.

```python
# Register a logged model
model_uri = f"runs:/{run_id}/model"
mv = mlflow.register_model(model_uri, "AnomalyDetectorCNN")
print(mv.version)  # e.g. "3"

# Transition to production
client.transition_model_version_stage(
    name="AnomalyDetectorCNN",
    version=mv.version,
    stage="Production",   # "Staging" | "Production" | "Archived"
    archive_existing_versions=True,
)

# Load latest production model
model = mlflow.pytorch.load_model("models:/AnomalyDetectorCNN/Production")

# List all versions
for v in client.search_model_versions("name='AnomalyDetectorCNN'"):
    print(v.version, v.current_stage, v.run_id)
```

---

## Search & compare runs programmatically

The `filter_string` syntax is a subset of SQL `WHERE` clauses. Supported prefixes: `metrics.`, `params.`, `tags.`, `attributes.` (run_id, status, start_time). Building a DataFrame lets you sort and slice runs however you like — useful for automated reporting or picking the best model without opening the UI.

```python
# Filter by metric threshold and tag
runs = client.search_runs(
    experiment_ids=[experiment.experiment_id],
    filter_string="metrics.val_loss < 0.05 AND tags.status = 'production'",
    order_by=["metrics.val_loss ASC"],
    max_results=10,
)

# Build a comparison DataFrame
import pandas as pd
df = pd.DataFrame([
    {"run_id": r.info.run_id, **r.data.params, **r.data.metrics}
    for r in runs
])
print(df.sort_values("val_loss"))
```

---

## Delete / cleanup runs

`client.delete_run()` is a **soft delete** — the run is hidden from the UI but still on disk. To actually free disk space, run `mlflow gc` from the command line. The `mlruns/` directory grows fast with large model artifacts; periodic GC is important for long-running projects.

```python
# Soft-delete a run (moves to 'deleted' lifecycle)
client.delete_run(run_id)

# Hard-delete all soft-deleted runs older than N days (CLI)
# mlflow gc --backend-store-uri file:///path/to/mlruns

# Delete a registered model version
client.delete_model_version(name="AnomalyDetectorCNN", version="1")

# Delete the whole registered model
client.delete_registered_model(name="AnomalyDetectorCNN")
```
