# 九、Task Notifications

# 1. 章节介绍和范围

## 通过中介对象进行通讯

到目前为止所描述的方法都需要创建一个通信对象。 通信对象的示例包括队列、事件组和各种不同类型的信号量。

当使用通信对象时，事件和数据不会直接发送到接收任务或ISR中，而是发送到通信对象。 同样，任务和 ISR 从通信对象接收事件和数据，而不是直接从发送事件或数据的任务或 ISR。 这在图 76 中进行了描述。

![Untitled](%E4%B9%9D%E3%80%81Task%20Not%20eddc1/Untitled.png)

## 任务通知——直接到任务通信

“任务通知”允许任务与其他任务交互，并与 ISR 同步，而不需要单独的通信对象。 通过使用任务通知，任务或 ISR 可以直接向接收任务发送事件。 这在图 77 中进行了描述。

![Untitled](%E4%B9%9D%E3%80%81Task%20Not%20eddc1/Untitled%201.png)

配置configUSE_TASK_NOTIFICATIONS→1(on FreeRTOSConfig.h)开启任务通知功能。

当 configUSE_TASK_NOTIFICATIONS 设置为 1 时，每个任务都有一个“通知状态”，可以是“待处理”或“未待处理”，以及一个“通知值”，它是一个 32 位无符号整数。 当任务收到通知时，其通知状态设置为待处理。 当任务读取其通知值时，其通知状态设置为未挂起。

一个任务可以在 Blocked 状态下和一定时间内等待它的通知状态变为待处理。

## 范围

- 任务的通知状态和通知值。
- 如何以及何时可以使用任务通知来代替通信对象，例如信号量。
- 使用任务通知代替通信对象的优点。

# 2. 任务通知的优点和限制

## 优点

- 性能优势
使用任务通知向任务发送事件或数据比使用队列、信号量或事件组执行等效操作要快得多。
- RAM占用空间优势
同样，使用任务通知向任务发送事件或数据所需的 RAM 比使用队列、信号量或事件组执行等效操作要少得多。 这是因为必须先创建每个通信对象（队列、信号量或事件组）才能使用它，而启用任务通知功能的固定开销仅为每个任务的 8 字节 RAM。

## 限制

无法在所有场景下使用，以下为无法使用的场景：

- 发送一个事件和数据到中断中
通信对象可用于将事件和数据从 ISR 发送到任务，以及从任务发送到 ISR。
任务通知可用于将事件和数据从 ISR 发送到任务，但不能用于将事件或数据从任务发送到 ISR。
- 使能更多的接收任务
通讯（队列，信号量或者事件组等）句柄只要能被获取，任何任务或ISR都可以访问和发送事件或数据。
而任务通知是直接发送给接收任务的，所以只能由接收通知的任务处理。 然而，这在实际情况中很少受到限制，因为虽然多个任务和 ISR 发送到同一个通信对象是很常见的，但很少有多个任务和 ISR 从同一个通信对象接收。
- 缓冲多个数据项
队列是一种通信对象，一次可以保存多个数据项。 已发送到队列但尚未从队列接收的数据在队列对象中缓冲。
任务通知通过更新接收任务的通知值向任务发送数据。 一个任务的通知值一次只能保存一个值。
- 广播
事件组是一种通信对象，可用于一次向多个任务发送事件。
任务通知是直接发送给接收任务的，所以只能由接收任务处理。
- 在阻塞状态等待发送完成
如果通信对象暂时处于无法向其写入数据或事件的状态（例如，当队列已满时，无法向队列发送更多数据），则尝试写入对象的任务可以 可选择进入 Blocked 状态以等待其写操作完成。
如果任务尝试向已经有通知待处理的任务发送任务通知，则发送任务不可能在阻塞状态下等待接收任务重置其通知状态。

# 3. 使用任务通知

任务通知是一个非常强大的功能，通常可以用来代替二进制信号量、计数信号量、事件组，有时甚至是队列。 这种广泛的使用场景可以通过使用xTaskNotify() API 函数发送任务通知和xTaskNotifyWait() API 函数接收任务通知来实现。

