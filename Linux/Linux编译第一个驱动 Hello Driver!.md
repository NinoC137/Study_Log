# Linux编译第一个驱动 Hello Driver!

## 什么是linux驱动

当你编写一个Linux设备驱动程序时，最终会生成一个文件，其文件扩展名为“.ko”，这个文件是可加载内核模块（Loadable Kernel Module，简称LKM）。下面我将详细解释一下：

1. **可加载内核模块（LKM）**：
   - Linux内核的一大特性就是可以动态地加载和卸载模块，这些模块通常是设备驱动、文件系统或其他内核功能。.ko文件就是这些模块的二进制形式。
   - LKM允许你在运行的Linux系统上添加新的功能和驱动，而无需重新编译整个内核。

2. **编写设备驱动**：
   - 设备驱动程序通常由C语言编写，使用Linux内核提供的API和数据结构。这些驱动程序负责与硬件设备通信，将其抽象为Linux内核可以理解的接口。
   - 设备驱动程序的主要功能包括初始化设备、响应设备中断、向用户空间提供接口等。

3. **编译为模块**：
   - 设备驱动程序可以编译为两种形式之一：编译进内核（built-in）或编译为可加载模块（.ko文件）。
   - 编译为模块的优点是它们在需要时可以动态加载和卸载，而不会增加内核的大小。

4. **加载和卸载模块**：
   - 使用insmod命令加载模块，使用rmmod命令卸载模块。
   - 模块加载时，Linux内核会将其加载到内核空间并执行初始化操作；模块卸载时，相应的资源会被释放。

5. **模块的生命周期**：
   - 一旦加载，模块会一直驻留在内核中，直到被卸载。
   - 模块可能会被其他内核模块或用户空间程序使用，或者与硬件设备进行交互。

总的来说，.ko文件是Linux设备驱动程序编译生成的二进制文件，它代表了一个可加载的内核模块，允许在运行的Linux系统上动态添加或移除设备驱动功能。



## 实现最简单的内核驱动程序hello_driver

1. **在系统中任意一个位置创建一个文件夹, 用于存放我们的hello_driver.c与Makefile文件**

> ![image-20240404181857610](E:/Typora_note/photos/image-20240404181857610.png)

2. **开始编写hello_driver.c文件**

   1. 首先需要加入必须的linux系统头文件	

   ``` c
   #include <linux/module.h>
   #include <linux/kernel.h>
   #include <linux/init.h>
   ```

   2. "module.h" 与"kernel.h"在linux内核模块中是必须的.

      "init.h"在module的初始化与退出时需要被调用.

   3. 接下来编写驱动函数, 模块的载入与卸载都需要一个函数句柄, 所以必须要有_init函数与 _exit函数

      ``` c
      static int __init hello_init(void) {
           printk(KERN_INFO "Hello World!\n");
           return 0;
      }
      
      static void __exit hello_exit(void) {
          printk(KERN_INFO "Goodbye World!\n");
      }
      ```

      由于我们需要将模块是否载入与卸载的信息打印出来, 所以此处用到了printk函数, 即将字符串在kernel log中打印出来, 并且这个log的等级是INFO等级.

      这样, 我们就可以通过dmesg指令来查看模块是否被正确载入或卸载.

   4. 最后, 我们要将模块的装载与卸载函数句柄传入

   ``` c
   module_init(hello_init);
   module_exit(hello_exit);
   ```

   5. 如果这是一个正式的模块, 我们也可以为其添加一些模块信息, 如下

   ``` c
   MODULE_LICENSE("GPL");
   MODULE_AUTHOR("NinoC137");
   MODULE_DESCRIPTION("Hello World Linux Device Driver");
   ```

   6. **完整代码如下:**

      ``` c
      #include <linux/module.h>
      #include <linux/kernel.h>
      #include <linux/init.h>
      
      static int __init hello_init(void) {
           printk(KERN_INFO "Hello Driver!\n");
           return 0;
      }
      
      static void __exit hello_exit(void) {
          printk(KERN_INFO "Goodbye Driver!\n");
      }
      
      module_init(hello_init);
      module_exit(hello_exit);
      
      MODULE_LICENSE("GPL");
      MODULE_AUTHOR("NinoC137");
      MODULE_DESCRIPTION("Hello World Linux Device Driver");
      ```

      

3. **开始编写Makefile文件**

这里我直接在对应linux平台本地环境下开发, 所以不需要配置交叉编译的环境, Makefile便可以直接如下写出

``` makefile
obj-m := hello_driver.o

all: hello_driver.c
        sudo make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
        sudo make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

4. **开始编译**

![image-20240404181924660](E:/Typora_note/photos/image-20240404181924660.png)

正常情况下不会出现报错.



## 测试内核驱动程序Hello_Driver

经过刚才的准备工作, 在编译之后, 文件夹之中就会有如下文件

![image-20240404182016466](E:/Typora_note/photos/image-20240404182016466.png)

开始测试驱动程序, 这里需要用到**insmod与rmmod**指令, 即装载/卸载内核驱动模块

![image-20240404182113569](E:/Typora_note/photos/image-20240404182113569.png)

正常运行下是不会直接打印出信息的, 所以我们需要用到**sudo dmesg**命令来查看kernel log

![image-20240404182156937](E:/Typora_note/photos/image-20240404182156937.png)

可见, 驱动被正确地装载与卸载了.

我们也可以使用**modinfo**命令来查看驱动的信息

![image-20240404182353203](E:/Typora_note/photos/image-20240404182353203.png)

----



## 在编译驱动时报错

### Kernel configuration is invalid

可能会在编译时出现这样的错误

``` sh
ERROR: Kernel configuration is invalid.
         include/generated/autoconf.h or include/config/auto.conf are missing.
         Run 'make oldconfig && make prepare' on kernel src to fix it.
```

**解决方法:**

1. 使用uname -r查看当前的内核版本

   ![image-20240404182652933](E:/Typora_note/photos/image-20240404182652933.png)

2. 进入/lib/modules中对应的内核文件夹的/build文件夹之中

​	![image-20240404182730956](E:/Typora_note/photos/image-20240404182730956.png)

3. 运行make menuconfig, 不需要做任何修改, 点击save后点击load, 然后exit退出即可

![image-20240404182832217](E:/Typora_note/photos/image-20240404182832217.png)

4. 重新编译我们的驱动中的makefile文件, 此时报错消失, 正常编译.
