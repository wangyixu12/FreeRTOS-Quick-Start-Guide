# Heap_4

## 堆策略

同Heap_1 和 Heap_2一样将堆数组分成若干小块。也是配置相同的参数。

Heap_4 使用first fit algorithm， 它可以合并相互连接的空闲空间，以减少内存碎片。

该算法确保 pvPortMalloc() 使用第一个足够大的空闲空间用来分配需要的内存。 

## 优点

因为能够合并相互连接的空闲块，所以减少了内存碎片的风险。**该策略适合重复分配和释放不同大小的内存块的应用。**

虽然Heap_4也是不具备确定性的，但是**它的执行效率快过大多数标准库的 malloc() 和 free().**

## 图解

![Untitled](Heap_4%206b7e3/Untitled.png)

- A：三个任务被创建
- B：一个任务被删除，原先的TCB和stack分区被释放后合并成一个完整的内存块
- C：创建了一个队列，算法优先选择了第一个合适的空间。
- D：应用代码使用pvPortMalloc()获得一个小的内存块，同C一样，算法优先选择第一个合适的空间
- E：删除C中创建的队列
- F：删除D中创建的内存块，由于空闲内存相互连接，所以合并成一个大块内存。

# 番外篇

## 设置Heap_4在内存阵列中的起始地址

一般情况下，它的其实地址是被连接器自动分配。

但是当 configAPPLICATION_ALLOCATED_HEAP → 1时，内存阵列必须由FreeRTOS的Application声明。ucHeap必须在application中声明。

不同编译器内的语法不尽相同：

1. GCC compiler

```c
uint8_t ucHeap[configTOTAL_HEAP_SIZE] __attribute__ ((section(".my_heap"))); 
// 将内存阵列放在名为.my_heap的section中
```

1. IAR:

```c
uint8_t ucHeap[configTOTAL_HEAP_SIZE] @ 0x20000000
// 将内存阵列放在绝对地址0x20000000上
```