但是，在大多数情况下，不需要 xTaskNotify() 和 xTaskNotifyWait() API 函数提供的全部灵活性，更简单的函数就足够了。 因此，提供 xTaskNotifyGive() API 函数作为 xTaskNotify() 的更简单但不太灵活的替代方案，而提供 ulTaskNotifyTake() API 函数作为 xTaskNotifyWait() 的更简单但不太灵活的替代方案。

## xTaskNotifyGive()

```c
/*
	xTaskNotifyGive() 直接向任务发送通知，并递增（加一）接收任务的通知值。
如果接收任务并不处于待处理状态，则调用 xTaskNotifyGive() 会将接收任务的
通知状态设置为待处理。
	xTaskNotifyGive() 实际上是作为宏实现的，而不是函数。作为比信号量更轻
量，很快速的选择。

Parameter:
	xTaskToNotify: 被发送通知的任务句柄。由[xTaskCreate()](%E4%B8%89%E3%80%81Task%20Man%20897b3.md)创建。
Return:
	xTaskNotifyGive() 是一个调用 xTaskNotify() 的宏。 宏传递给
xTaskNotify() 的参数设置为 pdPASS 是唯一可能的返回值。
*/
BaseType_t xTaskNotifyGive( TaskHandle_t xTaskToNotify );
```

## vTaskNotifyGiveFromISR()

```c
/*
Parameter:
	xTaskToNotify: 同上
	pxHigherPriorityTaskWoken: 如果被发送通知的任务在阻塞状态等待接收通
		知，那么发送通知将导致任务离开阻塞状态。如果调用 
		vTaskNotifyGiveFromISR() 导致任务离开阻塞状态，并且未阻塞任务的优先
		级高于当前执行任务（被中断的任务）的优先级，那么在内部，
			如果 vTaskNotifyGiveFromISR() 将此值设置为 pdTRUE，则应在退出中断
		之前执行上下文切换。这将确保中断直接返回到最高优先级的就绪状态任务。
		vTaskNotifyGiveFromISR()将设置*pxHigherPriorityTaskWoken为pdTRUE。
			与所有中断安全 API 函数一样，pxHigherPriorityTaskWoken 参数必须在
		使用前设置为 pdFALSE。
*/
void vTaskNotifyGiveFromISR( TaskHandle_t xTaskToNotify,
														 BaseType_t *pxHigherPriorityTaskWoken );
```

## ulTaskNotifyTake()

```c
/*
	ulTaskNotifyTake() 允许任务在阻塞状态等待其通知值大于零，并在任务返回
之前递减（减一）或清除任务的通知值。作为比信号量更轻量，很快速的选择。

Parameter:
	xClearCountOnExit:
		pdTURE - 通知值将在调用 ulTaskNotifyTake() 返回之前清零。
		pdFALSE - 当通知值大于零，通知值将在返回之前递减。
	xTicksToWait:
		保持在 Blocked 状态的最长时间。
Return:
	在被清零或者递减之前的通知值。
	如果xTicksToWait > 0, 而返回值为0，则表明任务进入过Blocked状态。
*/
uint32_t ulTaskNotifyTake( BaseType_t xClearCountOnExit, 
													 TickType_t xTicksToWait );
```

## xTaskNotify() & xTaskNotifyFromISR()

xTaskNotify() 是 xTaskNotifyGive() 功能更强的版本，可用于通过以下任何方式更新接收任务的通知值：

1. 增加（加一）接收任务的通知值，在这种情况下 xTaskNotify() 等价于 xTaskNotifyGive()。
2. 置位接受任务通知值的一位或多位。 这允许将任务的通知值用作事件组的更轻量级和更快的替代方案。
3. 将接收任务的通知值写入一个全新的数字，但仅当接收任务自上次更新后已读取其通知值。 这允许任务的通知值提供与长度为 1 的队列所提供的功能相似的功能。
4. 将接收任务的通知值写入一个全新的数字，即使接收任务自上次更新后尚未读取其通知值。 这允许任务的通知值提供与 xQueueOverwrite() API 函数提供的功能类似的功能。 由此产生的行为有时被称为“mailbox”。

