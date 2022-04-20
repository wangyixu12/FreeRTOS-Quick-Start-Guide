# 三、Task Management

# 任务函数原型

```c
void ATaskFunction(void *pvParameters)
{
	int32_t lVariableExample = 0;
	for (;;) {
	}
	vTaskDelete(NULL);
}
```

函数内容需要有个不会退出的死循环，且函数类型一定是void。在任何情况下均不允许返回。如果任务不再需要使用，应该被显示的删除。
每一个任务的func都可被用来创建多个任务，且每个任务均是独立执行的实例，它们拥有自己的stack和变量。

# 创建任务

```c
/*
创建任务原型。

参数解析：
	pvTaskCode: 任务函数
			pcName: 描述任务名称，增加代码可读性
usStackDepth: 内核分配任务的堆栈大小，单位为stack位数--如果stack是32bit，则实参为100意味着
							实际的stack大小为100*4 bytes.
pvParameters: 该参数将会分配进任务函数中
	uxPriority: 定义任务优先级(0~(configMAX_PRIORITIES-1))，数值越大，优先级越高。
							configMAX_PRIORITIES在FreeRTOSConfig.h中被定义。如果宏
							configUSE_PORT_OPTIMISED_TASK_SELECTION->1,将启动优化优先级方法，此时
							configMAX_PRIORITIES不可以超过32.建议configMAX_PRIORITIES尽量保持必要的最小值。
							因为configMAX_PRIORITIES越大，消耗的RAM也就越大。
pxCreatedTask:用来传递任务句柄，为后续如果需要使用修改任务优先级或者删除任务等操作提供任务句柄。
							如果没有使用，可以设置为NULL。

Return value:
	pdPASS: 任务创建成功
	pdFAIL: 由于没有足够的RAM用来分配给任务，导致任务创建失败。
*/
BaseType_t xTaskCreate( TaskFunction_t pvTaskCode,
												const char * const pcName,
												uint16_t usStackDepth,
												void *pvParamters,
												UBaseType_t uxPriority,
												TaskHandle_t *pxCreatedTask );
```

# 任务状态

![Untitled](%E4%B8%89%E3%80%81Task%20Man%20897b3/Untitled.png)

## State

### 1. The Blocked State

Blocked 状态是 Not Running 的子状态。

任务在Blocked State 等待两种类型事件发生：

1. 时间相关任务，例如任务设置了进入Blocked等待的Timeout
2. 同步事件发生，如其他事件或者中断的事件源。例如：queue，binary semaphores, counting semaphores, mutexes, recursive mutexes, event groups 和 获取到notifications.

事件可能同时拥有时间相关和同步事件相关属性。类似串口接收数据。

### 2. The Suspended State

Suspended 同样是 Not Running 的子状态。当任务处于Suspended状态时，调度器将无法使用它。仅有以下接口可以使任务进入 Suspended 和 退出 Suspended：

```c
// Enter Suspended State
void vTaskSuspend( TaskHandle_t xTaskToResume );
// Quit Suspended State
void vTaskResume( TaskHandle_t xTaskToResume );
// Quit Suspended State on interruption security
BaseType_t xTaskResumeFromISR( TaskHandle_t xTaskToResume );
```

大多数应用不使用Suspended 状态。

### 3. The Ready State

Ready 同样是 Not Running 的子状态。这种状态的任务处于可以Run，但还未Run的状态。

### 4. The Running State

Running 状态以为任务代码正在被执行。

## Delay API

使用Delay API 让 Running 状态下的任务进入 Blocked。

```c
/*
固定延时xTicksToDelay时间
Parameter:
	xTicksToDelay: Blocked 的时间，超时后进入 Ready state。一般使用pdMS_TO_TICKS()获取。
*/
void vTaskDelay( TickType_t xTicksToDelay );
```

