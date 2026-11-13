# WinTask

**High-Performance Priority-Based Task Scheduler & Processor for Windows**

WinTask is a Windows-native C++ proof-of-concept that explores high-throughput request ingestion, priority-based dispatch, bounded queue management and concurrent task processing. It models a server-oriented workload using one dedicated ingestor, three independent priority queues and three dedicated consumer threads. The project focuses on predictable resource usage, compact request representation, producer-consumer synchronization, processor affinity, cache-aware data placement, SIMD-assisted ingestion experiments, overload handling and measurable request-processing performance.

> **Performance objective:** WinTask uses **1,000,000 successfully completed requests per second** as a stretch target for a defined synthetic workload on the project machine. This is an objective to measure, not a guaranteed result or an established benchmark claim.

## Objectives

- Implement a bounded, priority-separated task-processing architecture for Windows.
- Route requests into High, Medium or Low queues based on their severity.
- Use independent single-producer/single-consumer (SPSC) ring buffers with fixed capacities.
- Process each priority tier using its own dedicated consumer thread.
- Use a compact, fixed-size request representation and preallocated queue storage.
- Apply explicit load shedding when a destination queue is full.
- Measure ingress-to-terminal-completion latency and successfully completed request throughput.
- Evaluate cache placement, CPU affinity and AVX2-assisted request-slot transfer on the target machine.
- Provide separate validation and benchmarking workflows through a command-line interface.
- Export per-request results and run/queue summaries to separate CSV files.
- Use `Visualizer.py` for offline validation, queue-specific latency percentile analysis and benchmark chart generation.

## Key Features

### Priority-separated request processing

WinTask classifies valid request severities into three priority tiers:

| Severity | Priority tier | Dedicated consumer |
|---:|---|---|
| 5 | High | High consumer |
| 3–4 | Medium | Medium consumer |
| 1–2 | Low | Low consumer |
| Outside 1–5 | Invalid | Not queued |

Each tier has its own queue and consumer. FIFO ordering is maintained within each queue. The design does not implement global preemption, cross-tier work stealing or a shared worker pool that continually selects the highest-priority request across all tiers. The `urgency` field is retained for diagnostics but does not reorder requests in this proof of concept.

### Bounded SPSC ring buffers

A single ingestor publishes requests to three independent bounded SPSC ring buffers. Each queue has exactly one producer and one assigned consumer.

| Queue | Capacity | Storage |
|---|---:|---:|
| High | 262,144 slots | 8 MiB |
| Medium | 524,288 slots | 16 MiB |
| Low | 524,288 slots | 16 MiB |
| **Total** | **1,310,720 slots** | **40 MiB** |

The additional 310,720 slots provide distributed headroom across the three queues. This headroom is not a shared reserve: an unused slot in one priority queue cannot be borrowed by another queue.

### Compact request representation

Each queue slot is designed to occupy exactly **32 bytes**, consisting of a 16-byte job descriptor and a 16-byte payload. Two slots therefore occupy 64 bytes under the project's cache-line layout assumption. Queue backing storage is intended to be 64-byte aligned while each slot retains its 32-byte stride. Compile-time layout checks are used to verify the required size and member offsets. The project does not rely on packing directives to obtain the intended layout.

### Concurrent processing and bounded overload behavior

The baseline runtime uses four dedicated threads:

| Thread | Responsibility | Configured processor assignment* |
|---|---|---:|
| Ingestor | Validates, timestamps, routes and publishes requests | 0 |
| High consumer | Processes the High queue | 2 |
| Medium consumer | Processes the Medium queue | 4 |
| Low consumer | Processes the Low queue | 6 |

The SPSC queues use explicit C++ acquire/release synchronization for request publication and slot reclamation. When a selected queue is full, the incoming request is rejected and counted. WinTask does not overwrite an occupied slot, borrow capacity from another tier or use a global overflow queue.

*Processor assignments are the project-machine configuration and are recorded during validation and benchmarking. They are not universal CPU topology identifiers.*

