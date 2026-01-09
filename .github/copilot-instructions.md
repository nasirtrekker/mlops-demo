# Copilot instructions for mlops-demo

This file gives concise, repo-specific guidance so an AI coding agent can be productive quickly.

1. Big picture
- Components: data ingestion (`src/data/*`), preprocessing (`src/data/preprocess.py`), model training/eval (`src/models/train.py`, `src/models/evaluate.py`), orchestration (`src/pipelines/pipeline.py` / Prefect), serving (`src/api/main.py` / FastAPI), monitoring (Prometheus + Grafana configured in `configs/`), and experiment tracking (`mlflow`, configured in `configs/mlflow_config.yaml`).
- Data flow: `data/raw` → preprocessing → `data/processed` (train/val/test CSVs) → `src/models/train.py` → saved models in `models/` and registered to MLflow (registered model name pattern: `news_classifier_{type}`).

2. Service boundaries & integration points
- API: `src/api/main.py` is the FastAPI app; endpoints: `/info`, `/predict`, `/metrics`. Auth uses `X-API-Key` header (see `Settings` in that file).
- Model registry: MLflow tracked in docker-compose `mlflow` service; code often loads `models:/<name>/latest` or falls back to local `models/news_classifier_*.joblib`.
- Orchestration: `src/pipelines/pipeline.py` is the Prefect flow; tasks call `download_bbc_dataset()`, `preprocess()`, `train_model()`, and `evaluate()`.
- Entrypoints: container entrypoint (`Dockerfile` writes `/app/entrypoint.sh`) optionally runs the pipeline when `RUN_PIPELINE=true` and then executes the app command (`uvicorn src.api.main:app ...`).

3. Developer workflows & commands (exact)
- Quick local dev (recommended):
  - `cp .env.example .env` then edit `.env` with `API_KEY`, `KAGGLE_USERNAME`, `KAGGLE_API_KEY`, etc.
  - `docker-compose up -d` (starts mlflow, prefect, prometheus, grafana, api, locust)
  - `docker-compose logs -f` to tail logs
  - Access API UI: `http://localhost:7860` (see `/docs`) and metrics at `/metrics`.
- Run pipeline locally (without Docker): `python -m src.pipelines.pipeline` (flow has __main__).
- Run unit tests: `pytest tests/test_model.py -v` (the Prefect flow also runs tests via a subprocess in `run_pytest`).

4. Project-specific conventions and gotchas
- Model names: training registers models as `news_classifier_{classifier_type}` and saves `models/news_classifier_{classifier_type}.joblib`. Code loads `latest` by default in the API.
- Prefect smoke test uses FastAPI `TestClient` (no separate uvicorn required) — `src/pipelines/pipeline.py` uses this for quick API checks.
- Data preprocessing expects a single CSV in `data/raw` (see `src/data/preprocess.py`); `download_bbc_dataset()` places `bbc-news-data.csv` there.
- Metrics: custom Prometheus metrics are implemented in `src/api/main.py` (singleton `Metrics`) — add labels there for new metrics.
- Entrypoint behavior: container entrypoint waits for MLflow/Prefect health and optionally runs pipeline (timeout guarded). See `Dockerfile` entrypoint script content.

5. Notable inconsistencies to watch for
- `pyproject.toml` claims `requires-python = ">=3.14"` while README and Dockerfile target Python 3.10 — prefer Python 3.10–3.11 for local/dev parity.
- `requirements.txt` and `pyproject.toml` have overlapping but non-identical dependency pins — use `requirements.txt` inside Docker images and for local `pip` installs.

6. Where to make common changes
- Add API endpoints or middleware: edit `src/api/main.py` (routes, auth, metrics).
- Add new Prefect tasks/flows: edit `src/pipelines/pipeline.py` and place task helpers under `src/pipelines/`.
- Change model training behavior or hyperparams: edit `src/models/train.py` and `configs/model_params.yaml`.
- Update MLflow server settings: edit `configs/mlflow_config.yaml` and docker-compose `mlflow` service.

7. Examples to reference in PRs
- Add new metric: follow pattern in `src/api/main.py` (define Counter/Histogram in `Metrics` singleton and use in middleware/endpoints).
- Add new dataset source: implement download function mirroring `src/data/download.py` and add a Prefect task that calls it.

If anything here is unclear or you'd like more detail on a specific area (tests, CI/CD, Kubernetes manifests), tell me which section to expand.
