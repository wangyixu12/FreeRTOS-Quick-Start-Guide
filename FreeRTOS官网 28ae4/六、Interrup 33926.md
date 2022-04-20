# 六、Interrupt Management

# 1. 简介和本章范围

Event需要被考虑的内容如下 ：

- Event应该如何被检测？— 通常使用中断，但也可以使用查询
- 当中断被使用时，如何决定中断内执行内容和中断外执行内容的多少？— 通常中断要设计的尽可能简短
- 事件如何传达给非中断代码？如何更好的构建代码，使其以最好的方式适应潜在的异步事件的处理？

FreeRTOS并不强加特定的事件处理策略给应用开发者，但也确实提供了允许以简单和可维护的方式实现所选策略的功能。

Task 优先级和 Interrupt 优先级的概念差别：

- Task 在 FreeRTOS 中是与硬件毫无关系的软件功能，Task 优先级由开发者通过软件分配，并通过软件算法（如调度器）调度。
- 虽然是由软件编写，但Interrupt 服务程序是一种硬件功能。因为其是由硬件控制哪一个Interrupt服务程序运行以及何时运行。Task仅允许在无Interrupt发生下运行，所以其优先级低于Interrupt。所以Task无法抢占Interrupt。

## 本章范围

- 哪一些 FreeRTOS API可以在中断服务程序中使用
- 将中断处理推迟到task的方法
- Binary信号量和计数信号量的区别
- 如何使用队列在中断服务函数中传输数据
- 一些FreeRTOS ports中有效的中断嵌套模型

# 2. 在ISR中可以使用的FreeRTOS API

## Interrupt Safe API

中断中调用的FreeRTOS API不允许使中断进入Blocked状态，所以FreeRTOS对部分功能提供了两个版本的函数。一个版本为了task，一个版本为了Interrupt（相较task版本的API，在其函数名称末尾添加“FromISR”）。所以绝不要再ISR中调用没有“FromISR”结尾的FreeRTOS API。

## 使用单独的Interrupt Safe API的好处

在中断中使用单独的API可以使任务代码更简略高效，从而ISR代码也更简略高校，使得中断入口更简单。

任务和中断使用同样API的弊端：

- 该API需要添加条件去判断是ISR还是task调用了自己。额外的逻辑通过函数引入新的路径，使代码更长，更复杂，更难测试。
- ISR和task所需要的参数可能有些不同
- 每一个FreeRTOS port都将要为决定如何执行上下文切换（task or ISR）提供机制
- 软件会更需要额外的使用更加复杂的非标准的中断入口代码，使架构不容易确定该执行上下文切换（task or ISR）

## 使用单独的Interrupt Safe API的弊端

有时我们有必要在task或者ISR中调用不属于FreeRTOS API但使用了FreeRTOS API的函数（如用户自己的API）。常见与使用第三方库。如果确实存在这样的问题，我们可以使用以下方法来解决这样的问题：

1. 推迟中断中的处理到任务中去，这样该API功能只能在任务的上下文切换被调用。
2. 如果您使用的是支持中断嵌套的FreeRTOS port，那么仅使用以“FromISR”结尾的API，即不在中断中使用第三方库。
3. 第三方库通常包括一个RTOS抽象层，可以实现该抽象层来测试调用函数的上下文切换，然后调用合适的API。

## xHigherPriorityTaskWoken

如果上下文切换由中断执行，则中断退出时运行的任务可能与进入中断时正在运行的任务不同—中断将中断一个任务，但返回到不同的任务中去。

一些FreeRTOS API函数可以将任务从阻塞状态移动到就绪状态。这已经在诸如xQueueSendToBack()之类的函数中提到了，如果有一个任务在阻塞状态等待数据在主题队列中有效，他将解除阻塞任务。

如果被FreeRTOS API函数接触阻塞的任务的优先级高于处于运行状态的任务的优先级，那么应该切换到更高优先级的任务中去，切换更高优先级任务的实际发生时间取决于调用API函数的上下文切换中。

- 如果是任务调用的API函数

