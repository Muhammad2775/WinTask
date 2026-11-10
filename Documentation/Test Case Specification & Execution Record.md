# WinTask — Test Case Specification & Execution Record

**Project:** WinTask — High-Performance Priority-Based Task Scheduler & Processor for Windows  
**Document version:** 0.2 — Revised 40-case Proof-of-Concept baseline  
**Status:** Prepared for execution; all tests are initially NOT RUN  
**Academic context:** Semester 5 — Operating Systems (Lab Project)  
**Platform:** Windows x64  
**Primary toolchain:** Clang 23.1.5, `x86_64-pc-windows-msvc`, MSVC STL, Microsoft Build Tools for Visual Studio 2026  
**Companion document:** Design & Specification Document.md

---

## 1. Purpose

This document defines the consolidated validation and benchmark cases for WinTask. It is the execution record for determining whether the implementation conforms to the approved design and for recording measured performance.

The suite contains **exactly 40 test cases** grouped by subsystem. A test case may contain several related assertions or parameterized sub-runs where they verify the same system property. Consolidation reduces administrative duplication; it does not remove the distinct correctness conditions that the case is intended to establish.

The test suite covers request layout, configuration, ingestion, SPSC queue behavior, concurrency, lifecycle, load shedding, the three synthetic job handlers, urgency semantics, processor affinity, diagnostics, CSV output and performance.


---

## 2. Test environment record

Complete this section before recording final results. The environment should remain consistent across comparative benchmark runs.

| Item | Recorded value |
|---|---|
| CPU model | NOT RECORDED |
| Physical core count | NOT RECORDED |
| Logical processor count | NOT RECORDED |
| RAM configuration | NOT RECORDED |
| Windows edition/build | NOT RECORDED |
| Compiler version | Clang 23.1.5 (verify actual installed build) |
| Compiler target | `x86_64-pc-windows-msvc` |
| Standard library | MSVC STL |
| Build tools | Microsoft Build Tools for Visual Studio 2026 |
| Language mode | C++20 baseline; C++23 only where explicitly enabled |
| Build configuration | Release for performance; validation configuration recorded separately |
| AVX2 availability | NOT VERIFIED |
| 64-bit atomic lock-free result | NOT VERIFIED |
| QPC frequency | NOT RECORDED |
| Processor-affinity mapping | Ingestor 0; High 2; Medium 4; Low 6 (verify during test run) |
| Source revision | NOT RECORDED |
| Test date/time | NOT RECORDED |

---

## 3. Test strategy

### 3.1 Test groups

| Group | Coverage | Test IDs | Count |
|---|---|---:|---:|
| A | Structure and configuration | TC-001–TC-004 | 4 |
| B | Request ingestion and data integrity | TC-005–TC-008 | 4 |
| C | SPSC queue correctness | TC-009–TC-016 | 8 |
| D | Concurrency and lifecycle | TC-017–TC-020 | 4 |
| E | Load shedding and accounting | TC-021–TC-024 | 4 |
| F | Job handlers and request semantics | TC-025–TC-032 | 8 |
| G | Windows platform and diagnostics | TC-033–TC-035 | 3 |
| H | Performance and scalability | TC-036–TC-040 | 5 |
| **Total** | | **TC-001–TC-040** | **40** |

### 3.2 Execution rules

- Structural and deterministic correctness cases should pass before high-volume benchmark runs begin.
- Concurrency cases shall use unique request IDs or another independent sequence-integrity method so loss, duplication and corruption can be detected.
- A test that exercises all three queues shall verify each queue independently as well as aggregate results.
- Queue-full behavior shall be tested without relying on timing luck; controlled fill/drain conditions shall be used.
- Performance tests shall use an optimized Release build, a recorded affinity mapping, a defined workload and a measured interval.
- Comparative tests shall use the same input sequence and workload settings for each compared method.
- CSV export shall occur after accepted work has drained and consumer threads have terminated.
- A failed correctness precondition invalidates dependent performance results; a benchmark must not be reported as successful if requests are silently lost or accounting does not reconcile.
- Optional experiments that are not part of the baseline 40 cases shall be clearly labelled exploratory and shall not replace a required case.

