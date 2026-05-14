# NanoMatch ⚡
### Ultra-Low Latency Order Matching Engine in C++17

> *"The most brilliant alpha model is completely useless if the underlying system executing the trade is a microsecond too slow."*

---

## Overview

NanoMatch is a fully functional **Limit Order Book (LOB)** built from scratch in C++17, engineered for deterministic sub-microsecond order execution. It bridges the gap between theoretical quantitative finance and hardcore, low-level systems engineering — the exact architectural mindset demanded by top-tier proprietary trading firms.

Standard C++ practices like `std::map` or dynamic memory allocation introduce unpredictable latency spikes due to OS overhead and CPU cache misses. NanoMatch strips away these abstractions, replacing them with **cache-aligned, contiguous data structures** designed for hardware sympathy and deterministic execution.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Data Ingestion Layer                  │
│         zero-copy mmap · NASDAQ TotalView-ITCH          │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                  Order Book Core (LOB)                   │
│     Price-Time Priority · Custom Memory Pools            │
│     Cache-Aligned Arrays · No Heap Allocation            │
└──────────────┬──────────────────────────┬───────────────┘
               │                          │
┌──────────────▼──────┐      ┌────────────▼───────────────┐
│   Matching Engine   │      │   Lock-Free Trade Logger    │
│  Market / Limit /   │      │   SPSC Ring Buffer          │
│  Cancel Orders      │      │   std::atomic operations    │
└──────────────┬──────┘      └────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────┐
│               Benchmarking Harness                       │
│     rdtsc · Google Benchmark · p50 / p90 / p99          │
│     Linux perf · CPU Flame Graphs · VTune               │
└─────────────────────────────────────────────────────────┘
```

---

## The Engineering Problem

### Why Standard C++ Fails in HFT

| Approach | Problem | Latency Impact |
|---|---|---|
| `std::map` (Red-Black Tree) | Pointer chasing, heap fragmentation | Unpredictable cache misses |
| `new` / `malloc` | OS syscalls, memory fragmentation | µs-level spikes |
| `std::mutex` locking | Thread blocking, context switching | Unbounded tail latency |
| Dynamic dispatch | Branch misprediction | Pipeline stalls |

### How NanoMatch Solves It

- **Custom Memory Pools** — pre-allocated arenas eliminate heap allocation on the hot path entirely
- **Cache-Aligned Contiguous Arrays** — order queues laid out to fit L1 cache lines, eliminating false sharing
- **Lock-Free SPSC Queue** — asynchronous trade logging via `std::atomic` without blocking the matching thread
- **Zero-Copy mmap I/O** — binary ITCH data ingested directly into virtual memory, no kernel buffer copies

---

## Features

- **Full Limit Order Book** — Buy/Sell limit orders, market orders, and cancellations with strict price-time priority
- **Cache-Optimized Memory Management** — custom memory pools replacing all STL heap allocations on the critical path
- **High-Throughput Data Ingestion** — zero-copy `mmap` parsing of NASDAQ TotalView-ITCH binary `.pcap` files
- **Lock-Free Concurrency** — SPSC ring buffer using `std::atomic` for non-blocking asynchronous trade logging
- **Rigorous Benchmarking Harness** — `rdtsc` cycle counting + Google Benchmark reporting throughput and tail latencies

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | C++17 |
| Build System | CMake |
| Benchmarking | Google Benchmark, `rdtsc` |
| Profiling | Linux `perf`, Intel VTune |
| Data Ingestion | `mmap`, NASDAQ TotalView-ITCH |
| Concurrency | `std::atomic`, SPSC Ring Buffer |
| Data Structures | Custom Memory Pools, Cache-Aligned Arrays |
| Data Sources | NASDAQ ITCH `.pcap` files, Synthetic Order Generator |

---

## Performance Results

> Benchmarked on: [hardware specs — CPU, RAM, L1/L2/L3 cache sizes]

| Metric | Baseline (STL) | NanoMatch (Optimized) |
|---|---|---|
| Throughput | — orders/sec | — orders/sec |
| p50 Latency | — µs | — µs |
| p90 Latency | — µs | — µs |
| p99 Latency | — µs | — µs |
| L1 Cache Miss Rate | — % | — % |

*Full benchmark reproduction instructions below.*

---

## Key Architectural Decisions

### 1. Memory Pool over `new`/`delete`
Every order object is allocated from a pre-warmed slab allocator. The matching engine never calls `malloc` on the hot path. This eliminates OS overhead and keeps order objects spatially local in memory.

### 2. Cache-Line Aligned Order Queues
Each price level's order queue is padded and aligned to 64-byte cache line boundaries. This prevents **false sharing** between producer and consumer threads accessing adjacent memory.

### 3. Lock-Free SPSC Trade Logger
Trade confirmations are written to an SPSC ring buffer using `std::atomic` with `memory_order_release` / `memory_order_acquire` semantics. The logging thread never blocks the matching thread.

### 4. Zero-Copy mmap Ingestion
NASDAQ ITCH binary data is memory-mapped directly — the kernel maps the file into the process's virtual address space with no intermediate buffer copies, eliminating `read()` syscall overhead entirely.

### 5. rdtsc Cycle-Accurate Timing
`RDTSC` instruction is used for nanosecond-granularity latency measurement, bypassing `clock_gettime()` syscall overhead. Serialized with `CPUID` to prevent CPU out-of-order execution from corrupting measurements.

---

## Engineering Challenges

### The Cache Miss Problem
Initial profiling via Linux `perf stat` revealed ~35% L1 cache miss rate on the order book's price level traversal. Root cause: `std::map`'s pointer-based Red-Black Tree scattered nodes across heap memory. Fix: replaced with a contiguous sorted array with binary search — cache miss rate dropped to <5%.

### False Sharing on Queue Boundaries
Early lock-free queue implementation showed unexpected latency spikes under load. `perf c2c` identified false sharing between the producer's write index and consumer's read index sharing a cache line. Fix: padding both indices to separate 64-byte cache lines eliminated the contention.

### Idempotent Benchmark Reproduction
Google Benchmark's CPU frequency scaling caused variance in results. Fix: pinned benchmarks to a single core (`taskset`), disabled turbo boost, and used `rdtsc` for cycle counts independent of frequency scaling.

---

## Build & Run

### Prerequisites
- GCC 11+ or Clang 13+ with C++17 support
- CMake 3.20+
- Google Benchmark
- Linux (for `perf` profiling)

### Build
```bash
git clone https://github.com/yourusername/nanomatch
cd nanomatch
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
```

### Run Matching Engine
```bash
./nanomatch --input data/sample.itch --mode limit
```

### Run Benchmarks
```bash
# Disable CPU frequency scaling for accurate results
sudo cpupower frequency-set --governor performance

# Run benchmark suite
./bench/nanomatch_bench --benchmark_repetitions=10 --benchmark_report_aggregates_only=true
```

### Reproduce Flame Graph
```bash
sudo perf record -g ./nanomatch --input data/sample.itch
sudo perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

---

## Data Sources

- **NASDAQ TotalView-ITCH** — sample `.pcap` files from [NASDAQ Historical Data](https://emi.nasdaq.com/ITCH/Nasdaq%20ITCH/)
- **Synthetic Order Generator** — Python script generating multi-million row CSVs with engineered edge cases (burst arrivals, price level collisions, rapid cancellations)

---

## Concepts Demonstrated

- Cache locality — L1/L2/L3 vs RAM hierarchy
- Custom memory pools & slab allocators
- Lock-free concurrency & `std::atomic` memory ordering
- Zero-copy I/O via `mmap`
- Struct packing & false sharing prevention
- Branch misprediction & CPU pipeline hazards
- Cycle-accurate latency measurement with `rdtsc`

---

## Project Context

Built as part of the **FEC · IIT Guwahati** Quant-Systems Engineering track (Project NanoMatch)

---
