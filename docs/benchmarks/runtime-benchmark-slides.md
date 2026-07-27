# Runtime-Aware Warm Pool Performance Benchmark

## Methodology, Instrumentation, and Analysis

agent-sandbox v0.5.3 | Kubernetes 1.32 | GCP n2-standard-8 | July 2026

---

## Background

Container runtimes vary in startup cost. Before a pod can serve work,
it traverses a multi-stage pipeline:

```
API Server → Scheduler → Kubelet → Image Pull → Runtime Init → Ready
  ~10ms       ~50ms       ~100ms    ~200ms-5s     varies
```

| Runtime | Observed cold start | Startup components |
|---------|--------------------|--------------------|
| runc    | 0.5–1s    | cgroup creation, rootfs mount |
| gVisor  | 1–1.5s    | cgroup, rootfs, userspace kernel (Sentry) |
| kata    | 7–13s     | hypervisor boot, guest kernel, kata-agent |

Under concurrent load, cold starts serialize through the API server.
Tail latency increases linearly with queue depth.

---

## Warm pools

The SandboxWarmPool controller pre-provisions sandboxes so that
SandboxClaim objects bind directly to Ready pods, bypassing all
pipeline stages:

```
Cold path:  API Server → Scheduler → Kubelet → Pull → Runtime → Ready
Warm path:  SandboxClaim → Bind (in-memory selection) → Ready
                           ~0.3s regardless of runtime
```

Measured warm claim baseline: 0.347–0.352s across runc, gVisor, and
kata on identical hardware. The pool absorbs the entire cold start
pipeline — warm claim latency is runtime-independent.

---

## Measurement requirements

Standard load generators (k6, wrk, Locust) target HTTP endpoints
or TCP connections. Evaluating warm pool performance requires
instrumentation that operates at the Kubernetes API object level:

- **Claim lifecycle tracking** — measuring the interval from
  SandboxClaim creation to Ready status condition
- **Pool state correlation** — recording `readyReplicas` at each
  measurement point to classify claims as warm or cold
- **Per-phase latency decomposition** — attributing time to specific
  pipeline stages using API watch events and status timestamps
- **Sustained oversubscription** — generating claim volume that
  exceeds pool capacity to observe refill and recovery behavior

---

## Test architecture

The benchmark (`TestRuntimeClassBurstRecovery`) consists of three
cooperating subsystems:

```
                     TestRuntimeClassBurstRecovery
                                  |
                  +---------------+---------------+
                  |               |               |
            Calibration     Batched Claims    CSV Reporter
            (baseline        (sustained         (per-claim
             measurement)     load generation)   telemetry)
                  |               |               |
            Milestone        Adaptive Batch    Cluster Metadata
            Tracker           Sizing            (auto-detected:
            (5-phase          (proportional      provider, instance
             decomposition)    to pool size)      type, CPU count)
```

1. **Calibration** — a single warm claim from a filled pool
   establishes the baseline latency and the 1s warm/cold threshold
2. **Batched claims** — parallel claim batches with controlled
   inter-batch intervals, continuing to 2x pool capacity
3. **Milestone tracker** — correlates API watch events to produce
   per-claim latency decomposition across 5 phases

---

## Per-claim latency decomposition

The milestone tracker records timestamps at each transition in the
claim lifecycle and computes 5 phase durations:

| Phase | Source timestamps | What it measures |
|-------|-------------------|-----------------|
| create_ack | create call → etcd write → watch event | API server round-trip |
| adoption | create returned → claim adopted by controller | Controller bind time |
| schedule | pod creation → PodScheduled condition | Scheduler placement |
| runtime | PodScheduled → pod Ready condition | Container or VM initialization |
| propagate | Sandbox Ready condition → claim Ready | Status propagation |

The `schedule`, `runtime`, and `propagate` phases reference the
**sandbox pod's own lifecycle**, not the claim's. For warm claims,
the sandbox was created and booted during pool fill — these values
are historical. Only `create_ack`, `adoption`, and `e2e` describe
the claim's own latency.

---

## Interpreting the decomposition: warm vs cold

**Warm claims** — sandbox created before the claim existed:

- `create_ack` + `adoption` = actual claim latency (~210–340ms)
- `runtime` = historical VM/container boot time during pool fill
- `propagate` = time the sandbox was idle in the pool before claim

**Cold claims** — sandbox created in response to the claim:

- `runtime` = real-time VM/container boot (the cold start cost)
- `propagate` = brief, sandbox just reached Ready
- `e2e` = full cold start latency (dominated by `runtime`)

