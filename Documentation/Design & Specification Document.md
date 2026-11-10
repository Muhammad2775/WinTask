# WinTask — Design & Specification Document

**Document type:** Combined Technical Requirements, High-Level Design and Low-Level Design  
**Project:** WinTask — High-Performance Priority-Based Task Scheduler & Processor for Windows  
**Document version:** 0.2 — Revised Proof-of-Concept baseline  
**Status:** Approved architectural baseline; implementation pending  
**Academic context:** Semester 5 — Operating Systems (Lab Project)  
**Platform:** Windows x64  
**Primary language:** C++20 (C++23 features may be used where they provide a concrete benefit and are supported by the selected toolchain)  
**Compiler/toolchain:** Clang 23.1.5 targeting `x86_64-pc-windows-msvc`, using the MSVC STL and Microsoft Build Tools for Visual Studio 2026  
**Implementation and validation window:** 10 days

---

## 1. Project definition

WinTask is a Windows-native, high-performance, priority-based task scheduler and processor. It is a proof-of-concept system intended to model how a bounded server-side task-processing component can ingest a large volume of requests, classify them by priority, dispatch them through independent queues, process them concurrently, and collect performance results.

The architecture uses one dedicated ingestion thread, three independent bounded single-producer/single-consumer (SPSC) ring buffers, and three dedicated consumer threads. The ingestor classifies each valid request into the High, Medium or Low tier. Each consumer processes only the queue assigned to it. Queue capacity is fixed, and requests that cannot be admitted are rejected through an explicit load-shedding policy.

WinTask emphasizes compact request representation, bounded memory use, clear producer/consumer ownership, C++ atomic synchronization, CPU affinity, cache-aware data placement, SIMD-assisted request ingestion, predictable overload behavior, and measurable throughput and end-to-end latency.

**WinTask is an application-level task scheduler and processor, not a replacement for the Windows kernel scheduler.** It does not enumerate, control or schedule arbitrary Windows processes, and it is not intended to be a production HTTP server. The server-oriented workload is simulated in-process.

### 1.1 Primary objective

Implement and evaluate a bounded, priority-separated request-processing architecture with a stretch performance objective of **1,000,000 successfully completed requests per second** under a defined synthetic workload on the project machine.

The one-million-requests-per-second figure is a target to measure, not a guaranteed capability. Offered request rate, accepted request rate, completed request rate, queue capacity and simultaneous execution are distinct concepts and shall be reported separately.

### 1.2 Secondary objectives

- Demonstrate priority-based request classification and dispatch across High, Medium and Low queues.
- Demonstrate SPSC producer/consumer ownership and synchronization using C++ atomics.
- Retain a compact, fixed-size 32-byte request slot.
- Bound pending-request storage through three preallocated ring buffers.
- Apply explicit load shedding when a destination queue is full.
- Run one dedicated consumer for each priority tier.
- Measure ingress-to-terminal-completion latency using the Windows high-resolution performance counter.
- Exercise three representative synthetic job types: session validation, telemetry processing and signature/bit-transformation computation.
- Collect per-consumer results without performing CSV formatting or file output in the measured processing path.
- Evaluate the effect of cache-line placement and AVX2-assisted slot transfer rather than assuming an optimization is beneficial.
- Keep the engine and its queue mechanism sufficiently self-contained to support later investigation or extraction into PDSTL.

### 1.3 Scope boundaries and exclusions

The following are outside the current proof of concept:

- A global overflow queue or cross-priority capacity borrowing.
- A Signal Tree or another alternative readiness-selection structure.
- A dynamic worker pool, work stealing or arbitrary worker creation.
- A GUI, dashboard, web interface or OS visualizer.
- Kernel modifications, drivers, kernel modules or filesystem minifilters.
- A production HTTP listener or actual HTTP response transport. HTTP 429 is only a simulated overload/rejection classification.
- Arbitrary executable launching or management of unrelated Windows processes.
- Cryptographically secure authentication or real signature verification.
- Intra-queue urgency reordering in the initial implementation.
- A general-purpose allocator, per-thread allocator caches or lock-free memory reclamation.
- A separate executable implementation of the engine in addition to the main WinTask application.

The three job handlers provide controlled synthetic processing. They are not security services and shall not be described as providing cryptographic protection.

---

## 2. Terminology

