## Plan: uv-only + Ruff + uv build/export (Python 3.14)

The approach is:

- **uv only** for Python/tooling entrypoints (env + deps + lock + run + build + export)
- **Ruff** for lint + format (Black-compatible)
- keep `requirements*.txt` only as **generated exports** from `uv.lock` via `uv export`

### Guiding principles
- **One source of truth:** `pyproject.toml` + `uv.lock`.
- **Reproducible automation:** CI/Docker must never “float” dependencies.
- **Fast iteration:** use caching (uv cache + Docker layer caching).
- **Python policy:** use the *latest Python version that the dependency set supports*; prefer Python **3.14**.
- **Canonical formatting:** Ruff formatter is the source of truth, even if it deviates slightly from Black.

### Deliverables
- A dependency compatibility report (direct dependencies) that records each package’s declared `Requires-Python` and the inferred max supported Python minor version; highlight Python 3.14 blockers.
- `.python-version` pinned to the chosen version (prefer **3.14**, otherwise highest version that locks/tests)
- `uv.lock` committed
- Ruff configured in `pyproject.toml` and used everywhere
- `requirements*.txt` generated via `uv export` (with CI guard against drift)
- CI uses `uv sync` + `uv run ...` + `uv build`
- Dockerfiles updated to uv best practices (multi-stage, lockfile-first, cache mounts)

### Steps
1. Audit dependency Python support (prefer 3.14, but follow reality)
   - For the full dependency set you intend to install, generate a report of **direct** dependencies and their declared `Requires-Python`.
   - Use this report to identify **hard blockers** for Python 3.14 (packages whose metadata includes `<3.14`).
   - Example findings from this repo’s current dependency list (from `extra-requirements*.txt`):
     - `vllm` declares `<3.14,>=3.10` → hard-caps at Python **3.13**
     - `faiss-gpu-cu12` declares `<3.14,>=3.10` → hard-caps at Python **3.13**
   - Treat classifiers as informational only; `Requires-Python` is the stronger signal.

2. Standardize Python and uv as the single entrypoint (prefer 3.14)
   - Decide the **highest supported Python** for the current dependency set:
     - try Python **3.14** first
     - if locking or builds fail due to dependency constraints / missing wheels, fall back to the next highest supported version
   - Set `requires-python` in `pyproject.toml` accordingly (prefer a range that reflects the supported set).
   - Pin the chosen interpreter with `uv python pin <version>` (creates `.python-version`).
   - Document/implement policy: **no `pip`, no `python -m venv`** in docs/scripts/CI.

3. Consolidate dependencies into uv-managed sources and lock deterministically
   - Ensure all dependencies are declared in `pyproject.toml` (including dev/test tooling like `pytest`, `pre-commit`, `ruff`).
   - Create and commit `uv.lock`.
   - Standardize sync semantics:
     - **Local dev:** `uv sync` (fast, convenient).
     - **CI/Docker:** `uv sync --locked` (error if lock is missing/out-of-date) or `uv sync --frozen` (do not update lock). Prefer `--locked` when you want to guarantee the lockfile is authoritative.

4. Adopt Ruff for lint + Black-compatible formatting everywhere
   - Configure Ruff in `pyproject.toml`:
     - `target-version = "py314"`
     - `line-length` aligned to repo norms
     - select/ignore rules to match existing lint intent
   - Replace existing tooling calls (`flake8`, `isort`, `black`) across `Makefile`, scripts, CI, and docs:
     - Lint gate: `ruff check`
     - Format gate: `ruff format --check`
     - Local autofix: `ruff check --fix` and `ruff format`
    - Best practice: keep Ruff formatter **stable** (avoid preview mode unless explicitly opting into changing formatting rules).
    - Policy: accept Ruff formatting as canonical and remove Black from the toolchain (do not try to force Black parity).

5. Make builds uv-native with `uv build`
   - Use `uv build` to produce wheel + sdist.
   - Ensure `[build-system]` in `pyproject.toml` is correct for the current build backend.
     - Optional: consider switching to `uv_build` as a backend only if it demonstrably reduces complexity; otherwise, keep the existing backend and simply standardize on `uv build` as the command.

6. Keep requirements files, but only as uv exports (interop artifacts)
   - Replace hand-maintained `requirements*.txt` with generated outputs from `uv export`.
   - Best practice for determinism:
     - Export from the lockfile with `--frozen`/`--locked` semantics.
     - Use `--format requirements-txt` and write to a stable path.
   - Add guardrails:
     - Treat exported files as **generated** (header comment + CI check that exports match `uv.lock`).
     - Do not use exported requirements as the primary install mechanism inside the repo.

