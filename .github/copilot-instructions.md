# GitHub Copilot Instructions for Marie-AI

This file provides context and guidelines for GitHub Copilot when working with the Marie-AI codebase.

## Project Overview

Marie-AI is an **Agentic Document Intelligence Platform** that orchestrates autonomous AI agents for document processing tasks including OCR, classification, NER, and document transformation. The architecture follows an agent-based design with modular executors.

## Language and Version Requirements

- **Python**: 3.10+ (strictly enforced in setup.py)
- **Key Feature**: Agent-based architecture with DAG-driven execution
- **Package Name**: `marie-ai` (PyPI), library name: `marie`

## Code Style and Standards

### General Guidelines
- Follow [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)
- Follow the [Zen of Python](https://zen-of-python.info/)
- Adhere to SOLID principles (see CODE_REVIEW_GUIDELINES.md)

### Current Linting/Formatting Tools
- **black**: Code formatter with `-S` flag (skip string normalization)
- **isort**: Import sorting with black profile
- **flake8**: Limited checks (E9, F63, F7, F82), max-line-length=127
- Pre-commit hooks configured in `.pre-commit-config.yaml`

### Future Migration (Planned)
- Migrating to **Ruff** for all linting/formatting
- Migrating to **uv** for package management
- See `.github/prompts/plan-uvOnlyRuffUvBuildMigration.prompt.md` for details

## Code Organization

### Directory Structure
```
marie/              # Main library code
marie_cli/          # CLI implementation
marie_server/       # Server implementation
tests/              # Test suite
docs/               # Documentation
examples/           # Example code
config/             # Configuration files
```

### Excluded from Linting
The following directories are excluded from most linting:
- `marie/storage`, `marie/core`, `marie/models` (legacy/external code)
- `marie/boxes/dit`, `marie/models/unilm` (model implementations)
- `marie/proto/docarray_v1`, `marie/proto/docarray_v2` (generated protobuf)
- `marie/resources/` (static resources)
- `hubble/resources/`, `tests/` (test code)

## Naming Conventions

### Variables
- Use **informative names** - no single letters like `k`, `v`, `x` (except in comprehensions)
- Global variables in `CAPS_CASE`
- Use appropriate underscores:
  - `_private`: Internal/private (single leading)
  - `__private`: Name mangling (double leading)
  - `variable_`: Avoid keyword conflicts (single trailing)

### Environment Variables
- All Marie environment variables start with `JINA_` prefix
- Example: `JINA_LOG_LEVEL`, `JINA_RANDOM_PORT`

### Functions and Classes
- Avoid renaming public APIs without deprecation strategy
- Single Responsibility Principle - one responsibility per class/function

## Type Hints and Documentation

### Type Hints
- Always add type hints to function signatures
- For imports only needed for type hints:
```python
if False:
    from PIL import Image

def func(arr: 'Image'):
    from PIL import Image
    ...
```

### Documentation
- Add docstrings to all public functions/classes
- Use codetags: `#TODO`, `#FIXME`, `#NOTE`, etc.
- Follow [PEP 350](https://www.python.org/dev/peps/pep-0350/#mnemonics) for codetags

## Testing Guidelines

### Test Organization
- **Unit tests**: For isolated component testing
- **Integration tests**: For Flow-based tests (goes under `/tests/integration`)
- Use pytest and pytest-xdist (`pytest -n auto`)

### Test Best Practices
- Add tests for new features
- Use `pytest tmpdir` fixture for temporary directories
- Test locally before pushing: `make test` or `pytest -n auto`
- Tests should be deterministic and isolated

## Common Patterns

### Random Ports
```python
from jina.helper import random_port
port = random_port()
```

### Function-Specific Imports
- Keep heavy imports inside functions when they're only needed there
- Reduces startup time and circular dependency issues

### Exception Handling
- Raise appropriate exceptions
- Use descriptive error messages
- Prefer specific exceptions over generic Exception

## Development Workflow

### Local Development
```bash
# Install in editable mode
pip install -e .

# Install with dev dependencies
pip install -e ".[devel]"

# Run linting/formatting
make quality        # Check code quality
make style         # Auto-fix style issues

# Run tests
make test          # Run all tests
pytest tests/unit/ # Run specific tests
```

### Pre-commit Hooks
```bash
# Install hooks
pre-commit install

# Run manually
pre-commit run --all-files
```

## Build and Release

### Building
- Uses setuptools with custom post-install commands
- `setup.py` handles auto-completion registration
- Complex dependency management via `extra-requirements.txt`

### Dependencies
- Core dependencies in `extra-requirements.txt` with tags:
  - `core`: Essential dependencies
  - `perf`: Performance enhancements (uvloop, etc.)
  - `standard`: Standard installation
  - `devel`: Development tools
  - `cicd`: CI/CD specific tools

### Platform-Specific
- uvloop excluded on Windows: `uvloop:perf,standard,devel` with conditional

## Agent Architecture Patterns

When working with Marie's agent-based architecture:

### Executor/Agent Pattern
```python
from marie import Executor, requests

class CustomExecutor(Executor):
    @requests(on='/endpoint')
    def process(self, docs, **kwargs):
        # Process documents
        return docs
```

### Flow/DAG Pattern
- Flows orchestrate multiple executors
- DAG-based execution for dependency management
- Support for parallel and sequential processing

### Job Scheduling
- Soft and hard SLAs supported
- Job orchestration with dependency-aware execution
- Fault-tolerant design

## Contribution Guidelines

### Pull Request Scope
- Keep PRs simple, unique, and well-defined
- Avoid unrelated changes in single PR
- Document new features appropriately

### Code Review Checklist
- Informative variable names
- Type hints present
- Docstrings added
- Tests included
- No breaking changes (or proper deprecation)
- Follows SOLID principles

## Security Considerations

- No hardcoded credentials
- Use dependency injection
- Validate user inputs
- Use `.secrets.baseline` for secret detection
- Pre-commit hook: `detect-secrets`

## Performance Considerations

- Lazy loading for heavy dependencies
- Async/await patterns for I/O operations
- Use uvloop for performance (when available)
- Consider memory footprint for large document processing

## Common Pitfalls to Avoid

1. **Don't modify existing tests** unless fixing actual bugs
2. **Don't rename public APIs** without deprecation
3. **Don't use single-letter variables** in production code
4. **Don't commit secrets** or sensitive data
5. **Don't premature optimize** - profile first
6. **Don't skip type hints** - they catch bugs early
7. **Don't hardcode dependencies** - use injection

## Useful Commands Reference

```bash
# Quality checks
make quality                    # Run all quality checks
make modified_only_fixup       # Check only modified files

# Testing
make test                       # Run full test suite
pytest -n auto                  # Parallel test execution
pytest -k "test_name"          # Run specific test

# Pre-commit
pre-commit run --all-files     # Run all hooks
pre-commit autoupdate          # Update hook versions

# Environment info
python collect_env.py          # Collect environment information
```

## Additional Resources

- [Code Review Guidelines](.github/CODE_REVIEW_GUIDELINES.md)
- [Contributing Guide](CONTRIBUTING.md)
- [Quick Start Guide](bootstrap.md)
- [Migration Plan](.github/prompts/plan-uvOnlyRuffUvBuildMigration.prompt.md)

## Notes for Copilot

- When suggesting code, always include type hints
- Prefer explicit over implicit
- Consider the agent-based architecture context
- Check exclusion lists before suggesting linting fixes
- Respect the black `-S` flag (single quotes for strings)
- Be mindful of Python 3.10+ features availability
- Consider cross-platform compatibility (Linux, macOS, Windows)
- Check if code should be excluded based on directory patterns