| Term | Meaning in WinTask |
|---|---|
| Request / job / task | One fixed-size unit submitted to WinTask for processing. These terms refer to the same work item in this document. |
| Ingestor | The single producer thread that validates basic request fields, captures ingress time, classifies priority and publishes requests to the corresponding queue. |
| Consumer | One dedicated thread that drains exactly one priority queue and processes its requests. |
| Queue slot | One 32-byte `QueueSlot`-equivalent record containing a descriptor and payload. |
| Severity | Input value used to select the High, Medium or Low queue. |
| Priority tier | The destination processing tier selected from severity: High, Medium or Low. |
| Urgency | Additional request metadata retained for diagnostics. It does not change FIFO ordering in this proof of concept. |
| Ingress timestamp | The 64-bit raw `QueryPerformanceCounter` value captured when a request enters the system. |
| SPSC | Single-producer/single-consumer: one thread publishes requests to a queue and exactly one assigned thread consumes them. |
| Load shedding | Rejecting a new request when its selected queue has no available slot. |
| Offered request | A request presented to the ingestion path, whether or not it is accepted. |
| Accepted request | A valid request successfully published into its selected queue. |
| Completed request | An accepted request whose handler reaches the successful terminal state. |
| Failed request | An accepted request whose processing reaches the failure terminal state. |
| Capacity rejection | A valid request rejected because its selected queue is full. |
| Aggregate capacity | Sum of the capacities of all three queues. It is not a shared pool. |
| End-to-end latency | Elapsed time from ingress timestamp capture to the request's terminal completion/failure point. |

---

## 3. Technical requirements

### 3.1 Functional requirements

| ID | Requirement |
|---|---|
| FR-01 | WinTask shall accept requests through one dedicated ingestion thread. |
| FR-02 | Each valid request shall be assigned to exactly one priority queue using its severity value. |
| FR-03 | Severity values 1–2 shall route to Low; 3–4 to Medium; and 5 to High. Values outside 1–5 shall be rejected as invalid input and shall not be published. |
| FR-04 | Each priority queue shall be an independently allocated, bounded SPSC ring buffer. |
| FR-05 | The ingestor shall be the only writer of each queue's tail counter; the assigned consumer shall be the only writer of its head counter. |
| FR-06 | Each queue shall have exactly one dedicated consumer thread. |
| FR-07 | Each consumer shall preserve FIFO processing order within its own queue. |
| FR-08 | A full destination queue shall reject an incoming request and increment the capacity-rejection accounting. No occupied slot shall be overwritten. |
| FR-09 | A request shall not be transferred to another priority queue when its selected queue is full. |
| FR-10 | Each accepted request shall be dispatched to the handler associated with its declared job type. |
| FR-11 | Consumers shall record terminal result data in preallocated, consumer-isolated diagnostics storage. |
| FR-12 | CSV generation shall occur after the benchmark workload has drained and all consumer threads have terminated. |
| FR-13 | The system shall maintain aggregate counts for offered, valid, accepted, successfully completed, failed, invalid and capacity-rejected requests. |
| FR-14 | The benchmark shall calculate successfully completed request throughput and ingress-to-terminal-completion latency. |
| FR-15 | Normal shutdown shall stop further ingestion, drain accepted work, terminate consumers and join all worker threads before final result export. |
| FR-16 | The command-line interface shall provide separate validation, benchmark, full-evaluation and exit choices. |

### 3.2 Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | The combined request slot shall have a compile-time verified size of exactly 32 bytes. |
| NFR-02 | Each queue's backing storage shall begin at a 64-byte-aligned address. Individual slots shall retain a 32-byte stride. |
| NFR-03 | Descriptor and payload member offsets shall be verified at build time. |
| NFR-04 | Queue publication and slot reclamation shall use explicit acquire/release memory ordering. |
| NFR-05 | Producer-tail and consumer-head control variables shall occupy separate cache-line-aligned locations. |
| NFR-06 | Queue capacities shall be powers of two, permitting masked physical indexing. |
| NFR-07 | The steady-state ingestion and processing paths shall not format or synchronously write CSV output. |
| NFR-08 | Diagnostics storage shall be allocated before the timed benchmark interval and partitioned by consumer. |
| NFR-09 | The ingestor and consumer threads shall use the processor-affinity mapping selected for the project machine. |
| NFR-10 | AVX2-assisted 32-byte slot transfer shall be evaluated on the target machine; instruction count shall not be treated as a cycle-count guarantee. |
| NFR-11 | Benchmark output shall distinguish offered, accepted, completed, failed, invalid and capacity-rejected requests. |
| NFR-12 | The selected Clang/MSVC STL toolchain shall be the project's build and validation target. GCC is not part of this project's primary toolchain matrix. |
| NFR-13 | Queue and engine storage shall be initialized before the measured interval. No queue backing-store allocation shall occur per request. |

