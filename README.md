# SAT-1: Interrupt-Driven Real-Time Ground Command System

### EEL 4775 – Real-Time Systems Final Integration Capstone

**Author:** Andrew VanAuken  
**University:** University of Central Florida  
**Course:** EEL 4775 – Real-Time Systems (Summer 2026)  
**Platform:** ESP32-S3 + FreeRTOS  
**Simulation:** Wokwi  
**Target Role:** Entry-Level Electrical Engineer

---

## Theme

An interrupt-driven satellite ground command system that uses FreeRTOS top-half and bottom-half processing to provide bounded response to spacecraft events, demonstrating real-time embedded system design for an Entry-Level Electrical Engineer role.

---

## Demo

- **GitHub Pages:** https://andrewvanauken.github.io/EEE4775-Capstone/
- **GitHub Repository:** https://github.com/AndrewVanAuken/EEE4775-Capstone
- **Wokwi:** https://wokwi.com/projects/468122086654300161
- **Video:** https://youtu.be/B9EnprE2-YU

---

# Project Overview

SAT-1 is a space-themed embedded system developed on the ESP32-S3 using FreeRTOS. A pushbutton connected to GPIO18 simulates a ground command sent to a satellite. Pressing the button generates a hardware interrupt that activates a minimal Interrupt Service Routine (ISR).

The ISR signals two bottom-half tasks using both a Binary Semaphore and a Direct Task Notification. Their wake-up latency is measured under idle and loaded processor conditions using the Wokwi Logic Analyzer.

Application 3 provides the system spine. Its interrupt-driven command path—the ISR, the two signaling mechanisms, and the bottom-half tasks is the primary subject of study. Although the periodic load tasks originate from Application 2, they are used solely to generate processor contention while evaluating the interrupt-driven behavior of the Application 3 system.

Together, the integrated system demonstrates interrupt handling, task synchronization, scheduling, timing analysis, and engineering documentation as one coherent real-time embedded project.

---

# System Architecture

## System Flow

```text
GPIO18 Button
      │
      ▼
Hardware Interrupt
      │
      ▼
IRAM_ATTR ISR
      │
      ├──────────────┐
      ▼              ▼
Binary Semaphore   Direct Notification
      │              │
      ▼              ▼
Bottom-Half Tasks (Priority 12)
      │
      ▼
Timing Results
```

The ISR responds immediately to the GPIO18 falling edge and performs only ISR-safe operations. It records a timestamp, applies debounce protection, generates a GPIO19 timing pulse, and signals both bottom-half tasks.

Longer processing and reporting are deferred to FreeRTOS task context. The Binary Semaphore and Direct Task Notification paths are measured independently so their wake-up behavior can be compared.

---

# Hardware / Test Setup

The interrupt-driven system was validated in Wokwi using an ESP32-S3 development board, a pushbutton connected to GPIO18, and a logic analyzer monitoring GPIO19.

The ISR toggles GPIO19 on entry and exit, allowing interrupt timing and wake-up latency to be measured while comparing the Binary Semaphore and Direct Task Notification implementations.

---

# Tasks and Timing Evidence

## WCET Measurement

To obtain accurate Worst-Case Execution Time (WCET) measurements for the background load tasks, the original Application 3 scaffold was instrumented with a boot-time self-test.

Before the periodic tasks begin executing, each load task runs multiple times in isolation on Core 1 without interference from other tasks. This allows the measured execution times to represent actual computation time rather than response time inflated by preemption.

These measured WCET values were then used to compute processor utilization.

| Task | Period | WCET | Utilization | Priority | Deadline |
|------|-------:|------:|------------:|---------:|---------:|
| Load Task A | 10 ms | 0.347 ms | 0.0347 | 15 | 10 ms |
| Load Task B | 20 ms | 1.977 ms | 0.0989 | 10 | 20 ms |
| Load Task C | 50 ms | 8.271 ms | 0.1654 | 5 | 50 ms |
| Load Task D | 100 ms | 6.334 ms | 0.0633 | 2 | 100 ms |
| Binary Semaphore Task | Event-driven | N/A | N/A | 12 | Latency evaluated experimentally |
| Direct Notification Task | Event-driven | N/A | N/A | 12 | Latency evaluated experimentally |

**Total Utilization**

