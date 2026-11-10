# WinTask — Design & Specification Document

**Document type:** Technical Requirements, High-Level/Low-Level Design & Architecture  
**Project:** WinTask  
**Document version:** 0.1 — Proof-of-Concept baseline  
**Status:** Approved for implementation planning  
**Platform:** Windows x64  
**Primary language:** C++23 (C++26 may be enabled where useful)  
**Compiler/toolchain:** Clang 23.1.5 targeting `x86_64-pc-windows-msvc`, using the MSVC STL and Microsoft Build Tools  
**Project window:** 10-day implementation and validation period

## 1. Purpose and project definition

WinTask is a Windows-native, priority-aware request and job-management proof of concept. It models a server-side request-processing engine using one dedicated ingestion thread, three independent bounded single-producer/single-consumer (SPSC) queues, and three dedicated consumer threads. Incoming jobs are routed into High, Medium or Low priority queues, processed by the corresponding consumer, checked against an application-level file/folder access policy where applicable, and recorded in preallocated diagnostics storage for offline reporting.

The project emphasizes compact request representation, bounded memory use, predictable queue ownership, atomic publication/reclamation, CPU affinity, load shedding, concurrency correctness and measurable throughput/latency.

**WinTask is not a clone of Windows Task Manager.** It manages application-level jobs/requests; it does not enumerate, control or schedule arbitrary Windows processes. It is also not a production HTTP server or a replacement for the Windows kernel scheduler.

### 1.1 Primary objective

Implement and evaluate a bounded request-processing architecture with a stretch performance objective of **1,000,000 successfully completed requests per second** under a defined synthetic workload on the project machine.

The figure is a target to test, not a guaranteed capability. Queue capacity, ingestion rate, accepted request rate, completed request rate and simultaneous execution are separate measurements.

### 1.2 Secondary objectives

- Demonstrate priority-separated request routing.
- Demonstrate producer/consumer synchronization using C++ atomics and SPSC ownership.
- Keep each queue slot compact and fixed at 32 bytes.
- Bound pending request storage through preallocated ring buffers.
- Apply an explicit load-shedding policy when a destination queue is full.
- Run one dedicated consumer per priority tier.
- Demonstrate application-level authorization for configured file/folder resources.
- Measure end-to-end ingress-to-terminal-completion latency using the Windows performance counter.
- Collect diagnostics without writing CSV data from the hot processing path.
- Keep the reusable queue, execution and request types sufficiently separate to support later extraction into PDSTL infrastructure.

### 1.3 Out of scope for this POC

The following are deliberately excluded from the 10-day implementation:

- A global overflow queue or cross-priority capacity borrowing.
- A Signal Tree or any other alternative readiness-selection structure.
- A dynamic worker pool, work stealing or arbitrary worker creation.
- A GUI, dashboard, OS visualizer or web interface.
- Kernel modifications, kernel drivers, kernel modules or filesystem minifilters.
- A production HTTP listener or actual HTTP response transport. HTTP 429 is a simulated overload/rejection classification only.
- Arbitrary executable launching or a general-purpose process manager.
- Native Windows security-token isolation between jobs and modification of real system ACLs.
- Cryptographically secure authentication/signature verification.
- Intra-queue urgency reordering in the initial implementation.
- A general-purpose allocator, per-thread allocator caches or lock-free memory reclamation.
- A separate `WinTask.cpp` executable entry point duplicating the benchmark's end-to-end demonstration.

The file/folder permission objective is implemented as **application-level authorization over a controlled test workspace**. This does not prevent an unrelated process running under the same Windows security identity from bypassing WinTask's application-level checks.

---

## 2. Terminology

| Term | Meaning in WinTask |
|---|---|
| Request / job | One fixed-size unit submitted to WinTask for processing. The terms are used interchangeably in this document. |
| Ingestor | The single producer thread that validates basic request fields, timestamps, classifies and routes jobs. |
| Consumer | One dedicated thread that drains exactly one priority queue. |
| Queue slot | One 32-byte `QueueSlot` containing a descriptor and payload. |
| Severity | Input field that selects the High, Medium or Low queue. It is a priority-classification value in this POC. |
| Urgency | Additional request metadata. It is recorded but does not reorder FIFO requests in this POC. |
| Resource policy ID | Identifier resolving to a preconfigured application-level resource/access policy. |
| Ingress timestamp | Raw `QueryPerformanceCounter` tick value captured by the ingestor. |
| Load shedding | Rejecting a new request when its destination ring buffer has no free slot. |
| Accepted request | A valid request successfully published into its destination queue. |
| Completed request | An accepted request that reaches the successful terminal state. Failed/denied requests are recorded separately in per-job status data. |
| Queue capacity | Maximum number of pending/in-flight slots owned by a particular ring buffer. |
| Aggregate capacity | Sum of the three independent queue capacities; it is not a shared pool. |