### 3.3 Performance objectives and interpretation

- **Primary stretch target:** 1,000,000 successfully completed requests per second for a declared workload, run duration, processor-affinity configuration and machine environment.
- **Aggregate pending capacity:** 1,310,720 slots across the three independent queues.
- **Queue-slot storage:** 40 MiB in total, excluding indices, diagnostics, executable/runtime state and other overhead.
- **Latency:** ingress-to-terminal-completion/failure, reported in microseconds with fractional precision derived from QPC ticks.
- **Synthetic processing estimate:** the originally proposed 300–700 ns processing interval is a workload-design estimate only. It is not an assumed measured result.
- **Throughput claims:** only successfully completed requests divided by the measurement interval qualify as completed throughput. Offered or accepted request rates shall not be reported as completed throughput.

---

## 4. Request data model and structural memory layout

### 4.1 Job categories and processing states

The initial job categories are:

| Job type | Purpose in the POC |
|---|---|
| `VALIDATE_SESSION` | Performs a deterministic synthetic validation operation using the request's user/session fields. It is not cryptographic authentication. |
| `TELEMETRY_LOG` | Performs a bounded synthetic telemetry-processing operation. The measured hot path does not synchronously write a log file. |
| `COMPUTE_SIGN` | Performs a deterministic bit-mixing/rotation workload over the supplied 64-bit input. It is not a cryptographic signature algorithm. |

The request state values are:

| State | Value | POC interpretation |
|---|---:|---|
| Ready | 0 | Published and waiting to be processed. |
| Running | 1 | Currently being processed by its assigned consumer. |
| Yielded | 2 | Reserved; yielding is not implemented. |
| Cancelled | 3 | Reserved; cancellation is not implemented. |
| Completed | 4 | Handler completed successfully. |
| Failed | 5 | Handler reached a failure outcome. |

The state occupies the lower three bits of the one-byte control field. The upper five bits remain available as flags. State operations must preserve the flag bits. The POC does not implement preemption, yielding or cancellation.

### 4.2 Approved 32-byte representation

The request representation retains a 16-byte descriptor and a 16-byte payload. The descriptor's 64-bit field is assigned to the ingress QPC timestamp. The payload retains the user identifier and synthetic session input; the former 32-bit microsecond timestamp position is repurposed as a compact request-specific parameter.

| Structure/member | Size | Offset within parent | Meaning |
|---|---:|---:|---|
| **JobDescriptor** | **16 B** | — | Request metadata and timing. |
| type | 1 B | 0 | Selects one of the three fixed job types. |
| severity | 1 B | 1 | Selects High, Medium or Low. |
| urgency | 1 B | 2 | Additional metadata, retained for reporting. |
| control | 1 B | 3 | State in lower three bits; upper five bits reserved for flags. |
| job_id | 4 B | 4 | Request identifier, unique within a benchmark run. |
| ingress_qpc_ticks | 8 B | 8 | Raw 64-bit QPC timestamp captured at ingress. |
| **JobPayload** | **16 B** | — | Compact operation-specific input. |
| user_id | 4 B | 0 | Synthetic requester/session identity input. |
| request_parameter | 4 B | 4 | Compact 32-bit parameter interpreted by the selected job type. |
| session_signature | 8 B | 8 | Synthetic 64-bit session/token input for deterministic processing. |
| **QueueSlot** | **32 B** | — | Descriptor followed immediately by payload. |

No path strings, dynamically allocated payloads or owning pointers are stored in a queue slot. The slot is a fixed-size, trivially transferable request record. The exact size and offsets are build-time requirements for the selected x64 target.

**Packing policy:** no packing pragma is required. The member order is intended to produce the stated natural layout for the selected target. Compile-time size/offset checks are authoritative. Queue backing storage is 64-byte aligned; the slot type itself remains 32 bytes and shall not be padded to 64 bytes.

### 4.3 Field semantics and constraints

| Field | Semantics |
|---|---|
| type | Selects `VALIDATE_SESSION`, `TELEMETRY_LOG` or `COMPUTE_SIGN`. Unsupported values are invalid. |
| severity | Values 1–2 map to Low, 3–4 to Medium, and 5 to High. |
| urgency | Retained as an 8-bit score and included in diagnostics. It does not reorder requests within a queue in this POC. |
| control | Contains the request state and reserved flags. A consumer sets Running and then a terminal state before releasing the slot. |
| job_id | Identifies the request in diagnostics and the CSV output. The benchmark avoids duplicate IDs within one run. |
| ingress_qpc_ticks | Captured once at ingress; used to calculate elapsed time until the terminal result is recorded. |
| user_id | Synthetic requester/session identifier used as input to the session-validation workload. |
| request_parameter | Compact job-specific input/metadata. Its interpretation is defined by the selected job type. |
| session_signature | Synthetic 64-bit input for the session/bit-transformation workload; it is not a real credential or cryptographic signature. |