```c
/*
延时xTimeIncrement - pxPreviousWakeTime时间，即可认为使用该函数的代码段以固定频率运行。
Note：
	在运行前需要获取pxPreviousWakeTime的值：
		TickType_t xLastWakeTime = xTaskGetTickCount()；

Parameter:
	pxPreviousWakeTime: 该参数假定该API被使用在一个固定周期频率的任务内，被当做下一次离开
											Blocked 状态的时间参考点。
			xTimeIncrement: 同上一个参数，该参数作为固定频率值。一般使用pdMS_TO_TICKS()获取。
*/
void vTaskDelayUntil( TickType_t * pxPreviousWakeTime, TickType_T xTimeIncrement );
```

# 时间测量和Tick中断

这里引出‘Time Slicing’的概念，即调度器执行的周期，是一种周期性的中断—tick interrupt。通过配置宏configTICK_RATE_HZ来决定时间片的中断频率。典型值为100—即100Hz（10ms）。

例如：configTICK_RATE_HZ → 1000。意味着时间片为1000Hz（1ms）。

图解：

![Untitled](%E4%B8%89%E3%80%81Task%20Man%20897b3/Untitled%201.png)

这里假定Task 1和Task 2的优先级相同，当时间片发生时，调度器会将RUNNING TASK → BLACKING, READY → RUNNING

***Note：***FreeRTOS API总是使用ticks来带作为特定的时间值，**一般使用pdMS_TO_TICKS()来获取一定时间对应的ticks数量。但值得注意的是，该方法仅在configTICK_RATE_HZ不大于1000时有效。使用pdMS_TO_TICKS()是被推荐的方法，这样即使tick的频率发生改变，也无需修改代码。**

例：

```c
// 获取200ms对应的时间片数量
TickType_t xTimeInTicks = pdMS_TO_TICKS(200);
```

# 空闲状态任务和空闲状态的钩子函数

## Idle Task

当vTaskStartScheduler()被调用时，Idle任务被调度器自动创建。

空闲任务拥有最低优先级，以确保它不会阻碍高优先级的任务进入Running状态。配置[configIDLE_SHOULD_YIELD](%E4%B8%89%E3%80%81Task%20Man%20897b3.md)，从而避免Idle任务占用太多处理器时间，让处理器跟高效的处理应用任务。

***Note： 如果应用使用了vTaskDelete()接口，那么Idle任务不能缺少处理时间（即被任务占用）。因为Idle任务是在任务被删除后清理内核资源的。***

## Idle Task Hook Function

Idle task 有对应的钩子函数，我们可以将特定的应用代码添加进这个钩子函数，从而在每次调用Idle时，自动的执行我们相对应的代码。

普遍的用法包含：

1. 执行低优先级，后台，或者连续处理功能。
2. 统计空闲处理器数量（Idle 任务仅在所有的高优先级任务没有被执行的情况下才能运行，所以统计处理器分配了多少时间点任务给到Idle，就能清晰的了解到有多少处理器时间是空闲的）
3. 让处理器进入一个低功耗模式，提供一个简单且自动的方法来省电。

空闲任务钩子函数的限制：

1. 空闲任务钩子函数绝不能进入Block或者Suspend状态。（以任何方法堵塞Idle任务，都可能导致其他任务无法进入Running状态）。（执行完任务后立即退出）
2. 如果一个应用使用了vTaskDelete()接口，那么Idle任务钩子函数必须始终在合理的时间内返回到他的调用函数中。这是由于Idle任务是有责任在任务被删除时释放内核资源。如果Idle任务始终在钩子函数中，那么清理就永远不会发生。

Idle任务钩子函数的原型如下：

```c
// The Idle task hook function name and prototype
// 配置 configUSE_IDLE_HOOK → 1 （FreeRTOSConfig.h）
void vApplicationIdleHook(void);
```

# 改变Task的优先级

