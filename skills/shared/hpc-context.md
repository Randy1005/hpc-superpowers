---
name: hpc-context
description: Shared HPC domain knowledge injected into all HPC verb skills. Covers C++, TBB, OpenMP, SYCL, CUDA C++, Taskflow, and CUDA library interfaces.
---

# HPC Context — C++ Parallel Computing Reference

Injected into all HPC verb skills to ground guidance in current library APIs, patterns, and known pitfalls.

> **Last updated:** 2026-04-07
> **Stale check:** If today's date is >7 days past "Last updated", run `/hpc-refresh-context` before proceeding.
> **Stale Signals:** See `## Stale Signals` section at the bottom for pending updates.

---

## Intel oneTBB 2021.x

> Training-knowledge snapshot through oneTBB 2021.6 (August 2025 cutoff).
> Cross-check against https://spec.oneapi.io/versions/latest/elements/oneTBB/source/index.html for the exact version in use.

### Core Parallel Algorithms

| Algorithm | API | When to use |
|-----------|-----|-------------|
| Element-wise parallel loop | `tbb::parallel_for(blocked_range<T>, body)` | Transforms, stencils, embarrassingly parallel loops |
| Loop with dynamic items | `tbb::parallel_for_each(begin, end, body)` | Graph traversal, BFS, dynamic work expansion; supports `feeder.add()` |
| Reduction | `tbb::parallel_reduce(range, identity, body, join)` | Sum, min, max, dot products |
| Reproducible reduction | `tbb::parallel_deterministic_reduce` | Same as reduce but deterministic split order |
| Prefix scan | `tbb::parallel_scan` | Exclusive prefix sums before parallel scatter |
| Sort | `tbb::parallel_sort(begin, end)` | Large arrays; not stable |
| Fixed fan-out | `tbb::parallel_invoke(f1, f2, ...)` | Two recursive halves, fixed N independent tasks |

**Partitioners:** `auto_partitioner` (default, adaptive), `simple_partitioner(grain)` (fixed), `static_partitioner` (round-robin, deterministic), `affinity_partitioner` (caches hints across repeated calls — best for repeated loops over same data).

**`parallel_pipeline`** (with `tbb::filter`) — legacy, still present but deprecated. Prefer `tbb::flow::graph` for new code.

### Task Scheduling Primitives

**`tbb::task_arena`** — isolated scheduler context with fixed thread count.
```cpp
tbb::task_arena arena(N);
arena.execute([&]{ tbb::parallel_for(...); });  // blocking; caller joins as worker
arena.enqueue(callable);                         // non-blocking submit
```
Use for: limiting TBB to N cores (co-scheduling with MPI/OpenMP), nested parallelism isolation, priority separation.

**`tbb::task_group`** — dynamic fork-join.
```cpp
tbb::task_group tg;
tg.run([&]{ /* task A */ });
tg.run([&]{ /* task B */ });
tg.wait();  // exceptions propagate here — wrap in try/catch
```

**`tbb::task_scheduler_observer`** — hooks for thread entry/exit in a `task_arena`. Override `on_scheduler_entry` / `on_scheduler_exit` for per-thread init (NUMA binding, affinity pinning). Must call `observe(true)` to activate.

**`tbb::global_control`** — process-level knobs: `max_allowed_parallelism`, `thread_stack_size`.

### Concurrent Containers

**`tbb::concurrent_vector<T>`**
- Thread-safe `push_back`, `grow_by(n)`, iteration; NO concurrent erase or positional insert.
- Never moves elements — pointers/references remain valid after push_back. Key differentiator from `std::vector`.
- `grow_by(n)` atomically reserves n slots and returns iterator range — use for bulk parallel insertion.
- `clear()` does NOT release memory; use `shrink_to_fit()` or swap-with-empty to reclaim.
- Higher per-element overhead than `std::vector` due to segment indirection.

