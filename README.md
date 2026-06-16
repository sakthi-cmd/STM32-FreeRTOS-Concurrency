# STM32 FreeRTOS Concurrency & Mutual Exclusion

This repository contains a production-ready implementation of real-time multi-threaded scheduling on an ARM Cortex-M4 microcontroller (**STM32L4 series**). It demonstrates advanced embedded operating system concepts, focusing on **Thread Safety, Dynamic Task Scheduling, Inter-Task Coordination, and Race Condition Prevention** using the CMSIS-RTOS v2 API layer built on top of FreeRTOS.

## Technical Architecture Overview

The system architecture decouples raw computational scaling from monitoring pipelines by orchestrating three concurrent tasks running via a priority-based, preemptive RTOS kernel.



### Concurrency Design & Task Execution Flow
1. **Task 1: Math Increment Stream A (`StartTask1`)**
   * *Priority:* Normal (`osPriorityNormal`)
   * *Execution Lifecycle:* Runs a strict 10,000-iteration pipeline. To simulate deep, multi-cycle asynchronous assembly instructions, explicit context-switch checkpoints (`osThreadYield()`) are introduced midway through the read-modify-write process.
2. **Task 2: Math Increment Stream B (`StartTask2`)**
   * *Priority:* Normal (`osPriorityNormal`)
   * *Execution Lifecycle:* Concurrently targets the same memory space as Task 1 across another 10,000 iterations, providing heavy resource contention to test real-time synchronization boundaries.
3. **Task 3: Diagnostic Telemetry Monitor (`StartTask3`)**
   * *Priority:* Normal (`osPriorityNormal`)
   * *Execution Lifecycle:* Periodically unblocks from an efficient sleep state (`osDelay`) to verify memory boundary integrity and stream localized state logs via the serial interface.

---

## The Critical Section Problem & Mutex Architecture

When multiple executing tasks concurrently access shared global variables, a **Race Condition** occurs. A standard increment operation (`counter++`) is non-atomic at the assembly level, translating to a discrete **Read-Modify-Write** hardware routine:



Without synchronization, forcing an explicit context switch (`osThreadYield()`) immediately after a register read allows the competing thread to load an outdated memory state, leading to massive, silent data corruption.

### Resolution via Mutual Exclusion (Mutex)
This firmware enforces strict mutual exclusion boundaries using an RTOS Mutex primitive (`counterMutexHandle`). 

```c
void StartTask1(void *argument)
{
    uint32_t temp;
    for(int i = 0; i < 10000; i++)
    {
        /* Enforce mutual exclusion lock over the shared memory lane */
        osMutexAcquire(counterMutexHandle, osWaitForever);
        
        temp = counter;
        osThreadYield(); // Context switch forced: Task 2 blocks safely on the locked Mutex

        temp = temp + 1;
        osThreadYield(); 

        counter = temp;
        
        /* Release lock to yield control back to the scheduler */
        osMutexRelease(counterMutexHandle);
        osThreadYield();
    }
}
