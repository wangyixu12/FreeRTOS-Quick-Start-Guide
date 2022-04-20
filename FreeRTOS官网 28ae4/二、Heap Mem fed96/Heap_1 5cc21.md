# Heap_1

## 堆策略：

该策略下内存仅在应用使用实时功能时由Kernel动态分配，且在整个应用周期中都存在。这意味着我们无需去考虑如determinism（确定性）和fragmentation（碎片化）这类的复杂内存问题，仅需要考虑分配大小和simplicity（简化性）。

它只执行pvPortMalloc(), 不执行vPortFree()。即应用从来没有被删除。**适用于禁用内存动态分配的商业或者对安全极为关注的系统**。该策略的内存分配总是确定的，且不存在碎片化的。

## 配置

堆数组总大小在 FreeRTOSConfig.h 中的 configTOTAL_HEAP_SIZE 定义。 该选项定义了一个大块内存用于应用内存开销。

## 图解

![Untitled](Heap_1%205cc21/Untitled.png)

- A：堆数组，在任务创建前已经根据configTOTAL_HEAP_SIZE分配。
- B：一个任务被创建
- C：三个任务被创建

-