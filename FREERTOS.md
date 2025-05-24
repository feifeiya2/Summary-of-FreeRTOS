# 关于队列机制解决，使用全局变量进行任务间通信存在的问题。

### 1. **互斥访问（原子操作）**

- **问题根源**：全局变量的并发读写可能导致竞态条件（如任务A写入时被任务B中断，读取到不完整数据）。
- **队列的解决方案**：
    - 队列内部通过临界区（Critical Section）或互斥锁（Mutex）实现原子操作，确保每次仅有一个任务访问队列。
    - 例如，`xQueueSend()`和`xQueueReceive()`函数在执行期间会暂时禁用中断或任务调度，防止操作被中断。

### 2. **数据拷贝而非共享**

- **问题根源**：全局变量直接共享内存，若发送方修改数据时接收方正在读取，会导致数据不一致。
- **队列的解决方案**：
    - 队列通过**数据拷贝**传递值而非引用。发送时，数据被复制到队列缓冲区；接收时，数据从缓冲区复制到接收任务的内存。
    - 发送后即使原始变量被修改，队列内的数据仍保持发送时的状态，确保接收方获取完整快照。

### 3. **阻塞机制与任务调度**

- **问题根源**：轮询全局变量会浪费CPU资源。
- **队列的解决方案**：
    - **阻塞等待**：当队列为空时，接收任务自动进入阻塞状态（不占用CPU），直到有数据到达；当队列满时，发送任务阻塞直到有空间。
    - **超时控制**：支持设置阻塞超时时间，防止任务永久挂起。
    - **任务优先级管理**：队列唤醒阻塞任务时，会根据任务优先级自动调度高优先级任务执行。

### 4. **中断安全**

- **问题根源**：中断中操作全局变量需复杂同步（如禁用中断）。
- **队列的解决方案**：
    - 提供中断专用API（如`xQueueSendFromISR()`和`xQueueReceiveFromISR()`），确保在中断上下文中安全操作队列。
    - 这些API通过调整中断上下文的任务唤醒逻辑，避免在中断中直接进行任务切换。

# 关于在队列集中，某队列还未加入队列集就开始写队列的问题。

**场景**：如果在**队列集**配置完成（创建队列集， 某队列加入队列集）之前，某队列开始写队列直至将队列空间用完，那么任务就无法通过队列集读取该队列， 则该队列就永远处于满的状态， 无法读出，也无法写入。

**解决**：在队列集配置（创建队列集， 某队列加入队列集）完成之后，再进行队列相关任务或中断函数等的写队列操作。





# ARM架构 cpu 内部寄存器：

![image-20250517171910193](../AppData/Roaming/Typora/typora-user-images/image-20250517171910193.png)

# 堆和栈

## 堆：

一块空闲的内存， 对其实现内存的分配和释放。

## 栈：

一块内存空间， cpu的sp寄存器指向这个栈，  可以用于 **函数调用** **局部变量** 多任务系统中**保存现场**

### 函数调用：

在c语言中， 调用函数时， 存在一种很常见的 情况， 函数中再次调用别的函数。  在c程序运行时， main函数调用 a函数，LR寄存器保存a的返回地址， 进入a函数开始运行，  若此时， a函数内部又调用了 b函数， 此时LR将 更新为b的返回地址。 为了防止返回地址 被覆盖，  在每次新调用的函数程序运行之前， 首先会先执行 ：将该函数的返回地址保存起来， 这里 保存的地方就是栈。 

 

### 局部变量：

在c语言中， 在函数运行的一开始，编译器会为函数在内存中开辟一块空间作为栈， 函数中的 局部变量 ，会优先存入寄存器中， 如果在变量声明前加 volatile 或 在 cpu的寄存器不够用的时候， 寄存器的值就会被存入栈中。



### 多任务系统：

在 freertos中， 每个函数都有自己的栈， 用来保存自己的函数调用关系， 局部变量 ， 现场。   在程序运行时， 任务A运行一半， 切换到任务B，  在切换之前，  需要将此时的 cpu寄存器的所有值（除了sp）， 保存到任务A 的 栈中，用于下一次任务切换回A时 恢复现场。 



# 任务创建：xTaskCreate()

```c
BaseType_t xTaskCreate(
    TaskFunction_t pxTaskCode,
    const char * const pcName,
    configSTACK_DEPTH_TYPE usStackDepth,
    void *pvParameters,
    UBaseType_t uxPriority,
    TaskHandle_t *pxCreatedTask
);
```