### 3.3 Status vocabulary

| Status | Meaning |
|---|---|
| NOT RUN | No execution has been recorded. |
| PASS | All expected results for the case were observed. |
| FAIL | One or more expected results were not observed. |
| BLOCKED | The case could not be run because a required prerequisite was unavailable. |
| INVALID | The run occurred, but the environment or execution conditions make its result unusable. |

---

## 4. Group A — Structure and configuration (4 tests)

| ID | Test case | Procedure / coverage | Expected result | Status |
|---|---|---|---|---|
| TC-001 | Request structure validation | Verify descriptor, payload and combined slot sizes; member offsets; descriptor/payload ordering; and trivially transferable request representation. | Descriptor is 16 bytes, payload is 16 bytes, slot is exactly 32 bytes, and all documented offsets match. | NOT RUN |
| TC-002 | Queue configuration validation | Verify all three configured capacities, power-of-two properties, masks, total slot count and calculated queue-storage footprint. | High is 262,144 slots; Medium and Low are 524,288 each; aggregate capacity is 1,310,720 slots and queue storage is 40 MiB. | NOT RUN |
| TC-003 | Queue and control alignment | Inspect queue backing-store base alignment, slot stride/alignment, and independent cache-line placement of head and tail controls. | Queue base is 64-byte aligned, slots retain a 32-byte stride, and head/tail controls do not share a cache line. | NOT RUN |
| TC-004 | Atomic and SIMD capability checks | Verify the target's 64-bit atomic lock-free property and AVX2 availability required by the configured build path. | Results are recorded explicitly. A non-lock-free counter implementation is not described as lock-free; unsupported AVX2 execution is not attempted. | NOT RUN |

---

## 5. Group B — Request ingestion and data integrity (4 tests)

| ID | Test case | Procedure / coverage | Expected result | Status |
|---|---|---|---|---|
| TC-005 | Severity routing validation | Submit representative requests for severity 1, 2, 3, 4 and 5. | Severities 1–2 route to Low, 3–4 to Medium and 5 to High; no request is routed to more than one queue. | NOT RUN |
| TC-006 | Invalid severity rejection | Submit severity values below and above the supported range, including boundary-invalid values. | Invalid requests are rejected before queue publication and counted separately from capacity rejection. | NOT RUN |
| TC-007 | Job-type classification and dispatch | Submit representative requests for each supported job type and verify handler selection. | Each supported type selects its corresponding handler; no type is silently treated as another type. | NOT RUN |
| TC-008 | Request-field preservation and ingress timestamp | Submit identifiable request records and verify all descriptor/payload fields after queue publication; capture ingress QPC values for a controlled sequence. | IDs, severity, urgency, control, user ID, request parameter and session input remain unchanged; ingress timestamps are captured and usable for elapsed-time conversion. | NOT RUN |

---

## 6. Group C — SPSC queue correctness (8 tests)

| ID | Test case | Procedure / coverage | Expected result | Status |
|---|---|---|---|---|
| TC-009 | Empty queue behavior | Initialize an empty queue and attempt to observe/consume work. | The queue is reported empty; no slot is read as a valid request. | NOT RUN |
| TC-010 | Enqueue, dequeue and FIFO ordering | Publish a known sequence of requests and consume the sequence. Include a single-item and multi-item sequence. | Every published request is consumed once and in publication order. | NOT RUN |
| TC-011 | Full queue handling | Fill a queue to its configured capacity and attempt one further publication. | The queue reports no capacity; the additional request is rejected and no occupied slot is overwritten. | NOT RUN |
| TC-012 | Wraparound and slot reuse | Repeatedly enqueue and dequeue beyond the physical end of the ring, reusing released slots. | Masked indexing wraps correctly; released slots are reused without corruption or ordering errors. | NOT RUN |
| TC-013 | Occupancy and counter invariants | Exercise empty, partially occupied, full and repeated wraparound states; include a controlled near-counter-wrap validation where practical. | Unsigned counter differences represent occupancy correctly and occupancy never exceeds capacity. | NOT RUN |
| TC-014 | Producer publication visibility | Publish requests containing distinct recognizable field patterns while the consumer reads under sustained operation. | The consumer never observes a partially initialized or mixed request record. | NOT RUN |
| TC-015 | Consumer reclamation safety | Hold a consumer on a selected request while the producer approaches the corresponding ring position. | The producer does not reuse or overwrite the slot until the consumer has completed access and released it. | NOT RUN |
| TC-016 | Sustained SPSC sequence-integrity stress | Run a high-volume producer/consumer sequence with unique IDs and verifiable payload patterns. | No accepted ID is lost, duplicated, reordered within the queue or returned with corrupted payload data. | NOT RUN |

