# GO_WORKFLOW.md

> **Note:** This file provides the Go-specific commands for Phase 4 of [WORKFLOW.md](../WORKFLOW.md). Read `WORKFLOW.md` first.

## 4.0 Map the Change to a Gate

- **Fast gate:** The change touches one package and adds or changes no exported identifier, `go.mod` entry, or shared configuration file.
- **Dependency gate:** The change adds or changes an exported identifier, a `go.mod` dependency or version, or a file more than one package imports.
- **Final gate:** The change is module-wide, or the task is the plan's last task.

## 4.1 Code Formatting
Ensure the code is properly formatted for the affected packages:
```bash
gofmt -l -w <package_path>
```
Check `goimports` availability before using it. Treat a missing binary as an Environment failure, not an Implementation failure, and fall back to `gofmt` alone:
```bash
command -v goimports || go install golang.org/x/tools/cmd/goimports@latest
goimports -w <package_path>
```

## 4.2 Run Tests
Under the Fast gate, run tests for the **affected packages** only:
```bash
go test ./<package_path>/...
```
Run tests with the race detector when the task touches concurrency, shared state, or locking, regardless of gate:
```bash
go test -race ./<package_path>/...
```
Under the Final gate, run the full module test suite, including the race detector:
```bash
go test -race ./...
```

## 4.3 Run Lint
Run `go vet` on the **affected packages** under every gate:
```bash
go vet ./<package_path>/...
```
Run the project linter using the repository's configuration file (for example `.golangci.yml`) if one exists. Ask the user for the exact configuration if none exists:
```bash
golangci-lint run --config <repo_config_path> ./<package_path>/...
```

## 4.4 Run Dependency Check
Under the Dependency gate, verify the module graph resolves and stays tidy:
```bash
go mod tidy
go mod verify
```

## 4.5 Run Full Build Check
Under the Dependency gate and the Final gate, ensure local package changes do not break downstream packages:
```bash
go build ./...
```

## 4.6 Final Test and Build Check
Before Phase 5 archives the plan, run the complete module check regardless of which packages the plan touched:
```bash
gofmt -l .
go vet ./...
golangci-lint run ./...
go test -race ./...
go build ./...
```

## 4.7 Suppression Rule
Do not suppress lint findings with `//nolint` unless there is a justified reason documented in the plan file.