---

## 3. Technical requirements

### 3.1 Functional requirements

| ID | Requirement |
|---|---|
| FR-01 | WinTask shall accept requests through one ingestion thread. |
| FR-02 | Each valid request shall be assigned to exactly one priority queue using its severity value. |
| FR-03 | Severity values 1–2 shall route to Low; 3–4 to Medium; 5 to High. Values outside 1–5 shall be rejected as invalid input and shall not be published to a queue. |
| FR-04 | Each priority queue shall be an independently allocated, bounded SPSC ring buffer. |
| FR-05 | The producer shall be the only writer of a queue's tail counter and the consumer shall be the only writer of its head counter. |
| FR-06 | Each queue shall have exactly one dedicated consumer thread. |
| FR-07 | A consumer shall process requests in FIFO order within its own queue. |
| FR-08 | A full destination queue shall reject the incoming request and increment the capacity-rejection counter. No request shall be silently overwritten. |
| FR-09 | The system shall not transfer requests to another priority queue when the selected queue is full. |
| FR-10 | Each accepted request shall pass through the defined job handler and, for file-facing operations, the application-level authorization check. |
| FR-11 | Consumers shall record terminal result data in preallocated, consumer-isolated diagnostics storage. |
| FR-12 | CSV generation shall occur after the measured run has drained and all worker threads have terminated. |
| FR-13 | The system shall maintain aggregate offered, accepted, successfully completed and capacity-rejected counts. |
| FR-14 | The benchmark shall calculate completed-request throughput and ingress-to-terminal-completion latency. |
| FR-15 | The system shall drain accepted work before normal shutdown and shall join all worker threads before exporting final results. |

### 3.2 Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | `QueueSlot` shall have a compile-time verified size of exactly 32 bytes. |
| NFR-02 | Queue backing storage shall begin at a 64-byte-aligned address. Individual slots shall retain a 32-byte stride; the slot type shall not be padded to 64 bytes. |
| NFR-03 | Descriptor and payload member offsets shall be verified using `static_assert` and `offsetof`. |
| NFR-04 | Queue publication and slot reclamation shall use explicit acquire/release ordering. |
| NFR-05 | Head and tail control variables shall be independently cache-line aligned to avoid sharing their control cache line. |
| NFR-06 | The ring-buffer capacities shall be powers of two and indexing shall use a mask. |
| NFR-07 | The steady-state ingestion/processing path shall not perform CSV formatting or synchronous CSV file output. |
| NFR-08 | Diagnostics storage shall be allocated before the timed benchmark interval and partitioned by consumer. |
| NFR-09 | Thread affinity shall use the project machine's explicitly selected processor mapping. |
| NFR-10 | AVX2 shall be used for the planned 32-byte slot-copy/ingestion path on the target machine, with its benefit measured rather than assumed. |
| NFR-11 | Benchmark reports shall distinguish offered, accepted, completed and rejected requests. |
| NFR-12 | The implementation shall compile with the selected Clang/MSVC STL toolchain; GCC is not a target compiler for this project. |

### 3.3 Performance objectives and claims

- **Primary stretch target:** 1,000,000 successfully completed requests/second for a stated workload, run duration, thread affinity and machine configuration.
- **Capacity:** 1,310,720 pending/in-flight slots across the three queues.
- **Latency:** measured from ingress timestamp to terminal completion/failure; report in microseconds with fractional precision derived from QPC ticks.
- **Processing-cost estimate:** the originally proposed 300–700 ns processing interval is a workload-design estimate only. It is not a guaranteed or accepted benchmark result until measured.
- **No result may claim 1M requests/second based only on offered or accepted requests.** The completed count and benchmark duration are the basis of completed throughput.

---

## 4. Request data model and structural memory layout

### 4.1 Job type and state values

```cpp
enum class JobType : std::uint8_t {
    VALIDATE_SESSION = 0,
    TELEMETRY_LOG    = 1,
    COMPUTE_SIGN     = 2
};

enum class JobState : std::uint8_t {
    Ready     = 0,
    Running   = 1,
    Yielded   = 2, // Reserved; not used by the POC execution path.
    Cancelled = 3, // Reserved; cancellation is not implemented in this POC.
    Completed = 4,
    Failed    = 5
};
```

