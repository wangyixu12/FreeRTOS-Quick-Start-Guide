# Heap_5

其释放和分配内存的算法和Heap_4是一样的，但Heap_5不限制内存仅使用一个静态的声明的内存队列用来分配，它可以从多个独立的内存空间中来分配内存。多使用在系统内存分配的内存块不连续的，不单一的场景中。

Heap_5是目前唯一一个需要在调用***pvPortMalloc()***前显示的初始化其内存分配表的算法。使用***vPortDefineHeapRegions()***来初始化Heap_5，且必须在内核对象被创建之前调用该函数。

```c
// The HeapRegion_t structure
typedef struct HeapRegion 
{ 
    /* The start address of a block of memory that will be part of the heap.*/ 
    uint8_t *pucStartAddress; 
 
    /* The size of the block of memory in bytes. */ 
    size_t xSizeInBytes; 
 
} HeapRegion_t;

/**
* pxHeapRegions 指向保存各个独立内存空间的数组开始位置。数组空间必须按顺序排列，最低的地址放在
*               数组的第一个位置，而最高的位置放在数组的最后一个位置。
*               数组的末尾由一个 pucStartAddress = NULL （参考如上的结构体） 的成员来标记结束
*/
void vPortDefineHeapRegions(const HeapRegion_t * const pxHeapRegions);
```

每一个独立的内存空间的类型都是 ***HeapRegion_t。*** 

如表达式可知，建立***HeapRegion_t***类型的数组，数组内容就是各个独立的内存空间。

## 例子

![Untitled](Heap_5%20b6b4b/Untitled.png)

A： 包含三个独立的内存块

```c
/* Define the start address and size of the three RAM regions. */ 
#define RAM1_START_ADDRESS    ( ( uint8_t * ) 0x00010000 ) 
#define RAM1_SIZE             ( 65 * 1024 ) 
 
#define RAM2_START_ADDRESS    ( ( uint8_t * ) 0x00020000 ) 
#define RAM2_SIZE             ( 32 * 1024 ) 
 
#define RAM3_START_ADDRESS    ( ( uint8_t * ) 0x00030000 ) 
#define RAM3_SIZE             ( 32 * 1024 ) 
 
/* Create an array of HeapRegion_t definitions, with an index for each of the three 
RAM regions, and terminating the array with a NULL address.  The HeapRegion_t 
structures must appear in start address order, with the structure that contains the 
lowest start address appearing first. */ 
const HeapRegion_t xHeapRegions[] = 
{ 
    { RAM1_START_ADDRESS, RAM1_SIZE }, 
    { RAM2_START_ADDRESS, RAM2_SIZE }, 
    { RAM3_START_ADDRESS, RAM3_SIZE }, 
    { NULL,               0         }  /* Marks the end of the array. */ 
}; 
 
int main( void ) 
{ 
    /* Initialize heap_5. */ 
    vPortDefineHeapRegions( xHeapRegions ); 
 
    /* Add application code here. */ 
}
```

B:  一般情况下，链接器需要一部分内存来存放RAM个数等信息。如图B所示，RAM1中包含了一部分链接器的变量信息，仅剩下一部分内存供Heap_5使用。因此，如果像如上代码这样声明，可能会导致存放变量的内存和Heap_5的内存重叠。为了避免这种现象，我们可以将RAM1_START_ADDRESS → 0x01nnnn。然而由于一下原因，并不推荐这样做：

1. 开始地址可能并不容易确认
2. 被使用的RAM的数量可能在未来的编译中被改变，这就需要修改一开始的起始地址
3. 编译工具可能并不知道“heap_5所使用的RAM覆盖了链接器所使用的RAM”，所以不会警告Coder

C：

```c
/* Define the start address and size of the two RAM regions not used by the  
linker. */ 
#define RAM2_START_ADDRESS    ( ( uint8_t * ) 0x00020000 ) 
#define RAM2_SIZE             ( 32 * 1024 ) 
 
#define RAM3_START_ADDRESS    ( ( uint8_t * ) 0x00030000 ) 
#define RAM3_SIZE             ( 32 * 1024 ) 
 
/* Declare an array that will be part of the heap used by heap_5.  The array will be 
placed in RAM1 by the linker. */ 
#define RAM1_HEAP_SIZE ( 30 * 1024 ) 
static uint8_t ucHeap[ RAM1_HEAP_SIZE ]; 
 
/* Create an array of HeapRegion_t definitions.  Whereas in Listing 6 the first entry 
described all of RAM1, so heap_5 will have used all of RAM1, this time the first 
entry only describes the ucHeap array, so heap_5 will only use the part of RAM1 that 
contains the ucHeap array.  The HeapRegion_t structures must still appear in start 
address order, with the structure that contains the lowest start address appearing 
first. */ 
const HeapRegion_t xHeapRegions[] = 
{ 
    { ucHeap,             RAM1_HEAP_SIZE }, 
    { RAM2_START_ADDRESS, RAM2_SIZE }, 
    { RAM3_START_ADDRESS, RAM3_SIZE }, 
    { NULL,               0         }  /* Marks the end of the array. */ 
};
```

这是一种更加方便维护的例子。它声明了***ucHeap***。这是一个一般变量，它成为来链接器分配到RAM1的数据的一部分。且它的大小和RAM1一样，同时被heap_5分配在RAM1的位置。以至于RAM1被***ucHeap***完全占满。

这样做的好处如下：

1. It is not necessary to use a hard coded start address.
2. 这个HeapRegion_t结构体的地址被链接器自动分配，所以它永远正确，即使在未来修改了RAM的数量。
3. Heap_5的分配的内存永远不可能覆盖链接器在RAM1中的位置。
4. 如果ucHeap太大，应用将无法链接（相当于编译器直接报错）。