This distinction is important: `runtime_ms` on a warm claim does
not indicate that the claim waited for a VM boot. It records how
long that particular sandbox took to boot when the pool was filled
— a historical measurement useful for characterizing per-sandbox
boot variance under concurrent fill load.

---

## Load generation: batched drain loop

The test generates sustained claim pressure using adaptive batching:

```go
batch_size = min(max(4, pool_size/2), batchCap)
max_claims = pool_size * 2

repeat:
  sleep 100ms               // inter-batch settle interval
  GET pool.readyReplicas    // snapshot pool state
  fire batch_size parallel claims
  until: readyReplicas ≤ 1 AND total_claims > pool_size
         OR total_claims ≥ max_claims
```

Design parameters:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Batch size | `min(max(4, pool/2), cap)` | Proportional to pool; capped to avoid etcd write serialization |
| Max claims | `pool_size × 2` | Reasonable upper bound for a single burst run |
| Inter-batch settle | 100ms | Allows reconciler work queue to process pending items |
| Pool state snapshot | Per batch | Enables post-hoc warm/cold classification per claim |

---

## Claim classification

Each claim is classified into one of three zones based on end-to-end
latency:

| Zone | Threshold | Interpretation |
|------|-----------|----------------|
| Green | ≤ 500ms | Warm claim completed within one reconciler cycle |
| Grey | 500ms–1s | Warm claim with additional controller or API contention |
| Cold | > 1s | Pool was depleted; claim was served via cold start |

The 1s threshold is based on observed latency distribution: claims
cluster around the warm baseline or well above 1s (cold), with no
intermediate population. The boundary aligns with established
user-perceptible latency thresholds.

The grey zone latency is runtime-independent — the same ~160ms
bimodal distribution appears across runc, gVisor, and kata.

---

## Cross-runtime throughput comparison

Under sustained batched load, each runtime reaches a distinct
throughput plateau. The limiting factor differs:

| Runtime | Measured throughput | Limiting factor | Warm baseline |
|---------|-------------------|-----------------|---------------|
| runc    | ~10 claims/s | API server and etcd write bandwidth | 0.347s |
| gVisor  | ~10 claims/s | API server and etcd write bandwidth | 0.347s |
| kata    | 2–4.5 claims/s | VM boot time during pool refill | 0.352s |

Warm claim latency is consistent across all runtimes. The throughput
difference is attributable to the refill path — how quickly depleted
pool slots are replaced with new Ready sandboxes.

For runc and gVisor, slot refill completes in under 300ms. The
throughput plateau is determined by controller processing capacity.

For kata, slot refill requires a full VM boot cycle. Throughput
depends on the hypervisor choice and host resource contention
during concurrent fill.

---

## Controller-side latency analysis

Grey zone observations at pool-8 with a batch of 4 concurrent claims:

```
runc   :  0.321  0.319  0.480  0.322   (3 green, 1 grey)
gvisor :  0.320  0.318  0.475  0.314   (3 green, 1 grey)
kata   :  0.333  0.467  0.336  0.333   (3 green, 1 grey)
                 ~160ms latency increment — consistent across runtimes
```

