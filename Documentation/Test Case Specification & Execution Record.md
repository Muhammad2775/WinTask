# WinTask — Test Case Specification & Execution Record

**Document type:** Test case specification and execution record  
**Project:** WinTask  
**Document version:** 0.1 — Proof-of-Concept baseline  
**Status:** Test cases specified; execution pending  
**Target:** Windows x64, Clang 23.1.5 + MSVC STL, Visual Studio 2026 Build Tools

> **Execution-status rule:** All test cases begin as `NOT RUN`. Do not change a status to `PASS` until the test has actually been executed and its evidence recorded. This document contains expected outcomes, not fabricated test results.

---

## 1. Purpose

This document defines the validation and benchmark cases for WinTask. It is used to verify the 32-byte request representation, three priority-separated SPSC queues, atomic publication/reclamation, worker behavior, load shedding, application-level file authorization, diagnostics and throughput/latency measurement.

`WinTaskValidator.cpp` executes correctness and integration tests. `WinTaskBenchmark.cpp` executes end-to-end workload and performance tests and generates `BenchmarkResults.csv` after the run has drained.

## 2. Test environment record

Complete this section for each formal test campaign.

| Environment item | Actual value |
|---|---|
| CPU model | i5-10400F/10600KF |
| Physical core / logical processor topology |6C/12T (Hyper-Threading) |
| Windows edition and build | Windows 11 Pro Build 22H2 22621.4317 |
| Clang version | Clang 23.1.5 (planned) |
| Target triple | `x86_64-pc-windows-msvc` (planned) |
| MSVC STL version | Latest |
| Visual Studio Build Tools version | Visual Studio 2026 Build Tools |
| Windows SDK version | 10.0.28000.0 |
| Build configuration | Release for performance; Debug/instrumented for correctness where applicable |
| AVX2 enabled | Yes |
| Ingestor affinity | Processor/core 0 (project-machine mapping) |
| High consumer affinity | Processor/core 2 (project-machine mapping) |
| Medium consumer affinity | Processor/core 4 (project-machine mapping) |
| Low consumer affinity | Processor/core 6 (project-machine mapping) |
| Queue capacities | High 262,144; Medium 524,288; Low 524,288 |
| Queue storage | 40 MiB total |
| Benchmark result file | `BenchmarkResults.csv` |

The processor identifiers above are the project-machine mapping selected by the user. Record the actual mapping used in the run; do not silently substitute another mapping.

---

## 3. Test strategy

### 3.1 Validation layers

1. **Compile-time validation:** structure sizes, offsets, type properties, queue-capacity powers of two, masks and atomic lock-free assumptions.
2. **Deterministic unit tests:** routing, ring-buffer empty/full behavior, FIFO order, wraparound, state transitions and policy decisions.
3. **Concurrent stress tests:** producer/consumer publication, no overwrite, no duplicates, no loss of accepted jobs and diagnostics isolation.
4. **Integration tests:** job handlers, file/folder authorization, graceful drain/shutdown and CSV output.
5. **Performance tests:** sustained submission, completed throughput, end-to-end latency, queue saturation and load-shedding accounting.

### 3.2 Test-run rules

- Correctness tests and optimized performance measurements must be reported separately.
- Allocate queue storage and diagnostics arrays before the timed interval.
- Create and pin worker threads and perform warm-up before starting the measured interval.
- Use a unique `job_id` per request within a benchmark run.
- Use the same workload and request distribution when comparing variants.
- Do not include CSV serialization or CSV disk output in the measured request-processing interval.
- Do not use artificial sleeps in the primary throughput benchmark.
- Allow the run to drain all accepted work before calculating final completed throughput.
- Record offered, accepted, successfully completed, failed and capacity-rejected counts.
- Never report offered or accepted requests as completed throughput.
- Preserve the generated CSV and console summary as execution evidence.

### 3.3 Status vocabulary

| Status | Meaning |
|---|---|
| NOT RUN | Test has not been executed. |
| PASS | Actual result matches the expected result and evidence is recorded. |
| FAIL | Actual result does not match expected behavior. |
| BLOCKED | Test cannot be run because a documented prerequisite is unavailable. |
| NOT APPLICABLE | Test is intentionally excluded from the current POC, with reason recorded. |

---

## 4. Compile-time and structural test cases

