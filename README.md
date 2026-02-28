> [!NOTE]
> This document is available in Traditional Chinese: [README.zh.md](./README.zh.md)

# skill-coverage-generate [![license](https://img.shields.io/github/license/pardnchiu/skill-coverage-generate)](LICENSE)

> A Claude Code Skill that automatically analyzes Go test coverage per function and generates table-driven test cases to bring projects to ≥90% coverage threshold.
> Invoke `/coverage-generate` from any Go project directory — it discovers source files, measures coverage, and writes missing test cases end-to-end.

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Usage](#usage)
- [CLI Reference](#cli-reference)
- [License](#license)

## Features

- **Automated Coverage Analysis**: Runs `go test -coverprofile` and parses per-function coverage to identify gaps below 90%
- **Smart Test Generation**: Creates table-driven tests covering nominal, nil input, boundary, error path, and context cancellation scenarios
- **Multi-Package Support**: Processes entire Go modules via `./...`, handling all packages in one run
- **Intelligent Exclusions**: Skips `main.go`, generated files (`*.pb.go`, `*_generated.go`), and `init()` functions automatically
- **Quality Gates**: Validates compilation with `go build ./...` and re-runs coverage after generation to confirm improvement

## Installation

```bash
git clone https://github.com/pardnchiu/skill-coverage-generate ~/.claude/skills/coverage-generate
```

## Usage

From within a Go project directory, invoke the skill:

```
/coverage-generate
```

Claude will run the full 4-phase workflow — discovery, analysis, generation, and validation — and report the before/after coverage delta.

## CLI Reference

### Workflow Phases

| Phase | Action | Tool |
|-------|--------|------|
| 1. Discovery | Find non-test Go source files | `find` |
| 2. Analysis | Generate coverage report | `go test -coverprofile=coverage.out ./...` |
| 3. Generation | Parse gaps and write tests | `go tool cover -func=coverage.out` |
| 4. Validation | Compile + re-measure coverage | `go build ./...`, `go test ./...` |

### Excluded Files

| Pattern | Reason |
|---------|--------|
| `*_test.go` | Already test files |
| `*.pb.go`, `*_generated.go` | Generated code |
| `main.go` | Integration test scope |
| `init()` functions | Not directly testable |

## License

This project is licensed under the [MIT LICENSE](LICENSE).
