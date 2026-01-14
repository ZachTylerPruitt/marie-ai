# Migration Plan: uv Package Manager and Ruff Linter/Formatter

## Overview

This document outlines the migration plan for Marie-AI to adopt modern Python tooling:
- **uv**: Fast Python package manager and installer (replacement for pip, pip-tools, virtualenv)
- **Ruff**: Fast Python linter and formatter (replacement for flake8, black, isort)

## Benefits

### uv Package Manager
- **10-100x faster** than pip for package installation
- Built-in virtual environment management
- Compatible with pip requirements files
- Deterministic dependency resolution
- Cross-platform consistency
- Written in Rust for performance

### Ruff
- **10-100x faster** than existing tools (flake8, black, isort, pylint)
- Single tool replaces multiple linters/formatters
- Drop-in replacement with minimal configuration
- Auto-fix capabilities for most rules
- Written in Rust for performance
- Actively maintained by Astral (same team as uv)

## Current State Analysis

### Package Management (Current)
- **Tool**: pip + setuptools
- **Files**: 
  - `requirements.txt` (minimal, just `.`)
  - `extra-requirements.txt` (detailed dependencies with tags)
  - `setup.py` (complex setup with custom commands)
  - `pyproject.toml` (minimal, just build system)
- **Issues**:
  - Slow dependency resolution
  - No lock file for reproducibility
  - Complex setup.py with custom post-install hooks

### Linting/Formatting (Current)
- **Tools**: flake8, black, isort
- **Files**:
  - `.pre-commit-config.yaml` (flake8, black, isort hooks)
  - `.pylintrc` (pylint configuration)
  - `Makefile` (quality, style, test targets)
- **Configuration**:
  - flake8: Limited checks (E9, F63, F7, F82), max-line-length=127
  - black: With `-S` flag (skip string normalization)
  - isort: Black profile
- **Issues**:
  - Multiple tools with separate configs
  - Slow execution on large codebase
  - Inconsistent formatting between tools

## Migration Strategy

### Phase 1: Add Ruff (Parallel to existing tools)

**Goal**: Introduce Ruff alongside existing linters to validate compatibility

**Steps**:
1. Add Ruff configuration to `pyproject.toml`:
   ```toml
   [tool.ruff]
   line-length = 127
   target-version = "py310"
   
   [tool.ruff.lint]
   select = [
       "E9",   # Runtime errors (from flake8)
       "F",    # Pyflakes (F63, F7, F82 from flake8)
       "I",    # isort
   ]
   ignore = []
   
   [tool.ruff.lint.isort]
   profile = "black"
   
   [tool.ruff.format]
   quote-style = "single"  # Match black -S behavior
   skip-magic-trailing-comma = true
   ```

2. Add Ruff to pre-commit hooks (parallel to existing):
   ```yaml
   - repo: https://github.com/astral-sh/ruff-pre-commit
     rev: v0.8.4
     hooks:
       - id: ruff
         args: [--fix]
       - id: ruff-format
   ```

3. Update Makefile to include Ruff targets:
   ```makefile
   quality-ruff:
       ruff check $(check_dirs)
       ruff format --check $(check_dirs)
   
   style-ruff:
       ruff check --fix $(check_dirs)
       ruff format $(check_dirs)
   ```

4. Run Ruff on codebase and fix any new issues
5. Compare outputs of Ruff vs existing tools
6. Document any differences or edge cases

**Validation**:
- All existing pre-commit hooks pass
- Ruff reports no new critical issues
- Code formatting is consistent with black output

### Phase 2: Add uv for Development

**Goal**: Enable developers to use uv for faster dependency installation

**Steps**:
1. Install uv globally: `pip install uv`

2. Create `pyproject.toml` with full project metadata:
   - Move dependencies from setup.py to pyproject.toml
   - Define `[project]` section with metadata
   - Define `[project.optional-dependencies]` for extras
   - Keep `[build-system]` initially as setuptools

3. Generate `uv.lock` file:
   ```bash
   uv pip compile pyproject.toml -o requirements.lock
   ```

4. Update documentation (README.md, bootstrap.md):
   ```bash
   # Traditional way
   pip install -e .
   
   # With uv (faster)
   uv pip install -e .
   ```

