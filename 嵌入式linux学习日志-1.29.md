# 嵌入式linux学习日志



1. Windows环境下, ssh登录时 报错**CreateProcessW failed error:2 posix_spawn: No such file or directory**

> 关闭VPN后再进行连接, 即可正常ssh登录目标机
>
> 怀疑是VPN设置的代理导致的



2. 在linux应用层进行编程时, 注册gpio口时显示资源被占用

>   File "/home/ninobox/LCD/my_LCD.py", line 163, in __ init __
>     writef("/sys/class/gpio/export", "%s" % num)
>   File "/home/ninobox/LCD/my_LCD.py", line 153, in writef
>     with open(file, 'w') as f: f.write(val)
>
> OSError: [Errno 16] Device or resource busy

**解决方案:**

> 使用指令 **cat /sys/kernel/debug/gpio**
>
> 查看gpio的资源占用情况, 发现有如下日志
>
> >  gpio-0   (ID_SDA              )
> >  gpio-1   (ID_SCL              )
> >  gpio-2   (SDA1                )
> >  gpio-3   (SCL1                )
> >  gpio-4   (GPIO_GCLK           )
> >  gpio-5   (GPIO5               )
> >  gpio-6   (GPIO6               )
> >  gpio-7   (SPI_CE1_N           |spi0 CS1            ) out hi ACTIVE LOW
> >  gpio-8   (SPI_CE0_N           |spi0 CS0            ) out hi ACTIVE LOW
> >  gpio-9   (SPI_MISO            )
> >  gpio-10  (SPI_MOSI            |sysfs               ) out lo
> >  gpio-11  (SPI_SCLK            |sysfs               ) out lo
>
> 其中的**gpio-8  spi0 CS0**即为当时的资源被占用的io口
>
> 与我手动申请的gpio-10对比, 可发现 占用此资源的**进程应为sysfs**
>
> 经试错发现, **是我在raspi-config之中开启了SPI接口**, 但是我对这个IO的操作并非是通过raspi-config的SPI-api进行的
>
> 所以出现了资源冲突
>
> **关闭raspi-config的SPI接口, 通过文件操作来进行IO的注册等操作, 即可正常使用**