The 64-bit ingress timestamp is stored in the descriptor's former generic 64-bit field. The payload's former 32-bit ingress timestamp field is named `request_parameter`; this avoids storing a redundant timestamp while preserving the 16-byte payload and 32-byte total slot.

### 4.4 Cache-line layout

The intended storage density is two 32-byte slots per assumed 64-byte cache line.

- Queue base: 64-byte aligned.
- Slot stride: 32 bytes.
- Slot starts: 32-byte aligned when the queue base is 64-byte aligned.
- Producer tail and consumer head: separate 64-byte-aligned control locations.
- A two-slot ownership/publication experiment may be evaluated if time permits, but it is not required for the baseline.

The two-slots-per-line layout is a density objective, not a guarantee that producer and consumer accesses will avoid cache-coherence traffic. A producer writing one slot while a consumer reads the adjacent slot can still cause line ownership transfers. The baseline retains compact slots; an experimental paired/padded variant may be compared without changing the required representation.

---

## 5. Queue topology and capacity

### 5.1 Queue capacities

| Priority queue | Severity mapping | Capacity | Mask | Storage |
|---|---:|---:|---:|---:|
| High | 5 | 262,144 (`2^18`) | `0x0003FFFF` | 8 MiB |
| Medium | 3–4 | 524,288 (`2^19`) | `0x0007FFFF` | 16 MiB |
| Low | 1–2 | 524,288 (`2^19`) | `0x0007FFFF` | 16 MiB |
| **Total** | — | **1,310,720** | — | **40 MiB** |

The aggregate capacity represents a one-million-slot baseline plus 310,720 additional slots, or 31.072% additional headroom. This headroom is deliberately distributed among the three queues.

**The additional capacity is not a shared reserve.** An empty Low queue cannot donate slots to a full High queue. No cross-queue borrowing or global overflow queue is part of this POC.

At an aggregate input rate of one million requests per second, 1,310,720 completely unused slots correspond to a theoretical 1.31072 seconds of buffered arrivals only under an idealized distribution that does not exhaust any individual queue first. This is a capacity illustration, not a traffic guarantee.

### 5.2 SPSC ownership

Each ring buffer has exactly one producer (the ingestor) and exactly one consumer (the thread assigned to that priority tier).

| Resource | Owner/writer | Other side |
|---|---|---|
| Tail publication counter | Ingestor | Consumer reads published value. |
| Head reclamation counter | Assigned consumer | Ingestor reads released value. |
| Slot before tail publication | Ingestor | Consumer must not read it. |
| Slot after tail publication and before head reclamation | Consumer reads/processes it | Ingestor must not overwrite it. |
| Slot after head release is observed | Ingestor may reuse it | Consumer has completed access to it. |

No generic MPMC queue or multi-producer reservation mechanism is required for this topology.

### 5.3 Counter and index rules

The logical head and tail counters are monotonically increasing unsigned 64-bit values. Physical indexing uses the counter masked by `Capacity - 1`. Since all capacities are powers of two, this maps the logical sequence onto the fixed ring storage.

Occupancy is derived from the unsigned difference between the producer's next tail position and the consumer's published head position. The invariant is that occupancy never exceeds the queue capacity. Counter wraparound shall be handled by unsigned arithmetic under this bounded-distance invariant.

The consumer's head is the **reclamation position** in this POC. The consumer processes the slot in place, records its terminal state and result, and only then publishes the next head position. The slot therefore remains owned by the consumer until processing and result recording are complete. This permits the consumer to update the slot state without racing with producer reuse.

---

## 6. Atomic synchronization and memory ordering

### 6.1 Publication and reclamation protocol

The queue protocol uses explicit C++ release/acquire ordering.

**Producer responsibilities:**

1. Read the consumer's published head with acquire semantics when refreshing the available-capacity view.
2. Determine whether the destination queue has space for the request or selected publication batch.
3. Write the complete request slot in the producer-owned tail position, including its Ready state.
4. Publish the advanced tail with release semantics.

**Consumer responsibilities:**

