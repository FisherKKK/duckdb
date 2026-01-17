# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

**Basic builds:**
- `make` or `make release` - Build optimized release version
- `make debug` - Build debug version with sanitizers and debug symbols
- `make reldebug` - Build RelWithDebInfo (optimized with debug info)

**Parallel builds:**
- `GEN=ninja make` - Use Ninja for faster parallel builds
- `CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make` - Limit parallel jobs to prevent system lockup

**Speed up rebuilds:**
- Install ccache (`apt install ccache` or `brew install ccache`) - automatically detected and used

## Testing

**Test execution:**
- `make unittest` - Run fast unit tests (~1 minute, debug build)
- `make allunit` - Run all unit tests (~1 hour, uses release build for speed)
- `./build/debug/test/unittest` - Run tests directly
- `./build/debug/test/unittest "[sqlitelogic]"` - Run specific test group
- `python3 scripts/run_tests_one_by_one.py build/debug/test/unittest --time_execution` - Run tests one at a time with timing

**Test types:**
- **Preferred:** SQL logic tests (`.test` files in `/test` directory) - use sqllogictest framework
- **C++ tests:** Only when testing exotic behavior (concurrent connections, etc.)
- Slower tests should use `.test_slow` extension or `[.]` tag in C++ tests

**Running a single test:**
- SQL logic test: `./build/debug/test/unittest test/sql/path/to/test.test`
- C++ test: `./build/debug/test/unittest "[test_group_name]"`

## Code Formatting

- `make format-fix` - Auto-format all code (runs clang-format and black)
- `make format-check` - Check formatting without fixing
- Use clang-format version 11.0.1: `python3 -m pip install clang-format==11.0.1`
- Format style: tabs for indentation, spaces for alignment, 120 column limit

## Architecture Overview

DuckDB follows a classic database architecture with clean separation between layers:

### Query Execution Pipeline

```
SQL String → Parser → Planner/Binder → Optimizer → Physical Plan Generator → Executor → Results
```

1. **Parser** (`src/parser/`) - Uses Postgres libpg_query parser, converts to custom AST with SQLStatements, Expressions, TableRefs
2. **Planner** (`src/planner/`) - Converts parsed AST to Logical Query Plan (tree of LogicalOperators), resolves symbols via Binder
3. **Optimizer** (`src/optimizer/`) - 40+ optimization passes (filter pushdown, join ordering, CSE, late materialization, etc.)
4. **Executor** (`src/execution/`) - Converts LogicalOperators to PhysicalOperators, executes using push-based vectorized model
5. **Catalog** (`src/catalog/`) - Metadata store for tables, schemas, functions, types
6. **Storage** (`src/storage/`) - Physical storage with RowGroups, column compression, buffer management, WAL, checkpoints
7. **Transaction** (`src/transaction/`) - MVCC transaction management with LocalStorage for uncommitted changes

### Vectorized Execution Model

- Data flows as **DataChunk** objects containing columnar **Vector** objects (typically 2048 rows)
- All operators process batches of rows, not row-at-a-time
- Enables SIMD operations and cache efficiency
- Physical operators implement `Init()`, `Execute()`, `Finalize()` lifecycle

### Extension System

Located in `extension/` directory:
- Extensions register functions/types via `ExtensionLoader` API
- Can be statically linked (core_functions) or dynamically loaded (parquet, json, etc.)
- Extensions implement `DUCKDB_EXTENSION_ENTRY` macro
- Build-time configuration in `extension/extension_config.cmake`

**Building with extensions:**
- `BUILD_BENCHMARK=1 BUILD_TPCH=1 make` - Build with benchmark and TPC-H extensions
- `BUILD_EXTENSIONS=parquet;json make` - Build specific extensions
- `SKIP_EXTENSIONS=parquet make` - Skip specific extensions

### Storage Architecture

```
DataTable
├── RowGroupCollection (persistent)
│   └── RowGroup (compressed column segments)
│       └── ColumnSegment (physical storage with compression)
└── LocalStorage (per-transaction uncommitted changes)
```

