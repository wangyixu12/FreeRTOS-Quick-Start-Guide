# 四、Queue Management

## 1. 章节介绍和范围

队列提供了一个task与Interrupt之间，以及自身的通讯机制。

### 范围

- 如何创建一个队列
- 队列如何管理所包含的数据
- 如何发送数据到队列
- 如何从队列中接收数据
- 阻塞队列的含义
- 如何阻塞多个队列
- 如何覆盖队列中的数据
- 如何清除队列
- 任务优先级如何影响写入和读取队列

## 2. 队列的特性

### 数据存储

一个队列可以容纳有限数量的固定大小的数据项。 队列可以容纳的最大项目数称为“长度”。 每个数据项的长度和大小都是在创建队列时设置的。
队列通常用作先进先出 (FIFO) 缓冲区，其中数据被写入队列的末端（尾部）并从队列的前端（头部）移除。下图演示了被当作FIFO的队列时如何被读取和写入的。当然也可以将数据写入队列的前端，并覆盖原本的数据。

![Untitled](%E5%9B%9B%E3%80%81Queue%20Ma%20e73e4/Untitled.png)

队列有两种方式可以被实现：

1. 复制队列
意味着发送到队列的数据被逐字节的复制到队列中。（类似数值类型的实参）
2. 引用队列
意味着队列只保存指向发送到队列的数据的指针，而不是数据本身。（类似指针类型的实参）

FreeRTOS 通过复制方法使用队列。复制队列被认为比引用队列更强大且更易于使用，因为：

- 堆栈变量可以直接发送到队列，即使该变量在声明它的函数退出后将不存在。
- 可以将数据发送到队列，而无需先分配缓冲区来保存数据，然后将数据复制到分配的缓冲区中
- 发送任务可以立即重新使用已发送到队列的变量或缓冲区。
- 发送任务和接收任务完全解耦——应用程序设计者不需要关心哪个任务“拥有”数据，或者哪个任务负责释放数据。
- 以复制的方式排队不会阻止队列也被用于按引用排队。例如，当排队数据的大小使得将数据复制到队列中变得不切实际时，可以将指向数据的指针复制到队列中。
- RTOS 完全负责分配用于存储数据的内存。
- 在内存保护系统中，任务可以访问的 RAM 将受到限制。在这种情况下，只有在发送和接收任务都可以访问存储数据的 RAM 时，才能使用按引用排队。复制排队不会施加这种限制；内核始终以完全权限运行，允许使用队列跨内存保护边界传递数据。

### 多任务访问

队列本身就是对象，任何知道它们存在的任务或 ISR 都可以访问这些对象。 任意数量的任务可以写入同一个队列，任意数量的任务可以从同一个队列中读取。 在实践中，一个队列有多个写入者是很常见的，但一个队列有多个读取者的情况就少得多了。

### 阻塞队列读取

当任务尝试从队列中读取时，它可以选择指定一个“阻塞”时间。 如果队列已经为空，这是任务将保持在阻塞状态以等待队列中可用数据的时间。 处于阻塞状态的任务正在等待队列中的数据可用，当另一个任务或中断将数据放入队列时，它会自动移动到就绪状态。 如果指定的阻塞时间在数据可用之前到期，任务也将自动从阻塞状态移动到就绪状态。
队列可以有多个读取器，因此单个队列可能会阻止多个任务在其上等待数据。 在这种情况下，当数据可用时，只有一个任务会被解除阻塞。 未阻塞的任务将始终是等待数据的最高优先级任务。 如果阻塞的任务具有相同的优先级，则等待数据时间最长的任务将被解除阻塞。

### 阻塞队列写入

就像从队列中读取一样，任务在写入队列时可以选择指定阻塞时间。 在这种情况下，阻塞时间是任务应该保持在阻塞状态以等待队列上可用空间的最长时间，如果队列已经满的话。
队列可以有多个写入者，因此一个完整的队列可能会阻止多个任务在其上等待完成发送操作。 在这种情况下，当队列上的空间可用时，只有一个任务会被解除阻塞。 未阻塞的任务将始终是等待空间的最高优先级任务。 如果阻塞的任务具有相同的优先级，则等待空间时间最长的任务将被解除阻塞。