如果configUSE_PREEMPTION→1，那么在API函数（在API函数退出之前）中会切换到更高优先级的任务中去。如下图所示，写入定时器命令队列导致在写入命令队列的函数退出之前就已经切换到RTOS的daemon task中了。

![Untitled](%E5%85%AD%E3%80%81Interrup%2033926/Untitled.png)

- 如果是中断调用的API函数

在中断内不会自动切换到更高级的任务。与任务相反的是，中断设置了一个变量来通知应用程序编写者应该执行上下文切换。中断safe API函数有一个被称为pxHigherPriorityTaskWoken的指针参数用于此目的。

如果应该执行上下文切换，那么中断safe API函数会将*pxHigherPriorityTaskWoken→pdTURE。为了检测这种情况已经发生，pxHigherPriorityTaskWoken指向的变量必须在第一次使用之前初始化为pdFALSE。

如果程序员选择不从ISR请求上下文切换，则更高优先级的任务将保持就绪态，直到调度器再一次运行—在最坏的情况下，将持续到下一次tick中断。

FreeRTOS API函数只能将*pxHighPriorityTaskWoken→pdTRUE。如果一个ISR调用了多个FreeRTOS API，那么可以在每个API函数调用中将同一个变量作为pxHighPriorityTaskWoken参数传递，并且该变量只需要在第一次使用之前初始化为pdFALSE。

上下文切换不会在中断Safe API中发生的原因（中断Safe API有哪些特性）：

1. 避免了不必要的上下文切换
在任务执行一个进程时，中断可能有必要执行多次。例如，任务是处理一个由中断驱动的UART接收字符串的场景；在每次接收到一个字符时，UART ISR都切换到任务是很浪费的，因为任务只有在接受到完整的字符串后才能执行处理。
2. 控制执行顺序
中断的发生可以是无序且无法预期的。专业的FreeRTOS用户可能希望暂时避免在其应用程序中的某个特定点意外的切换到不同的任务—尽管可以使用FreeRTOS调度器的锁定机制来实现。
3. 可移植性
    
    这是可用于所有FreeRTOS port的最简单的机制。
    
4. 高效
对于较小处理器架构的port只允许ISR的最后请求上下文切换，而笑出改限制将需要格外更复杂的代码。中断Safe API还允许在一个ISR内多次调用FreeRTOS API函数，而不会在同一ISR内产生多个上下文切换请求。
5. 在RTOS tick中断中执行
在tick中断内尝试上下文切换的结果取决于正在使用的FreeRTOS port。它最多也就导致不必要的调用了调度器。

pxHighPriorityTaskWoken参数的使用是可选的，如果不使用，需要将pxHighPriorityTaskWoken**→**NULL。

## portYIELD_FROM_ISR() & portEND_SWITCHING_ISR()

这两个宏被用于在ISR中请求上下文切换。两者以相同方式被使用，且做同样的事情（task中对应的版本为taskYIELD()）。一些port中仅提供二者之一的版本。

```c
portEND_SWITCHING_ISR(xHigherPriorityTaskWoken);

portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
```

从中断Safe API函数中传出的xHigherPriorityTaskWoken参数可以直接用作对portYIELD_FROM_ISR()的调用中的参数。

如果 portYIELD_FROM_ISR()的xHigherPriorityTaskWoken 参数为 pdFALSE（0），则不请求上下文切换，并且宏不起作用。 如果 portYIELD_FROM_ISR()的xHigherPriorityTaskWoken 参数不是 pdFALSE，则请求上下文切换，并且处于运行状态的任务可能会更改。 中断将始终返回到处于运行状态的任务，即使处于运行状态的任务在中断执行时发生了变化。

大多数 FreeRTOS port允许在 ISR 中的任何位置调用 portYIELD_FROM_ISR()。 一些 FreeRTOS port（主要是用于较小架构的端口）仅允许在 ISR 的最后调用 portYIELD_FROM_ISR()。

# 3. 延迟中断的处理

 通常认为最好的做法是使 ISR 尽可能短。有以下原因：