7. Update bootstrap/Docker/CI to use uv end-to-end
   - Refactor `bootstrap-marie.sh`, `build.sh`, and Dockerfiles (including `Dockerfiles/protogen.Dockerfile`) so they:
     - install uv once
     - run `uv sync` in build steps
     - use `uv run` for all Python commands (tests, tooling, CLIs)
   - In CI:
     - install uv (best practice: use `astral-sh/setup-uv`)
     - set up Python 3.14 from `.python-version` if the CI platform supports it
     - run `uv sync --locked` (or `--frozen`), then `uv run ruff ...`, `uv run pytest ...`, and `uv build`

   Docker best practices to apply while doing this:
   - **Pin uv itself** for reproducible builds (pin by version tag, or ideally by image digest when copying the uv binary).
   - Use **multi-stage builds**:
     - builder stage installs deps (and optionally builds wheels)
     - runtime stage contains only what’s needed to run
   - Enable **reproducible installs**:
     - use `uv sync --locked` or `uv sync --frozen`
     - add `--no-dev` for runtime images
   - Maximize layer caching (key pattern):
     - copy `pyproject.toml` + `uv.lock` first
     - run a dependency-only sync (best practice: `--no-install-project`) so dependency layers only change when lock or project metadata changes
     - copy the full source
     - run the final sync to install the project
   - Use BuildKit cache mounts for uv’s cache dir to speed up iterative builds.
   - If using cache mounts or CI environments with multiple mount points, set `UV_LINK_MODE=copy` to avoid hardlink issues.
   - Prefer **uv-native installs** inside Docker; keep `uv export` for interoperability/publishing (or specialized packaging like Lambda layers).

8. Pre-commit: use `uv-pre-commit` and `ruff-pre-commit` (scoped to relevant changes)
   - Use the official mirrored hooks:
     - `https://github.com/astral-sh/uv-pre-commit`
     - `https://github.com/astral-sh/ruff-pre-commit`
   - Pin hook `rev` values to known-good versions (no floating tags).
   - Recommended hook responsibilities:
     - `uv-lock`: keep `uv.lock` current when `pyproject.toml` changes
     - `uv-export`: regenerate `requirements*.txt` from `uv.lock` (include `--locked`/`--frozen` args)
     - `ruff-check` (optionally with `--fix`) and `ruff-format`
   - Only run lock/export when relevant files change:
     - Configure `uv-lock` to run only when dependency-definition files change.
       - During migration, that may include legacy inputs (e.g., `extra-requirements*.txt`) if they are still authoritative.
       - After migration, scope to `pyproject.toml` (and `uv.toml` if used).
     - Configure `uv-export` to run only when `uv.lock` changes.
     - Use `files:` patterns to enforce this.
   - Ordering best practice:
     - if using `ruff-check --fix`, run it **before** `ruff-format`.
   - CI best practice: run `pre-commit run --all-files` so the same hooks enforce policy in CI.

9. Formatting-only commits + blame hygiene
   - Ensure any repo-wide formatting churn happens in commits that contain **only** formatting changes.
   - Record those commit SHAs in `.git-blame-ignore-revs`.
   - Configure local git to respect it:
     - `git config blame.ignoreRevsFile .git-blame-ignore-revs`
   - Best practice: make formatting a single dedicated PR/commit series, merge it, then rebase feature work on top.

### Further considerations / risks
1. Docker + Python 3.14 readiness
   - Python 3.14 adoption may require adding OS build deps for packages that don’t have wheels yet.
   - Decide policy early:
     - either “build fails until deps support 3.14” (strict)
     - or temporarily allow a short-lived compatibility version for CI/runtime images (documented exception)

2. Selecting “latest supported Python”
   - Even with a preference for 3.14, some dependency sets will only lock/test on older versions.
   - Decide what constitutes “supported” for this repo (e.g., lock succeeds, unit tests pass, integration tests pass, Docker image builds).

3. Pre-commit developer experience
   - Running `uv-lock`/`uv-export` on every commit can be slow or noisy.
   - Consider scoping hooks via `files:` patterns (e.g., only when `pyproject.toml`/`uv.lock` change) and deciding whether `ruff-check` should run with `--fix` automatically or only in explicit developer workflows.

4. Ruff formatting churn
   - Expect one-time formatting churn during migration; coordinate it (ideally a dedicated commit/PR) to avoid long-lived rebase pain.
   - After the churn, enforce `ruff format --check` in CI to keep formatting stable.