**`tbb::concurrent_hash_map<K,V,HashCompare>`**
- Fine-grained per-bucket locking. Requires `accessor` (write) or `const_accessor` (read) — lock held until accessor destructs.
- Pitfall: holding an accessor while calling another map op on the same key = deadlock.

**`tbb::concurrent_unordered_map<K,V>`**
- Lock-free insert+find (striped internal locking). No concurrent erase.
- Faster than `concurrent_hash_map` for read-heavy; no `accessor` needed.

**`tbb::concurrent_queue<T>` / `tbb::concurrent_bounded_queue<T>`**
- Unbounded: lock-free push/try_pop. Bounded: blocking push/pop, supports backpressure.
- `size()` is approximate — do not use for exact checks.

**`tbb::concurrent_priority_queue<T>`**
- Coarse internal lock — does not scale beyond ~8–16 threads.
- For high-contention priority work: use level-banded `concurrent_vector` arrays instead.

### Enumerable Thread Specific Storage

```cpp
tbb::enumerable_thread_specific<int> ets(0);
tbb::parallel_for(..., [&](auto& range){
    auto& local = ets.local();
    for (auto x : range) local += x;
});
int total = ets.combine([](int a, int b){ return a + b; });
```
**Primary tool for per-thread accumulators** — eliminates false sharing and `concurrent_vector` push_back contention.

### Flow Graph

Dataflow model for complex dependency graphs. Key nodes:

| Node | Purpose |
|------|---------|
| `function_node<In,Out>` | Maps In→Out; configurable concurrency (`unlimited`, `serial`, N) |
| `continue_node<Out>` | AND-join: fires when all predecessors complete |
| `join_node<tuple<...>>` | Waits for one token from each input port |
| `broadcast_node<T>` | Fans out to all successors |
| `sequencer_node<T>` | Re-orders out-of-order tokens by sequence number |
| `limiter_node<T>` | Backpressure: limits in-flight tokens |
| `input_node<T>` | Source (replaces deprecated `source_node`) |

Overhead is high — avoid for simple fork-join. Use `parallel_invoke` or `task_group` instead.

### Memory Allocation

**`tbb::scalable_allocator<T>`** — drop-in STL allocator backed by TBB's per-thread memory pool. Avoids global heap lock. Requires `-ltbbmalloc`. Cannot mix with system `free()`.

**`tbb::cache_aligned_allocator<T>`** — allocates on cache-line boundaries (64 bytes). Prevents false sharing for per-thread data.

**`tbbmalloc_proxy`** — `LD_PRELOAD` to replace global `malloc`/`free`/`new`/`delete` with scalable allocator. No code changes needed.

### Breaking Changes: Legacy TBB → oneTBB 2021

| Removed | Replacement |
|---------|-------------|
| `tbb::task` low-level API (`allocate_root`, `spawn`, `wait_for_all`) | `task_group`, `parallel_invoke` |
| `tbb::task_scheduler_init` | `tbb::global_control`, `tbb::task_arena` |
| `tbb::flow::source_node` | `tbb::flow::input_node` |
| `tbb::atomic<T>` | `std::atomic<T>` |
| `tbb::tick_count` | `std::chrono` |

**Any code using `tbb::task::allocate_root()` / `spawn()` is broken on oneTBB 2021+ and must be rewritten.**

Namespace: `oneapi::tbb::` is canonical; `tbb::` remains as alias.

### Pitfalls

