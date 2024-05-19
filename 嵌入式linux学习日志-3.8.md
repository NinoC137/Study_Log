# 串口中断接收+数据包抓取

在本工程中, 实现了利用串口中断来抓取字符流, 并对字符流中的最新数据包进行抓取的功能.

**示例格式为Json数据串的开头' {' 与结尾 '}'**



## 效果展示

1. 发送单次数据包{create  by Nino}

   ![image-20240308170622586](E:/Typora_note/photos/image-20240308170622586.png)

2. 发送多个数据包{create by Nino}{2024}{03.08}

   ![image-20240308170635896](E:/Typora_note/photos/image-20240308170635896.png)

3. 夹杂无关数据123{create by Nino}456{2024}789{03.08}

![image-20240308170649868](E:/Typora_note/photos/image-20240308170649868.png)



可以看出, 每次都能准确获取到最新的数据包, 效果稳定.



## 如何使用? How to use?

将以下代码复制进main.c或其他函数之中.

**注意:**

​	要注意代码存放的位置能正确覆写原本的__weak函数, 否则可能会出现无法正常调用自己写的回调函数的情况.



``` c
/**
 * @file weakFunctionRewrite.c
 * @author NinoC137
 * @brief 替换stm32 hal库中的串口接收中断weak函数
 *        建议放在main.c或it.c文件之中, 防止无法重写原有的weak函数
 * 
 * @version 0.1
 * @date 2024-03-08
 * 
 * @copyright Copyright (c) 2024
 * 
 */

#define UART_STARTFLAG '{'
#define UART_ENDFLAG '}'

char uart3Buffer[128];

int dataPackageUpdate;

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
                    dataPackageUpdate = 1;
                }
                break;
            default:
                break;
        }
        HAL_UART_Receive_IT(&huart3, (uint8_t*)&uart3Buffer[uart3_RxFlag], 1);
    }
}
```



## 函数解析

在本项目中, 关键点在于:

1. 如何正确获取包头与包尾
2. 如何准确抓取包头与包尾中的数据
3. 如何处理包头与包尾之外的无关数据



在这里, 我采用了简易的状态机模式.



**判断新数据是否是包头 --> **

​						**不是包头 ---> 清空缓冲区, 数据作废**

​						**是包头 ---> index自增, 切换至下一状态, 准备接收包尾**

``` c
            case 0:
                if(uart3Buffer[uart3_RxFlag] == UART_STARTFLAG){
                    uart3_RxFlag++;
                    uart3_RxStatus = 1;
                }else{
                    memset(uart3Buffer, 0, sizeof(uart3Buffer));
                }
                break;
```

**判断新数据是否为包尾 --->**

​						**不是包尾 ---> index自增, 继续接收新字符**

​						**是包尾---> index归零, 切换至下一状态, 准备接收包头, 同时置1数据包更新标志位, 并清空包尾以后的数据, 防止无关数据干扰**	

``` c
            case 1:
                if(uart3Buffer[uart3_RxFlag] != UART_ENDFLAG){
                    uart3_RxFlag++;
                }else{
                    memset(&uart3Buffer[uart3_RxFlag + 1], 0, sizeof(uart3Buffer) - uart3_RxFlag);
                    uart3_RxFlag = 0;
                    uart3_RxStatus = 0;
                    dataPackageUpdate = 1;
                }
                break;
```

​	
