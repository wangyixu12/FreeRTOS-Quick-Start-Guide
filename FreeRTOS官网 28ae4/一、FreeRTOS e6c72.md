# 一、FreeRTOS的基本描述

## 要点：

1. **Definition: FreeRTOS Port**
    
    FreeRTOS 目前可以被20多种编译工具编译，并运行在30多种处理器上。这样每一种编译工具和处理器的组合被称为 **FreeRTOS Port**
    
2. **FreeRTOS通过 FreeRTOSConfig.h 文件进行配置**
3. **使用FreeRTOS API必须 include ‘FreeRTOS.h’.**
4. **FreeRTOS文件夹格式**
    
    ![Untitled](%E4%B8%80%E3%80%81FreeRTOS%20e6c72/Untitled.png)
    
5. **FreeRTOS的通用文件**
功能如其名
    
    ![Untitled](%E4%B8%80%E3%80%81FreeRTOS%20e6c72/Untitled%201.png)
    
6. **FreeRTOS Port 文件夹结构**
    
    ![Untitled](%E4%B8%80%E3%80%81FreeRTOS%20e6c72/Untitled%202.png)
    
7. **Demo 文件夹结构**
    
    ![Untitled](%E4%B8%80%E3%80%81FreeRTOS%20e6c72/Untitled%203.png)
    
8. **如何创建FreeRTOS项目**
    1. 修改对应Port的Demo
    2. 新建，参考1.4-Creating a New Project from Scratch
9. **数据类型：**
    1. TickType_t：
    FreeRTOS提供了一个周期性tick中断（Heartbeat），这个类型用来保存tick count value并指定时间的数据类型。
    TickType_t 可以是 unsigned 16-bit 类型，也可以是 unsigned 32-bit 类型。取决于是否置位了 ***configUSE_16_BIT_TICKS*** （再次强调，FreeRTOS配置选项在 **FreeRTOSConfig.h**）配置。
    2. BaseType_t:
    FreeRTOS框架最常用的基础类型，在32位架构中被定义成32-bit类型，在16位架构中被定义成16-bit类型，8位架构同理。
    通常被用在返回数值被限定的类型.
    3. FreeRTOS 需要显示的声明‘signed’和‘unsigned’，除非是用于保存字符类型或字符类型指针。FreeRTOS从不使用普通的int类型。
10. **变量命名：**
    
    前缀表示类型：
    
    c — char
    
    s — short
    
    l — long
    
    x — BaseType_t, structures, task handles, queue handles, etc.
    
    u — unsigned
    
    p — pointer
    
11. **函数命名**
    
    前缀表示行数返回类型:
    
    v — void
    
    x — BaseType_t
    
    pv — void *
    
    prv — private function (私有函数)
    
12. **宏命名**
大部分的宏用大写，前缀小写表示宏被定义的文件位置
    
    ![Untitled](%E4%B8%80%E3%80%81FreeRTOS%20e6c72/Untitled%204.png)
    
    ***注意： API的信号量通常用一系列的宏来调试，但它遵循函数命名规则，并不遵循宏命名规则***
    
    通用宏：
    
    ![Untitled](%E4%B8%80%E3%80%81FreeRTOS%20e6c72/Untitled%205.png)