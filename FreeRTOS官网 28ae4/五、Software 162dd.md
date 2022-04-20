# 五、Software Timer Management

# 1. 章节介绍和范围

Software Timer 被用于在未来的某个时间调度，或者以一个固定频率运行。该功能通过 Software Timer 回调函数被执行。

Software Timer 通过 FreeRTOS Kernel 被执行，和硬件Timer和Counter并没有关系，也无需硬件支持。

Software Timer 本身基本不占用系统时间，但其回调函数会占用。

配置：

1. 加入FreeRTOS/Source/timers.c文件到你的项目中
2. 设置 configUSE_TIMERS → 1 (on FreeRTOSConfig.h)

范围：

- Software timer的特性和Task的特性的比较
- RTOS的Deamon task
- Timer 命令队列
- 一次性的Software Timer和周期性的software timer之间的区别
- 关于Software timer的创建、开始、重置和改变周期

# 2. Software Timer 的回调函数

回调函数原型：

```c
void ATimerCallback(TimerHandle_t xTimer);
```

Software timer回调函数的执行时间必须短，且不能够进入Blocked状态（即执行完任务后退出）

***Note：***

Software timer回调函数在上下文切换中执行，所以它不能调用将使他进入Blocked状态的FreeRTOS API，例如vTaskDelay()。在xTicksToWait参数为0的情况下，可以调用如xQueueReceive()函数。

# 3. Software Timer的属性和状态

## 属性

1. Period (周期)
指Software timer开始到其回调函数被执行的时间
2. One-shot Timer
仅执行回调函数一次，可以手动重置
3. Auto-reload Timer
每次到期后自动重新开始
One-shot 和 Auto-reload Timer的时序对比
    
    ![Untitled](%E4%BA%94%E3%80%81Software%20162dd/Untitled.png)
    

## 状态

1. Dormant (休眠)
可以通过句柄被引用，但不会被执行
2. Running
在一定周期后会执行回调函数

下图展示了Dormant和Running在auto-reload和one-shot timer中可能的转换。他们关键的不同在于timer到期后，auto-reload会重新进入Running状态，而one-shot 则会进入Dormant状态。

![Untitled](%E4%BA%94%E3%80%81Software%20162dd/Untitled%201.png)

可以在任意时刻使用xTimerDelete()来删除Timer。

# 4. Software Timer的上下文切换

## RTOS Daemon Task

全部的Software timer回调函数均在同一个RTOS daemon task的上下文中执行。

Daemon task 是一个标准的FreeRTOS任务，当调度器开始时自动创建。它的优先级和堆栈大小通过设置configTIMER_TASK_PRIORITY和configTIMER_TASK_STACK_DEPTH配置（两个宏均位于FreeRTOSConfig.h中）。

**Software Timer回调函数必不能调用能够进入Blocked状态的FreeRTOS API。**

## Timer Command Queue

Software timer API函数从调用任务发送commands到Daemon task 所在的 ”timer command queue“。（命令包含：”start a timer”, “stop a timer” and “reset a timer”）

Timer command queue是一个标准的FreeRTOS队列，当调度器开始时自动创建。它的长度通过设置configTIMER_QUEUE_LENGTH配置（位于FreeRTOSConfig.h中）。

大致用法：

![Untitled](%E4%BA%94%E3%80%81Software%20162dd/Untitled%202.png)

## Daemon Task Scheduling

Daemon task 和其他FreeRTOS任务有着一样调度机制。当它是最高优先级运行时，它仅处理命令列表和软件定时器的回调函数。

命令发送到timer command queue时包含了时间戳。这个时间戳被用来说明应用任务发送命令和Daemon task 处理同一个命令之间所经历的任意时间。例如：如果”start a timer“命令中包含一个10ticks的周期信息，那么这个时间戳被用来确保从命令发出去后的10ticks后到期。而不是在daemon task 处理该命令后的10ticks后到期。

图解该意为，在t2时刻发送的start timer的命令是在t2时刻就开始计算该timer的周期，而非从t4时刻的daemon处理该条指令开始计算。

![Untitled](%E4%BA%94%E3%80%81Software%20162dd/Untitled%203.png)

# 5. 创建和开始Software Timer

## xTimerCreate()

```c
/*
创建Software timer
Parameter: 
	pcTimerName: 描述timer的名称，用于debug时我们能够识别的用途。
	xTimerPeriodInTicks: 该Timer的周期。使用pdMS_TO_TICKS()来获取被转化的ticks值。
	uxAutoReload: pdTRUE -> 重装载；pdFALSE -> 一次性时钟。
	pvTimerID: 每一个software timer都由同一个ID值。该值是一个void类型的指针，应用开发者可以
						 将其用于任意目的。例如：当多个software timer使用同一个回调函数时，ID可以用来
						 触发特定的存储内容。
						 pvTimerID 在任务被创建时赋值。
	pvCallbackFunction: Software Timer回调函数。

Return:
	Type: TimerHandle_t
	Value: 
		NULL: 创建失败，没有足够的heap去创建必要的数据结构。
		non-NULL: 创建成功，返回值是Timer的句柄，用来被其他FreeRTOS Timer API引用。创建成功时
							的Software timer处于Dormant状态
*/
TiemrHandle_t xTimerCreate(const char* const pcTimerName,
													 TickType_t xTimerPeriodInTicks,
													 UBaseType_t uxAutoReload,
													 void* pvTimerID,
													 TimerCallbackFunction_t pxCallbackFunction);
```

