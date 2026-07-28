# EEE4775CapstoneJuanLugo
The final integration capstone. Compose Apps 1–5 into one coherent themed system you can demo, document, and put in your portfolio. 
One-Sentence theme:
A dual-core roller-coaster launch and safety control system that coordinates operator commands, safety interlocks, ride-state transitions, and live monitoring, built to demonstrate real-time embedded software and industrial control skills for a Ride & Show Controls Engineer role.
## Demo video: https://youtu.be/s6daFCSxui0
## Architecture
===============================================================================
                    HULK LAUNCH & SAFETY CONTROL SYSTEM
===============================================================================

                         CORE 1 — CONTROL PLANE
-------------------------------------------------------------------------------

+---------------------------+
| Hulk Input Producer       |
| Priority 8                |
|---------------------------|
| Generates ride commands   |
| Records timestamp         |
| Increments heartbeat      |
+-------------+-------------+
              |
              | xQueueSend()
              | Queue<RideItem_t>
              v
+---------------------------+
| Ride State Machine        |
| Priority 8                |
|---------------------------|
| Locks restraints          |
| Confirms track clear      |
| Approves/blocks dispatch  |
| Updates ride state        |
+-------------+-------------+
              |
              | Sets DATA_PROCESSED bit
              v

 Producer Task                         State-Machine Task
      |                                        |
      | DATA_PRODUCED                          | DATA_PROCESSED
      +--------------------+-------------------+
                           |
                           v
              +---------------------------+
              | Event Group               |
              |---------------------------|
              | Waits for both bits       |
              | Synchronizes the pipeline |
              +-------------+-------------+
                            |
                            | xEventGroupWaitBits()
                            v
              +---------------------------+
              | Pipeline Coordinator      |
              | Priority 9                |
              |---------------------------|
              | Confirms cycle complete   |
              | Clears rendezvous bits    |
              +-------------+-------------+
                            |
                            | xTaskNotifyGive()
                            v
              +---------------------------+
              | Operator Display          |
              | Priority 12               |
              |---------------------------|
              | Shows current ride state  |
              | Reports safety status     |
              +---------------------------+


                       BUTTON INTERRUPT PATH
-------------------------------------------------------------------------------

 GPIO 18 Button
       |
       | Falling-edge interrupt
       v
+---------------------------+
| Button ISR                |
|---------------------------|
| Debounces input           |
| Records ISR timestamp     |
| Sends direct notification |
| Requests immediate yield  |
+-------------+-------------+
              |
              | vTaskNotifyGiveFromISR()
              v
+---------------------------+
| Operator Display          |
| Measures wakeup latency   |
+---------------------------+


                         CORE 0 — MONITORING
-------------------------------------------------------------------------------

+---------------------------+
| Web/Terminal Monitor      |
| Priority 4                |
|---------------------------|
| Queue depth               |
| Event-group bits          |
| Per-task heartbeats       |
| Current ride state        |
| Timing measurements       |
+---------------------------+
The producer sends typed ride commands through a FIFO queue to the ride state machine. The producer and state-machine tasks set event-group bits, and the coordinator waits for both stages to complete before notifying the operator display. The web or terminal monitor runs separately on Core 0 so networking and display activity do not interfere with the real-time Core-1 pipeline.

The button ISR also signals the operator-display task directly. The measured direct-notification wakeup latency was 12 µs minimum, 16 µs average, and 32 µs maximum over 68 samples.
## Tasks & timing (WCET evidence)

| Task | Period T | WCET C | U = C/T | Priority | Deadline |
|---|---:|---:|---:|---:|---:|
| Hulk Input Producer | 500 ms | 33 µs | 0.000066 | 8 | 500 ms |
| Ride State Machine | 500 ms | 21,762 µs | 0.043524 | 8 | 500 ms |
| Pipeline Coordinator | 500 ms | 5,515 µs | 0.011030 | 9 | 500 ms |
| Operator Display | 200 ms | 42,418 µs | 0.212090 | 12 | 200 ms |

Total Core-1 utilization:

U = 0.000066 + 0.043524 + 0.011030 + 0.212090

U = 0.266710

For four Core-1 tasks, the Rate-Monotonic sufficient bound is:

U_RM = 4(2^(1/4) - 1) = 0.756828

Because 0.266710 < 0.756828, the task set passes the Rate-Monotonic sufficient utilization test and is guaranteed schedulable under the assumptions of the test. The task set also passes the basic EDF utilization test because 0.266710 < 1.0.

The monitor task measured a WCET of 37,720 µs, but it runs on Core 0 and is therefore excluded from the Core-1 utilization calculation.
## Hazard analysis & standard mapping

