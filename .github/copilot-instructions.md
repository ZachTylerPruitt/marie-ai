## Marie-AI: AI agent quickstart

- **What this is:** Agentic document-intelligence platform built on a Jina-style `Flow` that wires together executors (OCR, classification, NER, overlay, indexing, etc.) behind gRPC/HTTP gateways. Runtime is driven by YAML configs in `config/service/*.yml` (see `marie.yml` for a representative flow: gateway on 51000/52000, storage, scheduler, toast tracking, auth keys, and executors defined with `py_modules`).
- **Key code layout:**
  - `marie/`: core engine (Flow orchestration, executor runtimes, scheduler, storage, messaging, metrics, discovery, logging). `orchestrate/flow/base.py` shows how deployments/gateways are assembled. Executors live under `marie/executor`, `marie/serve`, and related subpackages.
  - `marie_server/`: server wrappers and REST extensions for executors.
  - `marie_cli/`: CLI entrypoints (`marie` / `python -m marie`) and autocomplete assets.
  - `config/service/`: flow definitions and service configs; env interpolation uses `${{ ENV.* }}` for secrets such as RabbitMQ, S3, and Postgres.
  - `Dockerfiles/` + `build.sh`: CPU/CUDA images and gateway profiles; `docker-scripts/` has run/stop helpers.
  - `bootstrap-marie.sh` + `bootstrap.md`: one-stop deployer that brings up infra (RabbitMQ, MinIO/S3, ETCD, Postgres, LiteLLM) and app services (gateway + extract executors) via Compose; flags like `--infrastructure-only`, `--services-only`, `--no-litellm` control what starts.

### Dev setup & installs
- Python ≥ 3.10. `pip install -e .` uses the **standard** extras by default; toggle minimal sets with `MARIE_PIP_INSTALL_CORE=1` or perf-only with `MARIE_PIP_INSTALL_PERF=1`. Extras are defined/tagged in `extra-requirements.txt` (`core`, `perf`, `standard`, `devel`, `test`).
- For shell completions, the installer drops snippets from `marie/resources/completions` into your shell rc files.

### Quality & tests
- Formatting/lint: `make quality` (isort + flake8 check) or `make modified_only_fixup` for touched files; black is configured but usually run selectively.
- Tests: `make test` → `python -m pytest -n auto --dist=loadfile -s -v ./tests/`. Markers: `slow`, `asyncio`, `timeout`, `repeat`; filter heavy suites with `-m "not slow"`. `pytest.ini` sets `asyncio_mode=auto` and a 180s faulthandler timeout.

### Running services
- YAML flows in `config/service` are consumed by the CLI/containers (e.g., inside an image: `marie server --start --uses config/service/marie.yml`). Gateways expose HTTP+gRPC; executors reference `py_modules` that must stay importable from repo root.
- For end-to-end local env, prefer the bootstrap script: `./bootstrap-marie.sh --infrastructure-only --no-litellm` to start dependencies, then `./bootstrap-marie.sh --services-only` for gateway/extract. Service endpoints: HTTP 52000 / gRPC 51000; RabbitMQ 15672, MinIO 9000/9001, Grafana 3000, Postgres 5432 (see `bootstrap.md`).
- Docker builds: `build.sh` profiles `marie-gateway-cpu` and `marie-cuda`; edit `PROFILES` there to adjust tags/Dockerfiles. Compose overlays live under `Dockerfiles/docker-compose.*.yml`.

### Conventions & patterns
- Follow Conventional Commit types (see `CONTRIBUTING.md`), and use `black`/`isort`/`flake8` configs from `setup.cfg`/`pyproject.toml` when touching Python.
- Flows favor declarative wiring: keep executor args in YAML, use `with:` blocks for storage/scheduler/message settings, and reuse shared anchors (e.g., `&psql_conf_shared`, `&rabbitmq_conf_shared`).
- Many configs expect secrets via env (`AWS_MQ_*`, `RABBIT_MQ_*`, `S3_*`, `ENV.*` in YAML). Avoid hardcoding; surface new requirements in sample env files under `config/`.
- When adding executors, ensure they are registered/importable and update relevant flow YAMLs plus `config/service` examples; keep `py_modules` pointed at the module implementing the executor.
- Metrics/tracing hooks rely on extras (opentelemetry/prometheus); keep optional deps under the tagged extras instead of baking them into core.

### Useful references
- Quick-start & architecture/runbook: `README.md` and `bootstrap.md`.
- Flow/config samples: `config/service/*.yml` (start with `marie.yml`).
- Docker/ops helpers: `Dockerfiles/` and `docker-scripts/*.sh`.
- Contribution norms: `CONTRIBUTING.md`, `CODE_REVIEW_GUIDELINES.md`.
