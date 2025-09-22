## ML_deploy

This repository provides a clean, minimal, and production-ready ML deployment template with:

- Modular `src/` package
- Hydra configs
- PyTorch Lightning training
- ONNX export and FastAPI serving
- Docker + compose
- Tests and CI scaffold


## Quickstart

1. Install dependencies
```
python -m venv .venv && .venv\\Scripts\\activate  # On Windows PowerShell: .venv\\Scripts\\Activate.ps1
pip install -r requirements.txt
```

2. Train
```
python -m ML_deploy.train
```

3. Export to ONNX
```
python -m ML_deploy.onnx_export
```

4. Serve API
```
python -m app.main
```

5. Docker
```
docker build -t mlops-api:latest .
docker run -p 8000:8000 mlops-api:latest
```

## Architecture & Flow
- See the step-by-step Mermaid flow: [complete_flow.md](./complete_flow.md)
- Core steps:
  - Configure with Hydra → Train → Produce `models/best.ckpt` → Export `models/model.onnx` → Serve API → Test → (Optional) Containerize.

## Detailed Implementation

### 1) Configuration (`configs/`)
- Entrypoint defaults in `configs/config.yaml` with groups: `data`, `model`, `training`, `logging`, `export`.
- Override any value via CLI using Hydra: keys are dot-notation paths.

Examples:
```
python -m ML_deploy.train training.max_epochs=3 data.batch_size=16
python -m ML_deploy.train logging.use_wandb=true logging.project=ML_deploy logging.entity=<your-entity>
```

### 2) Training (`src/ML_deploy/train.py`)
- Uses `TextClassificationDataModule` (`data.py`) and `TextClassifier` (`model.py`).
- Produces checkpoints under `configs/training/default.yaml` → `training.checkpoint_dir` (e.g., `models/best.ckpt`).

Run:
```
python -m ML_deploy.train
```

Useful overrides:
```
python -m ML_deploy.train training.max_epochs=5 training.patience=3 training.limit_train_batches=0.2
```

### 3) ONNX Export (`src/ML_deploy/onnx_export.py`)
- Loads the best checkpoint and writes `models/model.onnx`.

Run:
```
python -m ML_deploy.onnx_export export.checkpoint=models/best.ckpt export.onnx_path=models/model.onnx
```

### 4) Serving API (`app/main.py` → `src/ML_deploy/api.py`)
- FastAPI app with endpoints: `/predict`, `/healthz`, `/metrics`.
- On startup, loads `Predictor` (`infer.py`).
- Environment variables:
  - `CHECKPOINT_PATH` (default `./models/best.ckpt`)
  - `TOKENIZER_NAME` (default `google/bert_uncased_L-2_H-128_A-2`)

Run locally:
```
python -m app.main
```

Call predict:
```
curl -X POST "http://localhost:8000/predict" -H "Content-Type: application/json" -d '{"text":"hello world"}'
```

### 5) Testing (`tests/`)
- `test_onnx_parity.py`: compares PyTorch logits vs ONNX outputs.
- `test_smoke.py`: baseline checks.

Run tests:
```
pytest -q
```

### 6) Containerization (optional)
- `Dockerfile` builds a production image. Optionally use `docker-compose.yml` if present.

Build and run:
```
docker build -t mlops-api:latest .
docker run --rm -p 8000:8000 -e CHECKPOINT_PATH=/app/models/best.ckpt mlops-api:latest
```

## Project Structure
```
├─ app/
│  └─ main.py               # FastAPI entrypoint
├─ src/ML_deploy/
│  ├─ __init__.py
│  ├─ data.py               # LightningDataModule
│  ├─ model.py              # LightningModule
│  ├─ train.py              # Hydra-driven trainer
│  ├─ infer.py              # Local inference helper
│  └─ onnx_export.py        # ONNX conversion
├─ configs/
│  ├─ config.yaml           # Hydra defaults
│  ├─ model/default.yaml
│  ├─ data/default.yaml
│  ├─ training/default.yaml
│  ├─ logging/default.yaml
│  └─ export/default.yaml
├─ tests/
│  └─ test_smoke.py
├─ Dockerfile
├─ docker-compose.yml
├─ requirements.txt
└─ extra_content/           # Previous repository files
```

## Configuration
Hydra overrides:
```
python -m ML_deploy.train training.max_epochs=3 data.batch_size=16
```

## API
- POST `/predict` {"text": "example"}
- GET `/healthz`

## Notes
- Replace W&B `entity` in `configs/logging/default.yaml` if you enable W&B logging.
- For production, add tests, linting CI, and secure secrets (OIDC/IAM roles).
