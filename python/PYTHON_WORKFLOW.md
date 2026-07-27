# PYTHON_WORKFLOW.md

> **Note:** This file provides the Python-specific commands for Phase 4 of [WORKFLOW.md](../WORKFLOW.md). Read `WORKFLOW.md` first.

## 4.1 Code Formatting
Ensure the code is properly formatted:
```bash
ruff format .
```

## 4.2 Run Tests
Run tests only for the **affected packages** — not the entire project unless the change is project-wide:
```bash
pytest <package_path>
```

## 4.3 Run Lint
Run the linter and type checker on the **affected packages** only:
```bash
ruff check <package_path>
mypy <package_path>
```

## 4.4 Run Full Project Check
Ensure local package changes do not break downstream dependencies:
```bash
mypy .
```

## 4.5 Suppression Rule
Do not suppress findings with `# noqa` or `# type: ignore` unless there is a justified reason documented in the plan file.
