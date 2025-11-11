# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NVIDIA Inference Xfer Library (NIXL) is a high-performance point-to-point communication library for AI inference frameworks. It abstracts communication mechanisms and memory access across heterogeneous devices (CPU, GPU) and storage types (file, block, object store) through a modular plugin architecture.

**Key Concepts:**
- **North Bound API (NB API)**: User-facing API where applications express transfer requests through buffer list primitives
- **South Bound API (SB API)**: Standardized interface between NIXL's Transfer Agent and backend plugins
- **Plugin Architecture**: Dynamically loadable or statically compiled backend plugins (UCX, GDS, POSIX, LibFabric, etc.)
- **Transfer Agent**: Manages memory registration, metadata exchange, and delegates transfers to optimal backend plugins

## Build System

NIXL uses **Meson** (with Ninja backend) as its build system. The project requires C++17 and CUDA support.

### Common Build Commands

```bash
# Configure build (release is default)
meson setup build

# Configure with custom prefix
meson setup build --prefix=/opt/nvidia/nvda_nixl

# Debug build
meson setup build --buildtype=debug

# Build
ninja -C build

# Install
ninja -C build install

# Run tests (only available in non-release builds)
ninja -C build test
```

### Important Build Options

```bash
# Custom UCX installation
meson setup build -Ducx_path=/path/to/ucx

# Custom CUDA paths
meson setup build -Dcudapath_inc=/path/to/cuda/include -Dcudapath_lib=/path/to/cuda/lib64

# Enable documentation build
meson setup build -Dbuild_docs=true

# Disable GDS backend
meson setup build -Ddisable_gds_backend=true

# Static plugins (comma-separated)
meson setup build -Dstatic_plugins=UCX,GDS

# Disable test builds
meson setup build -Dbuild_tests=false

# Disable example builds
meson setup build -Dbuild_examples=false

# Enable ETCD support (requires etcd-cpp-api)
meson setup build -Detcd_inc_path=/usr/local/include -Detcd_lib_path=/usr/local/lib
```

### Testing

```bash
# Run all tests (must be non-release build)
meson setup build --buildtype=debug
ninja -C build test

# Run specific test
./build/test/gtest/nixl_gtest

# Run GoogleTest unit tests
./build/test/gtest/unit/nixl_unit_test

# Run plugin-specific tests
./build/test/unit/plugins/ucx/nixl_ucx_test
./build/test/unit/plugins/cuda_gds/nixl_gds_test
```

### Python Bindings

```bash
# Install from PyPI (preferred)
pip install nixl[cu12]  # CUDA 12
pip install nixl[cu13]  # CUDA 13

# Build from source
pip install .
meson setup build
ninja -C build
pip install build/src/bindings/python/nixl-meta/nixl-*-py3-none-any.whl

# For CUDA 13
./contrib/tomlutil.py --wheel-name nixl-cu13 pyproject.toml
meson setup build
ninja -C build
pip install build/src/bindings/python/nixl-meta/nixl-*-py3-none-any.whl
```

### Rust Bindings

```bash
# Build with Meson
meson setup build -Drust=true
ninja -C build
ninja -C build install

# Or build manually
cargo build --release
cargo test
```

## Architecture

### Core Components

1. **nixlAgent** (`src/api/cpp/nixl.h`): Main transfer object exposing NB API to applications
   - Manages backend plugin lifecycle
   - Handles memory registration/deregistration
   - Coordinates metadata exchange between agents
   - Routes transfer requests to appropriate backend plugins

2. **Plugin Manager** (`src/core/plugin_manager.h`): Discovers, loads, and manages backend plugins
   - Dynamic plugin loading from configurable directories
   - Static plugin support for performance
   - API versioning for backward/forward compatibility

3. **Backend Plugins** (`src/plugins/`): Implement SB API for specific transports
   - **UCX** (`ucx/`): Network communication, supports local/remote/notifications
   - **GDS** (`cuda_gds/`): GPUDirect Storage for GPU-to-storage transfers
   - **GDS_MT** (`gds_mt/`): Multi-threaded GDS variant
   - **POSIX** (`posix/`): POSIX AIO/io_uring file operations
   - **LibFabric** (`libfabric/`): Fabric communication (EFA support)
   - **GPUNETIO** (`gpunetio/`): DOCA-based GPU network I/O
   - **Mooncake** (`mooncake/`): Mooncake transport backend
   - **HF3FS** (`hf3fs/`): HyperFile 3 filesystem backend
   - **OBJ** (`obj/`): S3 object storage backend
   - **GUSLI** (`gusli/`): G3+ User Space Access Library for block devices

4. **Transfer Request** (`src/core/transfer_request.h`): Represents prepared transfer operations
   - Contains descriptor lists for source and destination
   - Tracks backend-specific metadata handles
   - Supports READ/WRITE operations with optional notifications

