# 八、Event Group

# 1. 章节介绍和范围

前几章描述了 FreeRTOS 允许将事件传递给任务的功能。 此类功能的示例包括信号量和队列，它们都具有以下属性：

- 它们允许任务在阻塞状态下等待单个事件发生。
- 当事件发生时，它们解除对单个任务的阻塞——解除阻塞的任务是等待事件的最高优先级任务。

事件组是 FreeRTOS 的另一个功能，它允许将事件传达给任务。 与队列和信号量不同：

- 事件组允许任务在阻塞状态下等待多个事件的组合之一发生。
- 事件组在事件发生时解除阻塞等待同一事件或事件组合的所有任务。

事件组的这些独特属性使其可用于同步多个任务、将事件广播到多个任务、允许任务在阻塞状态等待一组事件中的任何一个发生，以及允许任务在阻塞状态下等待多个操作完成。

事件组还提供了减少应用程序使用的 RAM 的方法，因为通常可以用单个事件组替换许多二进制信号量。

事件组功能是可选的。 要包含事件组功能，请将 FreeRTOS 源文件 event_groups.c 构建为项目的一部分。

## 范围：

- 事件组的实际用途。
- 事件组相对于其他 FreeRTOS 功能的优缺点。
- 如何在事件组中设置位。
- 如何在阻塞状态下等待事件组中的位被设置。
- 如何使用事件组来同步一组任务。

# 2. 事件组的特性

## Event Groups, Event Flags and Event Bits

Event flag是一个布尔值（1 或 0），用于指示事件是否发生。 事件“group”是一组event flags。
一个事件flag只能为 1 或 0，允许将一个事件flag的状态存储在一个位中，并将一个事件组中所有事件flag的状态存储在一个变量中； 事件组中每个事件flag的状态由 EventBits_t 类型变量中的单个位表示。 因此，事件flags也称为事件“bits”。 如果 EventBits_t 变量中的某个位设置为 1，则该位表示的事件已经发生。 如果 EventBits_t 变量中的某个位设置为 0，则该位表示的事件尚未发生。
图 71 显示了如何将各个事件标志映射到 EventBits_t 类型的变量中的各个位。

![Untitled](%E5%85%AB%E3%80%81Event%20Gr%20af7f6/Untitled.png)

例如，如果事件组的值为 0x92（二进制 1001 0010），则仅设置事件位 1、4 和 7，因此仅发生了位 1、4 和 7 表示的事件。 图 72 显示了一个 EventBits_t 类型的变量，它设置了事件位 1、4 和 7，并且所有其他事件位都清零，给事件组一个值 0x92。

![Untitled](%E5%85%AB%E3%80%81Event%20Gr%20af7f6/Untitled%201.png)

## 有关 EventBits_t 数据类型的更多信息

configUSE_16_BIT_TICKS (on FreeRTOSConfig.h) 决定事件组中的事件位数：

- 如果 configUSE_16_BIT_TICKS 为 1，则每个事件组包含 8 个可用事件位。
- 如果 configUSE_16_BIT_TICKS 为 0，则每个事件组包含 24 个可用事件位。

configUSE_16_BIT_TICKS 配置用于保存 RTOS ticks counter的类型大小，因此似乎与事件组功能无关。 它对 EventBits_t 类型的影响是 FreeRTOS 内部实现的结果，并且当 FreeRTOS 在可以比 32 位类型更有效地处理 16 位类型的架构上执行时，configUSE_16_BIT_TICKS 应该设置为 1。

## 多任务访问

事件组本身就是对象，任何知道它们存在的任务或 ISR 都可以访问这些对象。 任何任务都可以设置同一事件组中的位，同样任何任务也都可以从同一事件组中读取位值。

## 实际使用事件组的例子

FreeRTOS+TCP TCP/IP 堆栈的实现提供了一个实际示例，说明如何使用事件组来同时简化设计并最大限度地减少资源使用。
TCP 套接字必须响应许多不同的事件。事件的示例包括接受事件、绑定事件、读取事件和关闭事件。套接字在任何给定时间可以预期的事件取决于套接字的状态。例如，如果一个套接字已经创建，但还没有绑定到一个地址，那么它可以期望接收一个绑定事件，但不会期望接收一个读取事件（如果它没有地址，它就无法读取数据） .
FreeRTOS+TCP 套接字的状态保存在名为 ***FreeRTOS_Socket_t*** 的结构中。该结构包含一个事件组，该组具有为套接字必须处理的每个事件定义的事件位。 FreeRTOS+TCP API 调用阻塞以等待一个事件或一组事件，只需阻塞事件组。
事件组还包含一个“中止”位，允许中止 TCP 连接，无论套接字当时正在等待哪个事件。

# 3. 使用事件组的事件管理

## xEventGroupCreate()

```c
/*
Return:
	NULL - 无法创建事件组，因为没有足够的堆内存来分配事件组数据结构。
	non-NULL - 事件组已成功创建。 返回值应存储为已创建事件组的句柄。
*/
EventGroupHandle_t xEventGroupCreate( void );
```