| 参数            | 类型                     | 说明                                                         |
| --------------- | ------------------------ | ------------------------------------------------------------ |
| `pxTaskCode`    | `TaskFunction_t`         | 任务函数指针。任务函数必须是返回类型为 `void`，参数为 `void *` 的函数。 |
| `pcName`        | `const char * const`     | 任务的名称（仅用于调试和跟踪，不影响功能）                   |
| `usStackDepth`  | `configSTACK_DEPTH_TYPE` | 任务栈的大小，以**字（word）为单位**，不是字节！例如在 STM32 上，1 word = 4 字节 |
| `pvParameters`  | `void *`                 | 传递给任务函数的参数，可以传结构体、变量指针等               |
| `uxPriority`    | `UBaseType_t`            | 任务优先级，越大优先级越高，`configMAX_PRIORITIES` 定义了最大优先级 |
| `pxCreatedTask` | `TaskHandle_t *`         | 任务句柄，用于以后控制（如挂起、删除等），可以传 `NULL` 忽略 |

返回值

- `pdPASS`：任务创建成功。
- `errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY`：内存不足，创建失败。

实例：

```c
  xTaskCreate(MyTask, "myfirsttask", 128, NULL, osPriorityNormal, NULL);
```

# 关于在 FreeRTOS中 任务优先级设置所需要考虑的问题：

总结：

STM32 使用 Cortex-M3/M4 的中断模型，FreeRTOS 规定：

> **只能在优先级值 ≥ `configMAX_SYSCALL_INTERRUPT_PRIORITY` 的中断中使用 FreeRTOS API（如信号量、队列）**

否则就会触发断言或导致崩溃。

       优先级数值：      0（最高） ←————— 5 —————→ 15（最低）
       
      			 ❌ 禁止用 FreeRTOS API
                    （0~4）
    
                ✅ 可以用 FreeRTOS API
                   （5~15）
                   
                   推荐中断配置：
    ─────────────────────────────────────────────
    任务              推荐优先级    是否可用FreeRTOS API
    ─────────────────────────────────────────────
    串口中断（USART）     6~10         ✅ 可以
    外部中断（按键）      10~14         ✅ 可以
    定时器中断（定时触发） 6~12          ✅ 可以
    ADC 中断              6~14        ✅ 可以
    DMA中断               6~14        ✅ 可以
    ─────────────────────────────────────────────
    
    【不要设置任何中断为 0~4，如果它用到了 FreeRTOS 的 API】


#  关于FreeRTOS任务调度与 OLED 显示刷新的时序问题 ：

在oled 第一行写 数据时 写了一般调度到了 task3  这时 虽然对于 程序来说  有保存现场的功能   但是对于 oled 却没有  所以oled 写task1 的 1写了一般被中断  这次的写1 的操作就完全作废了  即使 又调度回了task1  恢复了现场  但是 对于 oled来说 无法恢复现场  所以此时 就算程序恢复现场继续吧显示1 的操作进行完  也无法正常显示1

# 信号量创建 xSemaphoreCreateCounting（）

```c
SemaphoreHandle_t xSemaphoreCreateCounting(
    UBaseType_t uxMaxCount,
    UBaseType_t uxInitialCount
);
```

| 参数名           | 类型          | 含义                                 |
| ---------------- | ------------- | ------------------------------------ |
| `uxMaxCount`     | `UBaseType_t` | 计数信号量的最大计数值（最大资源数） |
| `uxInitialCount` | `UBaseType_t` | 初始计数值（初始可用资源数）         |



- **`uxInitialCount` ≤ `uxMaxCount`**，否则返回 NULL。
- 如果 `uxInitialCount == 0`，表示最初不可用，需要别的任务释放。

返回一个 `SemaphoreHandle_t`，即信号量句柄，用于后续对该信号量的操作（如 `xSemaphoreTake`、`xSemaphoreGive`）。

- 成功：返回一个有效的信号量句柄
- 失败：返回 NULL（如内存不足）

# 关于优先级反转的问题：

在使用信号量时  关于优先级反转的问题   我了解到  可以使用互斥量来解决 优先级反转的问题    原理是 优先级高的任务 可以暂时提高拿到互斥量的低优先级任务的优先级    我想具体了解一下     现在有 三个任务  task1 2 3     优先级分别是 0 1 2          task 1  和 task3   互斥，  task3 晚于 task1创建 所以 task1 先拿到 信号量   ，  但是 此时 task2 的优先级高于 task1 一直在运行 没有结束， 导致 task1  一直无法运行 从而导致 task3 迟迟无法获得 信号量   导致 优先级 反转的问题     这个场景中  我将 信号量改成 互斥量 就可以解决这个问题             我现在想知道  这里的原理  是因为操作系统 将 持有互斥量的任务提高到    所有等待这个互斥量任务中的 最高的一个优先级吗 。      还有， 这个继承高优先级的任务 是在释放互斥量的时候 恢复原来的优先级的吗



ChatGPT 说：

你描述得非常准确！你已经掌握了 FreeRTOS 中 **优先级反转问题**的核心，同时也正确理解了为什么 **互斥量（Mutex）**可以用来解决它。下面我将对你提出的问题进行系统、深入的解释：

------

🧠 一、什么是优先级反转（Priority Inversion）

