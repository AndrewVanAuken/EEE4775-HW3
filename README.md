# EEE4775 HW3
**Name**: Andrew VanAuken  
  
## Part A – Read a task table
| Task | Function | Period (ms) | WCET (ms) | Deadline (ms) |
| :--- | :--- | :---: | :---: | :---: |
| **T1** | sensor read | 10 | 1.0 | 10 |
| **T2** | control loop | 20 | 3.0 | 20 |
| **T3** | comms send | 50 | 5.0 | 50 |
| **T4** | logging | 100 | 10.0 | 100 |

### 1. CPU Fraction
- T1: 1.0 ms / 10 ms = 0.10 (10%)
- T2: 3.0 ms / 20 ms = 0.15 (15%)
- T3: 5.0 ms / 50 ms = 0.10 (10%)
- T4: 10.0 ms / 100 ms = 0.10 (10%)

**Total utilization:** 0.10 + 0.15 + 0.10 + 0.10 = 0.45 = 45%

**Idle fraction:** 1.0 - 0.45 = 0.55 = 55%

### 2. Priority ranking
T1 > T2 > T3 > T4

**Rule Used:** I used Rate Monotonic Scheduling, which assigns higher priority to tasks with shorter periods.

**Priority Bands:**
- T1: 10–14 (Sensor/IPC) because it reads sensors at the fastest rate to ensure immediate data capture.
- T2: 15–18 (Control) because it performs the real-time control calculations.
- T3: 5–9 (User) because communication is important but less time-critical than sensing and control.
- T4: 1–4 (Housekeeping) because logging is a background task that should not interfere with real-time operation.

### 3. Yield semantics
T2 should use `vTaskDelayUntil()` rather than `vTaskDelay()`. Since T2 is a periodic control loop, it needs to execute at a consistent interval. `vTaskDelayUntil()` schedules the next release relative to a fixed reference time, preventing timing drift and minimizing phase error. `vTaskDelay()` delays relative to when the task finishes running, which can cause the task period to gradually drift over time.

### 4. State-machine trace
**Running → Blocked:** Task finishes processing and executes vTaskDelayUntil() to wait until its next release time.

**Blocked → Ready:** The delay period expires (driven by the RTOS tick interrupt), which pushes the task into the queue.

**Ready → Running:** The scheduler selects T1 to execute because it is the highest-priority ready task.

---

## Part B – `xTaskCreate` defended
```c
xTaskCreate(
    control_task,            // Task function
    "T2_Control",            // Task name
    256,                     // Stack size (words)
    NULL,                    // Task parameters
    16,                      // Priority
    &ControlTaskHandle       // Task handle
)
```

- **Task Function (`control_task`):** The function that implements T2’s periodic control loop.
- **Task Name ("T2_Control"):** Gives the task a readable name for debugging.
- **Stack Size (`256`):** I estimated worst-case stack depth by summing the maximum nested function calls within the control loop, local variables, and allowing a buffer for interrupt context frames, arriving at around 1 KB. Given a 32-bit architecture 4 bytes/word, this gives 256 words.
- **Parameters (`NULL`):** NULL is used because this task does not need any startup input arguments.
- **Priority (`16`):** Places T2 in the 15–18 control priority band.
- **Task Handle (`&ControlTaskHandle`):** Stores the created task’s handle so it can be monitored or controlled later.

---

## Part C – Design your own
**Theme**: Space

### 1. Task Table
| Task | Function | Period (ms) | WCET (ms) | Deadline (ms) | Priority |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **star_tracker_task** | Reads star tracker and orientation sensor data | 15 | 2.0 | 15 | Highest |
| **attitude_control_task** | Computes spacecraft attitude corrections | 30 | 4.5 | 30 | Medium |
| **thruster_command_task** | Sends commands to reaction wheels or thrusters | 75 | 6.0 | 75 | Lowest |

### 2. Priority Justication
- **star_tracker_task** has the shortest period (15 ms) gets the highest priority. If it's ranked lower, longer math/actuation tasks will preempt it, causing hardware registers to overwrite themselves before data is read, blinding the spacecraft.
- **attitude_control_task** runs at a medium frequency (30 ms) to calculate orbital corrections. It sits below the sensor task to ensure it always reads fresh data, but above the thrusters so corrections are computed before actions are taken.
- **thruster_command_task** has the longest period (75 ms). Mechanical components react much slower than electronics, so physical actuation can safely run at the lowest priority without impacting the high-speed sensor capture and math loops.

### 3. Shared Resource
The shared resource is the attitude state structure, which stores the spacecraft's current orientation estimate. It is shared between `star_tracker_task`, which writes the updated telemetry, and `attitude_control_task`, which reads it. A FreeRTOS mutex guards this structure to ensure that the control task never reads partially updated, corrupted sensor data, while simultaneously providing priority to protect the system against unbounded priority inversion.

### 4. Concurrency Diagram
<img width="1201" height="806" alt="image" src="https://github.com/user-attachments/assets/95540652-800f-4f5b-b91d-e05ac2f12a75" />

---

## Part D – Industry anchor
### Zephyr

- Zephyr RTOS uses a hybrid scheduling model that supports cooperative scheduling, fixed-priority preemptive scheduling, and Earliest Deadline First (EDF) scheduling. It is commonly used in resource-constrained embedded systems such as IoT devices, wearable technology, smart home products, and industrial sensor networks. One major difference between Zephyr and FreeRTOS is its configuration-driven hardware abstraction system. Zephyr uses Devicetree and Kconfig to configure hardware and software features, while FreeRTOS typically relies on manual configuration through C header files such as `FreeRTOSConfig.h`. This approach makes Zephyr more portable across different hardware platforms and simplifies development for complex embedded systems with many peripherals and sensors.

---

## AI Usage

ChatGPT was used to help interpret the assignment, explain FreeRTOS concepts such as task states and scheduling behavior, assist with the `xTaskCreate()` parameter justifications in Part B, and help develop the space-themed tasks for Part C. Claude was used to help create and format the concurrency diagram. Google AI was used to research Zephyr RTOS and gather information for Part D.
