# CLAUDE.md

## Project Overview

**microhash** is a Go library providing consistent hashing and utility hash functions. It implements a ring-based consistent hash with virtual replicas, suitable for distributed systems (e.g., load balancing, distributed caching, sharding). The module path is `github.com/DTreshy/microhash` and the package name is `microhash`.

## Repository Structure

```
microhash/
├── consistenthash.go       # Core ConsistentHash ring implementation
├── consistenthash_test.go  # Tests and benchmarks for consistent hashing
├── hash.go                 # Hash utility functions (MurmurHash3, MD5)
├── hash_test.go            # Tests and benchmarks for hash functions
├── go.mod / go.sum         # Go module dependencies
├── .golangci.yml           # golangci-lint configuration
├── .github/workflows/
│   └── test.yml            # CI pipeline (lint, test, coverage)
├── renovate.json           # Renovate bot for dependency updates
├── LICENSE                 # MIT License
└── README.md               # Project badges
```

This is a single-package library — all `.go` files are in the root directory under `package microhash`. There are no sub-packages.

## Build & Development Commands

### Prerequisites
- Go 1.20+ (CI uses Go 1.24.4)

### Run tests
```sh
go test ./...
```

### Run tests with race detection and coverage (mirrors CI)
```sh
go test -race -coverprofile=coverage.out -covermode=atomic -timeout 2m ./...
```

### Run benchmarks
```sh
go test -bench=. -benchmem ./...
```

### Lint
```sh
golangci-lint run
```

The linter configuration is in `.golangci.yml`. Key enabled linters: `errcheck`, `govet`, `staticcheck`, `gosec`, `gocritic`, `wsl`, `dupl`, `misspell`, `unconvert`, `goconst`, `ineffassign`, `unused`, `gosimple`, `typecheck`.

## Architecture & Key Concepts

### ConsistentHash (consistenthash.go)
- **Ring-based consistent hashing** with configurable virtual replicas (minimum 100).
- Thread-safe via `sync.RWMutex`.
- Nodes are mapped to the ring using their string representation (`repr()`) plus a replica index.
- `Get(v)` finds the closest node clockwise on the ring; ties (hash collisions) are resolved with an inner hash using a prime multiplier.

### Public API
| Function/Method | Description |
|---|---|
| `New()` | Create a ConsistentHash with default replicas (100) and MurmurHash3 |
| `NewWithCustomHash(replicas, fn)` | Create with custom replica count and hash function |
| `Add(node)` | Add a node with default replicas |
| `AddWithReplicas(node, replicas)` | Add a node with explicit replica count |
| `AddWithWeight(node, weight)` | Add a node with weight (1-100, as percentage of max replicas) |
| `Get(v)` | Look up which node owns the given key |
| `Remove(node)` | Remove a node from the ring |
| `Hash(data)` | MurmurHash3 64-bit hash |
| `Md5(data)` | MD5 digest (returns bytes) |
| `Md5Hex(data)` | MD5 hex string |

### Key Constants
- `MaxWeight = 100` — maximum weight for `AddWithWeight`
- `minReplicas = 100` — minimum virtual replicas per node
- `prime = 16777619` — FNV prime used for inner hash disambiguation

### Dependencies
| Dependency | Purpose |
|---|---|
| `github.com/spaolacci/murmur3` | MurmurHash3 implementation (default hash function) |
| `github.com/stretchr/testify` | Test assertions (test-only) |

## Code Conventions

- **Package**: Single flat package `microhash` — no sub-packages.
- **Naming**: Standard Go conventions. Exported types use PascalCase, unexported use camelCase.
- **Node representation**: Nodes can be `any` type. Their identity on the ring is determined by `repr()`, which converts values to strings using `fmt.Stringer` interface, then reflection-based type switching.
- **Concurrency**: All `ConsistentHash` methods are goroutine-safe. Read operations use `RLock`, write operations use `Lock`.
- **Error handling**: `Get` returns `(value, bool)` pattern instead of errors. No sentinel errors are defined.
- **Testing**: Uses `testify/assert`. Tests verify distribution entropy, incremental transfer correctness, and minimal disruption on node failure. Benchmarks cover `New`, `Add`, and `Get` operations.
- **Linting**: Strict golangci-lint config with `errcheck` type assertion checking, `gocritic` with experimental/opinionated tags, `gosec` (excluding MD5 weakness warnings G401/G501), and `wsl` whitespace linting.

## CI Pipeline

The GitHub Actions workflow (`.github/workflows/test.yml`) runs on PRs to `master`:
1. Checkout code
2. Set up Go 1.24.4
3. Run `golangci-lint`
4. Run tests with race detection and coverage
5. Upload coverage to Codecov

## Things to Watch Out For

- The `wsl` linter enforces strict whitespace rules (blank lines before blocks, etc.). Follow existing code style for blank line placement.
- `errcheck` is configured with `check-type-assertions: true` — always handle type assertion results with the comma-ok pattern.
- `gosec` excludes `G401` and `G501` (weak crypto) since MD5 is intentionally provided as a utility, not for security.
- The `repr()` function uses reflection — when adding new node types, ensure they have a meaningful string representation (implement `fmt.Stringer` or rely on the type switch in `reprOfValue`).