| ID | Test | Procedure | Expected result | Status |
|---|---|---|---|---|
| TC-001 | Descriptor size | Build the layout assertions for `JobDescriptor`. | `sizeof(JobDescriptor) == 16`. | NOT RUN |
| TC-002 | Payload size | Build the layout assertions for `JobPayload`. | `sizeof(JobPayload) == 16`. | NOT RUN |
| TC-003 | Queue slot size | Build the layout assertions for `QueueSlot`. | `sizeof(QueueSlot) == 32`. | NOT RUN |
| TC-004 | Field offsets | Evaluate all specified `offsetof` assertions. | Descriptor and payload offsets match the Design & Specification Document. | NOT RUN |
| TC-005 | Trivially-copyable property | Compile `std::is_trivially_copyable_v<QueueSlot>` assertion. | Assertion succeeds. | NOT RUN |
| TC-006 | Queue capacity/mask | Compile-time check all capacities are powers of two and masks equal capacity minus one. | High: `0x0003FFFF`; Medium/Low: `0x0007FFFF`. | NOT RUN |
| TC-007 | Queue storage alignment | Allocate each queue and inspect the base address and slot addresses. | Base is 64-byte aligned; each 32-byte slot address is 32-byte aligned. | NOT RUN |
| TC-008 | Atomic property | Check the target build's 64-bit atomic lock-free property. | Property is verified; any failure is disclosed and the lock-free claim is removed. | NOT RUN |

---

## 5. Routing and request-data tests

| ID | Test | Procedure | Expected result | Status |
|---|---|---|---|---|
| TC-009 | Low routing | Submit valid jobs with severity 1 and 2. | All are published only to Low. | NOT RUN |
| TC-010 | Medium routing | Submit valid jobs with severity 3 and 4. | All are published only to Medium. | NOT RUN |
| TC-011 | High routing | Submit valid jobs with severity 5. | All are published only to High. | NOT RUN |
| TC-012 | Invalid severity | Submit severity 0 and a value greater than 5. | Invalid requests are not published to any queue. | NOT RUN |
| TC-013 | Job type validation | Submit each defined `JobType` value. | Each valid type reaches its corresponding handler. | NOT RUN |
| TC-014 | Unique job IDs | Submit a sequence with unique IDs. | Each result can be correlated to exactly one submitted request. | NOT RUN |
| TC-015 | Timestamp capture | Capture QPC at ingress and at completion for known test requests. | Ingress value is retained and completion delta is non-negative for the same run. | NOT RUN |
| TC-016 | Policy ID lookup | Submit valid and unknown `resource_policy_id` values. | Known IDs resolve to the configured policy; unknown IDs fail closed for protected operations. | NOT RUN |

---

## 6. SPSC ring-buffer correctness tests

Run the applicable tests for High, Medium and Low queues. Small-capacity test instances may be used to exercise wraparound quickly; production-capacity initialization is tested separately.

| ID | Test | Procedure | Expected result | Status |
|---|---|---|---|---|
| TC-017 | Initially empty | Initialize an empty queue and attempt consumer read. | Consumer observes empty; no slot is read. | NOT RUN |
| TC-018 | Single enqueue/dequeue | Publish one request and allow its consumer to process/release it. | Request is read once and in the correct slot. | NOT RUN |
| TC-019 | FIFO order | Publish a known sequence of IDs to one queue. | Consumer observes the same sequence order. | NOT RUN |
| TC-020 | Full queue | Fill a small test queue to its configured capacity. | Occupancy equals capacity; next enqueue is rejected without overwrite. | NOT RUN |
| TC-021 | Reuse after release | Fill, process/release slots and publish additional requests. | Released slots are safely reused. | NOT RUN |
| TC-022 | Wraparound | Drive head and tail beyond several physical array wraps. | Masked indexing remains correct; sequence and occupancy invariants hold. | NOT RUN |
| TC-023 | Capacity invariant | Stress a small queue while checking producer/consumer counters. | Unsigned occupancy never exceeds capacity; no unread/in-flight slot is overwritten. | NOT RUN |
| TC-024 | Cross-tier isolation | Fill or stall one queue while submitting to the other queues. | Each queue remains independent; no cross-tier slot borrowing occurs. | NOT RUN |
| TC-025 | Tail publication | Pause/interleave producer and consumer around slot construction and tail publication. | Consumer never observes partially initialized descriptor/payload data. | NOT RUN |
| TC-026 | Head reclamation | Pause/interleave consumer processing and producer reuse attempts. | Producer does not overwrite the slot before the consumer releases it. | NOT RUN |

---

## 7. Concurrent stress and lifecycle tests