### Synthetic workload and telemetry

The proof of concept includes three controlled job types:

- `VALIDATE_SESSION` — deterministic synthetic session-validation work.
- `TELEMETRY_LOG` — bounded synthetic telemetry-processing work.
- `COMPUTE_SIGN` — deterministic bit-mixing and rotation work over a 64-bit input.

These handlers provide representative processing workloads. The signature transformation is not cryptographic verification and does not provide security or authentication. Ingress timestamps are captured using the Windows high-resolution performance counter. Per-consumer diagnostics storage is preallocated and isolated by consumer. After accepted work has drained and consumers have terminated, the benchmark exports `BenchmarkResults.csv` (per-request terminal records) and `BenchmarkSummary.csv` (Overall and per-queue counters, occupancy, timing and environment metadata). The offline `Visualizer.py` utility consumes both files, validates their run identity and accounting, calculates latency statistics including P50/P90/P95/P99 independently for High, Medium and Low, and generates the visualization outputs. Completed throughput is calculated from successfully completed requests divided by the measurement interval. Offered, accepted, completed, failed, invalid and capacity-rejected requests are distinguished in the reported results.

## Architecture

At a conceptual level, WinTask follows this processing flow:

1. The in-process workload source presents requests to the single ingestor.
2. The ingestor validates supported request fields, captures the ingress timestamp and determines the priority tier from severity.
3. The request is published to the corresponding bounded SPSC queue.
4. The dedicated consumer for that tier observes and processes requests in FIFO order.
5. The consumer records the terminal status and its preallocated diagnostic result before releasing the queue slot.
6. After ingestion stops and accepted work drains, the benchmark collects the results and exports `BenchmarkResults.csv` and `BenchmarkSummary.csv`.
7. The user runs `Visualizer.py` manually to validate the CSV inputs, derive `LatencyPercentiles.csv` and generate the benchmark charts.

The project is an **application-level task scheduler and processor**, not a replacement for the Windows kernel scheduler. It simulates server-style request processing in-process; it does not implement a production HTTP server, external network ingress or management of arbitrary Windows processes. Any HTTP 429 terminology refers only to a simulated capacity-rejection event, not an actual HTTP response.

## Project Structure

The planned project layout contains six C++ source files and one essential offline Python visualization script:

| File | Responsibility |
|---|---|
| `WinTask.cpp` | Sole application entry point. Initializes the command-line interface and returns the application exit status. |
| `SystemInterface.hpp` | Command-line menu. Dispatches the user's selection to validation, benchmarking, full evaluation or exit. |
| `WinTaskValidator.hpp` | Contains the 40 documented test cases, grouped execution, pass/fail accounting and validation summary. Exposes `RunValidationSuite()` as the validation entry point. |
| `WinTaskBenchmark.hpp` | Workload generation, benchmark orchestration, processor-affinity configuration, measurement collection and CSV export. Exposes `RunBenchmarkSuite()` as the benchmark entry point. |
| `WinTaskUtilities.hpp` | Small shared C++ and Windows utilities, such as high-resolution timing and narrowly scoped processor-affinity helpers. |
| `WinTaskEngine.hpp` | Core runtime. Contains the `WinTaskEngine` class and the request model, configuration, internal SPSC queue implementation, priority routing, ingestion, consumers, synthetic job handlers, synchronization, diagnostics and lifecycle responsibilities. |
| `Visualizer.py` | Essential offline Python utility. Reads `BenchmarkResults.csv` and `BenchmarkSummary.csv`, validates the data, calculates Overall and per-queue latency statistics including P50/P90/P95/P99, and generates CSV/PNG outputs. It runs manually after the C++ benchmark. |

The intended dependency direction is:

- `WinTask.cpp` initializes `SystemInterface`.
- `SystemInterface.hpp` imports the validator and benchmark interfaces.
- `WinTaskValidator.hpp` and `WinTaskBenchmark.hpp` both use the same `WinTaskEngine` implementation.
- `WinTaskEngine.hpp` uses the shared utilities it requires.
- The engine does not depend on the CLI, validator or benchmark. `Visualizer.py` is outside the C++ include graph and operates only on completed benchmark CSV outputs.