---

## 7. Group D — Concurrency and lifecycle (4 tests)

| ID | Test case | Procedure / coverage | Expected result | Status |
|---|---|---|---|---|
| TC-017 | Three-queue simultaneous execution | Generate mixed-priority traffic and allow all three consumers to operate concurrently. | Each consumer processes only its assigned queue; all three queues make progress when work is available. | NOT RUN |
| TC-018 | Priority-tier isolation and independent progress | Apply an uneven workload, including a heavily loaded tier and a lightly loaded tier. | Queue capacities and consumer ownership remain independent; no cross-tier borrowing or work stealing occurs. | NOT RUN |
| TC-019 | Job-state transition validation | Observe representative successful and failed jobs through Ready, Running and terminal states. Also verify reserved states are not used by the POC path. | Valid requests reach the appropriate terminal state; Yielded and Cancelled remain unused. State/flag representation remains valid. | NOT RUN |
| TC-020 | Graceful shutdown and drain | Stop request generation while work is pending; allow accepted requests to finish and join consumers. | No accepted request is abandoned during normal shutdown; consumers terminate and shutdown completes after the queues drain. | NOT RUN |

---

## 8. Group E — Load shedding and accounting (4 tests)

| ID | Test case | Procedure / coverage | Expected result | Status |
|---|---|---|---|---|
| TC-021 | Per-tier saturation | Run a controlled fill-and-overflow sub-run for High, Medium and Low independently. | Each queue accepts up to its own capacity, rejects beyond capacity and does not consume another tier's unused slots. | NOT RUN |
| TC-022 | Load-shedding classification | Submit valid requests to a full destination queue and inspect rejection accounting/diagnostic classification. | Each capacity rejection is recorded exactly once and may be labelled as a simulated HTTP 429 event; no actual HTTP response is claimed. | NOT RUN |
| TC-023 | Request-accounting reconciliation | Complete a run containing valid accepted requests, invalid inputs, capacity rejections, successful handler results and controlled handler failures. | After drain: valid offered = accepted + capacity rejected; accepted = successful + failed; invalid requests are accounted separately. | NOT RUN |
| TC-024 | Queue recovery after saturation | Saturate a queue, stop or reduce ingress, allow the consumer to drain, then submit additional valid requests. | Occupancy falls as requests are reclaimed; newly available slots accept subsequent requests without requiring queue reinitialization. | NOT RUN |

---

## 9. Group F — Job handlers and request semantics (8 tests)