| Hazard | Possible Effect | Mitigation | Conceptual Standard Mapping |
|---|---|---|---|
| Dispatch occurs before safety checks are complete | The coaster could launch under unsafe conditions | The ride state machine blocks dispatch unless restraints are locked and the track is confirmed clear | ASTM F2291 — safety-related control systems and ride interlocks |
| Track-clear sensor fails or reports an invalid condition | The controller cannot confirm that the launch path is safe | The system remains outside the ready state, reports a fault, and denies dispatch | ASTM F2291 — fault detection and fail-safe control behavior |
| Operator button response is delayed | Important operator commands may be processed too slowly | The ISR performs minimal work, uses a direct task notification, and requests an immediate context switch | ASTM F2291 — operator controls and control-system response |
| Command queue becomes full | Normal ride commands may be delayed or dropped | The queue provides burst protection, uses a bounded send timeout, and reports dropped commands | ASTM F2291 — predictable control-system operation and fault reporting |
| Web-server activity interferes with control tasks | Increased latency or jitter could delay ride-state updates | The web server runs on Core 0 while the real-time control pipeline runs on Core 1 | ASTM F2291 — separation and reliability of safety-related control functions |
| A task stops running or becomes blocked | The control pipeline may stop progressing | Per-task heartbeat counters allow the monitor to detect a stalled task | ASTM F2291 — system monitoring and fault detection |
| Mechanical button bounce creates repeated interrupts | Duplicate or unintended commands may be generated | The button ISR uses a debounce interval to reject repeated edges | ASTM F2291 — reliable operator-input handling |
| Event-group bits are not cleared correctly | The coordinator may incorrectly report that a pipeline cycle is complete | Rendezvous bits are cleared after both required stages have completed | ASTM F2291 — control-state integrity and synchronization |


## Graceful degradation

The injected failure simulates a track-clear sensor that cannot confirm that the launch path is safe. The failure is detected when the state machine processes the `TRACK_CLEAR` command while fault injection is enabled.

The monitor reports:

[FAULT] track-clear sensor failed
[SAFETY] system remains not ready
[SAFETY] dispatch blocked - checks incomplete
## Build & run
This project was built in C using ESP-IDF and FreeRTOS for an ESP32-S3 DevKitC-1 in Wokwi.

To run the project:

1. Open the Wokwi project.
2. Start the simulation.
3. Open the serial monitor.
4. Press the button on GPIO 18 to test the interrupt and latency measurements.

For normal operation and WCET collection, use:

```c
#define LATENCY_TEST_MODE 0
#define USE_DIRECT_NOTIFICATION 1
#define INJECT_TRACK_FAULT 0

## Tailored for
Ride & Show Controls Engineer — The project demonstrates skills used in attraction-control work, including deterministic task scheduling, operator controls, safety interlocks, interrupt handling, state-machine design, inter-task communication, fault detection, timing analysis, live diagnostics, and safe rejection of invalid commands.
Final Reflection:
For this project, I built a Hulk-themed roller-coaster launch and safety control system using an ESP32-S3 and FreeRTOS. The final system combined several ideas from the earlier applications, including dual-core task separation, queues, event groups, direct task notifications, interrupt handling, WCET measurements, live monitoring, and fault injection. Bringing all of these features together helped me see the difference between making individual FreeRTOS features work and designing one complete real-time system.

One thing I would do differently is define the full system architecture and measurement plan before writing the final code. I initially focused on getting each task and IPC mechanism working, then added timing measurements, fault injection, and monitoring afterward. This made the code harder to organize because some task behavior had to be changed to collect accurate results. For example, the responder task could receive notifications from both the coordinator and the button ISR, which made it necessary to separate normal operation from the latency test. In a future project, I would first define each producer, consumer, data type, timing requirement, failure response, and measurement point. I would also create separate operating modes for normal control, performance testing, and fault injection from the beginning.

The most difficult part was integration. The queue, event group, direct notification, ISR, state machine, and monitor were each manageable by themselves, but combining them created interactions that were not always obvious. Timing measurements were also harder than expected because the measured value could include scheduling delays, background activity, logging overhead, or notifications from the wrong source. I had to make sure both the direct-notification and binary-semaphore tests used the same task priority, processor core, workload, and number of button presses. Another challenge was deciding what should run on each core. Moving the monitor to Core 0 helped isolate the real-time pipeline on Core 1, but it also meant the monitor was reading information updated by another core. This reinforced the importance of thinking about shared data and not only task functionality.

The fault-injection test was one of the most useful parts of the project. I simulated a failed track-clear sensor and verified that the ride remained in a safe state. Instead of crashing, freezing, or continuing with the launch, the controller blocked the dispatch command while the monitor, queue, display, and task heartbeats continued operating. This showed me that graceful degradation is more meaningful than simply detecting an error. A useful control system should identify the fault, communicate it clearly, reject unsafe behavior, and continue providing enough functionality for an operator to understand and reset the system.

The most valuable lesson I learned is that IPC primitives should be selected based on the communication contract. A queue was appropriate because the ride commands contained data and needed to remain in FIFO order. The event group was useful because the coordinator needed to wait for multiple Boolean conditions together. A direct task notification was appropriate for the button ISR because one producer was waking one known task with low overhead. My measurements supported this choice: the direct notification averaged 20 microseconds, while the binary semaphore averaged 22 microseconds. The difference was small, but the test demonstrated why design choices should be supported with measured evidence rather than assumptions.

The main idea I will carry into future projects and interviews is that real-time embedded design is not just about creating concurrent tasks. It requires clearly defined communication paths, bounded blocking, timing measurements, fault handling, and safe behavior when something goes wrong. This project gave me a stronger understanding of how to explain both what a system does and why its architecture was chosen.