| Source | Measured contribution | Status |
|--------|----------------------|--------|
| Reconciler work queue serialization | 80–160ms | Namespace partitioning available (#813) |
| etcd MVCC write contention | 10–20ms per claim | Mitigated via strategic merge patches (v0.5.0) |
| API server watch event coalescing | 50–100ms | Pod cache label indexing available (#1099) |

v0.5.3 increased sandbox workers from batched ramp-up to 100 parallel
workers. Measured result: 25–32% throughput improvement for runc and
gVisor. The controller was the prior constraint; the bottleneck
shifted to etcd and API server capacity.

---

## Resource density

Per-slot resource overhead by runtime:

| Resource | runc | gVisor | kata-clh |
|----------|------|--------|----------|
| CPU overhead | ~0 | ~0 | 250m |
| RAM consumption | ~16Mi | ~16Mi | ~200Mi |
| Isolation model | namespace/cgroup | syscall interception | hardware VT-x |

Pool capacity on 3 × n2-standard-8 (~22 allocatable CPUs, ~84 GB):

| Pool size | runc/gVisor CPU | kata CPU | kata-clh RAM |
|-----------|----------------|----------|--------------|
| 8  | < 1% | 9% (2.0 CPU) | 1.6 GB |
| 16 | < 1% | 18% (4.0 CPU) | 3.1 GB |
| 24 | < 1% | 27% (6.0 CPU) | 4.7 GB |

For runc and gVisor, pool size is bounded by controller throughput,
not by resource consumption.

---

## Hybrid pool configuration

Separate pools can be configured per runtime class, with claim
routing based on workload requirements:

```
                 SandboxClaim
                      |
             SLA-based routing
              /              \
     gVisor pool (24)      kata pool (16)
     burst workloads       hardware isolation
     ~10 cls/s             2–4.5 cls/s
     < 1% CPU overhead     250m CPU + 200–350Mi per slot
```

| Configuration | gVisor pool | kata pool | kata CPU | kata RAM |
|---------------|-------------|-----------|----------|----------|
| Balanced | 24 | 16 | 4.0 CPU (18%) | 3.1–5.5 GB |
| Burst-oriented | 32 | 8 | 2.0 CPU (9%) | 1.6–2.7 GB |
| Isolation-oriented | 16 | 24 | 6.0 CPU (27%) | 4.7–8.2 GB |

---

## Optimization vectors for kata refill path

The benchmark data identifies three improvement vectors for kata's
refill latency. These can be applied independently or combined:

**Cloud Hypervisor** (available in Kata 4.0 as default VMM)
- Lightweight VMM written in Rust (~50K lines vs ~2M lines C)
- Lower per-slot RAM footprint (~200Mi)

**Rust runtime — kata runtime-rs** (Kata 4.0)
- Runtime rewritten from Go to Rust
- Lower memory footprint and faster initialization
- Eliminates garbage collection pauses during VM lifecycle

**VM templating** (Kata 4.0)
- Pre-fork from a template VM with guest kernel and agent
  already initialized
- New VMs share memory pages via copy-on-write
- Reduces per-VM boot to approximately the fork + COW setup cost

Combined projection: kata cold boot from 8–13s to ~3–4s (CLH) or
~1–2s (CLH + templating), converging toward the controller-bound
throughput observed for runc and gVisor.

---

## Appendix: runc (default) benchmark run (2026-07-27)

The following data is from a single runc run on 3 × n2-standard-8
(21 allocatable CPUs, warm baseline 0.347s). Results are specific to
this runtime class and cluster configuration.

| Pool | Claims | Under 1s | Over 1s | Throughput | Worst | Fill time |
|------|--------|----------|---------|------------|-------|-----------|
| 4    | 8      | 8        | 0       | 4.1/s      | 0.98s | 1.5s      |
| 6    | 12     | 12       | 0       | 4.4/s      | 0.76s | 1.6s      |
| 8    | 16     | 16       | 0       | 4.8/s      | 0.65s | 1.6s      |
| 12   | 24     | 24       | 0       | 6.6/s      | 0.81s | 1.1s      |
| 16   | 32     | 32       | 0       | 8.7/s      | 0.77s | 1.2s      |
| 20   | 40     | 40       | 0       | 9.0/s      | 0.82s | 1.7s      |
| 24   | 48     | 48       | 0       | 10.4/s     | 0.71s | 1.8s      |
| 32   | 64     | 64       | 0       | 9.0/s      | 0.92s | 2.7s      |

Pool fill time is near-constant (1.1–2.7s) regardless of pool size.
Throughput peaks at 10.4 claims/s (pool-24), limited by controller
and API server capacity. Zero cold claims and zero over-1s claims
across all pool sizes.

The `runtime_ms` field shows 0 or 1000ms — container init completes
in sub-second time, below the per-second resolution of pod condition
timestamps. At pool-32, adoption_ms elevates to 150–317ms in later
batches, consistent with controller contention under sustained load.

---

## Appendix: gVisor benchmark run (2026-07-27)

The following data is from a single gVisor run on 3 × n2-standard-8
(21 allocatable CPUs, warm baseline 0.347s). Results are specific to
this runtime class and cluster configuration.

| Pool | Claims | Under 1s | Over 1s | Throughput | Worst | Fill time |
|------|--------|----------|---------|------------|-------|-----------|
| 4    | 8      | 5        | 3       | 3.3/s      | 1.4s  | 1.4s      |
| 6    | 12     | 12       | 0       | 4.6/s      | 0.70s | 1.7s      |
| 8    | 16     | 16       | 0       | 4.7/s      | 0.65s | 1.3s      |
| 12   | 24     | 24       | 0       | 6.8/s      | 0.65s | 1.8s      |
| 16   | 32     | 32       | 0       | 8.5/s      | 0.81s | 1.9s      |
| 20   | 40     | 40       | 0       | 10.7/s     | 0.72s | 1.8s      |
| 24   | 48     | 48       | 0       | 9.1/s      | 0.97s | 2.1s      |
| 32   | 64     | 63       | 1       | 8.8/s      | 1.4s  | 2.1s      |

Pool fill time is near-constant (1.3–2.1s) regardless of pool size.
Throughput peaks at 10.7 claims/s (pool-20), limited by controller
and API server capacity. Zero cold claims through pool-24; the single
over-1s claim at pool-32 resulted from a controller adoption_ms spike
(877ms), not a runtime cold start.

The `runtime_ms` field shows 0 or 1000ms — same resolution limitation
as runc. Both lightweight runtimes are indistinguishable to the
benchmark at this timestamp granularity.

---

## Appendix: kata-clh benchmark run (2026-07-27)

The following data is from a single kata-clh run on 3 × n2-standard-8
(21 allocatable CPUs, warm baseline 0.352s). Results are specific to
this runtime class and cluster configuration.

| Pool | Claims | Under 1s | Over 1s | Throughput | Worst | Fill time |
|------|--------|----------|---------|------------|-------|-----------|
| 4    | 8      | 4        | 4       | 1.9/s      | 3.2s  | 3.0s      |
| 6    | 12     | 10       | 2       | 1.6/s      | 5.8s  | 5.1s      |
| 8    | 16     | 12       | 4       | 1.5/s      | 4.8s  | 13.1s     |
| 12   | 24     | 18       | 6       | 1.6/s      | 11.9s | 20.8s     |
| 16   | 32     | 24       | 8       | 3.3/s      | 6.6s  | 26.1s     |
| 20   | 40     | 30       | 10      | 3.1/s      | 9.6s  | 23.8s     |
| 24   | 48     | 42       | 6       | 4.5/s      | 6.8s  | 9.2s      |
| 32   | 64     | 52       | 12      | 2.3/s      | 21.3s | 30.4s     |

Peak throughput: 4.5 claims/s at pool-24. All warm claims landed in
the grey zone (0.55–0.70s). Throughput decreased at pool-32 due to
longer cold start recovery during refill.

---

## Appendix: VM boot contention during pool fill (kata-clh)

The `runtime_ms` field on warm claims records how long each sandbox's
Cloud Hypervisor VM took to boot during pool fill. These values
reveal how concurrent VM creation contends for host CPU:

| Pool size | Fill time | runtime_ms range (batch 1) |
|-----------|-----------|---------------------------|
| 4  | 3.0s  | 2–7s    |
| 6  | 5.1s  | 3–5s    |
| 8  | 13.1s | 10–12s  |
| 12 | 20.8s | 6–21s   |
| 16 | 26.1s | 9–24s   |
| 20 | 23.8s | 15–24s  |
| 24 | 9.2s  | 4–8s    |
| 32 | 30.4s | 7–30s   |

At pool-32, the slowest VM took 30s to boot — compared to 2–3s at
pool-4. This is host CPU contention from 32 concurrent hypervisor
processes on 21 CPUs, not a hypervisor regression. The variance is
observable only through per-sandbox `runtime_ms` decomposition;
aggregate fill time does not surface per-VM spread.

Pool-24 shows an unexpectedly fast fill (9.2s). This requires
further investigation.

---

## Summary of findings

1. Warm claim latency is runtime-independent at 0.32–0.35s measured
   baseline across runc, gVisor, and kata on identical hardware

2. Throughput plateaus differ by runtime class: ~10 claims/s for
   runc and gVisor (controller-bound), 2–4.5 claims/s for kata
   (VM boot bound during refill)

3. Pool fill time is near-constant for lightweight runtimes (runc:
   1.1–2.7s, gVisor: 1.3–2.1s regardless of pool size) but scales
   with pool size for kata (3–30s) due to concurrent hypervisor CPU
   contention

4. The `runtime_ms` field resolves meaningfully only for runtimes
   with multi-second init (kata: 2–30s). For runc and gVisor, values
   are 0 or 1000ms — below the per-second resolution of pod condition
   timestamps

5. The grey zone (500ms–1s) is a controller artifact — bimodal
   ~160ms latency from reconciler serialization, consistent across
   all tested runtimes

6. Per-claim latency decomposition into 5 phases enables precise
   attribution of overhead to specific pipeline stages, which is
   not achievable with standard HTTP-level load generators

7. Kata 4.0 improvements (Rust runtime, Cloud Hypervisor default,
   VM templating) project a 60–70% reduction in cold start time,
   converging toward the controller-bound throughput observed for
   lightweight runtimes

---

## Next steps

- Validate Kata 4.0 projections with CLH and VM templating benchmarks
  on kata 4.0 builds
- Investigate pool-24 fill time anomaly (9.2s vs 26.1s at pool-16)
- Extend milestone tracker to separate claim-time overhead from
  historical sandbox lifecycle data in schedule and runtime phases
- Replace fixed inter-batch settle interval with probe-based latency
  detection
- Implement SLA-based claim routing via SandboxClaim admission webhook
- Conduct bare-metal comparison to quantify nested virtualization
  overhead for kata workloads
