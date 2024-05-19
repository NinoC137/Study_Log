# CMSIS-RTOS的消息队列及内存池

使用CMSIS-RTOS: FreeRTOS的消息队列及内存池实现了**传输自定义的结构体变量**



## 为什么要使用"内存池"这类内存管理模块?

对于消息队列来说, 要想传输结构体变量, 只能使用传入指针的方式来进行实现

如:

``` c
typedef struct JsonPackage{
    int counter;
    char JsonString[128];
} t_JsonPackage;

	t_JsonPackage *JsonPtr = osPoolAlloc(JsonQ_Mem);
	/*
	* 或者使用 
	* t_JsonPackage *JsonPtr = (t_JsonPackage *)malloc(sizeof(t_JsonPackage));
	*/
	osMessagePut(JsonQueueHandle, (uint32_t)JsonPtr, 0);
```

若是没有对此指针指向的内存区进行内存管理, 则会产生大量不可控的内存碎片, 有运行时的内存隐患

CMSIS-RTOS对内存的管理进行了统一化, 预先创建好需要的内存池, 将对应的指针变量分配进内存池之中, 便能够安全有效地进行内存的管理与控制

在接收sender发送出的指针, 并完成对应处理后, 便可调用对应的API来销毁这部分不需要的内存.

``` c
	osPoolFree(JsonQ_Mem, JsonBuffer);
```



## 对应模块的初始化

首先定义内存池及消息队列的属性及对应句柄

``` c
//必须是全局变量!
osPoolDef(JsonQ_Mem, 16, t_JsonPackage);
osMessageQDef(JsonQueue, 16, uint32_t);
osPoolId JsonQ_Mem;
osMessageQId JsonQueueHandle;

//创建于其他函数调用内存池/消息队列之前
JsonQ_Mem = osPoolCreate(osPool(JsonQ_Mem));
JsonQueueHandle = osMessageCreate(osMessageQ(JsonQueue), NULL);
```

**这里的句柄必须要是全局变量! 并且在需要调用的地方要extern进去**

原因很简单, 最浅显的原因就是, 不这样做的话, osMessagePut或osMessageGet函数无法正常传参

较为深层的原因更容易让人理解, 即: 若是内存池的地址是随机变化的, 那我们自然无法在一个"变化不停"的内存片上进行管理

从代码层次上来看, 链表的头自然不可以乱动.



## 消息队列的发送与接收

前文已说明了内存池的用处, 接下来就进入实战环节, 从代码的角度来感受用法

我这里实现的功能是, 接收到一个完整的Json数据串后, 将接收到的数据串传入消息队列之中, 然后在接收线程部分对其进行处理

``` c
//发送者
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart){
    if(huart == &huart3){
        static int uart3_RxFlag, uart3_RxStatus;
        switch(uart3_RxStatus){
            case 0:
                if(uart3Buffer[uart3_RxFlag] == UART_STARTFLAG){
                    uart3_RxFlag++;
                    uart3_RxStatus = 1;
                }else{
                    memset(uart3Buffer, 0, sizeof(uart3Buffer));
                }
                break;
            case 1:
                if(uart3Buffer[uart3_RxFlag] != UART_ENDFLAG){
                    uart3_RxFlag++;
                }else{
                    memset(&uart3Buffer[uart3_RxFlag + 1], 0, sizeof(uart3Buffer) - uart3_RxFlag);
                    uart3_RxFlag = 0;
                    uart3_RxStatus = 0;

                    t_JsonPackage *JsonPtr = osPoolAlloc(JsonQ_Mem);
                    memcpy(JsonPtr->JsonString, uart3Buffer, sizeof(uart3Buffer));
                    osMessagePut(JsonQueueHandle, (uint32_t)JsonPtr, 0);
                }
                break;
            default:
                break;
        }
        HAL_UART_Receive_IT(&huart3, (uint8_t*)&uart3Buffer[uart3_RxFlag], 1);
    }
}

//接收者
void UARTTask(void const *argument) {
    HAL_UART_Receive_IT(&huart3, (uint8_t*)&uart3Buffer, 1);

    osEvent JsonQueueEvt;	//接收消息的句柄
    t_JsonPackage *JsonBuffer = NULL;	//这里不需要分配内存, 因为等下就要将内存池中的发送者指针地址赋值于此变量

    for (;;) {
        JsonQueueEvt = osMessageGet(JsonQueueHandle, osWaitForever);	//若是没有接收到消息, 则阻塞式等候

        if(JsonQueueEvt.status == osEventMessage){
            JsonBuffer = JsonQueueEvt.value.p;	//获取发送者传入的结构体指针变量, 彻底完成消息的接收步骤
            uart3_printf("buffer: %s\r\n", JsonBuffer->JsonString);
            osPoolFree(JsonQ_Mem, JsonBuffer);	//处理完毕数据之后, 及时清理内存, 避免内存溢出
        }
    }
}
```