This is an initial conceptual overview of the planned organization and interaction of the files. It describes responsibilities and dependencies, not implementation code or a commitment to a particular internal class hierarchy. Related small types and operations may remain grouped in their assigned files.

## Command-Line Interface

The command-line interface is planned to provide four options:

1. **Run Validator**: Executes the 40 documented test cases and displays the validation summary.
2. **Run Benchmark**: Executes the configured workload and exports benchmark results.
3. **Run Full Evaluation**: Runs validation first and starts the benchmark only if validation succeeds.
4. **Exit**: Terminates the application cleanly.

The interface coordinates the selected operation; request ingestion, queue management and task processing remain responsibilities of the engine.

## System Requirements

- Windows x64.
- A processor and operating-system configuration compatible with the project's AVX2 benchmark path.
- Sufficient available memory for the 40 MiB queue storage and additional runtime/diagnostics allocations.
- Python 3 and Matplotlib for offline execution of `Visualizer.py`.
- A CPU topology that supports the configured thread-affinity arrangement on the project machine.

The target machine's processor model, logical-processor mapping, Windows build and available memory should be recorded with benchmark results.

## Compiler and Build Environment

The planned primary toolchain is:

- **Compiler:** Clang 23.1.5.
- **Target:** `x86_64-pc-windows-msvc`.
- **Standard library:** Microsoft STL.
- **Build environment:** Microsoft Build Tools for Visual Studio 2026.
- **Language baseline:** C++20; C++23 features may be used where explicitly supported and beneficial.
- **SIMD target:** AVX2 for the request-slot transfer experiment on the project machine.

GCC 16.1 is not part of WinTask's primary compiler configuration. Exact build commands and compiler options will be documented with the finalized build configuration; they should not be inferred from this conceptual project overview.

## Building WinTask

WinTask is planned as a single application with `WinTask.cpp` as its sole translation unit and the remaining project files providing header-based interfaces and implementation. The selected Clang/MSVC STL environment must be configured before building. The exact build command, required compiler switches and final executable configuration are intentionally left to the implementation/build-configuration stage. They will be added here once verified against the actual toolchain and source files.

## Running WinTask

After a successful build, launch `WinTask.exe` and select one of the four CLI options. The benchmark option generates `BenchmarkResults.csv` and `BenchmarkSummary.csv` after the workload has completed, accepted requests have drained and all consumer threads have terminated.

After the C++ benchmark finishes, run `Visualizer.py` manually from the location where the two input CSV files are available. The script validates both inputs, computes latency distributions and writes its derived CSV and PNG charts into the `Visualizations` directory. Visualization is an offline step and is not included in benchmark timing.

## Visualization

`Visualizer.py` is an essential project component for offline benchmark analysis. It consumes:

- `BenchmarkResults.csv` — one terminal result record for each accepted request.
- `BenchmarkSummary.csv` — four scope rows (`Overall`, `High`, `Medium`, `Low`) containing request accounting, queue capacity/peak occupancy, run durations and environment metadata.

The script validates matching run IDs, required columns, numeric values, record counts and request-accounting consistency before producing charts. It calculates minimum, mean, P50, P90, P95, P99 and maximum end-to-end latency for successful (`Completed`) requests, separately for High, Medium and Low, plus an Overall distribution. Percentiles use the nearest-rank method. A scope with no successful samples is reported as unavailable, not as zero.

The derived file and charts are written to `Visualizations/`:

- `LatencyPercentiles.csv` — one row each for Overall, High, Medium and Low, with sample count, minimum, mean, P50, P90, P95, P99 and maximum latency in microseconds.
- `QueueLatencyPercentiles.png` — P50/P90/P95/P99 comparison by priority queue.
- `LatencyDistribution.png` — latency distribution comparison, with outliers where practical.
- `CompletionTimeline.png` — successful completion counts/rates over elapsed time windows.
- `QueueUtilizationAndAccounting.png` — peak queue occupancy/utilization and request accounting.