- **Oversubscription:** mixing TBB + OpenMP + pthreads without arena coordination → use `task_arena` with explicit thread counts.
- **False sharing:** per-thread counters in `std::vector<int>` indexed by thread ID → use `enumerable_thread_specific` or `cache_aligned_allocator`.
- **`parallel_for` grain too fine:** grain=1 on tight inner loop defeats work stealing → use `auto_partitioner` or `grain = max(1, N/4/num_threads)`.
- **Nested parallelism:** `parallel_for` inside `parallel_for` body is fine but can cause priority inversion → use `task_arena::isolate()` to prevent inner tasks stealing outer.
- **`concurrent_vector::clear()` memory not released** → call `shrink_to_fit()` or swap-with-empty.
- **`concurrent_hash_map` accessor held too long** → deadlock on same-key access.
- **Flow graph for simple fork-join** → 5–20× higher overhead than `parallel_invoke`; avoid.
- **Ignoring `tbbmalloc`:** `std::make_unique` inside `parallel_for` serializes on global heap → link `libtbbmalloc`.

### Quick Reference

| Scenario | API |
|----------|-----|
| Parallel loop, known range | `parallel_for` + `blocked_range` |
| Loop with dynamic items | `parallel_for_each` + feeder |
| Reduction | `parallel_reduce` |
| Parallel sort | `parallel_sort` |
| Fixed N independent tasks | `parallel_invoke` |
| Dynamic fork-join | `task_group` |
| Dataflow pipeline | `tbb::flow::graph` |
| Per-thread accumulation | `enumerable_thread_specific` |
| Parallel results collection | `concurrent_vector` + `grow_by` |
| High-contention priority queue | Level-banded `concurrent_vector` arrays |
| Scalable heap allocation | `scalable_allocator` or `tbbmalloc_proxy` |
| False-sharing prevention | `cache_aligned_allocator` |
| Core count limiting | `task_arena` + `global_control` |
| Thread entry/exit hooks | `task_scheduler_observer` |

---

## CUDA C++ (CUDA 12.x)

> Training-knowledge snapshot through CUDA 12.x (August 2025 cutoff).
> Authoritative source: https://docs.nvidia.com/cuda/cuda-programming-guide/

### Thread Hierarchy

```
Grid → Blocks → Warps (32 threads) → Threads
```

| Built-in | Meaning |
|----------|---------|
| `threadIdx.{x,y,z}` | Thread index within block |
| `blockIdx.{x,y,z}` | Block index within grid |
| `blockDim.{x,y,z}` | Block dimensions |
| `gridDim.{x,y,z}` | Grid dimensions |
| `warpSize` | Always 32; use this, not a hardcoded literal |

**Warp:** 32 threads share a program counter (SIMT). Divergent branches execute all taken paths serially and reconverge. Warp-level primitives (`__shfl_sync`, `__ballot_sync`, `__reduce_add_sync`) require an `unsigned mask` — pass `0xffffffff` for full warp.

**Grid-stride loop (canonical pattern):**
```cpp
__global__ void kernel(float* a, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    for (int i = idx; i < n; i += blockDim.x * gridDim.x)
        process(a[i]);
}
```

### Memory Model

| Memory | Location | Scope | Latency | Notes |
|--------|----------|-------|---------|-------|
| Register | On-chip | Thread | 1 cycle | Spills → local memory (global) |
| Local | DRAM | Thread | ~600 cycles | Avoid large local arrays |
| Shared | On-chip SRAM | Block | ~5 cycles | 32 banks, 4B each; bank conflicts serialize |
| Global | DRAM | Device/Host | ~600 cycles | Coalesced access critical |
| Constant | DRAM + cache | All | ~5 cycles (hit) | Optimized for broadcast reads |
| Texture | DRAM + cache | All | — | Spatial 2D locality; use object API |
| L2 | On-chip | Device | — | Pin ranges with `cudaAccessPolicyWindow` |

**Shared memory bank conflicts:** 32 banks, 4B wide, cycling modulo 32. N threads accessing `base + k*N` hit same bank if N divides 32. Fix: pad arrays (`float smem[N][33]`).

**Coalescing:** consecutive threads → consecutive 128B-aligned addresses = single transaction. AoS → SoA for coalesced layouts.

**Dynamic shared memory:** `extern __shared__ float smem[];` + pass size as third launch param. Always `__syncthreads()` between write and read phases.

### Unified Memory

