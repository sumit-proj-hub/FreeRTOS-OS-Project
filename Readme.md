# Mid Evaluation Report

## Real Time Operating System
A Real-Time Operating System (RTOS) is a specialized operating system designed to manage tasks that must execute within strict timing constraints. Unlike general-purpose operating systems (GPOS) like Windows or Linux, which prioritize multitasking and user interaction, an RTOS focuses on predictability and determinism, ensuring that critical operations occur precisely when needed.

### Step 1: Environment Setup

#### Clone FreeRTOS Source
```bash
git clone --recurse-submodules https://github.com/FreeRTOS/FreeRTOS-Kernel.git
cd FreeRTOS-Kernel
```

#### Configure Build System
Use the template `FreeRTOSConfig.h` from the `template_configuration` directory.

Set key parameters:

```c
#define configUSE_PREEMPTION 1
#define configUSE_TIME_SLICING 1
#define configMAX_PRIORITIES 32
#define configUSE_STATS_FORMATTING_FUNCTIONS 1
```

**1. configUSE_PREEMPTION**

- **Function:** Enables/disables preemptive scheduling
- **Value 1 Behavior:**
  - Higher-priority tasks immediately preempt lower-priority ones
  - Uses priority-based scheduling matching SysBIOS policy
  - Requires careful interrupt priority configuration
- **Critical Implementation Note:**
  - Setting to 0 (cooperative) requires removing all preemption-dependent code and can cause unexpected aborts in some ports

**2. configUSE_TIME_SLICING**

- **Function:** Controls round-robin scheduling for equal-priority tasks
- **Value 1 Behavior:**
  - Tasks with same priority share CPU in fixed time slices
  - Time quantum = `portTICK_PERIOD_MS` (typically 1ms)
  - Maintains mutual exclusion within priority levels

```c
/* Enables fairness but increases context switches */
#define configUSE_TIME_SLICING 1 /* Recommended for multi-tasking apps */
```

**3. configMAX_PRIORITIES**

- **Function:** Defines number of available task priority levels
- **Value 32 Implications:**
  - Priorities range 0 (lowest) to 31 (highest)
  - Each priority level consumes 4 bytes RAM for ready list
  - Must align with `configKERNEL_INTERRUPT_PRIORITY`

```c
/* TI PDK default:16, Nordic SDK:3-7 */  
#define configMAX_PRIORITIES 32 /* For complex task hierarchies */
```

**4. configUSE_STATS_FORMATTING_FUNCTIONS**

- **Function:** Enables human-readable task statistics
- **Value 1 Enables:**
  - `vTaskList()`: ASCII table of tasks/states
  - `vTaskGetRunTimeStats()`: CPU utilization metrics

```c
/* Dependency chain */
#define configUSE_TRACE_FACILITY 1 /* Required */
#define configUSE_STATS_FORMATTING_FUNCTIONS 1 /* Enables debug CLI */
```

### Step 2: Scheduler Customization

#### Modify Scheduling Policy

Edit `tasks.c` to implement:

- Priority-based preemption (default)
- Round-robin for equal priorities

Add custom scheduling hooks:

```c
void vApplicationIdleHook(void) {
    // Custom idle task monitoring
}

void vApplicationTickHook(void) {
    // Custom tick handling
}
```

#### Add Scheduler APIs

Extend `task.h` with custom functions:

```c
void vTaskSetCustomPriority(TaskHandle_t xTask, UBaseType_t uxNewPriority);
void vTaskSetQuantum(TaskHandle_t xTask, TickType_t xQuantum);
```

### Step 3: Graphical Task Monitoring

#### Implement Data Collection

```c
void vTaskMonitor(void *pvParameters) {
    while(1) {
        TaskStatus_t *pxTaskStatusArray;
        UBaseType_t uxArraySize = uxTaskGetNumberOfTasks();
        pxTaskStatusArray = pvPortMalloc(uxArraySize * sizeof(TaskStatus_t));

        if(pxTaskStatusArray != NULL) {
            uxTaskGetSystemState(pxTaskStatusArray, uxArraySize, NULL);
            // Send data to GUI via IPC
            vSendMonitoringData(pxTaskStatusArray, uxArraySize);
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

#### Build Web Interface

Use FreeRTOS+TCP and HTTP server:

```c
// In main.c
xTaskCreate(http_server_task, "HTTP Server", 2048, NULL, 2, NULL);
```

#### Visualization Options

- Integrate Percepio Tracealyzer for professional tracing
- Create custom web UI with:

```xml
<div id="task-timeline"></div>
<script>
    // Use WebSocket to receive real-time task data
</script>
```

### Step 4: Custom IPC Implementation

#### Shared Memory IPC

```c
typedef struct {
    SemaphoreHandle_t xMutex;
    void *pvSharedBuffer;
    size_t xBufferSize;
} SharedMemoryIPC_t;

void vInitSharedMemoryIPC(SharedMemoryIPC_t *pxIPC, size_t xSize) {
    pxIPC->xMutex = xSemaphoreCreateMutex();
    pxIPC->pvSharedBuffer = pvPortMalloc(xSize);
    pxIPC->xBufferSize = xSize;
}
```

#### Event-Based Messaging

```c
// Custom event queue implementation
QueueHandle_t xCreateEventQueue(UBaseType_t uxQueueLength) {
    return xQueueCreateSet(uxQueueLength * 2);
}

BaseType_t xSendEvent(QueueHandle_t xQueue, Event_t *pxEvent) {
    return xQueueSendToBack(xQueue, pxEvent, portMAX_DELAY);
}
```

### Step 5: Integration & Testing

#### Create Demo Tasks

```c
void vDemoProducerTask(void *pvParams) {
    while(1) {
        // Use custom IPC
        xSendEvent(xEventQueue, &xEvent);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void vDemoConsumerTask(void *pvParams) {
    while(1) {
        xReceiveEvent(xEventQueue, &xEvent);
        // Process event
    }
}
```

#### Validation Steps

- Verify scheduler behavior using `uxTaskGetSystemState()`
- Test IPC mechanisms with boundary cases
- Monitor memory usage with `xPortGetFreeHeapSize()`
- Use Tracealyzer to validate real-time behavior

### Step 6: Documentation & Deployment

#### Build System Configuration

```text
add_subdirectory(FreeRTOS-Kernel)
target_link_libraries(your_project PRIVATE freertos_kernel)
```

#### Create Developer Guide

- Document custom scheduler APIs
- Provide IPC usage examples
- Include monitoring interface specifications

### Key Resources

- Official FreeRTOS documentation for scheduler fundamentals
- FreeRTOS kernel structure
- IPC verification research
- Task visualization tools