| ID | Test | Procedure | Expected result | Status |
|---|---|---|---|---|
| TC-027 | Sustained SPSC stress | Run one producer and one consumer per queue with a large deterministic request sequence. | Every accepted request reaches one terminal record; no duplicates, corruption or unexplained losses. | NOT RUN |
| TC-028 | Three-queue simultaneous stress | Generate a mixed severity stream while all three consumers run. | Correct routing and independent progress across all queues. | NOT RUN |
| TC-029 | Payload integrity | Populate every field with recognizable patterns and validate at the consumer. | Descriptor and payload values are unchanged by queue publication/consumption. | NOT RUN |
| TC-030 | State transitions | Process successful and failing test jobs. | Normal path: Ready → Running → Completed; failure path: Ready → Running → Failed. | NOT RUN |
| TC-031 | Reserved states | Inspect POC behavior for Yielded and Cancelled. | These values are not entered by the current runtime; they remain reserved. | NOT RUN |
| TC-032 | Graceful drain | Stop new submissions while requests remain queued/in-flight. | Accepted requests are processed to terminal state before workers exit. | NOT RUN |
| TC-033 | Worker join | Shut down the engine and join all three consumers. | No worker remains active before diagnostics export. | NOT RUN |
| TC-034 | Diagnostics isolation | Stress all consumers while each writes its own result array. | Each consumer writes only within its assigned array; no inter-consumer result collision. | NOT RUN |
| TC-035 | Diagnostics capacity | Configure result-array capacity to match maximum accepted test volume and exercise the limit. | No overwrite or out-of-bounds write; capacity sufficiency is verified before timed execution. | NOT RUN |

---

## 8. Load-shedding and accounting tests

| ID | Test | Procedure | Expected result | Status |
|---|---|---|---|---|
| TC-036 | High queue saturation | Prevent/slow High consumption and offer more High requests than available slots. | Requests beyond High capacity are rejected; existing slots remain intact. | NOT RUN |
| TC-037 | Medium queue saturation | Prevent/slow Medium consumption and exceed Medium capacity. | Requests beyond Medium capacity are rejected; no overwrite occurs. | NOT RUN |
| TC-038 | Low queue saturation | Prevent/slow Low consumption and exceed Low capacity. | Requests beyond Low capacity are rejected; no overwrite occurs. | NOT RUN |
| TC-039 | Per-tier rejection | Saturate one tier while the other queues retain free slots. | Only the saturated tier rejects; unused capacity in other tiers is not borrowed. | NOT RUN |
| TC-040 | Rejection classification | Trigger capacity rejection. | Diagnostics classify it as simulated overload/HTTP 429; no HTTP response is claimed or transmitted. | NOT RUN |
| TC-041 | Offered/accepted/rejected identity | Run a valid workload that includes capacity drops. | `offered == accepted + rejected_capacity`. | NOT RUN |
| TC-042 | Accepted terminal identity | Drain a run containing successful and failed/denied jobs. | `accepted == completed + failed` after full drain. | NOT RUN |
| TC-043 | No false throughput | Compare offered, accepted and completed counts after a run with rejections. | Throughput uses successfully completed count only. | NOT RUN |

---

## 9. Job-handler and authorization tests

All filesystem tests shall use a dedicated WinTask test workspace containing test-owned files and directories. Do not use system directories or real user data.

| ID | Test | Procedure | Expected result | Status |
|---|---|---|---|---|
| TC-044 | Session validation workload | Submit configured valid synthetic session records. | Handler produces the expected deterministic result and terminal status. | NOT RUN |
| TC-045 | Invalid synthetic session | Submit a test token/signature not present in the configured test profile. | Handler reports failure; no cryptographic-security claim is made. | NOT RUN |
| TC-046 | Telemetry job | Submit a telemetry job with an authorized test policy. | Event/record is constructed; any configured sandbox append operation is permitted. | NOT RUN |
| TC-047 | Compute job | Submit a compute/sign job with known input fields. | Deterministic transform result is reproducible for the same inputs. | NOT RUN |
| TC-048 | Authorized read | Attempt a read operation permitted by the selected policy. | Operation is authorized and the controlled test read succeeds. | NOT RUN |
| TC-049 | Authorized append/write | Attempt an append/write operation permitted by the selected policy. | Operation is authorized and the controlled test write succeeds. | NOT RUN |
| TC-050 | Denied read | Attempt a read not permitted by the selected policy. | Access is denied before the protected operation is performed. | NOT RUN |
| TC-051 | Denied write | Attempt a write not permitted by the selected policy. | Access is denied; target content remains unchanged. | NOT RUN |
| TC-052 | Unknown policy | Submit an unknown or unmapped policy ID for a protected operation. | Fail closed; access is denied. | NOT RUN |
| TC-053 | User/policy mismatch | Use a valid policy ID with a user ID not authorized for it. | Access is denied. | NOT RUN |
| TC-054 | Path traversal | Attempt a path containing traversal outside the approved test root. | Access is denied; no outside file is read or modified. | NOT RUN |
| TC-055 | Policy operation mismatch | Use a policy that allows Read but attempt Write/Append. | Access is denied. | NOT RUN |