**优先级反转**指的是：一个**低优先级**任务持有了一个关键资源（如信号量/互斥量），而一个**高优先级**任务需要等待该资源，但**中等优先级任务频繁执行**，导致低优先级任务无法运行、无法释放资源，最终让高优先级任务被“反转”到了更低的执行效率。

------

🎯 二、使用互斥量解决优先级反转的原理

FreeRTOS 的 **互斥量（Mutex）**具备 **优先级继承机制**（Priority Inheritance），与普通计数信号量不同。

✅ 互斥量中的优先级继承机制核心原理：

> 当一个高优先级任务（如 task3）阻塞在一个被低优先级任务（如 task1）持有的互斥量上时，RTOS **会临时将 task1 的优先级提升为 task3 的优先级**。

✅ 提升优先级的目的：

使得 task1 可以更快地执行并释放互斥量，避免被 task2（中优先级）抢占执行时间，从而减小高优先级任务的等待时间。

------

🧪 三、你描述的场景分析

🔧 任务创建与运行关系

| 任务名 | 优先级 | 是否使用互斥量 | 状态                         |
| ------ | ------ | -------------- | ---------------------------- |
| task1  | 0      | ✅ 持有互斥量   | 阻塞，等待运行机会释放互斥量 |
| task2  | 1      | ❌              | 一直运行，占据 CPU 时间片    |
| task3  | 2      | ✅ 等待互斥量   | 阻塞中，等待互斥量释放       |



🔁 使用 **普通信号量** 情况下：

- task3 被阻塞在信号量上
- task1 优先级低，始终无法运行释放信号量
- task2 优先级高，占用 CPU
- **task3 始终无法运行 → 典型的优先级反转现象**

🧠 改成 **互斥量（Mutex）** 后：

- task3（优先级 2）尝试获取 mutex，发现 task1（优先级 0）已经持有
- RTOS 检测到有高优先级任务等待 mutex，会**自动将 task1 的优先级提升为 2（等同于 task3）**
- task1 得以**优先执行并释放 mutex**
- task3 获得 mutex，继续执行
- 当 task1 释放 mutex 后，RTOS 会**恢复 task1 的原始优先级（0）**

------

🔄 四、你提出的问题回答总结

❓1. **是不是会把持有互斥量的任务优先级提升为等待该互斥量任务中的最高优先级？**

✅ 是的，FreeRTOS 实现了**优先级继承机制**：

> 任务A持有互斥量，任务B（优先级高）等待该互斥量，则任务A的优先级被临时提升为任务B的优先级，确保任务A能尽快完成工作并释放互斥量。

------

❓2. **释放互斥量时，是否会恢复原来的优先级？**

✅ 是的：

> 当任务A释放它持有的互斥量，并且没有更高优先级的任务在等待这个互斥量时，RTOS 会自动将任务A的优先级恢复为**最初的优先级**。

------

📌 五、FreeRTOS 中互斥量的创建方式（注意和普通信号量不同）

```c
SemaphoreHandle_t xMutex;
xMutex = xSemaphoreCreateMutex();  // 具有优先级继承能力
```

而不是 `xSemaphoreCreateCounting()` 或 `xSemaphoreCreateBinary()` —— 它们没有优先级继承功能。

------

🛠️ 六、实际开发建议

| 场景                         | 建议使用                                       |
| ---------------------------- | ---------------------------------------------- |
| 资源访问涉及多任务竞争       | 使用 `xSemaphoreCreateMutex()`（带优先级继承） |
| 计数、控制并发资源数量       | 使用 `xSemaphoreCreateCounting()`              |
| 同步中断和任务或简单事件通知 | 使用二值信号量或直接任务通知                   |

**在中断中不能使用互斥量**

## 信号量函数

使用信号量时，先创建、然后去添加资源、获得资源。使用句柄来表示一个信号量。

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter12.html#_12-2-1-创建)12.2.1 创建

使用信号量之前，要先创建，得到一个句柄；使用信号量时，要使用句柄来表明使用哪个信号量。 对于二进制信号量、计数型信号量，它们的创建函数不一样：

|          | 二进制信号量                                   | 计数型信号量                   |
| -------- | ---------------------------------------------- | ------------------------------ |
| 动态创建 | xSemaphoreCreateBinary 计数值初始值为0         | xSemaphoreCreateCounting       |
|          | vSemaphoreCreateBinary(过时了) 计数值初始值为1 |                                |
| 静态创建 | xSemaphoreCreateBinaryStatic                   | xSemaphoreCreateCountingStatic |

创建二进制信号量的函数原型如下：

```c
/* 创建一个二进制信号量，返回它的句柄。
 * 此函数内部会分配信号量结构体 
 * 返回值: 返回句柄，非NULL表示成功
 */
SemaphoreHandle_t xSemaphoreCreateBinary( void );

/* 创建一个二进制信号量，返回它的句柄。
 * 此函数无需动态分配内存，所以需要先有一个StaticSemaphore_t结构体，并传入它的指针
 * 返回值: 返回句柄，非NULL表示成功
 */
SemaphoreHandle_t xSemaphoreCreateBinaryStatic( StaticSemaphore_t *pxSemaphoreBuffer );
```