### 阻塞多个队列

队列可以分组到集合中，允许任务进入阻塞状态以等待数据在集合中的任何队列上可用。 

# 3. 队列的使用

### xQueueCreate()

```c
/*
使用队列前必须先创建队列。FreeRTOS 在创建队列时从 FreeRTOS 堆分配 RAM。 
RAM 用于保存队列数据结构和队列中包含的项目。 如果没有足够的堆 RAM 可用于创建
队列，xQueueCreate() 将返回 NULL。创建队列后，可以使用 xQueueReset() API 函
数将队列返回到其原始空状态。

Parameter:
	uxQueueLength: 队列最大可容纳的项目数
	uxItemSize: 可以存储在队列中的每个数据项的大小（以字节为单位）。

Return:
	NULL - 没有足够的堆栈来创建队列所需的数据结构和存储空间
	non-NULL - 创建成功，返回值为队列的句柄
*/
QueueHandle_t xQueueCreate(UBaseType_t uxQueueLength, UBaseType_t uxItemSize);
```

### xQueueSendToBack() & xQueueSendToFront()

```c
/*
xQueueSendToFront()被用来将数据发送到队列前端，而xQueueSendToBack()则
相反。xQueueSendToBack() & xQueueSend()相等。

Parameter:
	xQueue: 被发送数据的句柄，由xQueueCreate()创建。
	pvItemToQueue: 指向要复制到队列中的数据的指针。
								 队列可以容纳的每个项目的大小是在创建队列时设置的，因此会从 pvItemToQueue 将
							这么多字节复制到队列存储区中。
	xTicksToWait: 如果队列已满，任务应保持在 Blocked 状态以等待队列上可用空间的最长时间。
								如果 xTicksToWait 为零且队列已满，则 xQueueSendToFront() 和 
							xQueueSendToBack() 都将立即返回。
								块时间以滴答周期指定，因此它所代表的绝对时间取决于滴答频率。 宏 
							pdMS_TO_TICKS() 可用于将以毫秒为单位的时间转换为以滴答为单位的时间。
								将 xTicksToWait 设置为 portMAX_DELAY 将导致任务无限期等待（不会超时），前提
							是在 FreeRTOSConfig.h 中将 INCLUDE_vTaskSuspend 设置为 1。
Return:
	pdPASS - 数据发送成功
	errQUEUE_FULL - 在等待时间内，队列始终是满的。
*/
BaseType_t xQueueSendToFront(QueueHandle_t xQueue,
														 const void * pvItemToQueue,
														 TickType_t xTicksToWait);

BaseType_t xQueueSendToBack(QueueHandle_t xQueue,
														const void * pvItemToQueue,
														TickType_t xTicksToWait);
```

### xQueueReceive()

```c
/*
xQueueReceive() 用于从队列中接收（读取）项目。 收到的项目将从队列中删除。

Parameter:
	xQueue: 读取数据的句柄，由xQueueCreate()创建。
	pvBuffer: 指向将接收到的数据复制到其中的内存的指针。
						队列保存的每个数据项的大小是在创建队列时设置的。 pvBuffer 指向的内存必须至少大到足
					以容纳那么多字节。
	xTicksToWait: 如果队列已经为空，任务应保持在 Blocked 状态以等待队列上可用的数据的最长时间。
								如果 xTicksToWait 为零，则如果队列已经为空，则 xQueueReceive() 将立即返回。
Return:
	pdPASS - 数据接收成功
	errQUEUE_EMPTY - 在等待时间内，被接收的队列始终为空。
*/
BaseType_t xQueueReceive( QueueHandle_t xQueue,
													void * const pvBuffer,
													TickType_t xTicksToWait );
```