The benchmark's primary high-rate profile may use in-memory policy lookups and simulated operations. These filesystem integration cases verify the actual controlled file operations separately; they are not evidence of one million disk operations per second.

---

## 10. CPU affinity and platform tests

| ID | Test | Procedure | Expected result | Status |
|---|---|---|---|---|
| TC-056 | Ingestor affinity | Start the ingestor and query its assigned processor affinity. | It is assigned to the configured project processor/core 0. | NOT RUN |
| TC-057 | High consumer affinity | Start the High consumer and query its assigned affinity. | It is assigned to configured processor/core 2. | NOT RUN |
| TC-058 | Medium consumer affinity | Start the Medium consumer and query its assigned affinity. | It is assigned to configured processor/core 4. | NOT RUN |
| TC-059 | Low consumer affinity | Start the Low consumer and query its assigned affinity. | It is assigned to configured processor/core 6. | NOT RUN |
| TC-060 | Affinity failure handling | Use an invalid test affinity configuration or force the API failure path in a controlled test. | The affected test/benchmark run is marked failed/invalid; failure is not silently ignored. | NOT RUN |
| TC-061 | QPC frequency initialization | Query and cache QPC frequency once during initialization. | Frequency is valid and reused for all conversions in that run. | NOT RUN |
| TC-062 | QPC latency conversion | Use known tick deltas or compare conversion against an independently calculated value. | Conversion to fractional microseconds is correct within expected floating-point rounding. | NOT RUN |

---

## 11. Diagnostics and CSV tests