`JobState` is stored in the lower three bits of `JobDescriptor::control`. The upper five bits are reserved for flags. State helpers shall mask only the lower three bits and preserve the upper five flag bits. `Yielded` and `Cancelled` remain reserved enum values; the POC does not implement preemption, yielding or cancellation.

### 4.2 Approved 32-byte representation

The previous generic timestamp/policy arrangement is clarified as follows. The generic 64-bit descriptor field is assigned to the ingress QPC timestamp. The 32-bit payload field previously described as a microsecond timestamp is assigned to the resource policy identifier. This preserves the intended offsets and the 32-byte slot size.

```cpp
#include <cstddef>
#include <cstdint>
#include <type_traits>

struct JobDescriptor {
    JobType       type;               // Offset 0, 1 byte
    std::uint8_t  severity;           // Offset 1, 1 byte
    std::uint8_t  urgency;            // Offset 2, 1 byte
    std::uint8_t  control;            // Offset 3, 1 byte
    std::uint32_t job_id;             // Offset 4, 4 bytes
    std::uint64_t ingress_qpc_ticks;  // Offset 8, 8 bytes
};

struct JobPayload {
    std::uint32_t user_id;             // Offset 0, 4 bytes
    std::uint32_t resource_policy_id;  // Offset 4, 4 bytes
    std::uint64_t session_signature;   // Offset 8, 8 bytes
};

struct QueueSlot {
    JobDescriptor descriptor; // Offset 0, 16 bytes
    JobPayload    payload;    // Offset 16, 16 bytes
};

static_assert(sizeof(JobDescriptor) == 16);
static_assert(sizeof(JobPayload) == 16);
static_assert(sizeof(QueueSlot) == 32);
static_assert(offsetof(JobDescriptor, job_id) == 4);
static_assert(offsetof(JobDescriptor, ingress_qpc_ticks) == 8);
static_assert(offsetof(JobPayload, user_id) == 0);
static_assert(offsetof(JobPayload, resource_policy_id) == 4);
static_assert(offsetof(JobPayload, session_signature) == 8);
static_assert(offsetof(QueueSlot, descriptor) == 0);
static_assert(offsetof(QueueSlot, payload) == 16);
static_assert(std::is_trivially_copyable_v<QueueSlot>);
```

**Packing directive policy:** no `#pragma pack` is required or planned. The member order naturally yields the intended layout for the selected x64 target; compile-time assertions are the authority. The queue allocation base shall be 64-byte aligned. Since each slot is 32 bytes, each slot address is also 32-byte aligned when the base is 64-byte aligned. Do not apply `alignas(64)` to `QueueSlot`, because doing so would change the slot stride and invalidate the two-slots-per-cache-line layout.

### 4.3 Field semantics

| Field | Meaning | POC use |
|---|---|---|
| `type` | Job operation category | Selects one of the three fixed job handlers. |
| `severity` | Priority-classification input | Routes the request to High, Medium or Low. |
| `urgency` | Additional urgency score, 0–255 | Recorded in diagnostics; no intra-queue reordering in this release. |
| `control` | Packed state and flags | Lower 3 bits carry `JobState`; upper 5 bits are reserved flags. |
| `job_id` | Request identifier | Identifies the request in diagnostics and CSV. The benchmark must avoid duplicate IDs within one run. |
| `ingress_qpc_ticks` | Raw 64-bit QPC value | Captured at ingress and used to calculate end-to-end latency. |
| `user_id` | Synthetic/requesting user identifier | Used by the application-level authorization lookup. |
| `resource_policy_id` | 32-bit policy-table key | Resolves the permitted workspace/resource and operation rights. |
| `session_signature` | 64-bit synthetic session token/input | Used by the POC session/signature workload; not a secure cryptographic credential. |

`resource_policy_id` identifies preconfigured policy data held outside the queue slot. The slot contains no path strings, heap pointers or dynamically allocated payload. Policy records may contain the approved test root, configured file targets and operation permissions.

### 4.4 Cache-line layout

The intended density is two complete 32-byte slots per assumed 64-byte cache line. This is a storage-density choice, not a claim that producer writes and consumer reads are automatically free of coherence traffic.

- Queue base: 64-byte aligned.
- Slot stride: 32 bytes.
- Each slot start: 32-byte aligned.
- Producer tail control and consumer head control: separate 64-byte-aligned locations.
- A 64-byte ownership/batch mode may be evaluated as an experimental variant. It is not required for the baseline because retaining a pair until both jobs complete can increase per-job waiting time and reduce flexibility.
- The benchmark should compare compact 32-byte slots against an optional padded/paired experimental mode only if time permits. No mandatory queue-memory doubling is introduced.