### uxQueueMessagesWaiting()

```c
/*
uxQueueMessagesWaiting() 用于查询当前队列中的项目数。

Parameter:
	xQueue: 被查队列的句柄，由xQueueCreate()创建。
Return: 正在查询的队列当前持有的项目数。 如果返回零，则队列为空。
	
*/
UBaseType_t uxQueueMessagesWaiting( QueueHandle_t xQueue );
```

# 4. 从多个源中接收数据

在 FreeRTOS 设计中，一项任务从多个来源接收数据是很常见的。 接收任务需要知道数据来自哪里来确定应该如何处理数据。 一个简单的设计解决方案是使用单个队列来传输结构，其中包含数据值和结构字段中包含的数据源。 如下图所示：

![Untitled](%E5%9B%9B%E3%80%81Queue%20Ma%20e73e4/Untitled%201.png)

解析：

- 创建一个包含 Data_t 类型结构的队列。 结构成员允许在一条消息中将数据值和指示数据含义的枚举类型发送到队列。
- Controller task 被用于根据队列中的数据不同来相应系统的不同状态
- 数据eDataID为用来描述数据的源是什么（如CAN or HMI,etc.）。lDataValue用来保存传递的数据源的值。

# 5. 处理大型或者可变大小的数据

## 队列指针

如果存储在队列中的数据很大，那么最好使用队列来传递指向数据的指针，而不是逐字节地将数据本身复制到队列中和从队列中取出。 传输指针在处理时间和创建队列所需的 RAM 量方面都更有效。 但是，在对指针进行排队时，必须格外小心以确保：

1. 指向RAM的数据所有者必须清晰明确
通过指针在任务之间共享内存时，必须确保两个任务不会同时修改内存内容，或采取任何其他可能导致内存内容无效或不一致的操作。理想情况下，应该只允许发送任务访问内存，直到指向内存的指针被加载进队列中。然后在队列接收到数据后，只允许接收任务访问内存。
2. 被指向的RAM仍然有效
如果指向的内存是动态分配的，或者是从预先分配的缓冲区池中获得的，那么只有一个任务可以负责释放内存。 在内存被释放后，任何任务都不应尝试访问内存。
永远不应使用指针来访问已在任务堆栈上分配的数据。 堆栈帧更改后数据将无效。

## 使用队列发送不同类型和长度的数据

前几节展示了两种强大的设计模式； 将结构发送到队列，并将指针发送到队列。 结合这些技术，任务可以使用单个队列从任何数据源接收任何数据类型。 FreeRTOS+TCP TCP/IP 堆栈的实现提供了如何实现这一点的实际示例。
TCP/IP 堆栈在自己的任务中运行，必须处理来自许多不同来源的事件。 不同的事件类型与不同类型和长度的数据相关联。 在 TCP/IP 任务之外发生的所有事件都由 IPStackEvent_t 类型的结构描述，并发送到队列中的 TCP/IP 任务。 IPStackEvent_t 结构如清单 55 所示。IPStackEvent_t 结构的 pvData 成员是一个指针，可用于直接保存值或指向缓冲区。

![Untitled](%E5%9B%9B%E3%80%81Queue%20Ma%20e73e4/Untitled%202.png)

# 6. 从多个队列接收数据

## 队列集

通常应用程序设计需要单个任务来接收不同大小的数据、不同含义的数据以及来自不同来源的数据。 上一节演示了如何使用接收结构的单个队列以简洁有效的方式实现这一点。 但是，有时应用程序的设计者会处理限制其设计选择的约束，因此需要为某些数据源使用单独的队列。 例如，集成到设计中的第三方代码可能假定存在专用队列。 在这种情况下，可以使用“队列集”。

队列集允许任务从多个队列接收数据，而无需task依次轮询每个队列以确定哪个队列（如果有）包含数据。

与使用接收结构的单个队列实现相同功能的设计相比，使用队列集从多个源接收数据的设计不太整洁，效率也较低。 出于这个原因，建议仅在设计约束使其绝对必要时才使用队列集。

