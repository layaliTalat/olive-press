# Olive Press Production Line Simulation

Build on Ubuntu/WSL:

```bash
make
./olive config.txt
```

`src/main.c` creates 6 stage child processes and 1 Quality Control child. Five ordinary pipes connect the six stages. A shared status pipe sends stage events to QC; an alert pipe sends QC errors to the supervisor.

Change `config.txt` and rerun without rebuilding. Press Ctrl+C to stop.

## Configuration Contract

- `container_kg`: Initial mass of olives available in the reception container.
- `batch_size_kg`: Mass of each batch transferred to the production line.

Current validation:

    container_kg > 0 && batch_size_kg > 0

Reception automatically calculates and dispatches batches of size `batch_size_kg` (and a final partial batch if `container_kg` is not an exact multiple) until the container-empty sensor detects `0.00 kg` remaining.

## Stage Pipeline & Responsibilities

Stage function:

    run_stage(stage_id_t stage, int in_fd, int out_fd, int status_fd, const config_t *cfg);

### File descriptors

- Reception: `in_fd = -1`, `out_fd` sends batches to Cleaning.
- Cleaning, Washing, Crushing, Malaxation: read from `in_fd` and forward batches to `out_fd`.
- Pressing: reads from `in_fd`, `out_fd = -1`.
- All stages send `status_msg_t` to `status_fd`.

### Conveyor Signals

- `SIG_CONVEYOR_STOP = SIGUSR1`
- `SIG_CONVEYOR_RESUME = SIGUSR2`

Implemented flow:
1. Reception monitors the olive container level and sends `SIG_CONVEYOR_STOP` / `SIG_CONVEYOR_RESUME` to the Supervisor (`getppid()`).
2. Supervisor catches these signals and forwards them to the Cleaning process (`children[CLEANING]`).
3. Cleaning installs signal handlers to pause/resume conveyor intake while allowing in-flight batches already inside the downstream pipeline (`Washing` through `Pressing`) to finish processing cleanly.

## Current Status & Remaining Work

- Stage processing delays (`stage_time[]`), simulated sensor readings, container-empty detection, and conveyor signal controls are implemented.
- OpenMP computational benchmarking is implemented; OpenGL visualization is pending.
- Supervisor logging uses `olive.log`.



## Quality Control and Fault Monitoring (Member 3)

The Quality Control process monitors the production stages
using status messages received through a pipe.

Implemented features:
- Temperature monitoring using configured minimum and maximum limits.
- Pressure monitoring using configured minimum and maximum limits.
- Random sensor fault simulation using `fault_probability`.
- Actual processing time measurement using `CLOCK_MONOTONIC`.
- Processing delay detection with a tolerance of 20% or 0.1 seconds,
  whichever is greater.
- Monitoring of stage states: idle, running, stopped, completed, and error.
- Alert messages sent from Quality Control to the Supervisor.
- Supervisor records received alerts in `olive.log`.

## OpenMP Performance Benchmark

An independent computational benchmark was implemented in
`src/openmp_benchmark.c` to compare sequential and parallel execution.

Compile the benchmark:

```bash
gcc -O2 -fopenmp src/openmp_benchmark.c -o openmp_benchmark -lm
```

Run with different thread counts:

```bash
OMP_NUM_THREADS=1 ./openmp_benchmark
OMP_NUM_THREADS=4 ./openmp_benchmark
OMP_NUM_THREADS=8 ./openmp_benchmark
```

### Experimental Results

| Threads | Parallel Time (s) | Speedup |
|---------|-------------------|---------|
| 1       | 4.199165          | 1.22x   |
| 4       | 1.576607          | 2.96x   |
| 8       | 1.214332          | 3.71x   |

Speedup = Sequential Time / Parallel Time

The benchmark showed that parallel execution improved
performance for the independent computational workload.

Among the tested configurations, 8 threads achieved
the lowest parallel execution time.

Sequential and parallel numerical results were consistent
within floating-point rounding differences.

Note: These results are from individual benchmark runs.
They measure a simulated computational workload, not
the complete olive press pipeline throughput.


---

# Graphics, Testing and Final Delivery

## Overview

The graphics module provides a real-time monitoring dashboard for the Olive Press Control System using OpenGL and GLUT.

The dashboard receives live production updates through non-blocking FIFO-based Inter-Process Communication (IPC), allowing users to monitor the production system independently.

## Implemented Features

- Real-time OpenGL dashboard.
- Visualization of all six production stages.
- Custom machine icons.
- Color-coded machine states.
- Animated olive flow between production stages.
- Live temperature and pressure monitoring.
- Batch identification and processing time display.
- Dynamic total batch calculation.
- Completed batch counter.
- Active stage counter.
- Quality control alert counter.
- Real-time production progress bar.
- System event log with highlighted alerts.
- Non-blocking FIFO communication.
- Configuration file support.
- Makefile integration.