| ID | Test | Procedure | Expected result | Status |
|---|---|---|---|---|
| TC-063 | Per-job fields | Complete representative jobs of each priority/type. | Result rows contain JobID, PriorityClass, Urgency, LatencyMicroseconds and Status. | NOT RUN |
| TC-064 | Latency meaning | Compare saved ingress QPC with terminal completion QPC. | Exported latency includes queue waiting plus authorization/handler processing through terminal completion. | NOT RUN |
| TC-065 | Fractional latency | Process jobs with sub-microsecond measured durations where timer resolution permits. | CSV preserves fractional microseconds; values are not rounded to whole microseconds prematurely. | NOT RUN |
| TC-066 | CSV after shutdown | Complete a run and inspect output timing. | CSV is written only after consumers stop and result arrays are stable. | NOT RUN |
| TC-067 | CSV row count | Compare per-job result rows with accepted jobs that reached a terminal outcome. | One terminal record exists for each accepted and processed job. | NOT RUN |
| TC-068 | Aggregate summary | Compare CSV/summary counts with independently counted test inputs and statuses. | Offered, accepted, completed, failed and capacity-rejected totals reconcile. | NOT RUN |
| TC-069 | CSV parseability | Parse output with a standard CSV reader (for example, the user's Python analysis script). | File has consistent columns, valid delimiters and parseable numeric fields. | NOT RUN |

---

## 12. Performance and scalability benchmark cases

Performance tests shall be run in Release mode, after warm-up and thread-affinity setup. Record the exact request mix and run duration. Repeat each scenario sufficiently to report a distribution or at least multiple independent run results; do not select only the best run without disclosure.

| ID | Benchmark | Workload/procedure | Measurements | Status |
|---|---|---|---|---|
| TC-070 | Baseline mixed traffic | Generate a defined mixture of High/Medium/Low requests below queue saturation. | Offered/s, accepted/s, completed/s, latency distribution, rejection count. | NOT RUN |
| TC-071 | One-million request volume | Submit at least 1,000,000 uniquely identified requests, then drain. | Total duration, accepted/completed/failed/rejected counts, end-to-end throughput. | NOT RUN |
| TC-072 | One-million requests/second target | Run the defined workload with a measured interval and calculate successfully completed requests per second. | Completed requests/s; do not infer from offered rate. | NOT RUN |
| TC-073 | Priority distribution sensitivity | Test multiple declared severity distributions. | Per-tier accepted/completed counts, queue pressure and overall throughput. | NOT RUN |
| TC-074 | High-tier overload | Offer High traffic beyond High consumer capacity while lower tiers are not saturated. | High rejections and unaffected lower-tier operation. | NOT RUN |
| TC-075 | Queue headroom/recovery | Approach saturation, stop or reduce ingress, and allow consumers to drain. | Queue occupancy recovers; subsequent valid jobs are accepted again. | NOT RUN |
| TC-076 | AVX2 vs reference copy | Compare AVX2 slot-copy/ingestion with a correctness-equivalent reference copy under identical traffic. | Throughput and latency; verify identical request contents and counts. | NOT RUN |
| TC-077 | Single-slot vs pair publication | Compare ordinary publication with optional two-slot same-queue batching if implemented. | Throughput, median/tail latency, occupancy and rejection behavior. | NOT RUN |
| TC-078 | Compact vs experimental cache mitigation | Compare 32-byte baseline against any implemented paired/padded experimental variant. | Throughput and latency, plus memory footprint. | NOT RUN |
| TC-079 | Processing-cost sensitivity | Compare the three synthetic handler workloads or controlled workload weights. | Per-type latency and total completed throughput. | NOT RUN |
| TC-080 | Diagnostics overhead | Compare diagnostics enabled against a controlled measurement build with equivalent work and diagnostics disabled, if implemented safely. | Throughput/latency delta; no production result may omit required accounting. | NOT RUN |

### 12.1 Required benchmark summary

Record at least:

| Metric | Result |
|---|---|
| Workload request count | **Not run** |
| Severity distribution | **Not run** |
| Measurement duration | **Not run** |
| Requests offered | **Not run** |
| Requests accepted | **Not run** |
| Requests completed successfully | **Not run** |
| Requests failed/denied | **Not run** |
| Requests rejected for capacity | **Not run** |
| Completed throughput (requests/second) | **Not run** |
| Median ingress-to-completion latency | **Not run** |
| 95th percentile latency | **Not run** |
| 99th percentile latency | **Not run** |
| Maximum observed latency | **Not run** |
| High queue peak occupancy | **Not run** |
| Medium queue peak occupancy | **Not run** |
| Low queue peak occupancy | **Not run** |
| CSV path | **Not generated** |

Percentiles are computed offline from the per-job result records. They are analysis outputs, not additional hot-path instrumentation requirements.

---

## 13. Execution record template

Copy this block for each test campaign or benchmark scenario.

### Run record — [Run ID]

| Item | Value |
|---|---|
| Date/time | **Not recorded** |
| Executable | `WinTaskValidator.exe` / `WinTaskBenchmark.exe` |
| Build configuration | **Not recorded** |
| Commit/revision | **Not recorded** |
| Test case IDs | **Not recorded** |
| Machine/CPU | **Not recorded** |
| Windows build | **Not recorded** |
| Thread affinity mapping | **Not recorded** |
| Input/workload seed | **Not recorded** |
| Result file(s) | **Not recorded** |

**Observed result:**  
_To be completed after execution._

**Expected vs actual:**  
_To be completed after execution._

**Status:** `NOT RUN`

**Evidence:**  
_Add console output, CSV filename, relevant screenshots or reproducible notes after execution._

**Defect/reference:**  
_Complete if the test fails._

---

## 14. Defect and deviation log

| ID | Date | Test case | Description | Severity | Resolution | Status |
|---|---|---|---|---|---|---|
| — | — | — | No execution results recorded yet. | — | — | — |

Any deviation from the Design & Specification Document must be recorded here, including changes to structure size, queue capacity, queue ownership, memory ordering, affinity, timestamp representation, job semantics or throughput accounting.

---

## 15. Final test sign-off

| Acceptance item | Status |
|---|---|
| Structural assertions pass | NOT RUN |
| SPSC queue correctness passes | NOT RUN |
| Concurrency stress passes | NOT RUN |
| Queue saturation/load shedding passes | NOT RUN |
| Request accounting reconciles | NOT RUN |
| Job handlers behave deterministically | NOT RUN |
| File authorization tests pass | NOT RUN |
| CPU affinity verified | NOT RUN |
| QPC latency conversion verified | NOT RUN |
| CSV output validated | NOT RUN |
| Performance benchmark completed | NOT RUN |
| Final benchmark claims reviewed against completed counts | NOT RUN |

**Final disposition:** Pending implementation and test execution.

**Reviewer notes:**  
_To be completed after the test campaign._