5. **Memory Descriptors** (`src/infra/mem_section.h`): Describes memory regions
   - Supports DRAM, VRAM, BLK (block), FILE, OBJ (object store)
   - Contains address, length, device ID, and backend-specific metadata

### Plugin Development

When adding a new plugin (`src/plugins/your_plugin/`):

1. **Implement SB API** (`src/api/cpp/backend/backend_engine.h`):
   - Constructor/destructor
   - Capability indicators: `supportsLocal()`, `supportsRemote()`, `supportsNotif()`, `getSupportedMems()`
   - Connection management: `connect()`, `disconnect()`, `getConnInfo()`, `loadRemoteConnInfo()`
   - Memory management: `registerMem()`, `deregisterMem()`
   - Metadata management: `getPublicData()`, `loadRemoteMD()`, `loadLocalMD()`, `unloadMD()`
   - Transfer operations: `prepXfer()`, `postXfer()`, `checkXfer()`, `releaseReqH()`
   - Notification handling: `getNotifs()`, `genNotif()` (if `supportsNotif()` returns true)

2. **Implement Plugin Manager API** (`src/api/cpp/backend/backend_plugin.h`):
   - `get_plugin_name()`: Returns plugin name
   - `get_plugin_version()`: Returns version string
   - `create_engine()`: Factory function for backend instances
   - `destroy_engine()`: Cleanup function
   - `get_backend_mems()`: Returns supported memory types
   - `get_backend_options()`: Returns configuration parameters

3. **Build Integration**:
   - Create `meson.build` in plugin directory
   - Use `subdir_done()` to skip if dependencies unavailable
   - Install as shared library to `plugin_install_dir` with `name_prefix: 'libplugin_'`
   - Add tests in `test/gtest/unit/plugins/your_plugin/` or `test/unit/plugins/your_plugin/`

4. **Documentation**:
   - Create `README.md` with overview, dependencies, build instructions, API reference, and examples
   - Update main `README.md` if adding major new functionality

### Descriptor Lists

Core abstraction for transfers:
- **Memory space**: DRAM, VRAM, BLK, FILE, OBJ
- **Transfer descriptor**: (addr, len, devID, metadata pointer)
- **Registration descriptor**: (addr, len, devID, optional byte-array string)

Device ID meanings:
- DRAM: 0 or region ID
- VRAM: GPU ID
- BLK: Volume ID
- FILE: File descriptor (str contains path + access mode)
- OBJ: Object key (str contains extended key + bucket ID)

## Code Style

### Naming Conventions
- **lowerCamelCase**: Classes, structs, unions, class members, member functions
- **snake_case**: Function arguments, local variables, enums, type aliases (with `_t` suffix)
- **Private members**: Trailing underscore (e.g., `memberName_`)

### Class Declaration Order
1. `public` section
2. `protected` section
3. `private` section

Within each section:
1. Type definitions and nested classes
2. Static member variables
3. Constructors and destructor
4. Member functions
5. Data members

### Modern C++17 Practices
- Use `auto` for obvious types or verbose template types
- RAII for all resource management
- Smart pointers for ownership (`std::unique_ptr`, `std::shared_ptr`)
- Prefer `const` correctness
- Use `override` specifier for virtual method overrides
- Prefer anonymous namespaces over `static` for file-local symbols

### Standard Library Usage
- **Prefer STL types**: `std::vector`, `std::unordered_map`, `std::optional`, `std::variant`, `std::string_view`
- **Fallback to Abseil**: `absl::StrFormat`, `absl::StatusOr`, `absl::flat_hash_map`
- **⚠️ CRITICAL**: Never expose Abseil types in public APIs (plugins, agent APIs) - only STL types

### Error Handling
- **Control-path**: Use exceptions (`std::runtime_error`, `std::invalid_argument`)
- **Data-path**: Use error codes (`nixl_status_t`) - avoid exceptions in hot paths
- **Logging levels**:
  - `NIXL_ERROR`: Critical errors preventing normal operation
  - `NIXL_WARN`: Problems not preventing operation, recoverable errors
  - `NIXL_INFO`: Normal operation information, major state changes
  - `NIXL_DEBUG`: Detailed debugging, function entry/exit, intermediate values
  - `NIXL_TRACE`: Very detailed trace, per-packet processing, low-level operations

### Code Formatting
All code must be formatted with `.clang-format`:

```bash
# Format staged changes only (recommended)
git clang-format

# Format between commits
git clang-format HEAD~1

# Format specific files
git clang-format --diff path/to/file.cpp
```

CI will reject improperly formatted code.

## ETCD Integration

NIXL supports ETCD for distributed metadata exchange and coordination:

