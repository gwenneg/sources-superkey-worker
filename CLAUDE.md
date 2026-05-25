@AGENTS.md

# CLAUDE.md — Claude Code-specific guidance

## Verify before committing

```sh
make build        # must compile cleanly
make lint         # go vet + golangci-lint (gofmt, gci, bodyclose, forcetypeassert, misspell)
```

Both commands must pass with zero errors. CI runs the same checks on every push and PR.

## No tests exist

There are zero `*_test.go` files. Do not run `go test`. Do not add test files unless explicitly asked — there is no test framework in place. Verify correctness by reading code and confirming `make build && make lint` pass.

## Import ordering (gci)

The `gci` linter enforces import grouping. If `make lint` fails on import order, run `make gci` to auto-fix, then review the diff before committing.

## CI version mismatch

CI pins `go-version: "1.23"` while `go.mod` declares `go 1.24`. This is intentional. Do not change the CI Go version.