1. Read the producer's published tail with acquire semantics.
2. Treat equal local head and observed tail values as an empty queue.
3. Read the published request and transition its state to Running.
4. Execute the selected synthetic job handler.
5. Record the terminal state and per-job diagnostic result.
6. Publish the advanced head with release semantics, making the slot reusable.

The producer's acquire observation of the released head prevents premature slot reuse. The consumer's acquire observation of the released tail makes the producer's preceding slot writes visible before the consumer reads them.

This correctness argument is based on the C++ memory model. The x86-64 TSO behavior of the target machine is not a substitute for correct C++ atomic ordering.

### 6.2 Control-variable placement

The head and tail controls are independently cache-line aligned. Their independent writes must not share the same cache line. Queue slot storage remains separate from these controls.

### 6.3 Atomic capability requirement

The target build shall verify whether the 64-bit atomic counter operations used by the queue are lock-free on the project machine. The validation record shall capture the result. If the target does not provide the required lock-free behavior, the benchmark shall not describe the queue as lock-free.

### 6.4 Consumer waiting behavior

The consumer loop may use a bounded polling strategy appropriate to the benchmark. The selected strategy and any backoff behavior must be recorded because it can materially affect CPU usage, wake-up latency and throughput. The POC does not require a general-purpose blocking scheduler.

---

## 7. Ingestion, routing and load shedding

### 7.1 Ingestor responsibilities

The single ingestor:

- receives or generates the next request from the in-process workload source;
- increments offered-request accounting;
- validates the supported job type and severity range;
- captures the ingress QPC timestamp;
- selects the destination queue from severity;
- checks the selected queue's available capacity;
- writes the request record and publishes it;
- updates accepted or rejected accounting.

The benchmark workload source is in-process. No socket listener or external HTTP request parser is part of this POC.

### 7.2 Severity routing

| Severity | Priority tier | Destination consumer |
|---:|---|---|
| 1–2 | Low | Low consumer |
| 3–4 | Medium | Medium consumer |
| 5 | High | High consumer |
| Outside 1–5 | Invalid | Not queued |

Priority is implemented through tier classification and dedicated queue/consumer assignment. Each tier progresses independently. This is not global preemption or a single shared worker pool that always selects the highest-priority item across all tiers.

### 7.3 Load shedding

If the selected queue is full, the ingestor rejects the incoming request. It shall not overwrite an occupied slot, wait indefinitely for space, borrow capacity from another tier or place the request into a global fallback queue.

The rejection is recorded in diagnostics/accounting as a capacity rejection and may be labelled as a simulated HTTP 429 event for the server-overload demonstration. WinTask does not send an actual HTTP response.

### 7.4 AVX2-assisted ingestion

The intended SIMD path transfers request records in batches of two. Each `QueueSlot` is 32 bytes, matching one 256-bit AVX2 vector. Therefore, one 32-byte vector store corresponds to one complete slot, and two such stores correspond to a 64-byte pair. The two slots are not written by one 256-bit store.

A source-level intrinsic call does not imply a fixed one-cycle completion time. Alignment, load/store throughput, cache state, execution resources and memory behavior influence the result. The AVX2 path shall be compared against a correctness-equivalent scalar/reference path. Any incomplete final pair must be handled without publishing an uninitialized slot.

---

## 8. Worker and request-processing design

### 8.1 Thread topology and affinity

The baseline consists of four dedicated threads:

| Thread | Function | Configured processor/core assignment |
|---|---|---:|
| Ingestor | Validates, timestamps, routes and publishes requests | 0 |
| High consumer | Processes High queue | 2 |
| Medium consumer | Processes Medium queue | 4 |
| Low consumer | Processes Low queue | 6 |

These are the project-selected processor assignments for the target machine and shall be used consistently by the validator and benchmark. The run record shall capture the actual affinity configuration used. The POC does not implement a general-purpose CPU scheduler or dynamic affinity management.

### 8.2 Consumer responsibilities

Each consumer is responsible for exactly one priority queue. It observes published work, claims the next FIFO slot, transitions the state to Running, executes the corresponding handler, records the terminal result and releases the slot by advancing its head counter.

Consumers do not steal requests from another tier. A consumer that has no work in its assigned queue does not consume another priority tier's queue.

### 8.3 Job processing stages

The synthetic processing pipeline includes:

1. **Latency profiling:** read the ingress QPC timestamp and capture the terminal QPC timestamp to calculate ingress-to-completion latency.
2. **Synthetic session/signature transformation:** process the 64-bit session input with deterministic bitwise operations and rotations. This is a controlled CPU workload and does not perform cryptographic verification.
3. **Job-type handling:** execute the deterministic operation associated with `VALIDATE_SESSION`, `TELEMETRY_LOG` or `COMPUTE_SIGN`.
4. **Urgency retention:** preserve the urgency value in the request and diagnostic record.
5. **Terminal accounting:** record Completed or Failed and publish the corresponding diagnostic result before releasing the queue slot.

The originally estimated 300–700 ns workload is not hard-coded as an assumed duration. The measured result depends on the actual handler operations and test machine.

### 8.4 Urgency semantics

Urgency is retained as an 8-bit field and included in diagnostics. It does not reorder the FIFO queue, interrupt a running job, skip processing stages or suppress telemetry in the POC. This keeps the queue's FIFO property and the measurement path consistent. Any future urgency-based scheduling policy requires a separate design revision and measurement.

---

## 9. Timestamping, diagnostics and reporting

### 9.1 Ingress timestamp and latency

The descriptor stores the 64-bit raw QPC value captured by the ingestor. QPC frequency is obtained once during initialization and reused for conversions during the run.

For a request with ingress counter value `QPC_in` and terminal counter value `QPC_out`:

- elapsed ticks = `QPC_out - QPC_in`;
- elapsed seconds = elapsed ticks divided by the cached QPC frequency;
- reported latency = elapsed seconds converted to microseconds, retaining fractional precision.

The reported quantity is **ingress-to-terminal-completion/failure latency**. It includes time spent waiting in the priority queue and time spent in the consumer's processing stages. It is not merely handler execution time.

### 9.2 Per-consumer diagnostics storage

Each consumer writes result records into its own preallocated diagnostics array. The arrays are prepared before the measured run and are not shared for concurrent append operations.

The minimum per-job result fields are:

- Job ID;
- priority class;
- urgency;
- ingress-to-terminal-completion latency in microseconds;
- terminal status.

The diagnostics design shall avoid dynamic allocation and CSV formatting in the steady-state consumer path. Capacity shall be sufficient for the configured benchmark workload or the run shall be marked invalid if result storage cannot represent every required terminal record.

### 9.3 Aggregate counters

WinTask shall track at least:

- total requests offered;
- valid requests offered;
- accepted requests;
- successfully completed requests;
- failed requests;
- invalid requests;
- capacity-rejected requests.

After all accepted work has reached a terminal state and the queues have drained, the following accounting identities shall hold:

- valid requests offered = accepted requests + capacity-rejected requests;
- accepted requests = successfully completed requests + failed requests.

Invalid requests are counted separately and are not included in the valid-offer identity.

### 9.4 CSV output

The benchmark exports `TestResults.csv` after ingestion is stopped, all accepted work is drained, all consumers have terminated and diagnostics arrays are stable.

Minimum per-job columns:

`JobID, PriorityClass, Urgency, LatencyMicroseconds, Status`

The final output may include a separate aggregate summary or summary section/file if needed, but the per-job record schema remains stable for the user's offline Python analysis workflow. CSV generation is outside the measured hot path.

### 9.5 Completed throughput

Completed throughput is calculated as:

**Successfully completed requests ÷ measurement duration in seconds.**

The report shall show offered, accepted, completed, failed and rejected counts alongside the throughput result. A run that has not drained all accepted requests is incomplete and shall not be presented as a final completed-throughput result.

---

## 10. Cache, alignment and SIMD evaluation

### 10.1 Baseline memory arrangement

The baseline retains 32-byte request slots and 64-byte-aligned queue backing storage. This produces two slot positions per assumed 64-byte cache line and an aggregate 40 MiB queue-storage footprint.

### 10.2 Control-line separation

Head and tail counters are placed on separate cache-line-aligned locations. This is part of the baseline design because they are independently written by different threads.

### 10.3 Slot-line interference consideration

A producer and consumer can access different 32-byte slots that occupy the same 64-byte cache line. Although their C++ objects are distinct, cache coherence can still transfer ownership at cache-line granularity.

The initial implementation retains the compact 32-byte format. If the 10-day schedule permits, a paired-ownership or padded experimental variant may be evaluated. Such a variant is optional and shall not displace correctness work. It must be compared using both throughput and latency because retaining two requests as one ownership unit can delay reclamation of the first request.

### 10.4 SIMD evaluation

AVX2 is 256 bits wide, matching the size of one 32-byte slot. Two complete slots occupy 64 bytes and require two 256-bit transfers. The benchmark shall compare the AVX2-assisted path with a reference copy path using identical request contents and workloads. No fixed instruction-cycle claim is part of the specification.