以下描述使用队列集的步骤：

1. 创建队列集。
2. 将队列添加到集合中。
信号量也可以添加到队列集中。
3. 读取队列集合来确定哪一个队列中包含数据
当作为集合成员的队列接收到数据时，接收队列的句柄将被发送到队列集合，并在任务调用从队列集合中读取的函数时返回。因此，如果从队列集中返回队列句柄，则该句柄引用的队列已知包含数据，那么任务可以直接从队列中读取数据。

***注意：如果队列是队列集合的成员，则不要直接从队列中读取数据，必须首先从队列集合中读取队列的句柄。***

将configUSE_QUEUE_SETS→1 (on FreeRTOSConfig.h) 来启用队列集的功能。

## xQueueCreateSet()

```c
/*
Parameter:
	uxEventQueueLength: 
		当作为队列集合成员的队列接收数据时，接收队列的句柄将发送到队列集合。uxEventQueueLength 定
	义了正在创建的队列集在任何时候可以容纳的最大队列句柄数。
		只有当集合中的队列接收到数据时，队列句柄才会发送到队列集合。如果队列已满，则无法接收数据，因
	此如果集合中的所有队列都已满，则无法将队列句柄发送到队列集合。因此，队列集合一次必须容纳的最大
	项目数是集合中每个队列的长度之和。
		例如，如果集合中有 3 个空队列，每个队列的长度为 5，则集合中的队列总共可以在所有队列之前接收
	15 个项目（三个队列乘以每个队列 5 个项目）。设置已满。在该示例中，必须将 uxEventQueueLength
	设置为 15 以保证队列集可以接收发送给它的每个项目。
		信号量也可以添加到队列集中。为了计算必要的 uxEventQueueLength，二进制信号量的长度为 1，计
	数信号量的长度由信号量的最大计数值给出。
		作为另一个示例，如果队列集包含长度为 3 的队列和二进制信号量（长度为 1），则
	uxEventQueueLength 必须设置为 4。
Return:
	NULL - 无法创建队列集，因为 FreeRTOS 没有足够的堆内存来分配队列集数据结构和存储区域。
	non-NULL - 已成功创建队列集。 返回的值应存储为已创建队列集的句柄。
*/
QueueSetHandle_t xQueueCreateSet( const UBaseType_t uxEventQueueLength );
```

## xQueueAddToSet()

```c
/*
可以添加队列或者信号量

Parameter:
	xQueueOrSemaphore:正在添加到队列集中的[队列](%E5%9B%9B%E3%80%81Queue%20Ma%20e73e4.md)或[信号量](%E5%85%AD%E3%80%81Interrup%2033926.md)的句柄。
										队列句柄和信号量句柄都可以转换为 QueueSetMemberHandle_t 类型。
	xQueueSet: 队列集的句柄。由[xQueueCreateSet()](%E5%9B%9B%E3%80%81Queue%20Ma%20e73e4.md)创建。
Return:
	pdPASS - success
	pdFAIL - 无法将队列或信号量添加到队列集中, 队列和二进制信号量只有在它们为空时才能添加到集合
					中。 计数信号量只能在计数为零时添加到集合中。 队列和信号量一次只能是一个集合的成员。
*/
BaseType_t xQueueAddToSet( QueueSetMemberHandle_t xQueueOrSemaphore,
													 QueueSetHandle_t xQueueSet );
```

## xQueueSelectFromSet()