```cpp
cudaMallocManaged(&ptr, size);                             // accessible from CPU + GPU
cudaMemPrefetchAsync(ptr, size, deviceId, stream);         // prefetch to GPU before kernel
cudaMemPrefetchAsync(ptr, size, cudaCpuDeviceId);          // prefetch back to CPU
cudaMemAdvise(ptr, size, cudaMemAdviseSetReadMostly, dev);
cudaMemAdvise(ptr, size, cudaMemAdviseSetPreferredLocation, dev);
```
- Page-fault driven migration (Pascal+). First access incurs fault + migration.
- Always `cudaDeviceSynchronize()` before CPU reads UM data post-kernel.
- Oversubscription (UM > device memory) works but thrashes — avoid.
- Without prefetch, first-touch latency dominates.

### Streams and Concurrency

```cpp
cudaStream_t s;
cudaStreamCreate(&s);
cudaMemcpyAsync(d_ptr, h_ptr, size, cudaMemcpyHostToDevice, s);  // host must be pinned
kernel<<<grid, block, 0, s>>>(args);
cudaMemcpyAsync(h_out, d_out, size, cudaMemcpyDeviceToHost, s);
```
**Host memory must be pinned** (`cudaMallocHost`) for truly async transfers.

Cross-stream dependencies via events:
```cpp
cudaEvent_t e; cudaEventCreate(&e);
cudaEventRecord(e, s1);
cudaStreamWaitEvent(s2, e, 0);  // s2 waits for e before proceeding
```

**CUDA Graphs (10+):** Capture stream ops into a graph, instantiate once, launch repeatedly with near-zero overhead. Critical when kernels are short.
```cpp
cudaStreamBeginCapture(stream, cudaStreamCaptureModeGlobal);
// ... enqueue ops ...
cudaStreamEndCapture(stream, &graph);
cudaGraphInstantiate(&graphExec, graph, nullptr, nullptr, 0);
cudaGraphLaunch(graphExec, stream);
```

Avoid `cudaDeviceSynchronize()` in inner loops — use events and stream ordering instead.

### Occupancy and Launch Configuration

```cpp
kernel<<<gridDim, blockDim, sharedMemBytes, stream>>>(args);
```

Occupancy = active warps per SM / max warps per SM. Limiters: registers/thread, shared mem/block, max threads/SM, max blocks/SM.

```cpp
int minGridSize, blockSize;
cudaOccupancyMaxPotentialBlockSize(&minGridSize, &blockSize, kernel, sharedMem, 0);
```

**`__launch_bounds__(maxThreads, minBlocks)`** — caps register allocation to achieve target occupancy.

Rules of thumb:
- Block size must be multiple of 32. Common: 128, 256, 512.
- Avoid 1024 threads/block unless necessary — limits blocks per SM.
- Inspect register/spill counts: `nvcc --ptxas-options=-v`.
- Integer overflow: cast `blockIdx.x * blockDim.x` to `size_t` for large grids.

### Cooperative Groups

```cpp
#include <cooperative_groups.h>
namespace cg = cooperative_groups;

auto block = cg::this_thread_block();  block.sync();   // == __syncthreads()
auto warp  = cg::tiled_partition<32>(block);           warp.sync();
auto tile4 = cg::tiled_partition<4>(block);            tile4.shfl(val, 0);

// Grid-wide barrier (requires cooperative launch)
auto grid = cg::this_grid();  grid.sync();
cudaLaunchCooperativeKernel((void*)kernel, gridDim, blockDim, args, smem, stream);
// Constraint: all blocks must fit simultaneously on device
```

Collective ops: `tile.reduce(val, cg::plus<int>())`, `tile.scan(...)`, `tile.shfl(src, lane)`, `tile.shfl_down(src, delta)`.

Prefer over hand-rolled warp-shuffle patterns.

### CUDA Libraries