---

## 11. Source-file and module organization

The project uses the following six-file source layout. The structure is intentionally compact: logical responsibilities are grouped where their implementations are small, while the engine and evaluation components remain clearly separated.

| File | Responsibility |
|---|---|
| `WinTask.cpp` | The sole application entry point. Initializes the command-line interface and returns its exit status. It contains no separate engine, validator or benchmark implementation. |
| `SystemInterface.hpp` | Provides the command-line menu and dispatches the user's selection to validation, benchmarking, full evaluation or exit. It imports the validator and benchmark interfaces. |
| `WinTaskValidator.hpp` | Contains the 40 documented test cases, their grouped execution flow, pass/fail accounting and validation summary. It exercises the same engine used by the benchmark. |
| `WinTaskBenchmark.hpp` | Owns workload generation and benchmark orchestration, uses the engine, applies the configured affinity plan, collects measurements and exports `TestResults.csv`. |
| `WinTaskUtilities.hpp` | Groups small shared platform/utilities that do not justify separate files, including QPC timing support, configured thread-affinity support and narrowly scoped common helpers. It does not own the request model or scheduler. |
| `WinTaskEngine.hpp` | The core execution engine: configuration, job/state types, descriptor/payload/slot model, internal SPSC queue implementation, priority queues, ingestion/routing, consumer loops, synthetic job handlers, aggregate counters, per-consumer diagnostics storage and engine lifecycle. It uses the shared utilities. |

### 11.1 Conceptual dependency arrangement

The intended dependency direction is:

- `WinTask.cpp` initializes `SystemInterface`.
- `SystemInterface.hpp` imports `WinTaskValidator.hpp` and `WinTaskBenchmark.hpp`.
- `WinTaskValidator.hpp` and `WinTaskBenchmark.hpp` both depend on `WinTaskEngine.hpp`.
- `WinTaskEngine.hpp` depends on `WinTaskUtilities.hpp` for shared platform support.
- `WinTaskUtilities.hpp` does not depend on the engine, validator, benchmark or interface.

The engine does not import or depend on the CLI, validator or benchmark. The validator and benchmark use the same engine implementation rather than maintaining separate runtime implementations.

This file layout is an initial conceptual overview of how the project should be organized and how its responsibilities should interact. It is not source code or a commitment to a particular class hierarchy. Small related types and operations may remain grouped within their assigned file; additional files shall only be introduced when a concrete implementation or reuse requirement justifies them.

### 11.2 Command-line choices

The command-line interface shall expose four choices:

1. Run Validator — execute the 40 documented test cases and show the validation summary.
2. Run Benchmark — execute the configured workload and export benchmark results.
3. Run Full Evaluation — run validation first and start the benchmark only if validation succeeds.
4. Exit — terminate the application cleanly.

The menu coordinates these operations; it does not contain queue, scheduling or request-processing logic.

### 11.3 Build outputs

- `WinTask.exe` — the single application containing the CLI, validator, benchmark and engine.
- `TestResults.csv` — generated benchmark output after a completed run.

The validator's test summary may be displayed in the console. No separate validator or benchmark executable is required for this POC.

---

## 12. Build and runtime configuration

- Target: Windows x64.
- Compiler: Clang 23.1.5 targeting `x86_64-pc-windows-msvc`.
- Standard library: MSVC STL from the selected Microsoft Build Tools for Visual Studio 2026 installation.
- Language baseline: C++20; C++23 may be enabled for specific supported features.
- SIMD target: AVX2 for the planned request-slot transfer path on the project machine.
- Platform timing: Windows performance-counter facilities.
- Processor affinity: project-selected mapping of ingestor to 0 and High/Medium/Low consumers to 2/4/6, as recorded for the target machine.
- GCC 16.1 is not part of this project's primary build/validation matrix.
- Benchmark measurements shall use an optimized Release build. Correctness validation may additionally be performed in an instrumented or Debug build; timings from different build modes shall not be compared as equivalent.

The final run record shall capture the compiler version, target, STL/toolset, Windows build, CPU, build configuration, affinity mapping, workload distribution and measurement duration.

---

## 13. Known limitations and interpretation rules