---

## 5. Queue topology and capacity

### 5.1 Queue capacities

| Priority queue | Severity mapping | Capacity | Mask | Memory |
|---|---:|---:|---:|---:|
| High | 5 | 262,144 (`2^18`) | `0x0003FFFF` | 8 MiB |
| Medium | 3–4 | 524,288 (`2^19`) | `0x0007FFFF` | 16 MiB |
| Low | 1–2 | 524,288 (`2^19`) | `0x0007FFFF` | 16 MiB |
| **Total** | | **1,310,720** | | **40 MiB** |

The aggregate allocation is 1,000,000 baseline slots plus 310,720 additional slots (31.072% above the one-million baseline). The extra capacity is deliberate distributed headroom.

**The extra slots are not a global reserve.** Each queue has a fixed capacity. An empty Low queue cannot donate slots to a full High queue. No cross-queue borrowing or overflow queue is part of this POC.

At an aggregate input rate of 1,000,000 requests/second, 1,310,720 completely unused slots represent a theoretical 1.31072-second aggregate backlog only if requests are distributed so that no individual priority queue reaches its own capacity first. This is a capacity illustration, not a traffic guarantee.

### 5.2 SPSC ownership

Each ring buffer has exactly one producer (the ingestor) and exactly one consumer (the thread assigned to that tier).

| Queue resource | Exclusive writer/owner | Other side |
|---|---|---|
| `tail` publication counter | Ingestor | Consumer reads published value |
| `head` reclamation counter | Consumer | Ingestor reads released value |
| Slot before tail publication | Ingestor | Consumer must not read |
| Slot after tail publication and before head reclamation | Consumer processes it | Ingestor must not overwrite |
| Slot after head release is observed | Ingestor may reuse it | Consumer has finished with it |

No generic MPMC queue, per-slot atomic state machine, or `fetch_add` reservation is required for this single-producer/single-consumer arrangement.

### 5.3 Counter and index rules

Use monotonically increasing unsigned 64-bit logical counters. The physical slot index is calculated as:

```cpp
slot_index = counter & (Capacity - 1);
```

Queue capacities are powers of two and the mask is `Capacity - 1`. Occupancy is determined by the unsigned difference between the producer's next tail value and the consumer's published head value. The implementation shall maintain the invariant that occupancy never exceeds `Capacity`.

The queue's `head` is the **reclamation position** in this POC: the consumer processes the slot in place, writes its terminal state/diagnostic result, and only then releases the slot by publishing the next head. Thus the currently executing request occupies its slot until its processing is complete. This simplifies slot ownership and permits the consumer to update `JobState` without racing with producer reuse.

---

## 6. Atomic synchronization and memory ordering

### 6.1 Publication protocol

**Producer (ingestor):**

1. Read the consumer's published `head` with `memory_order_acquire` when a refreshed head value is needed for the full check.
2. Confirm that the destination queue has at least one free slot (or the required batch capacity).
3. Construct/write the complete `QueueSlot` in the producer-owned tail position, setting its state to `Ready`.
4. Publish the new tail with `memory_order_release`.

**Consumer:**

1. Read the published `tail` with `memory_order_acquire`.
2. If the local head equals the observed tail, the queue is empty.
3. Read the complete published slot and transition its state to `Running`.
4. Perform authorization and the job handler; transition the slot to `Completed` or `Failed`.
5. Write the request's diagnostic result to the consumer's private result array.
6. Publish the next head with `memory_order_release`, making the slot reusable.

The producer's acquire read of the consumer's released head prevents it from reusing a slot before the consumer has finished accessing it. The consumer's acquire read of the producer's released tail makes the producer's completed slot writes visible before the consumer reads the slot.

This is a C++ release/acquire synchronization design. It is compatible with x86-64 TSO, but correctness must be derived from the C++ memory model rather than relying solely on a hardware ordering assumption. The implementation should use the C++ atomic primitives explicitly.

### 6.2 Cache-line placement of indices

Place the producer-owned tail and consumer-owned head in separate cache-line-aligned storage. This targets false sharing on the frequently updated queue-control variables. The implementation may use a small aligned wrapper type for each atomic counter.

### 6.3 Atomic lock-free assumption

The selected x64 toolchain is expected to provide lock-free 64-bit atomics. The implementation shall verify the property on the target build (for example, with `std::atomic<std::uint64_t>::is_always_lock_free` or an equivalent runtime/build check) before describing the queue as lock-free. If the check fails, the benchmark report must disclose it rather than silently claiming lock-free behavior.

