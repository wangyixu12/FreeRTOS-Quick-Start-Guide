# Heap_3

## 堆策略

直接通过 ***malloc()*** 和 ***free()***。所以它决定的堆的大小通过链接器配置，***configTOTAL_HEAP_SIZE***对其无影响。该策略通过临时挂起FreeRTOS的调度器来保证malloc()和free()的线程安全。