# EEE4775CapstoneJuanLugo
The final integration capstone. Compose Apps 1–5 into one coherent themed system you can demo, document, and put in your portfolio. 
One-Sentence theme:
A dual-core roller-coaster launch and safety control system that coordinates operator commands, safety interlocks, ride-state transitions, and live monitoring, built to demonstrate real-time embedded software and industrial control skills for a Ride & Show Controls Engineer role.
## Demo video: https://youtu.be/s6daFCSxui0
## Architecture
HULK LAUNCH & SAFETY CONTROL SYSTEM

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