### 6.4 Busy-wait behavior

The POC may use a simple polling loop for empty-queue detection. Any pause/yield instruction or backoff policy must be documented and kept identical across comparable benchmark variants. The POC does not require a general event-driven wait mechanism. CPU consumption while idle should be noted when interpreting results.

---

## 7. Ingestion, routing and load shedding

### 7.1 Ingestor responsibilities

The single ingestor thread shall:

1. Obtain or receive the next request.
2. Validate the job type and severity range.
3. Capture `QueryPerformanceCounter` into `ingress_qpc_ticks`.
4. Determine the target priority queue from severity.
5. Initialize the slot descriptor, payload and `Ready` state.
6. Attempt bounded publication to the target queue.
7. Update offered/accepted/capacity-rejected counters.

The benchmark may generate requests internally. The POC does not require an HTTP server or network stack.

### 7.2 Severity routing

| Severity | Destination |
|---:|---|
| 1–2 | Low |
| 3–4 | Medium |
| 5 | High |
| 0 or greater than 5 | Invalid input; not enqueued |

### 7.3 Load shedding

If the selected queue is full, the ingestor rejects the request immediately. It shall not overwrite an unread/in-flight slot and shall not redirect the request to another queue.

The event increments the capacity-rejection counter. Diagnostics may classify it as a simulated HTTP `429 Too Many Requests` outcome, but no HTTP response is sent because no HTTP endpoint exists in this POC.

Rejection accounting is per request. If a producer-side batch contains two requests and the target queue cannot accept them, the number of rejected requests shall reflect the number actually not published. A batch implementation must not silently discard a request that could have fitted in the remaining slot.

### 7.4 AVX2 ingestion

One `QueueSlot` is 32 bytes, equal to one 256-bit AVX2 register. One AVX2 256-bit store can transfer one slot. Two adjacent slots require two 256-bit stores (two registers/stores), not one AVX2 store.

The destination queue base and slot stride are designed to make each slot address 32-byte aligned. The implementation shall confirm source/destination alignment and object-copy correctness. AVX2 is an optimization to test, not a guarantee that one source-level intrinsic completes in one CPU cycle. The scalar/reference copy path should remain available to the validator where practical.

Two-slot batching is permitted when two consecutive requests are destined for the same queue. If a batch is incomplete or requests route to different queues, the implementation shall publish the eligible request(s) without waiting indefinitely to form a pair. Batch publication must preserve the SPSC ownership protocol.

---

## 8. Worker and request-processing design

### 8.1 Thread topology and affinity

The target-machine mapping is:

| Thread | Assigned processor/core |
|---|---:|
| Ingestor | 0 |
| High consumer | 2 |
| Medium consumer | 4 |
| Low consumer | 6 |

These values are the user's selected processor mapping for the project machine and shall be centralized in configuration. `WinTaskValidator.cpp` and `WinTaskBenchmark.cpp` shall apply the agreed affinity configuration for relevant test/benchmark threads. The POC does not need an automatic topology-selection framework.

Thread creation, affinity setup and warm-up shall occur outside the timed request interval. Affinity setup failures shall fail or invalidate the associated benchmark run rather than being ignored.

### 8.2 Consumer loop

Each consumer drains only its assigned queue. Its basic lifecycle is:

`Empty check → acquire slot → Ready → Running → authorize → process → Completed/Failed → diagnostics append → release slot`

A consumer shall not inspect or modify another consumer's queue. No work stealing or cross-tier execution is part of the baseline.

### 8.3 Job types

| Job type | POC behavior | Authorization relationship |
|---|---|---|
| `VALIDATE_SESSION` | Validates a synthetic session/token test record using `user_id` and `session_signature`, then performs a small deterministic CPU workload. | Checks the configured policy for the session/resource read operation where a resource is configured. |
| `TELEMETRY_LOG` | Builds a compact telemetry/event record. The benchmark hot path records results in preallocated memory and exports them after the run. | Requires the configured append/write permission for a telemetry destination in filesystem integration tests. |
| `COMPUTE_SIGN` | Performs deterministic 64-bit rotation/mixing work on the input fields to simulate a signature/computation workload. | Requires the configured read/write permissions for its test input/output resource where file operations are exercised. |