```c
/*
	从队列集中读取队列句柄。
	当作为集合成员的队列或信号量接收数据时，接收队列或信号量的句柄被发送到队列集合，并在任务调用
xQueueSelectFromSet() 时返回。 如果从对 xQueueSelectFromSet() 的调用中返回句柄，则该句柄引用
的队列或信号量已知包含数据，调用任务必须直接从队列或信号量中读取。
	***注意：不要从作为集合成员的队列或信号量中读取数据，除非队列或信号量的句柄首先从对
xQueueSelectFromSet() 的调用中返回。 每次调用 xQueueSelectFromSet() 返回队列句柄或信号量句柄
时仅读取队列或者信号量其中一项。***

Parameter:
	xQueueSet: 队列或者信号量被接收的集合句柄。
	xTicksToWait: 
			如果集合中的所有队列和信号量都为空，则调用任务应保持在阻塞状态以等待从队列集中接收队列或信
		号量句柄的最长时间。
			如果 xTicksToWait 为零，则如果集合中的所有队列和信号量都为空，则 xQueueSelectFromSet() 
		将立即返回。
			块时间以滴答周期指定，因此它所代表的绝对时间取决于滴答频率。 宏 pdMS_TO_TICKS() 可用于将
		以毫秒为单位的时间转换为以滴答声指定的时间。
		将 xTicksToWait 设置为 portMAX_DELAY 将导致任务无限期等待（不会超时），前提是 
		FreeRTOSConfig.h 中的 INCLUDE_vTaskSuspend 设置为 1。
Return:
	NULL - 无法从队列集中读取句柄。
	non-NULL - 返回值将是已知包含数据的队列或信号量的句柄. 句柄作为 QueueSetMemberHandle_t 类型
		返回，该类型可以转换为QueueHandle_t 类型或 SemaphoreHandle_t 类型。
*/
QueueSetMemberHandle_t xQueueSelectFromSet( QueueSetHandle_t xQueueSet,
																						const TickType_t xTicksToWait );
```

# 7. 使用队列创建邮箱

嵌入式社区内的术语没有达成共识，“邮箱”在不同的 RTOS 中意味着不同的东西。在本书中，术语邮箱用于指代长度为 1 的队列。队列可能因其在应用程序中的使用方式而被描述为邮箱，而不是因为它与队列在功能上有所不同：

- 队列用于将数据从一个任务发送到另一个任务，或从中断服务例程发送到任务。发送者将一个项目放入队列，接收者从队列中移除该项目。数据通过队列从发送方传递到接收方。
- 邮箱用于保存可由任何任务或任何中断服务程序读取的数据。数据不通过邮箱，而是保留在邮箱中，直到被覆盖。发件人覆盖邮箱中的值。接收者从邮箱中读取值，但不从邮箱中删除值。

## xQueueOverwrite()

```c
/*
	与 xQueueSendToBack() API 函数一样，xQueueOverwrite() API 函数将数据
发送到队列。 与 xQueueSendToBack() 不同，如果队列已满，则 
xQueueOverwrite() 将覆盖已在队列中的数据。
	xQueueOverwrite() 只能用于长度为 1 的队列。 该限制避免了函数的实现需
要任意决定要覆盖队列中的哪个项目。

Parameter:
	xQueue: 数据被发送的句柄。由[xQueueCreate](%E5%9B%9B%E3%80%81Queue%20Ma%20e73e4.md)()创建。
	pvItemToQueue: 指向要复制到队列中的数据的指针。
			队列可以容纳的每个项目的大小是在创建队列时设置的，因此会从 
		pvItemToQueue 将这么多字节复制到队列存储区中。
Return:
		即使队列已满，xQueueOverwrite() 也会写入队列，因此 pdPASS 是唯一可
	能的返回值。
	
*/
BaseType_t xQueueOverwrite( QueueHandle_t xQueue, const void * pvItemToQueue );
```

## xQueuePeek()

```c
/*
	xQueuePeek() 用于从队列中接收（读取）项目，而不会从队列中删除该项目。
xQueuePeek() 从队列头部接收数据，不修改队列中存储的数据，也不修改队列中
数据的存储顺序。
它的参数和return和[xQueueReceive](%E5%9B%9B%E3%80%81Queue%20Ma%20e73e4.md)()一致。
*/
BaseType_t xQueuePeek( QueueHandle_t xQueue,
											 void * const pvBuffer,
											 TickType_t xTicksToWait );
```