| ID | Test case | Procedure / coverage | Expected result | Status |
|---|---|---|---|---|
| TC-025 | Session-validation handler | Run known valid and invalid synthetic session input combinations using the same declared validation rules. | The handler returns the expected deterministic outcome for each input; no cryptographic-security claim is made. | NOT RUN |
| TC-026 | Telemetry-processing handler | Submit representative telemetry jobs with known input fields and inspect their processing result. | The handler produces the expected deterministic in-memory processing outcome without synchronous CSV output in the consumer hot path. | NOT RUN |
| TC-027 | Compute/sign transformation handler | Submit known 64-bit inputs and repeat identical requests. | The bit-mixing/rotation transformation is deterministic; identical inputs under the same configuration produce identical results. | NOT RUN |
| TC-028 | Urgency-field preservation | Submit requests with different urgency values and verify the values in consumed records and diagnostics. | Urgency is preserved exactly and reported; it does not alter FIFO ordering in the POC. | NOT RUN |
| TC-029 | Terminal-state and status consistency | Exercise successful and failed handler outcomes and compare slot state with the result record. | Completed requests have Completed status; failed handler requests have Failed status; result and state agree. | NOT RUN |
| TC-030 | Unsupported job-type handling | Submit a request with an unsupported job-type value through the controlled validation path. | The request is rejected or marked invalid before an undefined handler is invoked; invalid input is accounted separately. | NOT RUN |
| TC-031 | Handler-failure containment | Force a controlled handler failure while other valid requests remain queued. | The failed request is recorded as Failed; the assigned consumer remains operational and continues processing subsequent requests. | NOT RUN |
| TC-032 | Mixed job-type processing | Submit all three supported job types across all priority tiers in a declared mixed workload. | Each request reaches the correct handler and priority consumer; IDs, job types and terminal results remain correctly associated. | NOT RUN |

---

## 10. Group G — Windows platform and diagnostics (3 tests)

| ID | Test case | Procedure / coverage | Expected result | Status |
|---|---|---|---|---|
| TC-033 | Thread-affinity validation | Start the ingestor and each consumer; inspect the configured affinity of all four threads. | Ingestor uses configured processor 0; High, Medium and Low consumers use configured processors 2, 4 and 6 respectively. The actual mapping is recorded. | NOT RUN |
| TC-034 | QPC timing and latency conversion | Verify QPC frequency initialization, monotonic ingress/terminal samples and conversion of a known tick delta to fractional microseconds. | Frequency is valid and cached; conversion is correct within expected numeric precision; measured latency is ingress-to-terminal rather than handler-only time. | NOT RUN |
| TC-035 | Diagnostics isolation and CSV validation | Run representative mixed jobs; verify consumer-isolated result records, terminal record completeness, output schema, row count, numeric parsing and aggregate reconciliation. | Every accepted request reaching a terminal result has one corresponding diagnostic record; `TestResults.csv` is generated after shutdown and is parseable by the offline analysis workflow. | NOT RUN |

---

## 11. Group H — Performance and scalability (5 tests)

Performance tests shall use an optimized Release build after initialization and affinity setup. Record the exact request mix, offered volume, measurement duration and run configuration. Repeat measurements sufficiently to avoid treating one favorable run as conclusive.

| ID | Benchmark | Workload / procedure | Measurements and expected interpretation | Status |
|---|---|---|---|---|
| TC-036 | Baseline mixed-traffic benchmark | Generate a declared mixture of High, Medium and Low requests below queue saturation. | Record offered, accepted, completed, failed and rejected counts; completed requests/second; and latency distribution. Use as the baseline result. | NOT RUN |
| TC-037 | One-million-request volume | Submit at least 1,000,000 uniquely identified requests and allow the accepted workload to drain. | Demonstrate request-volume handling and reconcile all counts. This tests total workload volume, not one-million-per-second throughput. | NOT RUN |
| TC-038 | One-million requests/second target | Run the declared benchmark profile over a measured interval and calculate successfully completed requests divided by that interval. | Report the measured completed requests/second. State whether the stretch target was reached; do not infer it from offered or accepted rate. | NOT RUN |
| TC-039 | Priority-distribution sensitivity | Compare balanced and deliberately skewed severity distributions while keeping other workload parameters controlled. | Record per-tier accepted/completed/rejected counts, peak occupancy, total completed throughput and latency. Explain tier-specific saturation where observed. | NOT RUN |
| TC-040 | AVX2 versus reference slot transfer | Compare the AVX2-assisted 32-byte slot transfer with a correctness-equivalent reference path using identical request data and workload settings. | Both paths preserve identical records and accounting. Report measured throughput/latency differences; do not assume a fixed cycle cost. | NOT RUN |

