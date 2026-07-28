# SAT-1: Interrupt-Driven Real-Time Ground Command System

### EEL 4775 – Real-Time Systems Final Integration Capstone

**Author:** Andrew VanAuken  
**University:** University of Central Florida  
**Course:** EEL 4775 – Real-Time Systems (Summer 2026)  
**Platform:** ESP32-S3 + FreeRTOS  
**Simulation:** Wokwi  
**Target Role:** Entry-Level Electrical Engineer

---

## One-Sentence Theme

An interrupt-driven satellite ground command system that uses FreeRTOS top-half/bottom-half processing to provide bounded response to spacecraft events, demonstrating real-time embedded system design for an Entry-Level Electrical Engineer role.

---

## Demo

- **GitHub Pages:** https://andrewvanauken.github.io/eee4775capstone
- **Wokwi:** https://wokwi.com/projects/468122086654300161
- **YouTube:** *(Add after recording.)*

---

## Project Overview

SAT-1 is a space-themed embedded system developed on the ESP32-S3 using FreeRTOS. A pushbutton connected to GPIO18 simulates a ground command sent to a satellite. When the button is pressed, a hardware interrupt immediately activates an Interrupt Service Routine (ISR), which signals dedicated bottom-half tasks responsible for processing the event.

This project demonstrates the top-half/bottom-half design pattern commonly used in real-time embedded systems. The ISR performs only the minimum time-critical work while longer processing is deferred to FreeRTOS tasks using both a Binary Semaphore and a Direct Task Notification. The performance of each signaling method is measured and compared using the Wokwi Logic Analyzer under both idle and loaded processor conditions.

The capstone builds upon Application 3 while incorporating periodic background tasks from Application 2 to create processor load during interrupt testing. The result is a complete real-time embedded application demonstrating interrupt handling, task synchronization, timing analysis, and engineering documentation.

---

## Architecture

### System Flow

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
 Bottom-Half Tasks (FreeRTOS)
      │
      ▼
Serial Output / Timing Results
```

*(Insert your system architecture or concurrency diagram here.)*

The ISR responds immediately to the hardware interrupt and performs only ISR-safe operations. It records the event, generates a timing pulse on GPIO19, and signals the bottom-half tasks. The bottom-half tasks calculate interrupt latency and report the results while periodic background tasks generate processor load for timing analysis.

---

## Tasks & Timing (WCET Evidence)

| Task | Period | WCET | Priority | Deadline |
|------|-------:|------:|---------:|---------:|
| Load Task A | 10 ms | *(Measured)* | 15 | 10 ms |
| Load Task B | 20 ms | *(Measured)* | 10 | 20 ms |
| Load Task C | 50 ms | *(Measured)* | 5 | 50 ms |
| Load Task D | 100 ms | *(Measured)* | 2 | 100 ms |
| Binary Semaphore Task | Event Driven | *(Measured)* | 12 | Immediate |
| Direct Notification Task | Event Driven | *(Measured)* | 12 | Immediate |

### Measured Interrupt Response

| Configuration | GPIO18 → GPIO19 |
|--------------|----------------:|
| Idle | 21.593 µs |
| Loaded | 21.593 µs |

### Measured Bottom-Half Wake-Up Latency

| Configuration | Binary Semaphore | Direct Task Notification |
|--------------|-----------------:|-------------------------:|
| Idle | 2,644 µs | 30 µs |
| Loaded | 10,425 µs | 18,996 µs |

**Processor Utilization**

Overall processor utilization was verified using the measured worst-case execution times (WCETs) of the periodic background tasks. The workload remained schedulable while allowing interrupt latency measurements under processor load.

---

## Hazard Analysis

| Hazard | Effect | Mitigation |
|---------|--------|------------|
| Long ISR execution | Increased interrupt latency | Keep the ISR minimal and defer processing to bottom-half tasks |
| Button bounce | Multiple interrupt events | Debounce logic inside the ISR |
| Missing scheduler yield | Delayed task execution | Use `portYIELD_FROM_ISR()` |
| Heavy processor load | Increased bottom-half latency | Fixed-priority scheduling and timing verification |

---

## Graceful Degradation

To demonstrate ISR safety, `portYIELD_FROM_ISR()` was temporarily removed during testing.

Although interrupts continued to be detected correctly, the bottom-half tasks no longer executed immediately after the interrupt completed. This increased wake-up latency while the system remained fully operational.

After verifying the behavior, `portYIELD_FROM_ISR()` was restored, returning the system to deterministic interrupt response.

---

## Build & Run

1. Open the Wokwi project.
2. Start the simulation.
3. Open the Serial Monitor.
4. Press the GPIO18 pushbutton.
5. Observe the interrupt latency measurements.
6. Enable the background workload (`WITH_LOAD = 1`) to compare timing under processor load.

---

## Tailored for

### Entry-Level Electrical Engineer

This project demonstrates practical skills applicable to entry-level embedded and electrical engineering positions, including:

- Interrupt handling
- FreeRTOS task synchronization
- Embedded C programming
- Real-time scheduling
- Timing analysis
- Logic analyzer debugging
- Hardware/software integration
- Technical documentation