- **BufferManager** - Memory pool with disk spillage for large datasets
- **StorageManager** - Coordinates persistent storage, WAL, checkpoints
- Supports both in-memory and persistent databases seamlessly

## C++ Code Guidelines

**Memory management:**
- Use smart pointers, avoid `malloc`/`new`/`delete`
- Prefer `unique_ptr` over `shared_ptr`

**Naming conventions:**
- Files: `lowercase_with_underscores.cpp`
- Types/Classes: `CamelCase` (e.g., `BaseColumn`)
- Variables: `lowercase_with_underscores` (e.g., `chunk_size`)
- Functions: `CamelCase` (e.g., `GetChunk`)

**Types:**
- Use `[u]int(8|16|32|64)_t` instead of `int`, `long`, etc.
- Use `idx_t` instead of `size_t` for indices/offsets/counts

**Other conventions:**
- Use `const` liberally, especially for non-trivial object parameters
- Do not import namespaces (`using std` forbidden)
- All functions in `src/` must be in `duckdb` namespace
- Use `override` or `final` for virtual methods
- Use C++11 range-based for loops
- Always use braces for if/loops, even single-line
- Use `D_ASSERT` for assertions (should never trigger on user input)
- Avoid magic numbers - use named `constexpr` variables
- Use `DConstants::INVALID_INDEX` instead of magic numbers like -1 for invalid indices

**Error handling:**
- Exceptions only for query-terminating errors (parser error, table not found)
- Use return values for expected errors
- Assert liberally with explanatory comments

## Key Source Locations

**Adding functionality:**
- Scalar functions: `extension/core_functions/scalar/`
- Aggregate functions: `extension/core_functions/aggregate/`
- Table functions: `extension/core_functions/table/`
- Operators: `src/execution/operator/` (physical) and `src/planner/operator/` (logical)
- Optimizations: `src/optimizer/` (each file is one optimization pass)

**Understanding features:**
- Type system: `src/common/types/`
- Vector operations: `src/common/types/vector.hpp`
- Expression system: `src/planner/expression/`
- Storage format: `src/storage/table/`

## Pull Request Workflow

1. **Before starting:** Discuss intended changes with core team on GitHub
2. **Formatting:** Run `make format-fix` before committing
3. **Testing:** Ensure all tests pass with `make allunit`
4. **Avoid:** Large pull requests, Draft PRs (use issues/discussions instead)
5. **Commit directly to main:** Never - always use fork and PR
6. **CI:** PRs auto-transition to draft on changes, mark "ready for review" for full CI run

## Common Development Tasks

**Debugging a query:**
1. Enable query logging: `PRAGMA enable_profiling='query_tree';`
2. Run query and examine logical/physical plans
3. Use `D_ASSERT` statements to verify assumptions
4. Build with `make debug` to enable sanitizers

**Adding a new scalar function:**
1. Create implementation in `extension/core_functions/scalar/<category>/`
2. Register via `ScalarFunctionSet` in extension's init function
3. Add tests in `test/sql/function/` as `.test` file
4. Test with `make unittest`

**Investigating storage layout:**
1. Start with `src/storage/data_table.hpp` → `row_group.hpp` → `column_segment.hpp`
2. Buffer management in `src/storage/buffer_manager.cpp`
3. Compression in `src/storage/compression/`

**Understanding an optimization:**
1. Each optimizer is a separate class in `src/optimizer/`
2. Check `src/optimizer/optimizer.cpp` for pass ordering
3. Optimizers transform LogicalOperator trees

## Build Configuration

The build system uses CMake with extensive configuration options via Makefile variables:
- Set `DISABLE_SANITIZER=1` to disable sanitizers (faster debug builds)
- Set `FORCE_ASSERT=1` for assertions in release builds
- Set `DISABLE_UNITY=1` to disable unity builds (better for incremental compilation)
- See `Makefile` for full list of configuration options

## Important Notes

- DuckDB uses libpg_query (Postgres parser) but has its own query engine
- The codebase enforces strict formatting - always run `make format-fix`
- Code coverage is important - aim to cover all code paths in tests
- Documentation: https://duckdb.org/docs/stable/
- Build guide: https://duckdb.org/docs/stable/dev/building/overview