**cuBLAS** — dense linear algebra (GEMM, GEMV, batched ops).
- `cublasSgemm`, `cublasDgemm`, `cublasGemmEx` (mixed precision).
- cuBLASLt (10.2+): flexible GEMM with epilogue fusion (bias, relu) — preferred for transformer workloads.
- One `cublasHandle_t` per thread/stream; not thread-safe to share.

**cuDNN** — DNN primitives (convolution, pooling, normalization, attention).
- v8+ backend/graph API replaces legacy `cudnnConvolution*` calls; supports kernel fusion.
- `NHWC` layout preferred on Volta+.
- Query workspace size before allocating: `cudnnGetConvolutionForwardWorkspaceSize`.

**cuSPARSE** — sparse linear algebra (SpMV, SpMM, sparse triangular solve).
- Generic API (10.1+): `cusparseSpMV`, `cusparseSpMM` with descriptor objects.
- Formats: CSR, CSC, COO, BSR, ELL.
- Always query buffer size before calling the compute function.

**Thrust** — high-level C++ parallel algorithms.
```cpp
thrust::sort(thrust::device, d_vec.begin(), d_vec.end());
thrust::reduce(thrust::cuda::par.on(stream), begin, end, 0.0f, thrust::plus<float>());
thrust::device_ptr<float> p(raw_cuda_ptr);  // wraps cudaMalloc pointer
```

**CUB** — block/device-level cooperative primitives for use inside kernels.
```cpp
__shared__ typename cub::BlockReduce<float, 256>::TempStorage temp;
float blockSum = cub::BlockReduce<float, 256>(temp).Sum(myVal);
```

**NCCL** — multi-GPU/multi-node collectives (AllReduce, Broadcast, etc.).

### Deprecations (CUDA 11–12.x)

| Deprecated | Replacement |
|------------|-------------|
| cuSPARSE legacy API (`cusparseScsrmv`, etc.) | Generic API (`cusparseSpMV`) |
| cuDNN legacy convolution API | cuDNN v8 graph/backend API |
| Texture reference API | Texture object API (`cudaTextureObject_t`) — removed in 12.0 |
| Surface reference API | Surface object API |
| Compute capability < 3.5 (Kepler) | Dropped in CUDA 12.0 |

### Pitfalls

- **Missing `__syncthreads()`** between shared mem write and read phases → race condition.
- **`__syncthreads()` in divergent code** (not all threads reach it) → undefined behavior.
- **Non-pinned host memory with `cudaMemcpyAsync`** → falls back to synchronous copy, no overlap.
- **UM without prefetch** → high first-touch latency from page faults.
- **CPU reads UM after kernel without `cudaDeviceSynchronize()`** → stale data.
- **Bank conflicts** → pad shared arrays by 1 (`[N][33]` instead of `[N][32]`).
- **Warp divergence** → restructure data to make branches warp-uniform; use predication.
- **Strided global access** → convert AoS to SoA for coalesced reads.
- **Integer overflow in index** → cast to `size_t` for large grids.
- **`cudaDeviceSynchronize()` in inner loops** → serialize all concurrency; use events instead.
- **`__shfl_sync` without correct mask** → undefined for inactive lanes; use `0xffffffff` for full warp.
- **Dynamic parallelism:** child kernels not visible to parent until device `cudaDeviceSynchronize()`; memory allocated by child must be freed explicitly from device.

### Performance Checklist

```
[ ] Accesses coalesce? (consecutive threads → consecutive 128B addresses)
[ ] Shared memory bank conflicts eliminated? (padding, transposed access)
[ ] Warp divergence minimized?
[ ] Occupancy checked? (registers, shared mem, block size)
[ ] __launch_bounds__ set if register-limited?
[ ] Host memory pinned for async transfers?
[ ] Streams overlap H2D / kernel / D2H?
[ ] UM prefetched before kernels?
[ ] Atomics replaced with reduction trees where possible?
[ ] No cudaDeviceSynchronize() in inner loops?
[ ] No large local arrays (register spill)?
[ ] SoA layout instead of AoS?
[ ] CUDA Graph used for repeatedly-launched short kernels?
```