创建计数型信号量的函数原型如下：

```c
/* 创建一个计数型信号量，返回它的句柄。
 * 此函数内部会分配信号量结构体 
 * uxMaxCount: 最大计数值
 * uxInitialCount: 初始计数值
 * 返回值: 返回句柄，非NULL表示成功
 */
SemaphoreHandle_t xSemaphoreCreateCounting(UBaseType_t uxMaxCount, UBaseType_t uxInitialCount);

/* 创建一个计数型信号量，返回它的句柄。
 * 此函数无需动态分配内存，所以需要先有一个StaticSemaphore_t结构体，并传入它的指针
 * uxMaxCount: 最大计数值
 * uxInitialCount: 初始计数值
 * pxSemaphoreBuffer: StaticSemaphore_t结构体指针
 * 返回值: 返回句柄，非NULL表示成功
 */
SemaphoreHandle_t xSemaphoreCreateCountingStatic( UBaseType_t uxMaxCount, 
                                                 UBaseType_t uxInitialCount, 
                                                 StaticSemaphore_t *pxSemaphoreBuffer );
```

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter12.html#_12-2-2-删除)12.2.2 删除

对于动态创建的信号量，不再需要它们时，可以删除它们以回收内存。

vSemaphoreDelete可以用来删除二进制信号量、计数型信号量，函数原型如下：

```c
/*
 * xSemaphore: 信号量句柄，你要删除哪个信号量
 */
void vSemaphoreDelete( SemaphoreHandle_t xSemaphore );
```

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter12.html#_12-2-3-give-take)12.2.3 give/take

二进制信号量、计数型信号量的give、take操作函数是一样的。这些函数也分为2个版本：给任务使用，给ISR使用。列表如下：

|      | 在任务中使用   | 在ISR中使用           |
| ---- | -------------- | --------------------- |
| give | xSemaphoreGive | xSemaphoreGiveFromISR |
| take | xSemaphoreTake | xSemaphoreTakeFromISR |

xSemaphoreGive的函数原型如下：

```c
BaseType_t xSemaphoreGive( SemaphoreHandle_t xSemaphore );
```

xSemaphoreGive函数的参数与返回值列表如下：

| 参数       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| xSemaphore | 信号量句柄，释放哪个信号量                                   |
| 返回值     | pdTRUE表示成功, 如果二进制信号量的计数值已经是1，再次调用此函数则返回失败； 如果计数型信号量的计数值已经是最大值，再次调用此函数则返回失败 |

pxHigherPriorityTaskWoken的函数原型如下：

```c
BaseType_t xSemaphoreGiveFromISR(
                        SemaphoreHandle_t xSemaphore,
                        BaseType_t *pxHigherPriorityTaskWoken
                    );
```

xSemaphoreGiveFromISR函数的参数与返回值列表如下：

| 参数                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| xSemaphore                | 信号量句柄，释放哪个信号量                                   |
| pxHigherPriorityTaskWoken | 如果释放信号量导致更高优先级的任务变为了就绪态， 则*pxHigherPriorityTaskWoken = pdTRUE |
| 返回值                    | pdTRUE表示成功, 如果二进制信号量的计数值已经是1，再次调用此函数则返回失败； 如果计数型信号量的计数值已经是最大值，再次调用此函数则返回失败 |

xSemaphoreTake的函数原型如下：

```c
BaseType_t xSemaphoreTake(
                   SemaphoreHandle_t xSemaphore,
                   TickType_t xTicksToWait
               );
```

xSemaphoreTake函数的参数与返回值列表如下：

| 参数         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| xSemaphore   | 信号量句柄，获取哪个信号量                                   |
| xTicksToWait | 如果无法马上获得信号量，阻塞一会： 0：不阻塞，马上返回 portMAX_DELAY: 一直阻塞直到成功 其他值: 阻塞的Tick个数，可以使用*pdMS_TO_TICKS()*来指定阻塞时间为若干ms |
| 返回值       | pdTRUE表示成功                                               |

xSemaphoreTakeFromISR的函数原型如下：

```c
BaseType_t xSemaphoreTakeFromISR(
                        SemaphoreHandle_t xSemaphore,
                        BaseType_t *pxHigherPriorityTaskWoken
                    );
```

xSemaphoreTakeFromISR函数的参数与返回值列表如下：

| 参数                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| xSemaphore                | 信号量句柄，获取哪个信号量                                   |
| pxHigherPriorityTaskWoken | 如果获取信号量导致更高优先级的任务变为了就绪态， 则*pxHigherPriorityTaskWoken = pdTRUE |
| 返回值                    | pdTRUE表示成功                                               |