## Production Pipeline

```text
Reception --> Cleaning --> Washing
                              |
                              v
Pressing <-- Malaxation <-- Crushing
```

### Machine States

| State | Color | Description |
|---|---|---|
| IDLE | Yellow | Waiting for production data |
| RUNNING | Green | Processing a batch |
| STOPPED | Gray | Production stage stopped |
| COMPLETED | Blue | Batch processing completed |
| ERROR | Red | Processing error detected |

## System Architecture

```text
Production Processes
         |
         v
     Status Pipe
         |
         v
 Quality Control Process
         |
         +--------------------+
         |                    |
         v                    v
    Supervisor          Dashboard FIFO
                              |
                              v
                       OpenGL Dashboard
                              |
                              v
                    Real-Time Visualization
```

The dashboard receives production status messages and quality control alerts through a separate named FIFO without consuming messages from the original quality control pipe.

## Graphics Module Structure

| File | Description |
|---|---|
| graphics/graphics.c | Main dashboard and rendering loop |
| graphics/graphics.h | Shared graphics definitions |
| graphics/graphics_state.c | Stage state management |
| graphics/ui_components.c | Dashboard components |
| graphics/ui_components.h | UI component declarations |
| graphics/machine_icons.c | Custom machine icons |
| graphics/machine_icons.h | Machine icon declarations |
| graphics/animation.c | Olive flow animation |
| graphics/animation.h | Animation declarations |
| graphics/live_data.c | FIFO reception, statistics and event logging |
| graphics/live_data.h | Live data interface |
| include/dashboard_ipc.h | Dashboard IPC message definitions |
| src/dashboard_ipc.c | Dashboard message publishing |
| src/quality.c | Quality control integration |
| Makefile | Build configuration |

## Requirements

The project was developed and tested using WSL2 Ubuntu.

Required tools and libraries:

- GCC
- GNU Make
- GDB
- OpenGL
- GLUT / FreeGLUT
- GLU
- Git

Install the required dependencies:

```bash
sudo apt update
sudo apt install build-essential gdb freeglut3-dev libgl1-mesa-dev libglu1-mesa-dev
```

## Build and Run

### Build the Project

```bash
make clean
make
```

The build generates two executables:

- `olive`: Production control system.
- `graphics_app`: OpenGL monitoring dashboard.

### Terminal 1 - Start the Dashboard

```bash
./graphics_app config.txt
```

Wait until the following message appears:

```text
[GUI] Dashboard IPC ready
```

### Terminal 2 - Start the Production System

```bash
./olive config.txt
```

Both executables must use the same configuration file and run from the same project directory.

### Standalone Operation

The production system can also operate without the graphical dashboard:

```bash
./olive config.txt
```

## Dynamic Production Configuration

The total number of batches is calculated using:

```text
Total Batches = ceil(Container Weight / Batch Size)
```

Production progress is calculated using:

```text
Progress (%) = (Completed Batches / Total Batches) * 100
```

### Default Configuration

| Parameter | Value |
|---|---|
| Container Weight | 200 kg |
| Batch Size | 25 kg |
| Total Batches | 8 |
| Fault Probability | 0.05 |
| Temperature Range | 20 - 35 |
| Pressure Range | 1 - 8 |

## Testing and Validation

Functional, integration, stability, performance and debugging tests were performed during development.

### Test Cases

| ID | Test Case | Status |
|---|---|---|
| TC01 | Successful Compilation | Completed |
| TC02 | Dashboard Initialization | Completed |
| TC03 | Live IPC Communication | PASS |
| TC04 | Batch Processing | PASS |
| TC05 | Conveyor Stop and Resume | PASS |
| TC06 | Sensor Monitoring | PASS |
| TC07 | Quality Control Alerts | PASS |
| TC08 | Olive Flow Animation | Completed |
| TC09 | Production Completion | PASS |
| TC10 | System Event Log | PASS |
| TC11 | GUI Shutdown Stability | Performed |
| TC12 | Standalone Operation | PASS |
| TC13 | Performance Testing | Completed |
| TC14 | Final Integration Testing | Pending final merge verification |
| TC15 | GDB Debugging | Completed |
| TC16 | Variable Batch Count | Fixed and retested |
| TC17 | Fault Injection | PASS |
| TC18 | Reception Synchronization | PASS |

Selected screenshots and execution logs are available in the `docs` directory. Additional tests were performed manually without separate screenshots.