```
U = 0.0347 + 0.0989 + 0.1654 + 0.0633 = 0.3623
```

Since

```
U = 0.3623 < 0.7568
```

the periodic task set satisfies the Rate-Monotonic schedulability bound.

The utilization is also below the EDF feasibility limit of 1.0, meaning the periodic workload is schedulable under both Rate-Monotonic Scheduling (RMS) and Earliest Deadline First (EDF).

The Binary Semaphore and Direct Notification tasks are event-driven rather than periodic, so they are excluded from the utilization calculation. Their performance is evaluated using measured wake-up latency following an interrupt.

---

# Interrupt Response Time

| Configuration | GPIO18 → GPIO19 |
|--------------|----------------:|
| Idle | 21.593 µs |
| Loaded | 21.593 µs |

The interrupt response remained essentially unchanged under processor load, demonstrating that ISR execution remained bounded while background tasks were active.

---

# Bottom-Half Wake-Up Latency

| Configuration | Binary Semaphore | Direct Task Notification |
|--------------|-----------------:|-------------------------:|
| Idle | 2,644 µs | 30 µs |
| Loaded | 10,425 µs | 18,996 µs |

The increased loaded latency occurred after the ISR while the bottom-half tasks waited to be scheduled. Load Task A executes at Priority 15, above the bottom-half Priority 12, allowing it to delay execution of the bottom-half tasks during processor load.

The loaded values represent end-to-end wake-up latency rather than the isolated execution overhead of each FreeRTOS signaling primitive.

---

# Hazard Analysis

| Hazard | Effect | Mitigation |
|---------|--------|------------|
| Long ISR execution | Increased interrupt latency | Keep the ISR minimal and defer processing to bottom-half tasks |
| Button bounce | Repeated interrupt events | Apply timestamp-based debounce logic |
| Missing scheduler yield | Delayed bottom-half execution | Use `portYIELD_FROM_ISR()` when a higher-priority task is awakened |
| Heavy processor load | Increased bottom-half latency | Fixed-priority scheduling and verify timing |
| Non-ISR-safe function call | Unpredictable behavior or system failure | Use only ISR-safe FreeRTOS APIs inside the ISR |

---

# Fault Injection and Graceful Degradation

To demonstrate graceful degradation, the scheduler yield call

```cpp
portYIELD_FROM_ISR(higher_woken);
```

was temporarily removed from the ISR.

The ISR continued detecting interrupts and successfully delivered both signals, but the awakened bottom-half tasks no longer executed immediately after the ISR returned. Wake-up timing became slower and less deterministic.

After restoring

```cpp
portYIELD_FROM_ISR(higher_woken);
```

the original deterministic timing behavior returned.

The experiment demonstrated that the system degraded in timing performance rather than failing completely.

---

# Build and Run

1. Open the Wokwi project.
2. Start the ESP32-S3 simulation.
3. Open the Serial Monitor.
4. Press the GPIO18 pushbutton.
5. Observe the Binary Semaphore and Direct Notification latency measurements.
6. Enable `WITH_LOAD = 1` to repeat the experiment under processor load.
7. Use the Wokwi Logic Analyzer to capture the GPIO18 and GPIO19 timing signals.

---

# Engineering Skills Demonstrated

This project demonstrates practical skills applicable to entry-level embedded and electrical engineering positions, including:

- Interrupt handling
- FreeRTOS task synchronization
- Embedded C programming
- Real-time scheduling
- Worst-Case Execution Time (WCET) analysis
- Interrupt latency measurement
- Logic analyzer debugging
- Hardware/software integration
- Hazard analysis
- Technical documentation

---

# Conclusion

This project demonstrates the implementation and evaluation of an interrupt-driven real-time embedded system using FreeRTOS on the ESP32-S3.

Timing measurements, schedulability analysis, and fault injection confirmed bounded interrupt response while illustrating how processor loading affects bottom-half scheduling latency.

Worst-case execution times measured in isolation produced a processor utilization of **U = 0.3623**, well below the Rate-Monotonic utilization bound of **0.7568**, demonstrating schedulability under both RMS and EDF.

Finally, removing `portYIELD_FROM_ISR()` showed that the system degraded gracefully by increasing task wake-up latency without interrupt detection failure, illustrating an important real-time software engineering principle.
