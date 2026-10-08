# FS_WORKFLOW.md

> **Note:** This file provides the F#/.NET-specific commands for Phase 4 of [WORKFLOW.md](../WORKFLOW.md). Read `WORKFLOW.md` first.

## 4.0 Map the Change to a Gate

- **Fast gate:** The change touches one project and adds or changes no public API, signature file (`.fsi`), `<Compile Include>` order, `.fsproj` entry, or `Directory.*`/`global.json` file.
- **Dependency gate:** The change adds or changes a public API, a `.fsi` signature file, a `PackageReference` or version, the `<Compile Include>` order, a `Directory.Build.props`/`Directory.Packages.props`/`global.json` file, or a file more than one project references.
- **Final gate:** The change is solution-wide, or the task is the plan's last task.

F# compiles files in `<Compile Include>` order. Record any change to that order in the Execution Log's Changes & Decisions field.

## 4.1 Code Formatting
Ensure the code is properly formatted for every affected project with Fantomas. Restore local tools from `.config/dotnet-tools.json` when it exists. Treat a missing `fantomas` tool as an Environment failure, not an Implementation failure:
```bash
dotnet tool restore
dotnet fantomas <affected_files_or_directory>
```

## 4.2 Run Tests
Under the Fast gate, run tests for the **affected project** only:
```bash
dotnet test <project_path>/<name>.fsproj
```
Under the Dependency gate, also run tests for every project that references the changed project.
Under the Final gate, run the full test suite:
```bash
dotnet test
```

## 4.3 Run Lint
The F# compiler is the primary linter. Build the **affected project** with warnings as errors:
```bash
dotnet build <project_path>/<name>.fsproj --warnaserror
```
Run FSharpLint when the repository or tool manifest configures it. Check tool availability before using it; treat a missing `fsharplint` tool as an Environment failure:
```bash
dotnet fsharplint lint <project_path>/<name>.fsproj
```

## 4.4 Run Dependency and Build Checks
Under the Dependency gate, verify the project and its package references resolve cleanly:
```bash
dotnet restore <project_path>/<name>.fsproj --force-evaluate
```
Under the Dependency gate and the Final gate, ensure local project changes do not break downstream projects:
```bash
dotnet build --no-restore
```

## 4.5 Toolchain Recording
Record the .NET SDK and Fantomas versions in the plan's Discovered Gotchas & Constraints section when a task's result depends on a specific toolchain version:
```bash
dotnet --version
dotnet fantomas --version
```
Treat a mismatch between the recorded toolchain (including `global.json`) and the toolchain installed for verification as an Environment failure, not an Implementation failure.

## 4.6 Final Project Check
Before Phase 5 archives the plan, run the complete project check regardless of which projects the plan touched:
```bash
dotnet fantomas --check .
dotnet build --warnaserror
dotnet test
dotnet build --no-restore
```

## 4.7 Suppression Rule
Do not suppress warnings with `#nowarn`, `fsharplint:disable` comments, or `.editorconfig` severity overrides unless there is a justified reason documented in the plan file.