---

## 12. Required benchmark summary

Complete this summary for each final benchmark run.

| Metric | Result |
|---|---|
| Workload request count | NOT RECORDED |
| Severity distribution | NOT RECORDED |
| Job-type distribution | NOT RECORDED |
| Measurement duration | NOT RECORDED |
| Requests offered | NOT RECORDED |
| Valid requests offered | NOT RECORDED |
| Requests accepted | NOT RECORDED |
| Requests completed successfully | NOT RECORDED |
| Requests failed | NOT RECORDED |
| Invalid requests | NOT RECORDED |
| Capacity-rejected requests | NOT RECORDED |
| Completed throughput (requests/second) | NOT RECORDED |
| Median ingress-to-terminal latency | NOT RECORDED |
| 95th percentile latency | NOT RECORDED |
| 99th percentile latency | NOT RECORDED |
| Maximum observed latency | NOT RECORDED |
| High queue peak occupancy | NOT RECORDED |
| Medium queue peak occupancy | NOT RECORDED |
| Low queue peak occupancy | NOT RECORDED |
| Ingestor/consumer affinity mapping | NOT RECORDED |
| CSV output path | NOT GENERATED |

Latency percentiles may be calculated offline from the recorded per-job results. Offline analysis is not an additional hot-path instrumentation requirement.

---

## 13. Test execution record template

Copy this record for each test campaign or benchmark scenario.

### Run record — [Run ID]

| Item | Value |
|---|---|
| Date/time | NOT RECORDED |
| Executable/menu operation | WinTask.exe — Validator / Benchmark / Full Evaluation |
| Build configuration | NOT RECORDED |
| Source revision | NOT RECORDED |
| Test case IDs | NOT RECORDED |
| Machine/CPU | NOT RECORDED |
| Windows build | NOT RECORDED |
| Compiler/STL/toolset | NOT RECORDED |
| Thread-affinity mapping | NOT RECORDED |
| Workload seed and distribution | NOT RECORDED |
| Measurement duration | NOT RECORDED |
| Result file(s) | NOT RECORDED |

**Observed result:**  
_To be completed after execution._

**Expected versus actual:**  
_To be completed after execution._

**Status:** `NOT RUN`

**Evidence:**  
_Add console output, CSV filename, relevant screenshots or reproducible notes after execution._

**Defect/reference:**  
_Complete if the test fails or is blocked._

---

## 14. Defect and deviation log

| ID | Date | Test case | Description | Severity | Resolution | Status |
|---|---|---|---|---|---|---|
| — | — | — | No execution results recorded yet. | — | — | — |

Any deviation from the Design & Specification Document shall be recorded here, including changes to request layout, queue capacity, ownership, memory ordering, processor affinity, timestamp representation, job semantics, source-file responsibilities or throughput accounting.

---

## 15. Final test sign-off

| Acceptance item | Status |
|---|---|
| Request layout and configuration tests pass | NOT RUN |
| Ingestion and request-field integrity tests pass | NOT RUN |
| SPSC queue correctness tests pass | NOT RUN |
| Concurrency and lifecycle tests pass | NOT RUN |
| Load-shedding and accounting tests pass | NOT RUN |
| All three synthetic job handlers behave as specified | NOT RUN |
| Urgency metadata and FIFO semantics are correct | NOT RUN |
| Processor affinity is verified | NOT RUN |
| QPC latency conversion is verified | NOT RUN |
| Diagnostics and CSV output are validated | NOT RUN |
| One-million-request volume run is completed | NOT RUN |
| One-million requests/second target is measured and reported | NOT RUN |
| AVX2/reference comparison is recorded | NOT RUN |
| Final benchmark results and environment are documented | NOT RUN |

**Final result:** NOT RUN  
**Executed by:** NOT RECORDED  
**Execution date:** NOT RECORDED  
**Remarks:** To be completed after implementation and execution.
