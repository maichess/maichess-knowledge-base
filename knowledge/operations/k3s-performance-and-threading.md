# K3s Cluster — Performance & Threading

**Status:** Reference / known issues + applied fixes (as of 2026-06-11)
**Relates to:** [deployment-and-environments.md](deployment-and-environments.md)

## Problem: pods appeared single-threaded despite multi-core nodes

Observed symptom: all services appeared to use only one CPU core under load, even on the 16-core
`maichess-mega` node. Profiling showed each pod's event loops and thread pools were sized to 1.

### Root cause — JVM services (engine-service, move-validator-service)

JVM 8u191+ enables **container-awareness** (`-XX:+UseContainerSupport`) by default. It reads the
cgroup CPU quota from the container's cgroup and uses it as `Runtime.availableProcessors()`. With
a `cpu: "1"` limit the JVM sees one processor and creates correspondingly tiny thread pools:

- **Netty gRPC event loop** — defaults to `2 × availableProcessors()`, so 2 threads.
- **ZIO fiber executor** — defaults to `availableProcessors()` threads.
- **Result:** concurrent gRPC requests queue behind a single Netty thread; ZIO fibers serialize.

With `cpu: 500m` (move-validator's previous limit) the effect is the same: the JVM rounds down
to 1 processor.

### Root cause — .NET services (match-manager, bot-arena, analysis, match-maker)

On modern Linux kernels with **cgroup v2** and **containerd**, `Environment.ProcessorCount`
follows the cgroup CPU quota. With `cpu: 500m` it returns 1. The .NET ThreadPool minimum thread
count defaults to `ProcessorCount`, so I/O-bound workloads that spawn many async continuations
can saturate the (1-thread) pool minimum and stall.

### Fix applied in `maichess-deploy/helm/maichess/values.yaml` (2026-06-11)

**JVM services — `-XX:ActiveProcessorCount=N`**

`ActiveProcessorCount` is an explicit JVM override that bypasses the cgroup detection. It tells
the JVM to create N-sized thread pools regardless of what the cgroup reports. The pods still
receive their cgroup CPU allocation — this does not bypass scheduling, it only stops the JVM from
under-sizing its pools.

```yaml
engineService:
  resources:
    limits:
      cpu: "2"         # raised from "1" to give the search threads real quota
      memory: "2Gi"
  env:
    JAVA_TOOL_OPTIONS: "-XX:MaxRAMPercentage=75.0 -XX:ActiveProcessorCount=4"

moveValidatorService:
  resources:
    limits:
      cpu: "1"         # raised from 500m
      memory: "512Mi"
  env:
    JAVA_TOOL_OPTIONS: "-XX:ActiveProcessorCount=4"
```

**Why 4 and not the node's full 16?** The value should be proportional to the CPU *limit*, not
the node size. The engine pod has a 2-CPU limit; setting 4 threads means the JVM is slightly
overcommitted within the cgroup but has headroom for burst. Tune this value up if profiling shows
queuing on the executor.

**Caution:** `ActiveProcessorCount` also affects the G1GC parallel GC thread count. Setting it
too high (e.g., 16) on a 2-CPU limit causes GC threads to contend with each other and the
application threads. Stay ≤ 2× the CPU limit.

**.NET services — `DOTNET_THREADPOOL_MINTHREADS`**

Sets the ThreadPool minimum thread count as an environment variable. Overrides the
`ProcessorCount`-derived default.

```yaml
matchMakerService:
  env:
    DOTNET_THREADPOOL_MINTHREADS: "4"

botArenaService:
  env:
    DOTNET_THREADPOOL_MINTHREADS: "4"

analysisService:
  env:
    DOTNET_THREADPOOL_MINTHREADS: "4"
```

These services are I/O-bound (gRPC, MongoDB, Redis) so a minimum of 4 threads allows them to
sustain concurrent async continuations without stalling on the ThreadPool injection delay.

## Further optimization opportunities

### JVM thread pool tuning

- The ZIO runtime in engine-service and move-validator-service creates a fixed pool sized to
  `availableProcessors()`. After the `ActiveProcessorCount` fix this is now 4 threads, which is
  reasonable for 2 CPU cores. If engine-service is still bottlenecked under heavy load, consider
  explicitly sizing the ZIO runtime:
  ```scala
  override val bootstrap: ZLayer[Any, Nothing, Unit] =
    Runtime.setExecutor(Executor.makeDefault(8))
  ```
  Read the ZIO docs on blocking vs non-blocking executors before changing this.
- Netty's event loop group in gRPC-Java can also be sized explicitly via
  `NettyServerBuilder.workerEventLoopGroup(new NioEventLoopGroup(N))`.

### .NET ThreadPool max threads

`DOTNET_THREADPOOL_MINTHREADS` only sets the floor. The ThreadPool max is `32767` by default;
CPU-bound work can create too many threads. For CPU-heavy .NET services (none currently — all are
I/O-bound), also set `DOTNET_THREADPOOL_MAXTHREADS`.

### Prometheus metrics to watch

- `jvm_threads_current` / `jvm_threads_peak` — confirms pool sizing is effective.
- `grpc_server_handled_total` + latency histograms — confirms requests are not queuing.
- .NET: `dotnet_threadpool_queue_length` (if OTEL metrics are enabled) — confirms no ThreadPool
  starvation.

### Known remaining gap

- **match-manager-service** is not listed above — it was not given `DOTNET_THREADPOOL_MINTHREADS`
  yet. Add it if match-manager starts showing threading issues under load (same pattern as the
  other three .NET services).

## Background: cgroup v2 detection differences by runtime

| Runtime | How it detects CPU count | Override |
|---|---|---|
| JVM (8u191+) | `cfs_quota_us / cfs_period_us` via cgroup v1 or v2 | `-XX:ActiveProcessorCount=N` |
| .NET 5+ | `sched_getaffinity` or cgroup v2 `cpu.max` | `DOTNET_THREADPOOL_MINTHREADS` env var |
| Node.js | `os.cpus()` → `/proc/cpuinfo` (ignores cgroup quota) | N/A — not affected |
| Go | `GOMAXPROCS` defaults to `runtime.NumCPU()` (ignores quota unless autopilot used) | `GOMAXPROCS=N` env var |