## Performance Testing

Performance measurements were collected using GNU Time.

```bash
/usr/bin/time -v ./olive config.txt
```

### Standalone Performance Results

| Metric | Result |
|---|---|
| Execution Time | 53.05 seconds |
| User CPU Time | 0.00 seconds |
| System CPU Time | 0.02 seconds |
| CPU Utilization | 0% (rounded) |
| Maximum Resident Set Size | 2100 KB |
| Major Page Faults | 0 |
| Minor Page Faults | 311 |
| Voluntary Context Switches | 655 |
| Involuntary Context Switches | 0 |
| Swaps | 0 |
| Exit Status | 0 |

The standalone production simulation completed successfully with low measured CPU utilization.

Integrated production performance and GUI resource usage were also measured separately.

These measurements do not independently establish hard real-time deadline guarantees.

## GDB Debugging

GNU Debugger was used to inspect the supervisor process and monitor program execution.

```bash
gdb ./olive
```

The debugging procedure included:

- Setting breakpoints.
- Inspecting local variables.
- Examining the call stack.
- Continuing program execution.
- Checking program termination.

## Bug Identification and Resolution

### Dynamic Batch Count Mismatch

During variable batch count testing, the production system was configured to process four batches instead of eight.

However, the dashboard continued displaying eight total batches and incorrectly reported 50% progress after production completion.

**Root Cause:**

The dashboard always loaded the default `config.txt` file while the production system used an alternative configuration file.

**Solution:**

The dashboard was modified to accept a configuration file path through command-line arguments.

```bash
./graphics_app tests/config_batch_test.txt
```

```bash
./olive tests/config_batch_test.txt
```

The corrected dashboard successfully displayed four total batches and calculated the corresponding production progress.

### Reception Synchronization

During an initial fault injection test, the Reception stage remained in the IDLE state despite production activity.

The test was repeated after ensuring that the dashboard FIFO was initialized before starting production.

The repeated test successfully received all expected Reception status updates for the four batches.

The exact cause of the initial missing updates was not conclusively established.

## Fault Injection Testing

Fault injection was tested using an alternative configuration:

```text
fault_probability = 1.0
```

The dashboard displayed simulated sensor violations, including:

- Temperature above the maximum threshold.
- Temperature below the minimum threshold.
- Pressure above the maximum threshold.
- Pressure below the minimum threshold.

During the documented test, the dashboard displayed 21 quality control alerts while production was still running.

## Test Evidence

### TC01 - Successful Compilation

![Successful Compilation](docs/evidence/TC01_Successful_Compilation.png)

### TC02 - Dashboard Initialization

![Dashboard Initialization](docs/evidence/TC02_Dashboard_Initialization.png)

### TC09 - Production Completion

All eight batches were processed successfully, reaching 100% production progress.

![Production Completed](docs/evidence/TC09_Production_Completed.png)

### TC13 - GUI Resource Usage

![GUI Resource Usage](docs/evidence/TC13_GUI_Resource_Usage.png)

### TC13 - Integrated Performance Testing

![Integrated Performance](docs/evidence/TC13_Integrated_Performance_Test.png)

### TC15 - GDB Debugging

![GDB Debugging](docs/evidence/TC15_GDB_Debugging.png)

### TC15 - GDB Normal Exit

![GDB Normal Exit](docs/evidence/TC15_GDB_Normal_Exit.png)

### TC16 - Variable Batch Count After Fix

![Dynamic Batch Configuration](docs/evidence/TC16_Variable_Batch_Count_After_Fix.png)

### TC17 - Fault Injection Testing

![Fault Injection](docs/evidence/TC17_Fault_Injection_Test.png)

### TC18 - Reception Synchronization

![Reception Synchronization](docs/evidence/TC18_Reception_Synchronization.png)

## Known Limitations

- Both executables must use the same configuration file.
- Both executables must run from the same project directory.
- The dashboard should be initialized before starting production.
- FIFO messages may be missed if the dashboard is unavailable or the FIFO buffer becomes full.
- The dashboard does not replay previous production events.
- The event log maintains a limited in-memory history.
- Olive flow animation represents production activity rather than exact physical batch movement.
- Performance measurements depend on the execution environment.
- Final verification of the merged team implementation is required.

## Conclusion

The graphics module provides a functional real-time visualization of the Olive Press Control System.

The implementation integrates OpenGL rendering, FIFO communication, production statistics, sensor monitoring, quality control alerts, animation and event logging.

Testing helped identify and resolve the dynamic configuration mismatch and verify Reception status synchronization.

The graphical dashboard improves system observability while maintaining separation between production and monitoring processes.

---