1. 即使task的优先级非常高，它也仅能运行在硬件没有发生中断服务的情况下。
2. ISR 可以扰乱（添加“抖动”）task的开始时间和执行时间。
3. 根据运行 FreeRTOS 的架构，在执行 ISR 时，可能无法接受任何新中断，或者至少是新中断的子集。
4. 程序员需要考虑task和ISR同时访问资源（如变量、外设和内存缓冲区）的后果并加以防范。
5. 一些FreeRTOS port允许中断嵌套，但中断嵌套会增加复杂性并降低可预测性。中断越短，嵌套的可能性就越小。

## 含义

中断服务程序必须记录中断的原因，并清除中断。 中断所需的任何其他处理通常可以在task中执行，从而允许中断服务程序尽可能快地退出。 这称为“延迟中断处理”，因为中断所需的处理从 ISR“延迟”到task。

中断延迟处理还允许程序员相对于应用程序中的其他task优先处理，并可以使用所有 FreeRTOS API 函数。

如果中断处理延迟到的task的优先级高于任何其他task的优先级，则处理将立即执行，就像在 ISR 本身中执行的处理一样。时序如下图：

![Untitled](%E5%85%AD%E3%80%81Interrup%2033926/Untitled%201.png)

如图可知，如果没有使用中断延迟处理的手段，那么t2→t4时刻均处于ISR阶段。

关于何时执行 ISR 中的中断所需的所有处理比较好，以及何时将部分处理推迟到task比较好，没有绝对的规则。 在以下情况下，将中断中的处理推迟到task最有用：

- 中断所需的处理需要很长时间。 例如，如果中断只是存储模数转换的结果，那么几乎可以肯定这最好在 ISR 内部执行。但如果转换结果还必须通过软件过滤器，那么过滤器的处理最好在task中执行。
- 中断中进程需要去处理例如“写入控制台或分配内存”这种在 ISR 内部无法执行的操作
- 中断处理不是确定性的——这意味着事先不知道处理需要多长时间。

# 4. 用于同步的二进制信号量

二进制信号量 API 的中断安全版本可用于在每次发生特定中断时解除阻塞任务，从而有效地使任务与中断同步。 这允许在同步任务中实现大部分中断事件处理，只有非常快速和短的部分直接保留在 ISR 中。 如上一节所述，二进制信号量用于将中断处理推迟到task中处理。

下图更新了[图48](%E5%85%AD%E3%80%81Interrup%2033926.md)的描述:

![Untitled](%E5%85%AD%E3%80%81Interrup%2033926/Untitled%202.png)

用于中断延迟处理的task使用一个阻塞“take”调用一个进入Blocked状态去等待时间发生的信号量。当事件发生，ISR对相同的信号量使用“give”操作以解除task的堵塞，task进而处理这个事件。

“Taking信号量”和“giving信号量”是根据使用场景而具有不同含义的概念。在这种中断同步场景中，二进制信号量在概念上可以被认为是一个长度为 1 的队列。这个队列在任何时候最多可以包含一个项目，所以总是空的或满的（因此是二进制的）。通过调用 xSemaphoreTake()，延时处理中断进程的task尝试读取这个队列。如果队列为空，则task进入阻塞状态。当事件发生时，ISR 使用 xSemaphoreGiveFromISR() 函数将令牌（信号量）放入队列，使队列满。这会导致task退出阻塞状态并移除令牌，使队列再次为空。当task完成其处理后，它再次尝试从队列中读取，并发现队列为空，重新进入阻塞状态以等待下一个事件。该时序如下图。

![Untitled](%E5%85%AD%E3%80%81Interrup%2033926/Untitled%203.png)

上图展示了中断”giving”了一个信号量，task“taken”了这个信号量（即使中断延迟处理的task并未在一开始就“taken”），但task并没有返回这个信号量。所以二进制信号量只是类似queue。

## xSemaphoreCreateBinary()

```c
/*
Return:
	Value:
		NULL - 信号量由于没有足够的栈用来创建FreeRTOS的信号量数据结构而
					 未创建成功。
		non-NULL - 创建成功，返回值应存储为信号量句柄。
*/
SemaphoreHandle_t xSemaphoreCreateBinary(void);
```

## xSemaphoreTake()