## 消息队列的意义

1. **解耦模块**：消息队列可以将不同模块之间的通信解耦，使它们之间的依赖性降低。模块之间通过发送和接收消息进行通信，而不需要直接调用彼此的函数或知道彼此的内部实现细节。
2. **异步通信**：消息队列支持异步通信模型，发送者和接收者之间的操作可以是非阻塞的。发送任务可以继续执行而不必等待接收任务处理消息，这样可以提高系统的并发性和响应性。
3. **缓解资源竞争**：使用消息队列可以避免多个任务同时访问共享资源而引起的竞争条件。任务通过向消息队列发送消息来共享数据，而不是直接访问共享资源，从而减少了竞争的可能性。
4. **处理并发性**：消息队列允许多个任务同时发送和接收消息，从而支持系统中的并发性。这使得系统能够更有效地利用多核处理器或多线程环境。
5. **缓解通信速度不匹配**：发送者和接收者之间的通信速度可能不匹配，消息队列可以允许发送者和接收者以不同的速度进行操作。消息队列会在发送者和接收者之间提供一个缓冲区，使得消息在被发送者生产和接收者消费之间可以被缓存。
6. **容错和数据传输**：消息队列通常具有容错性，即使接收者不可用或暂时无法处理消息，消息也可以被缓存起来，直到接收者准备好接收。此外，消息队列通常支持不同类型的消息数据，如结构体、指针等，使得可以传输各种类型的数据。



## debug记录

1. UART串口外设在不加互斥锁时, 大量printf请求导致hardfault.

> 若UART的打印函数不加入互斥锁Mutex, 则在大批量打印请求时必定出现卡死的情况, 加入互斥锁后彻底解决。
>
> ``` c
> osMutexDef(printfMutex);
> osMutexId printfMutex;
> 
> void setup(){
>     printfMutex = osMutexCreate(osMutex(printfMutex));	//放于初始化函数中
> }
> 
> void uart3_printf(const char* format, ...) {
>     osMutexWait(printfMutex, portMAX_DELAY);
>     char buffer[64];  // 缓冲区用于存储格式化后的字符串
>     va_list args;
>     va_start(args, format);
> 
>     vsnprintf(buffer, sizeof(buffer), format, args);  // 格式化字符串到缓冲区
>     va_end(args);
> 
>     for (size_t i = 0; buffer[i] != '\0'; ++i) {
>         HAL_UART_Transmit(&huart3, (uint8_t *) &buffer[i], 1, HAL_MAX_DELAY);
>     }
>     osMutexRelease(printfMutex);
> }
> ```

2. 对于数组与数组间的数据传输, 对memcpy的利用

> ``` c
>                     t_JsonPackage *JsonPtr = osPoolAlloc(JsonQ_Mem);
>                     memcpy(JsonPtr->JsonString, uart3Buffer, sizeof(uart3Buffer));
>                     osMessagePut(JsonQueueHandle, (uint32_t)JsonPtr, 0);
> ```
>
> 在上述函数中, 有两个数组 JsonPtr->JsonString(char [128]) 与 uart3Buffer[128] 数组
>
> 无法直接使用类似"JsonPtr->JsonString = uart3Buffer"或""JsonPtr->JsonString = &uart3Buffer[0]""的赋值式
>
> 应使用memcpy(JsonPtr->JsonString, uart3Buffer, sizeof(uart3Buffer));