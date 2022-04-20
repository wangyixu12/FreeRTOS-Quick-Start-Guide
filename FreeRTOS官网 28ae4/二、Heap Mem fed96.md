# 二、Heap Memory Management

## 前置内容：

**FreeRTOS 对RAM的申请是动态的：**

FreeRTOS 的 Kernel 对象（tasks, queues, semaphores and event groups）内存都不是在编译时被静态的分配的，而是在运行时被**动态分配**。当创建Kernel 对象时分配RAM，在删除Kernel对象时释放RAM。 *这样的实际减少了前期设计和规划的工作量，同时简化了API 和减少了对RAM的占用。*

FreeRTOS 的动态内存分配使用 **pvPortMalloc()** 和 **vPortFree()** 去替换标准库中的 malloc() 和 free()。这是由于 malloc() 和 free() 有以下缺点，使其并不合适用于小型嵌入式实时系统：

1. 它们在小型嵌入式系统中并不总是可用
2. 其实现需要占用大量的代码空间
3. 他们几乎不是 thead-safe
4. 不具有确定性，在不同的调用中就有不同的时间开销
5. 经常产生碎片
6. 将使编译链变得复杂
7. 当使用它们而发生内存越界时，他们很难被debug

FreeRTOS 提供五个堆管理算法的源码，heap_1.c, heap_2.c, heap_3.c, heap_4.c 和 heap_5.c, 位于 ‘***FreeRTOS/Source/portable/MemMang***’ 文件夹中，以下将介绍各个策略。

## 相关缩写：

TCB — task control block (创建任务时分配内存块的其中一部分)

创建内存

## 5种堆策略

[Heap_1](%E4%BA%8C%E3%80%81Heap%20Mem%20fed96/Heap_1%205cc21.md)

[Heap_2](%E4%BA%8C%E3%80%81Heap%20Mem%20fed96/Heap_2%20573d0.md)

[Heap_3](%E4%BA%8C%E3%80%81Heap%20Mem%20fed96/Heap_3%20172bf.md)

[Heap_4](%E4%BA%8C%E3%80%81Heap%20Mem%20fed96/Heap_4%206b7e3.md)

[Heap_5](%E4%BA%8C%E3%80%81Heap%20Mem%20fed96/Heap_5%20b6b4b.md)

## 与堆相关的实用的函数

```c
/**
* Return: 剩余未被分配的堆空间大小。
* 可以用来优化堆的大小。
* Note: 对heap_3无效。
*/
size_t xPortGetFreeHeapSize( void );
```

```c
/**
* Return: 返回从系统开始到调用该函数时，最小的从未被使用的堆大小
* 用来表明程序距离耗尽堆空间还差多少空间。
* Note: 仅对heap_4 和 heap_5有效。
*/
size_t xPortGetMinimumEverFreeHeapSize( void );
```

Malloc 失败的钩子函数：

***pvPortMalloc***() 可以被应用代码直接调用，也可以被FreeRTOS Kernel 对象每次创建的过程中调用。但是当其请求的内存块不够或者不存在时，其将返回NULL，且不会创建相应的对象（如果是Kernel调用）。所以有如下钩子函数在***pvPortMalloc***()返回NULL时被调用。

```c
/**
* 配置： configUSE_MALLOC_FAILED_HOOK -> 1  /FreeRTOSConfig.h
* 该函数为Weaken声明，可以在应用中合适的位置重新实现该函数。
* 如下为钩子函数原型。
*/
void vApplicationMallocFailedHook( void );
```