```c
/*
获取信号量。除了递归互斥锁以外的FreeRTOS信号量均可以使用xSemaphoreTake()获取。

Parameter：
	xSemaphore：需要被“taken”的信号量句柄（由[xSemaphoreCreateBinary](%E5%85%AD%E3%80%81Interrup%2033926.md)()创建）
	xTicksToWait： 如果信号量不可用，应保持Blocked的时间。
								（。。。同之前以往的同名参量）
Return：
	Value：
		1.pdPASS -- 成功获得信号量
		2.pdFALSE -- 信号量无效
*/
BaseType_t xSemaphoreTake(SemaphoreHandle_t xSemaphore, TickType_t xTicksToWait);
```

## xSemaphoreGiveFromISR()

```c
/*
发出信号量。二进制和计数式信号量都可以使用xSemaphoreGiveFromISR()去‘given’一个信号量。
Parameter:
	xSemaphore: 需要“given”的信号量句柄（由[xSemaphoreCreateBinary](%E5%85%AD%E3%80%81Interrup%2033926.md)()创建）
	pxHigherPriorityTaskWoken: 如[前文](%E5%85%AD%E3%80%81Interrup%2033926.md)介绍。
Return:
	Value:
		1. pdPASS -- 成功发送了这个信号量
		2. pdFAIL -- 失败
*/
BaseType_t xSemaphoreGiveFromISR(SemaphoreHandle_t xSemaphore, BaseType_t *pxHigherPriorityTaskWoken);
```

# 5. 计数信号量

正如二进制信号量可以被认为是长度为 1 的队列一样，计数信号量也可以被认为是长度大于 1 的队列。 task对存储在队列中的数据不感兴趣——只对队列中元素的数目感兴趣。 configUSE_COUNTING_SEMAPHORES→1(FreeRTOSConfig.h 中) 便可以使用计数信号量。

计数信号量典型被用于处理两件事：

1. 计数事件
创建用于计数事件的计数信号量，初始值为0。
在这种事件中，事件处理程序将在每次事件发生时“given”一个信号量——导致信号量的计数值在每次“given”时递增。 task每次处理事件时都会“taken”一个信号量——导致信号量的计数值在每次“taken”时递减。 计数值是已发生的事件数与已处理的事件数之差。 这种机制如图 55 所示。

![Untitled](%E5%85%AD%E3%80%81Interrup%2033926/Untitled%204.png)

1. 资源管理
创建用于管理资源的计数信号量，使其初始计数值等于可用资源的数量。 
在这种情况下，计数值表示可用资源的数量。 为了获得对资源的控制，task必须首先“taken”一个信号量——递减信号量的计数值。 当计数值达到零时，没有空闲资源。 当任务完成资源后，它会“given”信号量——增加信号量的计数值。

## xSemaphoreCreateCounting()

```c
/*
创建计数信号量
Parameter:
	uxMaxCount: 信号量的最大值，实际上是队列长度
			当信号量用于计数或锁存事件时，uxMaxCount 是可以锁存的最大事件数。
			当信号量用于管理对资源集合的访问时，应将 uxMaxCount 设置为可用资源
		的总数。
	uxInitialCount: 信号量创建后的初始计数值。
			当信号量用于计数或锁存事件时，uxInitialCount 应该设置为零——因为可
		能在创建信号量时还没有发生任何事件。
			当信号量用于管理对资源集合的访问时，应将 uxInitialCount 设置为等于
		uxMaxCount——因为据推测，当信号量被创建时，所有资源都可用。
Return:
	Value:
		NULL - 无法创建信号量，因为 FreeRTOS 没有足够的堆内存来分配信号量数
			据结构。
		non-NULL - 创建成功，返回值应该被保存为信号量的句柄。
*/
SemaphoreHandle_t xSemaphoreCreteCounting(UBaseType_t uxMaxCount, UBaseType_t uxInitialCount);
```

# 6. 延迟工作到RTOS Daemon Task中

我们也可以使用 xTimerPendFunctionCallFromISR() API 函数将中断处理推迟到 RTOS daemon task——无需为每个中断创建单独的task。 将中断处理推迟到daemon task称为“集中延迟中断处理”。