5. Add GitHub Actions workflow using uv:
   ```yaml
   - name: Set up uv
     uses: astral-sh/setup-uv@v4
     with:
       version: "latest"
   
   - name: Install dependencies
     run: uv pip install -e ".[devel]"
   ```

**Validation**:
- Project installs successfully with uv
- All tests pass with uv-installed dependencies
- Development workflow is faster

### Phase 3: Replace Existing Linters with Ruff

**Goal**: Remove flake8, black, isort in favor of Ruff

**Steps**:
1. Remove old linter configurations:
   - Remove flake8 from `.pre-commit-config.yaml`
   - Remove black from `.pre-commit-config.yaml`
   - Remove isort from `.pre-commit-config.yaml`
   - Archive `.pylintrc` (rename to `.pylintrc.old`)

2. Update Makefile:
   - Replace `quality` target with `quality-ruff`
   - Replace `style` target with `style-ruff`
   - Keep old targets commented for reference

3. Update CI/CD workflows:
   - Replace linter steps with Ruff
   - Update caching strategies

4. Remove dependencies:
   - Remove flake8, black, isort from `extra-requirements.txt`
   - Add ruff to `extra-requirements.txt` under `devel` tag

5. Update CONTRIBUTING.md and CODE_REVIEW_GUIDELINES.md:
   - Document new linting/formatting commands
   - Update style guide references

**Validation**:
- All pre-commit hooks pass
- CI/CD pipelines pass
- Code quality is maintained
- Documentation is accurate

### Phase 4: Migrate Build System to uv (Optional)

**Goal**: Use uv as the primary build backend (most disruptive)

**Steps**:
1. Update `pyproject.toml` build-backend:
   ```toml
   [build-system]
   requires = ["hatchling"]
   build-backend = "hatchling.build"
   ```

2. Migrate setup.py logic:
   - Move version detection to `__about__.py` or similar
   - Move post-install hooks to separate scripts
   - Document manual steps if hooks can't be automated

3. Update packaging:
   ```bash
   uv build
   ```

4. Update release workflows:
   - Use uv for building distributions
   - Test on PyPI test server first

**Validation**:
- Package builds successfully
- Installation works on clean environments
- All extras install correctly
- Post-install hooks work (or documented as manual)

## Rollback Plan

### If Ruff Issues Found
1. Keep pre-commit hooks with both tools
2. Revert to flake8/black/isort as primary
3. File issues with Ruff project
4. Retry after Ruff updates

### If uv Issues Found
1. Keep documentation for both pip and uv
2. Continue using pip in CI/CD
3. Allow uv as optional developer tool
4. Retry after uv matures

## Implementation Timeline

- **Week 1**: Phase 1 - Add Ruff (parallel)
- **Week 2**: Phase 2 - Add uv for development
- **Week 3**: Phase 3 - Replace existing linters
- **Week 4+**: Phase 4 - Migrate build system (optional)

## Success Criteria

- [ ] CI/CD pipelines are faster (>2x speedup)
- [ ] Developer workflow is faster (>5x dependency install)
- [ ] Code quality is maintained or improved
- [ ] All tests pass
- [ ] Documentation is updated
- [ ] Team is trained on new tools
- [ ] No regression in functionality

## References

- [uv Documentation](https://docs.astral.sh/uv/)
- [Ruff Documentation](https://docs.astral.sh/ruff/)
- [Migrating from Black to Ruff](https://docs.astral.sh/ruff/formatter/#black-compatibility)
- [Migrating from isort to Ruff](https://docs.astral.sh/ruff/faq/#how-does-ruff-compare-to-isort)
- [Migrating from Flake8 to Ruff](https://docs.astral.sh/ruff/faq/#how-does-ruff-compare-to-flake8)

## Notes

- Maintain Python 3.10+ requirement (current: >=3.10)
- Preserve black's string normalization skip behavior (`-S` flag)
- Keep extensive directory exclusions from current config
- Consider impact on external contributors
- Monitor Ruff and uv for breaking changes
- Test thoroughly on all supported platforms (Linux, macOS, Windows)
