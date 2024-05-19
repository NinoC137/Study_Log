# ADC专题

## ADC-continous + DMA导致程序卡死的情况

**涉及函数:**

``` c
/**
  * @brief  Enable ADC, start conversion of regular group and transfer result through DMA.
  * @note   Interruptions enabled in this function:
  *         overrun (if applicable), DMA half transfer, DMA transfer complete.
  *         Each of these interruptions has its dedicated callback function.
  * @note   Case of multimode enabled (when multimode feature is available): HAL_ADC_Start_DMA()
  *         is designed for single-ADC mode only. For multimode, the dedicated
  *         HAL_ADCEx_MultiModeStart_DMA() function must be used.
  * @param hadc ADC handle
  * @param pData Destination Buffer address.
  * @param Length Number of data to be transferred from ADC peripheral to memory
  * @retval HAL status.
  */
HAL_ADC_Start_DMA(ADC_HandleTypeDef* hadc, uint32_t* pData, uint32_t Length)
```

其中的Length代表了 从ADC寄存器转入到内存中的数据长度.

例如: 

1. 将ADC配置为8bit模式, 那么Length应该为1Byte的倍数(单个rank, 若**多个rank的话要同步*rank数量**).
2. 将ADC配置为12bit或更高, 则Length应该为2Byte的倍数(单个rank), 更高位的ADC配置则同理.

若此时, **Length的数据不够长**, 如Length=2或4等,且**ADC的读取速度又过快**(即ADC的预分频不够高, ADC完成一次的速度太快), 则很容易出现"代码卡死"的情况.

**此时进入Debug模式, 会发现程序经常停在DMA相关的函数之中**.



**而造成这一现象的原因为:**

ADC完成读取的速度过快, 并且DMA传输一次的速度也非常快, 同时又由于我们将其配置为了连续模式, 所以就会导致程序进入

**"完成ADC读取+DMA传输一次  ---> CPU开启DMA传输 ---> 完成ADC读取+DMA传输一次"**这样的死循环之中, 占用大量CPU的带宽, 导致程序无法正常执行.



**解决方法:**

1. 增大ADC的预分频值, 即降低单次ADC的读取速度, 以此减慢DMA传输完成的频率.(不建议)
2. 增大Length的值, 即加大单词DMA传输的数据量, 同样减少DMA传输完成的频率.