xTimerPendFunctionCall() 和 xTimerPendFunctionCallFromISR() API 函数使用相同的计时器命令队列向daemon task发送“exeute function”命令。 发送到daemon task的函数然后在daemon task的上下文中执行。

集中延迟处理的优点：

- 降低资源利用率
无需对每一个延迟中断创建独立的task
- 简化用户模型
延迟中断处理函数使用标准C函数

缺点：

- 灵活性降低
无法独立设置每一个延迟中断处理task的优先级。[Daemon task的优先级如第五章所述。](%E4%BA%94%E3%80%81Software%20162dd.md)
- 确定性降低
xTimerPendFunctionCallFromISR() 将命令发送到计时器命令队列的后面。 在‘ execute function’命令由xTimerPendFunctionCallFromISR()发送到queue之前这条命令已经被daemon task处理。

使用Daemon task作为延迟中断处理task的时序图：

![Untitled](%E5%85%AD%E3%80%81Interrup%2033926/Untitled%205.png)

## xTimerPendFunctionCallFromISR()

```c
/*
xFunctionToPend 参数必须符合该原型
*/
void vPendableFunction( void *pvParameter1, uint32_t ulParameter2 );
/*
Parameter:
	xFunctionToPend:在Daemon task中执行的函数指针（原型如上）
	pvParameter1: 作为参数1传递给xFunctionToPend。类型为void*意味这可以
		传递任何数据类型。
	ulParameter2: 作为参数2传递给xFunctionToPend。
	pxHigherPriorityTaskWoken:如[上文](%E5%85%AD%E3%80%81Interrup%2033926.md)介绍。如果 RTOS daemon task处于阻塞状
		态以等待定时器命令队列上的数据可用，那么写入定时器命令队列将导致
		daemon task离开阻塞状态。 如果Daemon task的优先级高于当前执行task
		（被中断的task）的优先级，则在内部，xTimerPendFunctionCallFromISR() 
		会将*pxHigherPriorityTaskWoken 设置为 pdTRUE。
			如果 xTimerPendFunctionCallFromISR() 将此值设置为 pdTRUE，则必须
		在中断退出之前执行上下文切换。 这将确保中断直接返回到daemon task，因为
		daemon task将是最高优先级的就绪状态任务。
Return:
	Value:
		1.pdPASS -- 'execute function'命令已经被写到了timer command queue.
		2.pdFAIL -- 'execute fucntion'命令无法写到timer command queue.(
				timer command queue已经满了)
*/
BaseType_t xTimerPendFunctionCallFromISR( PendedFunction_t xFunctionToPend,
																					void *pvParameter1,
																					uint32_t ulParameter2,
																					BaseType_t *pxHigherPriorityTaskWoken );
```

# 7. 在中断服务程序中使用队列

二进制和计数信号量用于与事件通讯，而队列不仅用于事件通讯，还用于传输数据。

xQueueSendToFrontFromISR() 是可在中断服务例程中安全使用的 xQueueSendToFront() 版本，xQueueSendToBackFromISR() 是可在中断服务例程中安全使用的 xQueueSendToBack() 版本，xQueueReceiveFromISR() 是 xQueueReceive() 可以在中断服务例程中安全使用。

```c
/*
Parameter:
	xQueue: 选择被发送数据到Queue的句柄。由xQueueCreate()创建。
	pvItemToQueue: 指向被复制到队列的数据的指针
	pxHigherPriorityTaskWoken: 如[上文](%E5%85%AD%E3%80%81Interrup%2033926.md)介绍。
Return:
	1. pdPASS - 成功发送数据到queue
	2. errQUEUE_FULL - 失败，queue已经满了
*/
BaseType_t xQueueSendToFrontFromISR( QueueHandle_t xQueue,
																		 void *pvItemToQueue
																		 BaseType_t *pxHigherPriorityTaskWoken);

/*
与xQueueSendFromISR()等效。
参数同上
*/
BaseType_t xQueueSendToBackFromISR( QueueHandle_t xQueue,
																		void *pvItemToQueue
																		BaseType_t *pxHigherPriorityTaskWoken);
```