## xEventGroupSetBits()

```c
/*
	xEventGroupSetBits() API 函数设置事件组中的一个或多个位，通常用于通
知任务已发生由正在设置的一个或多个位表示的事件。
Parameter:
	xEventGroup: 被设置位的事件组句柄，由[xEventGroupCreate](%E5%85%AB%E3%80%81Event%20Gr%20af7f6.md)()创建
	uxBitsToSet: 位掩码，指定要在事件组中设置为 1 的一个或者多个事件位。
		事件组的值通过将事件组的现有值与 uxBitsToSet 中传递的值进行按位或运
		算来更新。
Return: 调用 xEventGroupSetBits() 返回时事件组的值。 
		***请注意，返回的值不一定会设置 uxBitsToSet 指定的位，因为这些位可能
	在返回前已被不同的任务再次清除。***
	
*/
EventBits_t xEventGroupSetBits( EventGroupHandle_t xEventGroup,
																const EventBits_t uxBitsToSet );
```

## xEventGroupSetBitsFromISR()

```c
/*
	xEventGroupSetBitsFromISR() 是 xEventGroupSetBits() 的中断安全版本。
	give信号量是一种确定性操作，因为事先知道give信号量最多可以导致一个任务
离开阻塞状态。 当在事件组中设置位时，事先不知道有多少任务将离开阻塞状态，
因此在事件组中设置位不是确定性操作。
	FreeRTOS 设计和实施标准不允许在中断服务例程内或在禁用中断时执行非确定性
操作。 因此，xEventGroupSetBitsFromISR() 不会直接在ISR中设置事件位，而是
将操作推迟到 RTOS Deamon task中执行。

Parameter:
	xEventGroup: 被设置位的事件组句柄，由[xEventGroupCreate](%E5%85%AB%E3%80%81Event%20Gr%20af7f6.md)()创建
	uxBitsToSet: 位掩码，指定要在事件组中设置为 1 的一个或者多个事件位。
		事件组的值通过将事件组的现有值与 uxBitsToSet 中传递的值进行按位或运
		算来更新。
	pxHigherPriorityTaskWoken: xEventGroupSetBitsFromISR() 不会直接在ISR中
		设置事件位，而是通过发送命令到定时器命令队列的方式将操作推迟到RTOS 
		Deamon task。如果Deamon task处于阻塞状态以等待定时器命令队列上的数据可
		用，那么写入定时器命令队列将导致Deamon task离开阻塞状态。如果Deamon
		task的优先级高于当前执行任务（被中断的任务）的优先级，则
		xEventGroupSetBitsFromISR() 会将 *pxHigherPriorityTaskWoken设置为pdTRUE。
			如果 xEventGroupSetBitsFromISR() 将此值设置为 pdTRUE，则应在退出
		中断之前执行上下文切换。这将确保中断直接返回到Deamon task，因为Deamon
		task将是最高优先级的就绪状态任务。
Return:
	pdPASS - 数据成功发送到计时器命令队列
	pdFALSE - 由于队列已满而无法将“set bits”命令写入定时器命令队列
*/
BaseType_t xEventGroupSetBitsFromISR( EventGroupHandle_t xEventGroup,
																			const EventBits_t uxBitsToSet,
																			BaseType_t *pxHigherPriorityTaskWoken );
```

## xEventGroupWaitBits()

调度器用来确定任务是否进入阻塞状态以及任务何时离开阻塞状态的条件称为“ 解锁条件”。 解锁条件由 uxBitsToWaitFor 和 xWaitForAllBits 参数值的组合指定：

- uxBitsToWaitFor 指定要测试事件组中的哪些事件位
- xWaitForAllBits 指定是使用按位 OR 测试，还是使用按位 AND 测试

如果在调用 xEventGroupWaitBits() 时满足解除阻塞条件，则任务不会进入阻塞状态。

任务使用 uxBitsToWaitFor 参数指定要测试的位，并且任务可能需要在满足其解锁条件后将这些位清零。 可以使用 xEventGroupClearBits() API 函数清除事件位，但在以下情况下，使用该函数手动清除事件位将导致应用程序代码出现竞争条件：

- 有多个任务使用同一个事件组。
- 由不同的任务或中断服务程序在事件组中设置位。

提供 xClearOnExit 参数是为了避免这些潜在的竞争条件。 如果 xClearOnExit 设置为 pdTRUE，那么事件位的测试和清除对调用任务来说是一个原子操作（不会被其他任务或中断中断）。