---

## Taskflow v3.x

> Training-knowledge snapshot, accurate through Taskflow v3.6–v3.7 (August 2025 cutoff).
> Cross-check against https://taskflow.github.io/taskflow/ for the exact version in use.

### Core Task Graph Constructs

**`tf::Taskflow`** — a directed acyclic graph (DAG) of tasks. Reusable across runs.

```cpp
tf::Taskflow tf;
auto [A, B, C, D] = tf.emplace(
    []{ /* A */ }, []{ /* B */ }, []{ /* C */ }, []{ /* D */ }
);
A.precede(B, C);   // A → B, A → C
D.succeed(B, C);   // B → D, C → D
```

**`tf::Executor`** — owns the thread pool. Create once per application; never per task.

```cpp
tf::Executor executor;              // hardware_concurrency threads
tf::Executor executor(8);           // explicit thread count
executor.run(tf).wait();            // run and block
executor.run_n(tf, 3).wait();       // run N times
executor.run_until(tf, pred).wait();
executor.wait_for_all();
```

**`tf::Task`** — lightweight handle to a graph node. `.precede()`, `.succeed()`, `.name()`, `.type()`.

### Task Types

| Type | Callable signature | Use when |
|------|--------------------|----------|
| Static | `[]{}` | Most tasks |
| Dynamic (Subflow) | `[](tf::Subflow& sf){}` | Generate work at runtime |
| Condition | `[]() -> int {}` | Branches, loops without external flags |
| Module | `tf.composed_of(inner_tf)` | Reuse a Taskflow as an atomic unit |

**Subflow:** `sf.detach()` runs children async without joining before parent's successors — only use if children's output is not needed downstream.

**Condition:** return value selects successor index (0-based). First pipe of a pipeline must be SERIAL; condition tasks must return in `[0, num_successors)`.

### Parallel Algorithms

All return a `tf::Task` and accept a partitioner as last arg.

```cpp
tf.for_each(begin, end, [](T& x){ ... });
tf.for_each_index(0, N, 1, [&](int i){ ... });
tf.reduce(begin, end, init, [](T a, T b){ return a+b; });
tf.transform_reduce(begin, end, init, reduce_op, transform_op);
tf.sort(begin, end);
tf.transform(in_begin, in_end, out_begin, [](T x){ return ...; });
tf.find_if(begin, end, result, pred);   // v3.3+
tf.min_element(begin, end, result);     // v3.3+
```

**Partitioners:** `tf::GuidedPartitioner` (default), `tf::StaticPartitioner(chunk)`, `tf::DynamicPartitioner(chunk)`.

### Pipeline

```cpp
tf::Pipe p1{tf::PipeType::SERIAL,   [](tf::Pipeflow& pf){ if (done) pf.stop(); }};
tf::Pipe p2{tf::PipeType::PARALLEL, [](tf::Pipeflow& pf){ /* transform */ }};
tf::Pipeline pl(/*max_tokens=*/4, p1, p2, p3);
tf::Task t = tf.composed_of(pl);
```

- First pipe must be `SERIAL`; call `pf.stop()` when input is exhausted.
- `pf.token()` — zero-based token index.
- `pf.input<T>()` / `pf.output<T>()` — typed per-token slot (same slot, overwritten in place).
- `tf::ScalablePipeline` (v3.3+) — pipe count determined at runtime.
- Set max_tokens ≤ num_workers to avoid spinner waste.

### Executor Configuration

```cpp
// CPU affinity via WorkerInterface (v3.3+)
struct PinWorkers : tf::WorkerInterface {
    void scheduler_prologue(tf::Worker& w) override {
        // pin w.id() to a core via pthread_setaffinity_np
    }
};
tf::Executor exec(8, std::make_shared<PinWorkers>());
```

