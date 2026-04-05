# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hybrid Python-C++ module using Pybind11 and Modern CMake. C++ side implements a zero-copy RAII wrapper around NumPy arrays and performs GIL-free, multi-threaded in-place image transformations via `std::thread`. The shared object (`.so`) is imported directly from Python.

## Build Commands

```bash
# Configure and build
cmake -B build && cmake --build build

# Build with ThreadSanitizer
cmake -B build -DCMAKE_CXX_FLAGS="-fsanitize=thread" && cmake --build build

# Clean rebuild
rm -rf build && cmake -B build && cmake --build build
```

The compiled `.so` lands in `build/`. Import from Python with `PYTHONPATH=build uv run python ...`.

## Python Environment

Uses `uv` with `pyproject.toml` and `uv.lock` for dependency management.

```bash
# Install all deps (creates .venv automatically)
uv sync

# Add a new dependency
uv add <package>

# Run scripts
uv run python python/bench.py
```

## Test Commands

```bash
# C++ unit tests (Google Test via ctest)
ctest --test-dir build --output-on-failure

# Python tests
uv run pytest python/ -v

# Run a single Python test file
uv run pytest python/test_zerocopy.py -v

# Run benchmarks
uv run pytest python/test_bench.py -v --benchmark-enable

# Manual benchmark
uv run python python/bench.py

# Full acceptance validation
uv run python python/run_validation.py

# All tests (C++ + Python)
ctest --test-dir build --output-on-failure && uv run pytest python/ -v
```

## Architecture

Three C++ layers, one Python interface:

1. **Pybind11 Bindings** (`src/module.cpp`) — Module entry point. Binds `ImageBuffer` class and `process_quadrants` function. The `process_quadrants` binding uses `py::call_guard<py::gil_scoped_release>()` to release the GIL for the entire C++ call.

2. **RAII Buffer Wrapper** (`include/image_buffer.hpp`) — `ImageBuffer` class. Constructor accepts `py::array_t<uint8_t, py::array::c_style | py::array::forcecast>`, extracts raw pointer + dimensions from `py::buffer_info`, validates shape (3D, 3-4 channels, writable). Stores only C++ primitives (`uint8_t*`, `int`) — no `py::` types as members. Destructor is defaulted (non-owning view; NumPy owns the memory).

3. **Transform Kernels** (`src/transforms.cpp`) — `transform_quadrant()` operates on a row range via raw pointer arithmetic. `process_quadrants()` partitions image into 4 row-ranges, dispatches 4 `std::thread` instances, joins all in a scope-exit loop. No thread touches the Python API.

**Data flow**: Python allocates NumPy array → passes to `ImageBuffer` (zero-copy, same pointer) → C++ threads mutate pixels in-place with GIL released → control returns to Python with the original array modified.

## Build System

- CMake ≥3.16, C++17
- Pybind11 and GoogleTest fetched via `FetchContent` (not git submodules)
- `pybind11_add_module` target compiling `src/module.cpp` and `src/transforms.cpp`
- GoogleTest target in `tests/` for C++ unit tests, discovered via `gtest_discover_tests()`

## Interaction Rules

- Do NOT provide, suggest, or write any code unless the user explicitly asks for it. Discuss, explain, and plan — but no code until requested.

## Commits

Use [Conventional Commits](https://www.conventionalcommits.org/): `<type>(<scope>): <description>`. Common types: `feat`, `fix`, `docs`, `build`, `test`, `refactor`, `perf`, `chore`.

## Key Constraints

- **Zero-copy invariant**: `ImageBuffer.data_ptr()` must equal `ndarray.ctypes.data`. Any code path that copies the buffer is a bug.
- **GIL discipline**: All code running inside `std::thread` must use only raw C++ types. No `py::object`, `py::array_t`, or any Python API calls from threads.
- **Thread safety**: Quadrant row-ranges must be non-overlapping and cover `[0, H)` exactly. `ImageBuffer` members read during threading must be effectively immutable after construction.
- **RAII scope**: `ImageBuffer` manages access lifetime, not allocation lifetime. All threads must be joined before the `ImageBuffer` (and thus the buffer view) goes out of scope.