1. The three queues have fixed capacities and cannot borrow unused capacity from one another.
2. A dedicated consumer per priority tier means an idle consumer does not process another tier's requests.
3. Priority is represented by classification into dedicated queues and consumers; there is no global preemption or shared-pool priority arbitration.
4. FIFO is maintained within each queue. Urgency is recorded but does not reorder requests in this POC.
5. The system simulates server-style request processing in-process; it does not implement network ingress or an HTTP server.
6. HTTP 429 is a simulated capacity-rejection label, not a transmitted protocol response.
7. The session/signature transformation is synthetic and provides no cryptographic security.
8. The 1,000,000 requests/second figure is a stretch target, not a promised outcome. Results are meaningful only with their workload, run duration, processor affinity, build configuration and accounting counts.
9. The 64-byte cache-line assumption is the target-machine optimization assumption, not a universal guarantee for every processor.
10. The original 300–700 ns processing estimate is provisional until measured.
11. The additional 310,720 slots are distributed headroom, not capacity that can be dynamically reassigned between queues.
12. Optional cache-line ownership experiments are not required for acceptance of the baseline POC.

---

## 14. POC acceptance criteria

The POC is ready for its final demonstration when:

- [ ] The request structures satisfy the documented size and offset requirements; the complete slot is 32 bytes.
- [ ] All three ring buffers initialize with the specified capacities and masks.
- [ ] Queue backing storage and control variables satisfy the documented alignment requirements.
- [ ] The ingestor routes valid severity values to the correct queue and rejects invalid values.
- [ ] The SPSC queues pass empty, FIFO, full, wraparound, publication and reclamation validation.
- [ ] Concurrent stress validation detects no lost, duplicated or corrupted accepted request IDs.
- [ ] Each consumer processes only its assigned queue.
- [ ] State transitions and terminal result records are correct.
- [ ] Queue-full conditions reject rather than overwrite requests.
- [ ] QPC-based ingress-to-terminal latency is calculated and exported with fractional-microsecond precision.
- [ ] Diagnostics storage can represent every terminal result in the configured run.
- [ ] Offered/accepted/completed/failed/invalid/rejected accounting reconciles after drain.
- [ ] The configured processor-affinity mapping is applied and verified.
- [ ] Both the scalar/reference and AVX2 request-copy paths, if enabled, preserve identical request data.
- [ ] The CLI's validation, benchmark, full-evaluation and exit choices behave as specified.
- [ ] `WinTask.exe` builds with the selected Clang/MSVC STL toolchain.
- [ ] The benchmark exports a valid `TestResults.csv` after all consumers terminate.
- [ ] Performance results state the workload and do not confuse offered rate with completed throughput.

---

## 15. Technical references

The following platform and language references support the implementation decisions. The project-specific requirements in this document remain the controlling design baseline.

1. Microsoft Learn — [QueryPerformanceCounter](https://learn.microsoft.com/en-us/windows/win32/api/profileapi/nf-profileapi-queryperformancecounter): high-resolution performance-counter acquisition.
2. Microsoft Learn — [QueryPerformanceFrequency](https://learn.microsoft.com/en-us/windows/win32/api/profileapi/nf-profileapi-queryperformancefrequency): performance-counter frequency acquisition.
3. C++ reference — [std::memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order): acquire/release synchronization and visibility of data published through atomics.
4. Microsoft Learn — [SetThreadAffinityMask](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setthreadaffinitymask): Windows thread processor-affinity API.

---

## 16. Change-control notes

This revision establishes the following baseline:

- WinTask is a **High-Performance Priority-Based Task Scheduler & Processor for Windows**.
- The six-file layout is `WinTask.cpp`, `SystemInterface.hpp`, `WinTaskValidator.hpp`, `WinTaskBenchmark.hpp`, `WinTaskUtilities.hpp` and `WinTaskEngine.hpp`.
- `WinTaskEngine.hpp` owns the core engine and internal SPSC implementation.
- `SystemInterface.hpp` is the CLI entry to validation, benchmarking, full evaluation and exit.
- The validator and benchmark use the same engine implementation.
- The 40-case validation baseline is defined in the companion Test Case Specification & Execution Record.
- The request slot remains 32 bytes; its 64-bit descriptor timestamp is the ingress QPC value.
- The payload's 32-bit middle field is request-specific metadata, not a second timestamp or a policy identifier.
- Urgency remains diagnostic metadata; FIFO order is retained inside each priority queue.
- The POC retains three fixed SPSC queues, one ingestor and three dedicated consumers. No global overflow queue, Signal Tree or dynamic worker pool is introduced.

Any later change to slot size, queue capacities, ownership, memory ordering, thread topology, job semantics, source-file responsibilities or throughput accounting shall be recorded in this document before the implementation is considered conformant to this baseline.