The 64-bit transformation is **not cryptographic verification, authentication or a secure signature algorithm**. It is a deterministic CPU-work simulation. The POC must not claim security based on `std::rotl`, `std::rotr`, XOR or multiplication alone. A future security-oriented version may replace it with a reviewed keyed MAC/signature implementation and an appropriately sized key/digest representation; that is outside the current POC.

The proposed 300–700 ns total processing interval is not hardcoded as a delay. The handler performs actual deterministic operations, and the measured cost is reported. Artificial sleeps are not permitted in the throughput benchmark.

### 8.4 Urgency field

`urgency` is retained and recorded. It does not reorder FIFO requests, change priority queues, bypass authorization or suppress diagnostics in this POC. Any urgency-based local reordering is post-POC work and would require a separate scheduling policy and fairness tests.

---

## 9. File and folder access-control subsystem

### 9.1 Scope and trust boundary

WinTask implements an application-level access policy for file/folder operations initiated by WinTask's own job handlers. It does not change NTFS ACLs and does not create a separate Windows security identity for every job.

The initial implementation uses a controlled test workspace and a preconfigured policy table. The request carries only `user_id` and `resource_policy_id`; paths and permission sets remain outside the 32-byte slot.

### 9.2 Policy lookup

The authorization service resolves `(user_id, resource_policy_id)` to a policy record containing:

- Approved workspace root or configured resource paths.
- Permitted operation bits (Read, Create, Write/Append, as applicable).
- Active/disabled policy state for the test configuration.

The required operation is derived from the job's actual file operation. Authorization must occur before the operation. All file operations performed by WinTask job handlers must pass through the policy service.

### 9.3 Path constraints

The POC shall use test-owned directories/files only. It shall reject paths outside the approved root, including straightforward `..` traversal attempts. The implementation should normalize/resolve paths consistently and avoid treating a string-prefix match as sufficient proof that a path is inside the approved directory. Reparse-point/symlink edge cases should be documented as a known POC limitation unless they are specifically tested and handled.

### 9.4 Benchmark separation

Filesystem access can dominate the cost of a high-rate in-memory dispatch benchmark. Therefore:

- The primary throughput profile shall use preloaded/in-memory policy records and deterministic in-memory job work, measuring dispatch and authorization lookup without synchronous per-request disk I/O.
- Separate authorization integration tests shall perform actual controlled file/folder operations in the sandbox and verify allowed/denied outcomes.
- Results from the in-memory dispatch profile must not be described as one million real disk operations per second.

---

## 10. Timestamping, latency and diagnostics

### 10.1 Ingress timestamp

At ingress, capture a raw 64-bit `QueryPerformanceCounter` value and store it in `JobDescriptor::ingress_qpc_ticks`. Query `QueryPerformanceFrequency` once during initialization and cache the result.

The QPC tick delta is:

`latency_ticks = completion_qpc_ticks - ingress_qpc_ticks`

Convert to microseconds using the cached frequency:

`latency_us = latency_ticks * 1,000,000 / qpc_frequency`

Use an arithmetic approach that avoids unintended integer overflow and preserves fractional microseconds in the CSV. The timestamp measures **end-to-end ingress-to-terminal-completion latency**, which includes queue waiting, authorization and the handler work. It is not solely consumer execution time.

### 10.2 Per-consumer diagnostics

Each consumer writes results only to its own preallocated diagnostics array. No consumer writes into another consumer's result region. The arrays are sized before the benchmark begins and do not grow during timed processing.

A compact record should retain at least:

- `job_id`
- Priority tier
- `urgency`
- Terminal status (Completed or Failed, with an authorization-denied reason where applicable)
- Raw latency QPC ticks or a value sufficient to export fractional `LatencyMicroseconds`

Diagnostics arrays must have sufficient capacity for the maximum number of accepted requests expected in the run. If an array capacity is reached, this must be treated as a benchmark configuration failure; diagnostics must not silently overwrite earlier records.

### 10.3 Aggregate counters

Maintain the following aggregate counts:

- `RequestsOffered`
- `RequestsAccepted`
- `RequestsCompleted` (successful terminal completions)
- `RequestsRejectedCapacity`

Per-job diagnostic status provides the failed/denied outcomes. At a fully drained run, the accounting identity should be:

`RequestsAccepted = RequestsCompleted + RequestsFailed`

and:

`RequestsOffered = RequestsAccepted + RequestsRejectedCapacity`

These identities assume the benchmark offers only validly formed requests. Invalid-input test cases are handled separately by validation tests and shall not be mixed into the capacity-rejection count.

### 10.4 CSV output

The primary per-job result file is `BenchmarkResults.csv`
 Minimum columns:

`JobID,PriorityClass,Urgency,LatencyMicroseconds,Status`

A run-summary section/file or accompanying summary output shall retain offered, accepted, successfully completed, rejected-capacity and failed counts, elapsed benchmark time, and completed throughput. If a single CSV is used, the summary row format must be documented so it cannot be confused with per-job rows.

CSV output is produced after workers stop and the result arrays are stable. No CSV formatting, allocation or file writing occurs on the measured consumer hot path.

### 10.5 Throughput definition

`CompletedThroughput = RequestsCompleted / MeasurementDurationSeconds`

The report shall also show offered, accepted and rejected rates or counts so overload shedding cannot inflate the completed-throughput claim. A run that has not drained all accepted jobs is incomplete and cannot be used for final completed-throughput reporting.

---

## 11. Cache and SIMD evaluation

### 11.1 Baseline layout

The baseline retains 32-byte slots and 64-byte-aligned queue storage. This yields two slots per assumed 64-byte cache line and a 40 MiB aggregate ring-buffer footprint.

### 11.2 Control-line separation

Head and tail atomics shall occupy separate cache lines. This is mandatory in the baseline because the two counters are independently written by different cores.

### 11.3 Slot-line interference experiment

The 32-byte slot arrangement may permit the producer and consumer to touch different halves of one cache line near the producer/consumer boundary. Cache coherence operates at cache-line granularity, so this may create avoidable line ownership transfers.

The POC shall preserve the 32-byte representation. If time permits, benchmark one experimental mitigation:

- Producer writes two adjacent same-queue slots and publishes the pair together.
- Consumer retains/releases the two-slot group together.

This is not a required baseline behavior. It may increase the time a completed first item occupies a slot while the second item is processed. The experiment must compare throughput and latency, not only store count.

### 11.4 AVX2 claims

AVX2 is 256 bits wide. A 32-byte slot corresponds to one 256-bit vector. Two slots occupy 64 bytes and require two 256-bit stores. Source-level intrinsic count shall not be translated into an assumed one-cycle completion time. Report the measured result on the target machine.

---

## 12. Source-file and module organization

The POC uses a small, flat source layout. No extra directory hierarchy is required.

| File | Responsibility |
|---|---|
| `WinTaskTypes.hpp` | `JobType`, `JobState`, descriptor, payload, queue slot, state/flag helpers and compile-time layout assertions. |
| `WinTaskConfig.hpp` | Queue capacities/masks, severity routing constants, batch width, target processor assignments and benchmark configuration. |
| `WinTaskQueues.hpp` | Fixed-capacity ring buffer, aligned head/tail controls, enqueue/publication and consumer/reclamation operations. |
| `WinTaskAuthorization.hpp` | Resource-policy records, `(user_id, resource_policy_id)` lookup and permission checks for controlled test resources. |
| `WinTaskDiagnostics.hpp` | Per-consumer result records/arrays, counters, latency conversion and post-run CSV export. |
| `WinTaskPlatform.hpp` | Windows QPC/QPF wrappers, thread-affinity wrapper and any narrowly scoped Windows-specific support. |
| `WinTaskEngine.hpp` | Ingestor, priority routing, consumer loops, three job handlers, load shedding, lifecycle and run/shutdown orchestration. |
| `WinTaskValidator.cpp` | Standalone validation executable: layout, queue behavior, routing, synchronization, authorization, state transitions, accounting and shutdown tests. |
| `WinTaskBenchmark.cpp` | Standalone end-to-end demonstration and benchmark: generate workload, pin threads, run the engine, drain work, collect metrics and export CSV. |

### 12.1 Is `WinTask.cpp` required?

**No separate `WinTask.cpp` is required for the current POC.** `WinTaskBenchmark.cpp` is the end-to-end executable entry point and demonstrates the integrated WinTask engine. `WinTaskValidator.cpp` is the independent correctness executable. The reusable engine implementation resides in `WinTask.Engine.hpp` and its supporting headers.

A separate `WinTask.cpp` would become justified only if WinTask later gains a distinct application/service entry point, an interactive server-control mode or a runtime executable that is intentionally separate from the benchmark harness. It should not be added merely to duplicate the benchmark's simulation.

### 12.2 Build outputs

- `WinTaskValidator.exe`
- `WinTaskBenchmark.exe`
- `BenchmarkResults.csv` (generated after a completed benchmark run)

Generated results are not source files and should not be committed as if they were implementation modules. Preserve representative results separately when preparing the report.

---

## 13. Build and runtime configuration