##  互斥量函数

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter13.html#_13-2-1-创建)13.2.1 创建

互斥量是一种特殊的二进制信号量。

使用互斥量时，先创建、然后去获得、释放它。使用句柄来表示一个互斥量。

创建互斥量的函数有2种：动态分配内存，静态分配内存，函数原型如下：

```c
/* 创建一个互斥量，返回它的句柄。
 * 此函数内部会分配互斥量结构体 
 * 返回值: 返回句柄，非NULL表示成功
 */
SemaphoreHandle_t xSemaphoreCreateMutex( void );

/* 创建一个互斥量，返回它的句柄。
 * 此函数无需动态分配内存，所以需要先有一个StaticSemaphore_t结构体，并传入它的指针
 * 返回值: 返回句柄，非NULL表示成功
 */
SemaphoreHandle_t xSemaphoreCreateMutexStatic( StaticSemaphore_t *pxMutexBuffer );
```

要想使用互斥量，需要在配置文件FreeRTOSConfig.h中定义：

```c
#define configUSE_MUTEXES 1
```

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter13.html#_13-2-2-其他函数)13.2.2 其他函数

要注意的是，互斥量不能在ISR中使用。

各类操作函数，比如删除、give/take，跟一般是信号量是一样的。

```c
/*
 * xSemaphore: 信号量句柄，你要删除哪个信号量, 互斥量也是一种信号量
 */
void vSemaphoreDelete( SemaphoreHandle_t xSemaphore );

/* 释放 */
BaseType_t xSemaphoreGive( SemaphoreHandle_t xSemaphore );


/* 获得 */
BaseType_t xSemaphoreTake(
                   SemaphoreHandle_t xSemaphore,
                   TickType_t xTicksToWait
               );
```





# 队列：

## 队列函数

使用队列的流程：创建队列、写队列、读队列、删除队列。

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter11.html#_11-2-1-创建)11.2.1 创建

队列的创建有两种方法：动态分配内存、静态分配内存，

- 动态分配内存：xQueueCreate，队列的内存在函数内部动态分配

函数原型如下：

```c
QueueHandle_t xQueueCreate( UBaseType_t uxQueueLength, UBaseType_t uxItemSize );
```

| **参数**      | **说明**                                                     |
| ------------- | ------------------------------------------------------------ |
| uxQueueLength | 队列长度，最多能存放多少个数据(item)                         |
| uxItemSize    | 每个数据(item)的大小：以字节为单位                           |
| 返回值        | 非0：成功，返回句柄，以后使用句柄来操作队列 NULL：失败，因为内存不足 |

- 静态分配内存：xQueueCreateStatic，队列的内存要事先分配好

函数原型如下：

```c
QueueHandle_t xQueueCreateStatic(*
              		UBaseType_t uxQueueLength,*
              		UBaseType_t uxItemSize,*
              		uint8_t *pucQueueStorageBuffer,*
              		StaticQueue_t *pxQueueBuffer*
           		 );
```

| **参数**              | **说明**                                                     |
| --------------------- | ------------------------------------------------------------ |
| uxQueueLength         | 队列长度，最多能存放多少个数据(item)                         |
| uxItemSize            | 每个数据(item)的大小：以字节为单位                           |
| pucQueueStorageBuffer | 如果uxItemSize非0，pucQueueStorageBuffer必须指向一个uint8_t数组， 此数组大小至少为"uxQueueLength * uxItemSize" |
| pxQueueBuffer         | 必须执行一个StaticQueue_t结构体，用来保存队列的数据结构      |
| 返回值                | 非0：成功，返回句柄，以后使用句柄来操作队列 NULL：失败，因为pxQueueBuffer为NULL |

示例代码：

```c
// 示例代码
 #define QUEUE_LENGTH 10
 #define ITEM_SIZE sizeof( uint32_t )
 
 // xQueueBuffer用来保存队列结构体
 StaticQueue_t xQueueBuffer;

// ucQueueStorage 用来保存队列的数据

// 大小为：队列长度 * 数据大小
 uint8_t ucQueueStorage[ QUEUE_LENGTH * ITEM_SIZE ];

 void vATask( void *pvParameters )
 {
	QueueHandle_t xQueue1;

	// 创建队列: 可以容纳QUEUE_LENGTH个数据，每个数据大小是ITEM_SIZE
	xQueue1 = xQueueCreateStatic( QUEUE_LENGTH,
							ITEM_SIZE,
                            ucQueueStorage,
                            &xQueueBuffer ); 
  }
```

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter11.html#_11-2-2-复位)11.2.2 复位

队列刚被创建时，里面没有数据；使用过程中可以调用 **xQueueReset()** 把队列恢复为初始状态，此函数原型为：

```c
/*  pxQueue : 复位哪个队列;
 * 返回值: pdPASS(必定成功)
*/
BaseType_t xQueueReset( QueueHandle_t pxQueue);
```

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter11.html#_11-2-3-删除)11.2.3 删除