## xTimerStart()

```c
/*
开始Software timer。被用来将Software timer从Dormant状态转变会Running，或者重置一个已经开始
的Software timer。
xTimerStart()可以在调度器开始前被调用，但如果这样做了，Software timer 实际上并没有开始，直
到调度器开始后开始工作。
***Note: 绝不能在中断服务进程中调用xTimerStart(),应该使用ISR安全版本--xTimerStartFromISR()***

Parameter:
	xTimer: Software Timer的句柄,即xTimerCreate()的返回值。
	xTicksToWait: xTimerStart使用timer commmand queue去发送'start a timer'的命令至daemon
		task。该参数规定了调用任务进入blocked等待命令进入command queue的时间。
				当该值为0时，即使timer command queuex是满的，TimerStart()也立即返回。
				如果 INCLUDE_vTaskSuspend -> 1，且设置该值为portMAX_DELAY将会导致任务永久的处于
		Blocked状态，直至timer command queue有空间装入新的command。
				在调度器启用前就调用xTimerStart()，那么该值将被忽视，就像该值被设为0.
Return：
	Value：
		pdPASS: command成功发送到timer command queue.
				如果Daemon task的优先级高于调用该函数的任务，那么调度器将会确保函数返回前command
			已经被处理。那是因为Daemon task在timer command queue中有数据时会立马抢占cpu。
				如果xTicksToWait有确定值，那么在未超时或者命令为进入timer command queue前，调用该
			函数的任务都将处于Blocked状态。
		pdFALSE: 失败，当command无法被写入timer command queue时(Queue为满)。
*/
BaseType_t xTimerStart(TimerHandle_t xTimer, TickType_t xTicksToWait);
```

# 6. The Timer ID

每一个software timer都有一个ID，它可以被应用开发者用作任何目的。这个ID被存储在类型为(void*)中，所以他可以直接保存整数类型值，任何对象的指针，又或者作为函数指针。

与其他software timer API函数不同，vTimerSetTimerID() 和 pvTimerGetTimerID()接口是直接访问software timer，而不是发送command至timer command queue。

## vTimerSetTimerID()

```c
/*
Parameter:
	xTimer: Software timer的句柄（由[xTimerCreate()](%E4%BA%94%E3%80%81Software%20162dd.md)返回）。
	pvNewID: 新的定时器ID。
*/
void vTimerSetTimerID(const TimerHandle_t xTimer, void *pvNewID);
```

## pvTimerGetTimerID()

```c
/*
Parameter:
	xTimer: Software timer的句柄（由[xTimerCreate()](%E4%BA%94%E3%80%81Software%20162dd.md)返回）。
Return：
	Value:
		Software timer的ID。
*/
void *pvTimerGetTimerID(TimerHandle_t xTimer);
```

# 7. 改变Timer的周期

## xTimerChangePeriod()

```c
/*
***如果xTimerChangePeriod()被使用时，timer已经处于running，那么timer将会使用新的周期值去重
新计算到期时间。重新计算的到期时间从调用xTimerChangePeriod()开始，与timer开始时间无关。
如果xTimerChangePeriod()被调用时，timer处于Dormant，那么timer将会计算一个到期时间，并转
换到running状态。
在ISR服务进程中，不允许调用xTimerChangePeriod(),应该调用ISR-Safe版本--
xTimerChangePeriodISR().***

Parameter:
	xTimer: Software timer的句柄（由[xTimerCreate()](%E4%BA%94%E3%80%81Software%20162dd.md)返回）。
	xNewTimerPeriodInTicks: 新的周期，使用pdMS_TO_TICKS()获取ticks值。
	xTicksToWait: 该函数发送命令至deamon task的timer command queue中。所有再其queue满时需要
		等待一段时间queue能接受命令。
			内容如同[xTimerStart()](%E4%BA%94%E3%80%81Software%20162dd.md)。
Return: 返回内容类似[xTimerStart()](%E4%BA%94%E3%80%81Software%20162dd.md)。
	Value:
		pdPASS: 成功
		pdFALSE: 失败
*/
BaseType_t xTimerChangePeriod(TimerHnadle_t xTimer,
															TickType_t xNewTimerPeriodInTicks,
															TickType_t xTicksToWait);
```

# 8. 重置Software Timer

重置Software timer意味着重新开始定时器，定时器的到期时间根据重置时刻开始计算，而不是timer开始的时间。如下图：

![Untitled](%E4%BA%94%E3%80%81Software%20162dd/Untitled%204.png)

## xTimerReset()

```c
/*
xTimerReset()也可以时处于Dormant状态的timer转变为running。
***Note：禁止在ISR进程中调用该函数，应该使用ISR-Safe版本 xTimerResetFromISR().***
Parameter:
	xTimer:使句柄的Software timer重置或者开始工作。（由[xTimerCreate()](%E4%BA%94%E3%80%81Software%20162dd.md)返回）。
	xTicksToWait: 内容如同[xTimerStart()](%E4%BA%94%E3%80%81Software%20162dd.md)。
Return:
	Value: 内容如同[xTimerStart()](%E4%BA%94%E3%80%81Software%20162dd.md)。
*/
BaseType_t xTimerReset(TimerHandle_t xTimer, TickType_t xTicksToWait);
```