```bash
# Environment variables
export NIXL_ETCD_ENDPOINTS="http://localhost:2379"
export NIXL_ETCD_NAMESPACE="/nixl/agents"  # Optional, default: /nixl/agents

# Running with ETCD
./build/examples/nixl_etcd_example

# nixlbench with ETCD
./build/benchmark/nixlbench/nixlbench --etcd-endpoints http://localhost:2379 --backend UCX
```

ETCD is required for:
- Multi-node communication
- Cloud-native/containerized deployments
- Dynamic agent discovery

## Key Files

### Public API Headers
- `src/api/cpp/nixl.h`: Main agent API
- `src/api/cpp/nixl_types.h`: Type definitions
- `src/api/cpp/nixl_params.h`: Configuration parameters
- `src/api/cpp/nixl_descriptors.h`: Memory descriptor types
- `src/api/cpp/backend/backend_engine.h`: SB API interface
- `src/api/cpp/backend/backend_plugin.h`: Plugin manager interface

### Core Implementation
- `src/core/plugin_manager.h`: Plugin discovery and loading
- `src/core/transfer_request.h`: Transfer request management
- `src/core/agent_data.h`: Agent internal data structures
- `src/infra/mem_section.h`: Memory section abstractions

### Utilities
- `src/utils/serdes/serdes.h`: Serialization/deserialization
- `src/utils/common/nixl_log.h`: Logging macros
- `src/utils/common/nixl_time.h`: Timing utilities
- `src/utils/ucx/ucx_utils.h`: UCX helper functions

### Examples
- `examples/cpp/`: C++ usage examples
- `examples/python/`: Python binding examples
- `benchmark/nixlbench/`: Comprehensive benchmarking tool

## Benchmarking

The `nixlbench` tool provides comprehensive performance testing:

```bash
# Build nixlbench
cd benchmark/nixlbench
meson setup build && cd build && ninja

# Basic UCX benchmark
./nixlbench --etcd-endpoints http://localhost:2379 --backend UCX --initiator_seg_type VRAM

# Storage benchmark
./nixlbench --backend GDS --filepath /mnt/storage/testfile

# S3 benchmark
./nixlbench --backend OBJ --obj_bucket_name my-bucket

# Multi-threaded with progress threads
./nixlbench --backend UCX --num_threads 4 --enable_pt --progress_threads 2
```

Communication patterns: `pairwise`, `manytoone`, `onetomany`, `tp` (tensor parallel)

## Dependencies

### Core Requirements
- C++17 compatible compiler (GCC, Clang)
- Meson build system (>= 0.64.0)
- Ninja build tool
- UCX library (tested with 1.20.x)
- CUDA Toolkit (>= 12.8 for GPU features)
- Abseil C++ library (subproject)
- Taskflow library (subproject)

### Optional Dependencies
- ETCD C++ API: For distributed coordination
- GDRCopy: For maximum UCX performance
- LibFabric: For EFA and other fabrics
- DOCA SDK: For GPUNetIO backend
- AWS SDK C++: For S3 object storage backend
- io_uring: For POSIX async I/O
- NVSHMEM: For NVSHMEM worker in nixlbench

### Python Requirements
- Python 3.12+
- pybind11
- meson, ninja
- PyTorch (for benchmarks)
- tomlkit (for build scripts)

## Development Workflow

1. **Before contributing**: Review existing issues and PRs to avoid duplicates
2. **For significant changes**: Open an issue for design discussion first
3. **Code changes**:
   - Follow code style guide in `docs/CodeStyle.md`
   - Format with `git clang-format` before committing
   - Add comprehensive tests for new features
   - Update documentation where needed
4. **Testing**: Run tests with `ninja -C build test` (debug build required)
5. **Commits**: Sign-off with DCO (`git commit -s`)
6. **Pull requests**: Fill out template completely, be responsive to feedback

Review process is thorough (2-4 rounds typical) to maintain quality, security, and performance standards.

## UCX GPU Device API

NIXL detects and uses UCX GPU Device API when available (requires UCX with GPU device support and CUDA):
- GPU-side API: `ucp/api/device/ucp_device_impl.h`
- Host-side API: `ucp/api/device/ucp_host.h`
- When available, adds `-DHAVE_UCX_GPU_DEVICE_API` compile flag

## Plugin Installation

Plugins are installed to: `<prefix>/lib/plugins/` or `<prefix>/lib/x86_64-linux-gnu/plugins/`

For dynamic loading, ensure plugin directory is in the library search path or set during runtime.

## Telemetry

NIXL includes telemetry support for performance analysis (see `docs/telemetry.md`):
- Event tracking for transfers, memory operations, and backend calls
- Configurable through agent configuration
- Can be exported for analysis

## License

Apache 2.0 License - All contributions must include SPDX headers and DCO sign-off.