删除队列的函数为 **vQueueDelete()** ，只能删除使用动态方法创建的队列，它会释放内存。原型如下：

```c
void vQueueDelete( QueueHandle_t xQueue );
```

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter11.html#_11-2-4-写队列)11.2.4 写队列

可以把数据写到队列头部，也可以写到尾部，这些函数有两个版本：在任务中使用、在ISR中使用。函数原型如下：

```c
/* 等同于xQueueSendToBack
 * 往队列尾部写入数据，如果没有空间，阻塞时间为xTicksToWait
 */
BaseType_t xQueueSend(
                                QueueHandle_t    xQueue,
                                const void       *pvItemToQueue,
                                TickType_t       xTicksToWait
                            );

/* 
 * 往队列尾部写入数据，如果没有空间，阻塞时间为xTicksToWait
 */
BaseType_t xQueueSendToBack(
                                QueueHandle_t    xQueue,
                                const void       *pvItemToQueue,
                                TickType_t       xTicksToWait
                            );


/* 
 * 往队列尾部写入数据，此函数可以在中断函数中使用，不可阻塞
 */
BaseType_t xQueueSendToBackFromISR(
                                      QueueHandle_t xQueue,
                                      const void *pvItemToQueue,
                                      BaseType_t *pxHigherPriorityTaskWoken
                                   );

/* 
 * 往队列头部写入数据，如果没有空间，阻塞时间为xTicksToWait
 */
BaseType_t xQueueSendToFront(
                                QueueHandle_t    xQueue,
                                const void       *pvItemToQueue,
                                TickType_t       xTicksToWait
                            );

/* 
 * 往队列头部写入数据，此函数可以在中断函数中使用，不可阻塞
 */
BaseType_t xQueueSendToFrontFromISR(
                                      QueueHandle_t xQueue,
                                      const void *pvItemToQueue,
                                      BaseType_t *pxHigherPriorityTaskWoken
                                   );
```

这些函数用到的参数是类似的，统一说明如下：

| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| xQueue        | 队列句柄，要写哪个队列                                       |
| pvItemToQueue | 数据指针，这个数据的值会被复制进队列， 复制多大的数据？在创建队列时已经指定了数据大小 |
| xTicksToWait  | 如果队列满则无法写入新数据，可以让任务进入阻塞状态， xTicksToWait表示阻塞的最大时间(Tick Count)。 如果被设为0，无法写入数据时函数会立刻返回； 如果被设为portMAX_DELAY，则会一直阻塞直到有空间可写 |
| 返回值        | pdPASS：数据成功写入了队列 errQUEUE_FULL：写入失败，因为队列满了。 |

🎯 `pxHigherPriorityTaskWoken` 的作用详解

这是一个 **输出参数**（`BaseType_t *` 类型），用于告诉 FreeRTOS：

> “由于发送数据后，是否有更高优先级的任务因为接收到数据被唤醒了？”

✅ 如果唤醒了更高优先级任务，会把 `*pxHigherPriorityTaskWoken` 设为 `pdTRUE`

你需要在中断结束前检查它的值，并调用：

```
c


复制编辑
portYIELD_FROM_ISR(pxHigherPriorityTaskWoken);
```

来**立即切换到更高优先级的任务**，实现低延迟的任务响应。

------

🧠 为何要这样设计？

中断不能直接进行任务切换，它只能设置“切换请求”。FreeRTOS 通过 `pxHigherPriorityTaskWoken` 这一机制，让你决定是否要在中断后切换上下文。

------

✅ 正确使用示例：

```
c复制编辑void USART1_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    uint8_t ch = USART1->DR;

    // 将数据放入队列
    xQueueSendToBackFromISR(xRxQueue, &ch, &xHigherPriorityTaskWoken);

    // 如果有更高优先级的任务被唤醒，则触发一次上下文切换
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

------

🚫 常见错误

❌ 忘记写 `portYIELD_FROM_ISR()`：

会导致尽管有高优任务就绪了，但任务切换推迟到下一次 tick，**影响实时性**。





### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter11.html#_11-2-5-读队列)11.2.5 读队列

使用 **xQueueReceive()** 函数读队列，读到一个数据后，队列中该数据会被移除。这个函数有两个版本：在任务中使用、在ISR中使用。函数原型如下：

```c
BaseType_t xQueueReceive( QueueHandle_t xQueue,
                          void * const pvBuffer,
                          TickType_t xTicksToWait );

BaseType_t xQueueReceiveFromISR(
                                    QueueHandle_t    xQueue,
                                    void             *pvBuffer,
                                    BaseType_t       *pxTaskWoken
                                );