xTaskNotifyFromISR() 是可在中断服务例程中使用的 xTaskNotify() 版本，因此具有额外的 pxHigherPriorityTaskWoken 参数。
如果接收任务尚未处于待处理状态，调用 xTaskNotify() 将始终将接收任务的通知状态设置为待处理。

| eNotifyAction (enum) | description |
| --- | --- |
| eNoAction | 接收任务的通知状态设置为待处理，而通知值不被更新。xTaskNotify()无需使用ulValue参数。
将任务通知作为二进制信号量更快、更轻便的代替方案。 |
| eSetBits | 接收任务的通知值与 xTaskNotify() ulValue 参数中传递的值进行按位或运算。 例如，如果 ulValue 设置为 0x01，那么将在接收任务的通知值0位置位；如果 ulValue 是 0x06（二进制 0110），那么将在接收任务的通知值中的第1和第2位置位。
将任务通知作为事件组更快、更轻便的代替方案。 |
| eLncrement | 接收任务的通知值递增。 未使用 xTaskNotify() ulValue 参数。
eIncrement 操作允许将任务通知用作二进制或计数信号量的更快、更轻量级的替代方案，并且等效于更简单的 xTaskNotifyGive() API 函数。 |
| eSetValueWithoutOverwrite | 如果接收任务在调用 xTaskNotify() 之前有一个待处理的通知，则不采取任何操作并且 xTaskNotify() 将返回 pdFAIL。
如果在调用 xTaskNotify() 之前接收任务没有待处理的通知，则接收任务的通知值设置为 xTaskNotify() ulValue 参数中传递的值。 |
| eSetValueWithOverwrite | 接收任务的通知值设置为在 xTaskNotify() ulValue 参数中传递的值，无论接收任务在调用 xTaskNotify() 之前是否有待处理的通知。 |

```c
/*
Parameter:
	xTaskToNotify: 被发送通知的任务句柄。由[xTaskCreate()](%E4%B8%89%E3%80%81Task%20Man%20897b3.md)创建。
	ulValue: 取决于[eNotifyAction](%E4%B9%9D%E3%80%81Task%20Not%20eddc1.md)的值
	[eNotifyAction](%E4%B9%9D%E3%80%81Task%20Not%20eddc1.md): 枚举类型，指定如何更新接收任务的通知值。
Return:
	除了在eAction->eSetValueWithoutOverwrite下可能返回pdFALSE之外，均返回
	pdTRUE.
*/
BaseType_t xTaskNotify( TaskHandle_t xTaskToNotify,
												uint32_t ulValue,
												eNotifyAction eAction );
BaseType_t xTaskNotifyFromISR( TaskHandle_t xTaskToNotify,
															 uint32_t ulValue,
															 eNotifyAction eAction,
															 BaseType_t *pxHigherPriorityTaskWoken );
```

## xTaskNotifyWait()

xTaskNotifyWait() 是 ulTaskNotifyTake() 的更强大版本。它可以使调用任务的信号量状态处于待处理，直至超时发生。 xTaskNotifyWait() 提供了在调用任务的通知值中在进入函数和退出函数时清除位的选项。

```c
/*
Parameter:
	ulBitsToClearOnEntry: 如果在调用该函数之前，任务没有待处理的通知，那么
		任务的通知值将被清除ulBitsToClearOnEntry中设置的位。
		例如，如果ulBitsToClearOnEntry为 0x01，则任务通知值的0位将被清除。
	ulBitsToClearOnExit: 任务因为有待处理任务的通知而返回之前，清除任务
		通知值对应ulBitsToClearOnExit的位。
	pulNotificationValue: 用于传递任务的通知值（在任务退出前，根据
	ulBitsToClearOnExit参数清除对应位之前的通知值）。该参数可选，无需要
	可以设置为NULL。
	xTicksToWait: 任务进入Blocked状态的最长时间。
Return:
	pdTRUE - 接受到通知而返回。
	pdFALSE - Blocked超时而返回。
*/
BaseType_t xTaskNotifyWait( uint32_t ulBitsToClearOnEntry,
														uint32_t ulBitsToClearOnExit,
														uint32_t *pulNotificationValue,
														TickType_t xTicksToWait );
```