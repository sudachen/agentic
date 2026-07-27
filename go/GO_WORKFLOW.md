# GO_WORKFLOW.md

> **Note:** This file provides the Go-specific commands for Phase 4 of [WORKFLOW.md](../WORKFLOW.md). Read `WORKFLOW.md` first.

## 4.1 Code Formatting
Ensure the code is properly formatted:
```bash
gofmt -l -w .
goimports -w .
```

## 4.2 Run Tests
Run tests only for the **affected packages** — not the entire module unless the change is module-wide:
```bash
go test ./<package_path>/...
```

## 4.3 Run Lint
Run `go vet` and the project linter on the **affected packages** only:
```bash
go vet ./<package_path>/...
golangci-lint run ./<package_path>/...
```

## 4.4 Run Full Build Check
Ensure local package changes do not break downstream dependencies:
```bash
go build ./...
```

## 4.5 Suppression Rule
Do not suppress lint findings with `//nolint` unless there is a justified reason documented in the plan file.