```

参数说明如下：

| **参数**     | **说明**                                                     |
| ------------ | ------------------------------------------------------------ |
| xQueue       | 队列句柄，要读哪个队列                                       |
| pvBuffer     | bufer指针，队列的数据会被复制到这个buffer 复制多大的数据？在创建队列时已经指定了数据大小 |
| xTicksToWait | 果队列空则无法读出数据，可以让任务进入阻塞状态， xTicksToWait表示阻塞的最大时间(Tick Count)。 如果被设为0，无法读出数据时函数会立刻返回； 如果被设为portMAX_DELAY，则会一直阻塞直到有数据可写 |
| 返回值       | pdPASS：从队列读出数据入 errQUEUE_EMPTY：读取失败，因为队列空了。 |

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter11.html#_11-2-6-查询)11.2.6 查询

可以查询队列中有多少个数据、有多少空余空间。函数原型如下：

```c
/*
 * 返回队列中可用数据的个数
 */
UBaseType_t uxQueueMessagesWaiting( const QueueHandle_t xQueue );

/*
 * 返回队列中可用空间的个数
 */
UBaseType_t uxQueueSpacesAvailable( const QueueHandle_t xQueue );
```

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter11.html#_11-2-7-覆盖-偷看)11.2.7 覆盖/偷看

当队列长度为1时，可以使用 **xQueueOverwrite()** 或 **xQueueOverwriteFromISR()** 来覆盖数据。

注意，队列长度必须为1。当队列满时，这些函数会覆盖里面的数据，这也以为着这些函数不会被阻塞。

函数原型如下：

```c
/* 覆盖队列
 * xQueue: 写哪个队列
 * pvItemToQueue: 数据地址
 * 返回值: pdTRUE表示成功, pdFALSE表示失败
 */
BaseType_t xQueueOverwrite(
                           QueueHandle_t xQueue,
                           const void * pvItemToQueue
                      );

BaseType_t xQueueOverwriteFromISR(
                           QueueHandle_t xQueue,
                           const void * pvItemToQueue,
                           BaseType_t *pxHigherPriorityTaskWoken
                      );
```

如果想让队列中的数据供多方读取，也就是说读取时不要移除数据，要留给后来人。那么可以使用"窥视"，也就是**xQueuePeek()\**或\**xQueuePeekFromISR()**。这些函数会从队列中复制出数据，但是不移除数据。这也意味着，如果队列中没有数据，那么"偷看"时会导致阻塞；一旦队列中有数据，以后每次"偷看"都会成功。

函数原型如下：

```c
/* 偷看队列
 * xQueue: 偷看哪个队列
 * pvItemToQueue: 数据地址, 用来保存复制出来的数据
 * xTicksToWait: 没有数据的话阻塞一会
 * 返回值: pdTRUE表示成功, pdFALSE表示失败
 */
BaseType_t xQueuePeek(
                          QueueHandle_t xQueue,
                          void * const pvBuffer,
                          TickType_t xTicksToWait
                      );

BaseType_t xQueuePeekFromISR(
                                 QueueHandle_t xQueue,
                                 void *pvBuffer,
                             );
```

# 队列集：

### 创建队列集

函数原型如下：

```c
QueueSetHandle_t xQueueCreateSet( const UBaseType_t uxEventQueueLength )
```

| **参数**      | **说明**                                                     |
| ------------- | ------------------------------------------------------------ |
| uxQueueLength | 队列集长度，最多能存放多少个数据(队列句柄)                   |
| 返回值        | 非0：成功，返回句柄，以后使用句柄来操作队列NULL：失败，因为内存不足 |

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter11.html#_11-5-2-把队列加入队列集)11.5.2 把队列加入队列集

函数原型如下：

```c
BaseType_t xQueueAddToSet( QueueSetMemberHandle_t xQueueOrSemaphore,

                QueueSetHandle_t xQueueSet );

 
```

| **参数**          | **说明**                       |
| ----------------- | ------------------------------ |
| xQueueOrSemaphore | 队列句柄，这个队列要加入队列集 |
| xQueueSet         | 队列集句柄                     |
| 返回值            | pdTRUE：成功pdFALSE：失败      |

### [#](https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter11.html#_11-5-3-读取队列集)11.5.3 读取队列集

函数原型如下：

```c
QueueSetMemberHandle_t xQueueSelectFromSet( QueueSetHandle_t xQueueSet,

                        TickType_t const xTicksToWait );