- Target: Windows x64.
- Compiler: Clang 23.1.5 with `x86_64-pc-windows-msvc` target.
- Standard library: MSVC STL supplied with the selected Visual Studio 2026 Build Tools installation.
- Language baseline: C++26
- ISA: AVX2 for the planned SIMD path on the target machine.
- OS APIs: Windows performance counter and thread-affinity facilities.
- GCC 16.1 is not part of this project's primary build/validation matrix.
- Build mode for benchmark: optimized/release configuration. Correctness validation may additionally run in a debug/instrumented configuration.

The precise compiler, STL, SDK, linker and Windows build versions used for the final recorded benchmark must be written into the run record. Do not compare debug and release timings as equivalent results.

---

## 14. Known limitations and interpretation rules

1. The three queues have fixed capacities and cannot borrow unused capacity from one another.
2. A dedicated consumer per priority tier means a busy Low queue does not use idle High/Medium consumer capacity.
3. Strict queue-tier separation can produce per-tier imbalance; aggregate free slots do not guarantee that a particular tier can accept more work.
4. FIFO is maintained inside each queue. `urgency` does not reorder a queue in the POC.
5. The system simulates a server request workload in-process; it does not implement network ingress or an HTTP server.
6. HTTP 429 is a simulated capacity-rejection classification, not a transmitted protocol response.
7. The session/signature operation is synthetic and provides no cryptographic security.
8. File authorization is application-level and applies only to WinTask-mediated operations.
9. The primary throughput workload uses in-memory policy lookup and synthetic processing; actual filesystem integration is validated separately.
10. One million requests/second is not a promised outcome. A result is meaningful only with its workload distribution, run duration, CPU affinity, build configuration and accounting counts.
11. The 64-byte cache-line assumption is the project's target-machine optimization assumption; the implementation should label it as such rather than presenting it as a universal guarantee for every processor.
12. The 300–700 ns processing estimate is provisional until measured.

---

## 15. Acceptance criteria for the POC

The POC is ready for its final demonstration when:

- All data-layout assertions pass and `QueueSlot` is exactly 32 bytes.
- All three ring buffers initialize with the specified capacities and masks.
- The producer routes valid severity values to the correct queue.
- The SPSC queues pass FIFO, wraparound, full and empty tests.
- Concurrent stress validation shows no duplicate, lost or corrupted accepted request IDs.
- Queue-full conditions reject rather than overwrite requests.
- Each consumer processes only its assigned queue.
- State transitions and terminal status records are correct.
- File-policy allow/deny tests pass inside the controlled workspace.
- QPC-based ingress-to-completion latency is exported with fractional-microsecond precision.
- Diagnostics arrays do not overflow for the configured benchmark run.
- Offered/accepted/completed/rejected/failed accounting reconciles after drain.
- Both executables build with the selected Clang/MSVC STL toolchain.
- The benchmark exports `BenchmarkResults.csv` after all consumers terminate.
- Performance results state the workload and do not confuse offered rate with completed throughput.

## 16. External technical references

These references describe relevant platform/language behavior; the project-specific architecture and requirements above remain the controlling specification.

1. Microsoft Learn — [QueryPerformanceFrequency](https://learn.microsoft.com/en-us/windows/win32/api/profileapi/nf-profileapi-queryperformancefrequency): performance-counter frequency is fixed at system boot and can be cached after initialization.
2. Microsoft Learn — [QueryPerformanceCounter](https://learn.microsoft.com/en-us/windows/win32/api/profileapi/nf-profileapi-queryperformancecounter): high-resolution performance-counter acquisition.
3. C++ reference — [std::memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order): acquire/release synchronization and visibility of non-atomic data published through an atomic.
4. Microsoft Learn — [SetThreadAffinityMask](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setthreadaffinitymask): Windows thread processor affinity API.

## 17. Change-control notes

The following changes are explicitly part of this document baseline:

- `custom_field` is semantically assigned to `ingress_qpc_ticks` (64-bit QPC timestamp).
- The payload's former 32-bit `ingress_timestamp` position is assigned to `resource_policy_id`.
- `QueueSlot` remains 32 bytes; no packing pragma is used.
- `WinTask.cpp` is omitted from the POC; benchmark and validator are the two executable entry points.
- Urgency remains recorded metadata but does not reorder FIFO queues.
- The POC uses application-level file authorization and synthetic computation, not cryptographic verification or native per-job Windows security isolation.

Any later change to slot size, queue capacities, ownership model, memory-order protocol, job-state semantics or benchmark accounting must be recorded here before implementation is treated as conforming to this specification.