The timeline chart is a time-series view; the authoritative completed-throughput metric remains successful completions divided by total measurement duration. Python visualization runs manually after the C++ benchmark and does not participate in the measured request-processing path. Python 3 and Matplotlib are required; the exact installed versions should be recorded for reproducibility.

## Validation

WinTask's validation suite contains **40 consolidated test cases** covering:

- Request structure, configuration and alignment.
- Ingestion, priority routing and data integrity.
- SPSC queue correctness, publication, reclamation and wraparound.
- Concurrent execution and engine lifecycle.
- Queue saturation, load shedding and accounting consistency.
- Synthetic job handlers and request-state semantics.
- Windows timing, processor affinity and diagnostics.
- Dual-CSV schema, run identity, accounting and offline visualization validation.
- Performance and scalability workloads, including queue-specific latency percentiles.

The companion `Test Case Specification & Execution Record.md` defines the procedures, expected results and execution-status fields. Tests are initially pending and must not be represented as passed until actually executed.

## Benchmarking

The benchmark evaluates request volume, completed throughput, ingress-to-terminal latency, priority-distribution sensitivity and AVX2-assisted slot transfer against a correctness-equivalent reference path. Each completed run produces `BenchmarkResults.csv` and `BenchmarkSummary.csv`. `Visualizer.py` performs offline validation and derives High, Medium, Low and Overall latency distributions, including P50, P90, P95 and P99.

The primary throughput measure is:

**Successfully completed requests / measurement duration in seconds.**

A workload that is merely offered or accepted is not evidence of completed throughput. Benchmark reports should include workload distribution, run duration, build configuration, CPU information, affinity mapping and accepted/completed/rejected counts. Results should be repeated and interpreted in the context of the target machine and workload. The 1,000,000 requests/second figure is a stretch target. WinTask does not claim to have achieved it until a valid benchmark run demonstrates it.

## Current Limitations

- The three queue capacities are fixed and independently reserved; capacity cannot be borrowed between priority tiers.
- Each priority tier has one dedicated consumer. Consumers do not steal work from other tiers.
- Priority selection occurs through severity classification and dedicated queues; there is no global preemption or cross-tier priority arbitration.
- FIFO ordering is preserved within each queue. Urgency is recorded but does not reorder work in this proof of concept.
- The workload is generated in-process. There is no production HTTP server or external network request listener.
- HTTP 429 is a simulated overload classification only.
- The job handlers are synthetic workloads and do not perform real cryptographic authentication or signature verification.
- The 300–700 ns processing estimate is provisional and must be measured rather than assumed.
- The 64-byte cache-line assumption and processor-affinity mapping are specific to the target-machine evaluation configuration.
- The million-requests-per-second objective is not a guaranteed capability.
- `Visualizer.py` performs offline analysis only; it is not a live dashboard and does not affect request processing.
- Successful-service latency percentiles exclude failed requests; failures remain included in request accounting and per-request terminal records.
- The completion timeline is a visualization of time-windowed completions, not a replacement for the aggregate completed-throughput calculation.

## Future Development

Potential post-POC work may include further evaluation of cache-line ownership strategies, alternative queue and scheduling policies, improved memory management, additional workload types and investigation of whether the independently useful queue or execution components can be adapted for other applications. Such work is outside the current proof-of-concept acceptance scope unless explicitly added through a later design revision.

## Disclaimer

This project is designed strictly for learning purposes and must not be considered a competitor to a professional application or production-grade task-processing system. It is an academic proof of concept intended to explore operating-system concepts and performance-oriented C++ engineering. It is not intended for deployment in production environments or for handling real security-critical workloads.

## Contact

For questions, feedback, feature requests, or technical discussions, please use the repository's **Discussions** section.

For urgent matters, contact the repository owner at:\
[**muhammad.moazzam2775@gmail.com**](mailto:muhammad.moazzam2775@gmail.com)