```

| **参数**     | **说明**                                                     |
| ------------ | ------------------------------------------------------------ |
| xQueueSet    | 队列集句柄                                                   |
| xTicksToWait | 如果队列集空则无法读出数据，可以让任务进入阻塞状态，xTicksToWait表示阻塞的最大时间(Tick Count)。如果被设为0，无法读出数据时函数会立刻返回；如果被设为portMAX_DELAY，则会一直阻塞直到有数据可写 |
| 返回值       | NULL：失败，队列句柄：成功                                   |

## 发问：

如果我只创建一个队列  然后多个任务写这一个队列  虽然是不同任务发来的数据  但是我可以 对所有任务 数据进行规定 ， 比如  所有任务写队列的数据都是结构体  然后结构体内有一个元素是专门区分是哪个任务发来的数据，   这样 不会出现 多个读队列函数 共同阻塞的问题  也可以实现区分 数据来源   这个效果是不是和队列集 是一样的



ChatGPT 说：

你的思路是**非常合理且实用的**，也是在实际开发中非常常见的一种简化做法。我们来详细对比你描述的方法和 FreeRTOS 队列集的本质差异与适用场景。

------

✅ 你的方法回顾

你打算：

- **只创建一个队列**
- **多个任务向这个队列发送数据**
- 所有发送的数据是结构体，如：

```
c复制编辑typedef struct {
    uint8_t source_id; // 区分来源任务
    uint8_t payload[10]; // 实际数据
} Message_t;
```

- 接收端任务通过 `xQueueReceive()` 统一接收，然后依据 `source_id` 判断数据来源。

------

✅ 这样做可以达到的效果

| 功能                                      | 是否支持                  |
| ----------------------------------------- | ------------------------- |
| 支持多个任务发送数据                      | ✅ 是的                    |
| 支持接收端识别数据来源                    | ✅ 通过结构体字段实现      |
| 避免多个 `xQueueReceive()` 造成的多次阻塞 | ✅ 一个队列 + 一个阻塞接收 |



------

✅ 与队列集的比较

| 特性           | 你的方法（单队列+区分字段） | 队列集（QueueSet）                   |
| -------------- | --------------------------- | ------------------------------------ |
| 数据来源区分   | ✅ 通过 `source_id` 字段     | ✅ 通过返回的句柄                     |
| 接收端阻塞次数 | ✅ 一次 `xQueueReceive()`    | ✅ 一次 `xQueueSelectFromSet()`       |
| 系统开销       | ✅ 更低（一个队列）          | ⛔ 更多句柄、需要队列集对象           |
| 灵活性         | ✅ 简洁、代码集中            | ✅ 更通用，可监听信号量、多个模块分开 |
| 可读性         | ✅ 结构化代码                | ✅ 适合复杂模块解耦                   |



------

✅ 你的方法适合什么场景？

✅ 推荐使用你这种方案的情况：

- 数据结构统一、清晰（结构体中包含来源字段）
- 队列数量不多，只需要一个接收端处理所有消息
- 系统资源受限，尽量少用内存、队列句柄
- 所有任务间关系明确，可以用代码统一约定格式

------

❌ 不适合你的方法的情况：

🔻 建议使用 **队列集** 的情况：

- 接收的数据种类差异较大，不便统一为一个结构体（如串口任务、CAN 任务、按键任务）
- 每个子模块或任务独立，发送不同类型数据
- 队列的数量较多，队列大小不一，处理方式也差异大
- 希望通过队列句柄能直接路由到某个子处理逻辑模块（更模块化）

------

✅ 小结

你的这个方法确实可以实现 **类似队列集的“统一接收 + 来源识别”的效果**，**而且开销更低、实现更直接**。

✅ 优点：

- 简洁高效，结构清晰
- 更容易调试
- 占用资源更少

⛔ 局限：

- 结构体要统一管理，耦合性高
- 灵活性略逊于队列集（无法监听信号量）

------

🎯 建议：

如果你控制整个任务系统结构，而且数据类型能统一，**推荐优先使用你这种“单队列+结构体+来源字段”的方式**，它比队列集更轻量、简单好用。

但如果你希望系统解耦、模块间独立、动态组合监听，或者数据种类复杂，再考虑使用队列集更合适。

## 补充：关于队列集监听信号量的问题

```c
QueueSetHandle_t queueSet = xQueueCreateSet(10);  // 创建队列集

xQueueAddToSet(uartQueue, queueSet);
xQueueAddToSet(canQueue, queueSet);
xSemaphoreAddToSet(gpioBinarySemaphore, queueSet); // 因为它是“伪队列”，也可以加入
```

然后在主任务中：

```c
QueueSetMemberHandle_t activatedHandle = xQueueSelectFromSet(queueSet, portMAX_DELAY);

if (activatedHandle == uartQueue) {
    // 从 uartQueue 中读取数据
} else if (activatedHandle == gpioBinarySemaphore) {
    // 表示发生了中断
    xSemaphoreTake(gpioBinarySemaphore, 0);
}
```

将二值信号量加入队列集的目的，是为了在**一个任务中统一监听多个事件来源（如多个队列和信号量）**，使代码结构更清晰、管理更集中。

相比起用多个 `xQueueReceive()` 和 `xSemaphoreTake()` 分别监听多个队列/信号量，**队列集能避免复杂的多路等待逻辑**。

**功能上并没有增加新能力** —— 不管你用不用队列集，每个队列和信号量都可以单独工作。
 难点只在于**监听方式更优雅统一**，尤其适合“多个数据来源、一个任务统一处理”的场景。