Work stealing always enabled (structural). Control granularity via partitioners and task fan-out.

### CUDA Support

**`tf::cudaFlow`** — builds CUDA Graph explicitly.

```cpp
tf.emplace([&](tf::cudaFlow& cf){
    auto h2d = cf.memcpy(d_in, h_in, N * sizeof(float));
    auto k   = cf.kernel(grid, block, 0, my_kernel, d_in, d_out, N);
    auto d2h = cf.memcpy(h_out, d_out, N * sizeof(float));
    h2d.precede(k); k.precede(d2h);
});
```

**`tf::cudaFlowCapturer`** — records library calls (cuBLAS, cuDNN) via stream capture.

```cpp
tf.emplace([&](tf::cudaFlowCapturer& cap){
    auto gemm = cap.on([&](cudaStream_t s){ cublasSgemm(handle, ..., s); });
});
```

**GPU parallel algorithms (v3.2+):** `cf.for_each`, `cf.reduce`, `cf.sort`, `cf.transform` — compile to CUDA kernels, appear as `cudaTask` nodes.

### Heterogeneous CPU+GPU DAG

```cpp
tf::Task cpu_prep   = tf.emplace([&]{ prepare(h_input, N); });
tf::Task gpu_kernel = tf.emplace([&](tf::cudaFlow& cf){ /* ... */ });
tf::Task cpu_post   = tf.emplace([&]{ analyze(h_output, N); });
cpu_prep.precede(gpu_kernel);
gpu_kernel.precede(cpu_post);
```

Multi-GPU: call `cudaSetDevice(n)` inside each cudaFlow lambda.

### When to Use Taskflow vs TBB

| Prefer Taskflow | Prefer TBB |
|-----------------|------------|
| Complex CPU+GPU DAGs | Fine-grained recursive tasks (µs scale) |
| Static DAGs run many times (compiled once) | NUMA-aware thread pool isolation (`task_arena`) |
| Conditional loops inside the graph | Concurrent containers (`concurrent_vector`, etc.) |
| Pipeline with typed token slots | Mature algorithms with well-tested edge cases |

### Key API Changes by Version

| Version | Change |
|---------|--------|
| v3.1 | `parallel_for` → `for_each` (old name removed) |
| v3.2 | CUDA parallel algorithms added to `cudaFlow` |
| v3.3 | `WorkerInterface`, `ScalablePipeline`, `find_if`/`min_element`/`max_element` |
| v3.4 | `tf::AsyncTask` for fire-and-forget without a graph |
| v3.5–3.6 | `executor.async()`, `executor.silent_async()`, `tf::corun` |
| v3.7 | `tf::Runtime` + `corun` for inline recursive scheduling (preferred over nested `executor.run()`) |

### Pitfalls

- **Dangling captures:** tasks run async — capture by value or ensure lifetime.
- **Modifying graph while running:** UB — always `wait_for_all()` first.
- **Nested `executor.run().wait()` inside a task:** deadlock risk — use `tf::Runtime::corun()` instead (v3.7+).
- **Condition task out-of-range return:** UB — ensure return ∈ `[0, num_successors)`.
- **New `Taskflow` per hot-loop iteration:** expensive — reuse the graph object.
- **`Subflow::detach()` with dependent successors:** data race — only detach truly independent subflows.
- **Pipeline first pipe = PARALLEL:** unsupported, UB.
- **Forgetting `pf.stop()`:** pipeline runs forever.
- **Shared mutable state in parallel tasks:** use `std::atomic` or reduce pattern.
- **Taskflow does not provide concurrent containers:** use TBB's `concurrent_vector` alongside Taskflow.

---

## OpenMP

*Content to be added in a future update.*

---

## SYCL

*Content to be added in a future update.*

---

## Stale Signals

*No signals yet. Scheduled agent will append findings here weekly.*