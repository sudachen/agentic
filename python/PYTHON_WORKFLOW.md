# PYTHON_WORKFLOW.md

> **Note:** This file provides the Python-specific commands for Phase 4 of [WORKFLOW.md](../WORKFLOW.md). Read `WORKFLOW.md` first.

## 4.0 Map the Change to a Gate

- **Fast gate:** The change touches one package and adds or changes no public function, class, entry point, or dependency.
- **Dependency gate:** The change adds or changes a public function, class, entry point, `pyproject.toml`/`setup.cfg` dependency, or a file more than one package imports.
- **Final gate:** The change is project-wide, or the task is the plan's last task.

## 4.1 Code Formatting
Ensure the code is properly formatted for every affected package:
```bash
ruff format <package_path>
```

## 4.2 Run Environment Setup
Activate or create the project's virtual environment before running any check. Treat a missing interpreter or missing dependency as an Environment failure, not an Implementation failure:
```bash
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
```

## 4.3 Run Tests
Under the Fast gate, run tests for the **affected package** only:
```bash
pytest <package_path>
```
Under the Dependency gate and the Final gate, also run the project's coverage command and record the result in the Execution Log's Verification field:
```bash
pytest --cov=<package_path> <package_path>
```

## 4.4 Run Lint and Type Checking
Run the linter and type checker on the **affected package** under the Fast gate:
```bash
ruff check <package_path>
mypy <package_path>
```

## 4.5 Run Packaging and Dependency Check
Under the Dependency gate, verify the package still builds and its dependency metadata resolves:
```bash
python -m build --sdist --wheel
pip check
```

## 4.6 Run Full Project Check
Under the Dependency gate and the Final gate, ensure local package changes do not break downstream packages:
```bash
mypy .
```

## 4.7 Final Project Check
Before Phase 5 archives the plan, run the complete project check regardless of which packages the plan touched:
```bash
ruff format --check .
ruff check .
mypy .
pytest --cov=.
```

## 4.8 Suppression Rule
Do not suppress findings with `# noqa` or `# type: ignore` unless there is a justified reason documented in the plan file.