```c
/*
Parameter:
				 pxTask: 由[xTaskCreate](%E4%B8%89%E3%80%81Task%20Man%20897b3.md)()创建的任务句柄
	uxNewPriority: 新优先级
*/
void vTaskPrioritySet( TaskHandle_t pxTask, UBaseType_t uxNewPriority );

/*
Brief:
	需要设置 INCLUDE_uxTaskPriorityGet -> 1 (FreeRTOSConfig.h)
Parameter:
	pxTask: 由[xTaskCreate](%E4%B8%89%E3%80%81Task%20Man%20897b3.md)()创建的任务句柄。如果为当前任务本身，该值可以设置为NULL。
Return:
	目前被请求分配的优先级。
*/
UBaseType_t uxTaskPriorityGet( TaskHandle_t pxTask );
```

# 删除Task

```c
/*
Brief:
	需要设置INCLUDE_vTaskDelete -> 1 (FreeRTOSConfig.h)。在IDLE任务中释放被分配的内存，所以
	调用vTaskDelete()后必须要有个完整的idle任务。
Parameter:
	pxTakeToDelete: 由[xTaskCreate](%E4%B8%89%E3%80%81Task%20Man%20897b3.md)()创建的任务句柄。删除自身时，可以填NULL。
*/
void vTaskDelete( TaskHandle_t pxTakeToDelete );
```

# 调度算法

## 配置调度算法

通过配置 ***configUSE_PREEMPTION*** & ***configUSE_TIME_SLICING*** (FreeRTOSConfig.h) 来改变调度算法。

***configUSE_TICKLESS_IDLE*** 同样也可以影响调度算法，因为他可使tick中断完全关闭。configUSE_TICKLESS_IDLE专为那些最小化能耗的应用使用。一般情况下默认为0。

## 基于时间片的Prioritized Pre-emptive Scheduling

该算法也称为“Fixed Priority Pre-emptive Scheduling with Time Slicing”.

宏配置如下：

![Untitled](%E4%B8%89%E3%80%81Task%20Man%20897b3/Untitled%202.png)

算法条目解释：

1. Fixed Priority
不会在调度器中修改任务的优先级，但不限制任务中修改自身或者其他任务的优先级。
2. Pre-emptive
当另一个任务的优先级高于目前Running状态下的任务时，调度器迫使当前Running的任务转变为Ready，而使优先级更高的任务进入Running状态。
3. Time Slicing
时间片使优先级相同的任务分享系统处理时间。

配置configIDLE_SHOULD_YIELD (FreeRTOSConfig.h) 来设置IDLE任务执行时间:

| Constant | Value | Description |
| --- | --- | --- |
| configIDLE_SHOULD_YIELD | 0 | IDLE 任务将在整个时间片中处于Running状态，除非它被更高级的任务打断 |
|  | 1 | 如果其他Idle优先级的任务处于Ready状态，那么Idle任务将在每次完成后自动挂起 |

configIDLE_SHOULD_YIELD → 1:

![Untitled](%E4%B8%89%E3%80%81Task%20Man%20897b3/Untitled%203.png)

configIDLE_SHOULD_YIELD → 0:

![Untitled](%E4%B8%89%E3%80%81Task%20Man%20897b3/Untitled%204.png)

## Prioritized Pre-emptive Scheduling (without Time Slicing)

除了不使用时间片分享处理器时间意外，算法均与上文相同。

配置：

![Untitled](%E4%B8%89%E3%80%81Task%20Man%20897b3/Untitled%205.png)

当不使用时间片时，调度器将在以下情况下选择新的任务进入Running状态：

1. 更高优先级的任务进入Ready状态
2. 当前任务进入了Blocked或者Suspended状态

仅建议对FreeRTOS有经验的人使用这种不使用时间片的算法。

以下为使用这种算法的任务时间图：

![Untitled](%E4%B8%89%E3%80%81Task%20Man%20897b3/Untitled%206.png)

## Co-operative Scheduling

Co-operative 算法将使调度器仅在当前运行任务进入Blocked和显式的调用了taskYIELD()后才起作用。任务不会被抢占，所以时间片也不能被使用。

配置：

![Untitled](%E4%B8%89%E3%80%81Task%20Man%20897b3/Untitled%207.png)

以下为使用这种算法的任务时间图：

![Untitled](%E4%B8%89%E3%80%81Task%20Man%20897b3/Untitled%208.png)