```c
/*
	xEventGroupWaitBits() API 函数允许任务读取事件组的值，如果尚未设置事件
位，则可选择在阻塞状态等待事件组中的一个或多个事件位被设置。

Parameter:
	xEventGroup: 被等待读取事件组句柄，由[xEventGroupCreate](%E5%85%AB%E3%80%81Event%20Gr%20af7f6.md)()创建。
	uxBitsToWaitFor: 位掩码，指定要在事件组中测试的一个或多个事件位。
	xClearOnExit: 如果调用任务的解锁条件已经满足，并且 xClearOnExit 设置为
		pdTRUE。那么在调用任务退出 xEventGroupWaitBits() API 函数之前，由
		uxBitsToWaitFor 指定的事件位将在事件组中清零为 0。
			如果 xClearOnExit 设置为 pdFALSE，则事件组中事件位的状态不会被
		xEventGroupWaitBits() API 函数修改。
	xWaitForAllBits: xWaitForAllBits指定当设置了uxBitsToWaitFor参数指定的
		一个或多个事件位时，或者仅当设置了uxBitsToWaitFor参数指定的所有事件位
		时，调用任务是否应从阻塞状态中移除。
			pdFALSE - 意味着任意事件为1时就满足解除阻塞条件。
			pdTRUE - 意味着要同时满足所有事件条件才能解除阻塞条件。
	xTicksToWait: 保持阻塞状态下的最长时间。
			设置为0时立马返回，除非调用时，恰好满足解锁条件。
Return: 如果因为调用任务的解锁条件被满足而返回，那么返回值是事件组在调用
	任务的解锁条件满足时的值（如果 xClearOnExit 为 pdTRUE，则在任何位被自动
	清除之前的值）。
		如果xEventGroupWaitBits()是因为xTicksToWait参数指定的出块时间到期而
	返回的，那么返回值就是超时时事件组的值。 在这种情况下，返回值将不满足解
	锁条件。
	
*/
EventBits_t xEventGroupWaitBits( const EventGroupHandle_t xEventGroup,
																 const EventBits_t uxBitsToWaitFor,
																 const BaseType_t xClearOnExit,
																 const BaseType_t xWaitForAllBits,
																 TickType_t xTicksToWait );
```

# 4. 使用事件组同步任务

事件组可用于创建同步的关键点：

- 每个必须参与同步的任务都在事件组中分配了一个唯一的事件位。
- 每个任务在到达同步点时都会设置自己的事件位。
- 设置了自己的事件位后，事件组上的每个任务都会阻塞以等待代表所有其他同步任务的事件位也被设置。

但是在这种情况下不能使用 xEventGroupSetBits() 和 xEventGroupWaitBits() API 函数。 如果它们被使用，那么位的设置（表明一个任务已经到达它的同步点）和位的测试（以确定其他同步任务是否已经到达它们的同步点）将作为两个独立的操作来执行。 要了解为什么会出现问题，请考虑任务 A、任务 B 和任务 C 尝试使用事件组进行同步的场景：

1. 任务A和任务B已经到达同步点，所以他们的事件位在事件组中被置位，它们处于阻塞状态等待任务C的事件位也被置位。
2. 任务 C 到达同步点，并使用 xEventGroupSetBits() 设置其在事件组中的位。 一旦任务 C 的位置位，任务 A 和任务 B 就会离开阻塞状态，并清除这三个事件位。
3. 然后任务 C 调用 xEventGroupWaitBits() 等待所有三个事件位都被置位，但此时所有三个事件位都已被清除，任务 A 和任务 B 已离开各自的同步点，因此同步 失败了。

要成功地使用事件组创建同步点，事件位的设置和事件位的后续测试必须作为单个不间断操作执行。 为此提供了 xEventGroupSync() API 函数。

## xEventGroupSync()

```c
/*
	提供 xEventGroupSync() 以允许两个或多个任务使用事件组相互同步。 该功
能允许任务设置一个事件组中的一个或多个事件位，然后作为一个单一的不间断操
作等待同一事件组中的事件位组合被设置。
	xEventGroupSync() uxBitsToWaitFor 参数指定调用任务的解锁条件。 
如果 xEventGroupSync() 由于满足解除阻塞条件而返回，则 uxBitsToWaitFor 
指定的事件位将在 xEventGroupSync() 返回之前清零。

Parameter:
	xEventGroup: 需要同步的事件组的句柄，由[xEventGroupCreate](%E5%85%AB%E3%80%81Event%20Gr%20af7f6.md)()创建
	uxBitsToSet: 位掩码，指定要在事件组中**设置**为 1 的事件位或事件位。 事件
			组的值通过将事件组的现有值与 uxBitsToSet 中传递的值进行按位或运算
			来更新。
	uxBitsToWaitFor: 位掩码，指定要在事件组中**测试**的一个或多个事件位。
	xTicksToWait: 保持阻塞状态的最长时间。
Return:
	如果 xEventGroupSync() 因为调用任务的解锁条件被满足而返回，那么返回值
是事件组在调用任务的解锁条件满足时的值。 在这种情况下，返回值也将满足调
用任务的解锁条件。
	如果 xEventGroupSync() 是因为 xTicksToWait 参数指定的阻塞时间到期而返
回，则返回值是阻塞时间到期时事件组的值。 在这种情况下，返回值将不满足调用
任务的解锁条件。
*/
EventBits_t xEventGroupSync( EventGroupHandle_t xEventGroup,
														 const EventBits_t uxBitsToSet,
														 const EventBits_t uxBitsToWaitFor,
														 TickType_t xTicksToWait );
```