## 在ISR中使用Queue的注意事项

队列提供了一种将数据从中断传递到任务的简单方便的方法。但如果数据传递的频率很高，则使用队列这种方式的效率不高。

适用于生产代码的更高效的技术包括：

- 使用直接内存访问 (DMA) 硬件来接收和缓冲字符。 这种方法实际上没有软件开销。 然后可以使用”direct to task notifications”的方法解除task的blocked状态（该task仅在检测到传输中断后才处理缓冲区）
- 将每个接收到的字符都复制到线程安全的 RAM 缓冲区。 同样，“direct to task notifications”可用于在接收到完整消息后，或者检测到传输中断后，解除blocked task（该task用于处理缓冲区数据）。
- 直接在 ISR 中处理接收到的字符，然后使用queue将处理好的数据（而不是原始数据）发送给task。

# 8. 中断嵌套

RTOS 中断嵌套方案将可用的中断优先级分为两组 - 一组将被 RTOS 临界区屏蔽，另一组永远不会被 RTOS 临界区屏蔽并因此始终启用。FreeRTOSConfig.h 中的 configMAX_SYSCALL_INTERRUPT_PRIORITY设置定义了两个组之间的边界。

| Constant | Description |
| --- | --- |
| configMAX_SYSCALL_INTERRUPT_PRIORITY （较旧的port） or configMAX_API_CALL_INTERRUPT_PRIORITY（较新的port） | 设置可以调用中断安全 FreeRTOS API 函数的最高中断优先级。
该优先级高于configKERNEL_INTERRUPT_PRIORITY时可以使用中断嵌套功能。 |
| configKERNEL_INTERRUPT_PRIORITY | 设置tick中断使用的中断优先级，必须始终设置为可能的最低中断优先级。
如果 FreeRTOS port 没有使用 configMAX_SYSCALL_INTERRUPT_PRIORITY 常量，那么任何使用FreeRTOS中断safe  API 函数的中断的优先级也必须为configKERNEL_INTERRUPT_PRIORITY。 |

每个中断源都有一个数字优先级和一个逻辑优先级：

- 数字优先级
分配给中断的序列号
- 逻辑优先级
实际中断的优先级
如果两个不同优先级的中断同时发生，那么处理器将先执行逻辑优先级较高的中断，然后再执行逻辑优先级较低的中断。
如果允许中断嵌套，那么高逻辑优先级的中断可以嵌套低逻辑优先级的中断。

之所以区分两种优先级名称，是由于不同的处理器架构上，数值和逻辑优先级并非总是正相关。如ARM上，数值优先级越低，逻辑优先级越高。

## 中断嵌套例子

**前提**：

- 处理器有七个中断优先级。
- 分配了数字优先级 7 的中断比分配了数字优先级 1 的中断具有更高的逻辑优先级。
- configKERNEL_INTERRUPT_PRIORITY 设置为 1。
- configMAX_SYSCALL_INTERRUPT_PRIORITY 设置为3。

**图示：**

![Untitled](%E5%85%AD%E3%80%81Interrup%2033926/Untitled%206.png)

**解释：**

- 优先级处于1-3的中断会因为cpu处于临界区（使用了有限资源的代码段，即被上锁等）而被阻止执行。处在这部分的中断可以使用interrupt-safe FreeRTOS API函数。
- 优先级大于等于4的中断不会受制与临界区，当中断触发时，对应的中断代码也会立即响应。处在这部分的中断不可以使用任何FreeRTOS API。
- 通常对时间精度要求很高的功能会保持优先级高于configMAX_SYSCALL_INTERRUPT_PRIORITY，以确保中断能及时相应。

## **[在 ARM Cortex-M 内核上运行 RTOS](https://www.freertos.org/RTOS-Cortex-M3-M4.html)**

**注意：** *本节有关中断嵌套的信息适用于使用 Cortex-M3、Cortex-M4、Cortex-M4F、Cortex-M7、Cortex-M33 和 Cortex-M23 时。它不适用于不包含 BASEPRI 寄存器的 Cortex-M0 或 Cortex-M0+* 

内核正如之前所述，Cortex-M的数字优先级和逻辑优先级是负相关的。

当定义了configASSERT()，FreeRTOS Cortex-M port会自动检查ARM Cortex-M 中断控制器 (NVIC)配置的正误。

Cortex-M 中断控制器允许使用最多 8 位来指定每个中断优先级。但大多数Cortex-M内核的微控制器只允许访问其中一个子集。例如，TI Stellaris Cortex-M3 和 ARM Cortex-M4 微控制器实现了三个优先级位。这提供了八个唯一的优先级值。

如果您的项目包含 CMSIS 库头文件，则检查 __NVIC_PRIO_BITS 定义以查看有多少优先级位可用。

### 抢占优先级和次优先级

**Cortex-M 硬件细节**

8位优先级寄存器分为两部分：抢占优先级和子优先级。分配给每个部分的位数是可配置的。抢占优先级定义了一个中断是否可以抢占一个已经在执行的中断。子优先级决定了当两个具有相同抢占优先级的中断同时发生时，哪个中断将首先执行。

**使用 RTOS 时的相关配置**

建议将所有优先级位分配为抢占优先级位，不留任何优先级位作为子优先级位。任何其他配置都会使 configMAX_SYSCALL_INTERRUPT_PRIORITY 设置和分配给各个外设中断的优先级之间的直接关系复杂化。

大多数系统默认为所需配置，但 STM32 驱动程序库明显例外。 ***如果您使用带有 STM32 驱动程序库的 STM32，则通过调用NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 ) 确保所有优先级位都分配为抢占优先级位；在 RTOS 启动之前。***

由于在ARM Cortex-M内核中，数字优先级和逻辑优先级负相关。因此，任何使用 RTOS API 函数的ISR都必须将其优先级设置为数值上等于或大于configMAX_SYSCALL_INTERRUPT_PRIORITY 设置的值。这确保了中断的逻辑优先级等于或小于 configMAX_SYSCALL_INTERRUPT_PRIORITY 设置。

Cortex-M 中断默认具有零优先级值。零是可能的最高优先级值。因此，**切勿将使用Interrupt-safe RTOS API 的中断的优先级保留为默认值。**

### Cortex-M 内部优先级描述

- 硬件相关
ARM Cortex-M 内核将中断优先级值存储在其 8 位中断优先级寄存器的最高有效位中。例如，如果 ARM Cortex-M 微控制器的实现仅实现三个优先级位，则这三个位分别向上移位为位 5、6 和 7。位 0 到 4 可以取任何值，但为了将来的验证和最大兼容性，它们应该设置为 1

![Untitled](%E5%85%AD%E3%80%81Interrup%2033926/Untitled%207.png)

- RTOS相关
如上所述，使用 RTOS API 的ISR的逻辑优先级必须等于或低于 configMAX_SYSCALL_INTERRUPT_PRIORITY 设置的逻辑优先级（较低的逻辑优先级意味着较高的数值）。
**Note: *CMSIS 和不同的微控制器制造商提供了可用于设置中断优先级的库函数。一些库函数期望中断优先级在 8 位字节的最低有效位中指定，而其他库函数期望中断优先级指定已经转移到 8 位字节的最高有效位。检查被调用函数的文档以查看您的情况需要哪个函数，因为出错可能会导致意外行为。***

### 关键内容

RTOS 内核使用 ARM Cortex-M 内核的 BASEPRI 寄存器实现临界区。这允许 RTOS 内核仅屏蔽一部分中断，因此提供了灵活的中断嵌套模型。
BASEPRI 是一个位掩码。将 BASEPRI 设置为一个值会屏蔽所有逻辑优先级等于和低于该值的中断。但是BASEPRI被设为0时，不屏蔽任何中断。因此不能使用 BASEPRI 来屏蔽数字优先级为 0 的中断

从中断中安全地调用的 FreeRTOS API 函数使用 BASEPRI 来实现中断安全临界区。进入临界区时，BASEPRI 设置为 configMAX_SYSCALL_INTERRUPT_PRIORITY，退出临界区时设置为 0。