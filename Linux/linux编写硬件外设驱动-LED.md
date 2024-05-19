# linux编写硬件外设的驱动--LED

在我们的linux驱动编写时, 若是使用了最小核心板/系统板, 这些系统模块通常都有官方的SDK, 将其中的GPIO或其他外设引出.

这样我们可以省去硬件内存映射的步骤, 直接操作其中注册好的设备.



## 知识点

1. **内核态的文件打开/关闭与读写**
2. **虚拟设备到真实设备的映射, 硬件抽象的实现, 便于移植/修改**
3. **体验将多个代码文件共同编译成一个ko内核驱动模块**
4. **使用C语言程序来与用户态的驱动进行交互, 从而控制内核模块**



## 哪里可以找到这些硬件设备

设备文件通常都存储在**/sys/class目录**下

![image-20240406145444697](E:/Typora_note/photos/image-20240406145444697.png)



## 编写LED内核驱动的前置知识

在LED设备的目录下, 有以下分区

![image-20240406175021405](E:/Typora_note/photos/image-20240406175021405.png)

其中brightness代表了当前的亮度, 可读可写, 写入要求为**字符'1'或'0'**.

其中trigger代表了LED的亮灭触发源, 可读可写, 可以通过cat指令来获取可以写入的变量范围.

### 在终端中通过命令行来控制LED

在终端下, 我们可以通过echo指令来为brightness赋值

``` bash
echo 1/0 > /sys/class/leds/led0/brightness
```



### 实现最简单的C语言LED控制程序

对于我们的C语言程序来说, 这brightness也是一种文件而已, 要对其赋值, 自然要用到

1. open函数
2. write函数
3. read函数(如有必要)

**注意: open/write时要在root用户下, 否则没有权限, 导致打开文件失败**

``` c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char** argv){
    int fd;
    char status;

    if (argc != 3) 
	{
		printf("Usage: %s <dev> <on | off>\n", argv[0]);
		return -1;
	}

    fd = open(argv[1], O_RDWR);
    if(fd == -1){
        printf("can not open file %s\r\n", argv[1]);
        return -1;
    }

    if(0 == strcmp(argv[2], "on")){
        status = '1';
        write(fd, &status, 1);
    }else{
        status = '0';
        write(fd, &status, 1);
    }

    close(fd);

    return 0;
}
```

编写完成之后, 进行简单的编译

``` bash
gcc -o led_test led_test.c
```

**如果发现报错:	找不到头文件 \<stddef.h> 可以尝试sudo apt-get upgrade一下, 可能是内核更新导致的. 内核版本可通过uname -r指令查看**

编译完成后, 就可以直接对其进行操作了

![image-20240406180152857](E:/Typora_note/photos/image-20240406180152857.png)

如果没有在root用户或者sudo前缀下直接打开led0, 就会报错无法打开文件

![image-20240406180333223](E:/Typora_note/photos/image-20240406180333223.png)

正常写入则不会有报错提示(不过也没有其他消息提示, 建议结合自己的硬件观察效果, 或者加一个read函数)

![image-20240406180440142](E:/Typora_note/photos/image-20240406180440142.png)



## 开始编写LED内核驱动模块

类似地, 我们内核驱动模块的关键点依旧在于file_operation中的成员函数的实现

### LED驱动模块 -- 本体

``` c
#include "linux/module.h"
#include "linux/kernel.h"
#include "linux/init.h"

#include "linux/fs.h"
#include "linux/errno.h"
#include <linux/miscdevice.h>
#include <linux/major.h>
#include <linux/mutex.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/stat.h>
#include <linux/init.h>
#include <linux/device.h>
#include <linux/tty.h>
#include <linux/kmod.h>
#include <linux/gfp.h>

#include "led_opr.h"

static int major;   //确定主设备号
static struct class *led_class; //创建led设备
struct led_operations *p_led_opr;

#define MIN(a, b) (a < b ? a : b)

static ssize_t led_drv_read(struct file *file, char __user *buf, size_t size, loff_t *offset){
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	return 0;
}

static ssize_t led_drv_write(struct file *file, const char __user *buf, size_t size, loff_t *offset){
    int err;
    char status;
    struct inode *inode = file_inode(file);
    int minor = iminor(inode);

	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    err = copy_from_user(&status, buf, 1);

    p_led_opr->ctl(minor, status);

	return 1;
}

static int led_drv_open(struct inode *node, struct file *file){
    int minor = iminor(node);
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    
    p_led_opr->init(minor);

    return 0;
}

static int led_drv_close (struct inode *node, struct file *file)
{
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	return 0;
}

struct file_operations led_drv = {
    .owner = THIS_MODULE,
    .open = led_drv_open,
    .release = led_drv_close,
    .read = led_drv_read,
    .write = led_drv_write,
};

static int __init led_init(void){
    int err;
    
    printk("my led driver loading");
    major = register_chrdev(0, "my_led", &led_drv);

    led_class = class_create(THIS_MODULE, "my_led_class");
    err = PTR_ERR(led_class);
    if(IS_ERR(led_class)){
        printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
        printk("led class create failure\n");
        unregister_chrdev(major, "my_led");
        return -1;
    }

    p_led_opr = get_board_led_opr();

    device_create(led_class, NULL, MKDEV(major, 0), NULL, "my_led0");

    return 0;
}

static void __exit led_exit(void){
    printk("my led driver removing\n");

    device_destroy(led_class, MKDEV(major, 0));
    class_destroy(led_class);
    unregister_chrdev(major, "my_led");
}

module_init(led_init);
module_exit(led_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("NinoC137");
MODULE_DESCRIPTION("My led driver!");
```

### LED驱动模块 -- 驱动对象的定义

可以看到, 在驱动程序中, 有如下代码

``` c
#include "led_opr.h"
struct led_operations *p_led_opr;
.....
    p_led_opr->ctl(minor, status);
.....
    p_led_opr->init(minor);
.....
    p_led_opr = get_board_led_opr();
.....
```

是因为在驱动模块中, 我们进行了简单的抽象.

目的是将我们虚拟的软件驱动模块的"读/写/初始化"映射到真实的硬件中.(当然我这里也没有直接映射到物理内存上面, 而是映射给了SDK中已有的设备)



**驱动对象抽象层代码如下:**

1. **led_opr.h**

``` c
#ifndef _LED_OPR_H
#define _LED_OPR_H

struct led_operations
{
    int (*init) (int which);
    int (*ctl) (int which, char status);
};

struct led_operations *get_board_led_opr(void);

#endif
```

2. **board_demo.c**

``` c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#include <linux/errno.h>
#include <linux/miscdevice.h>
#include <linux/kernel.h>
#include <linux/major.h>
#include <linux/mutex.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/stat.h>
#include <linux/init.h>
#include <linux/device.h>
#include <linux/tty.h>
#include <linux/kmod.h>
#include <linux/gfp.h>
#include "led_opr.h"

#define LED_FILE "/sys/class/leds/led0/brightness"

static int board_demo_led_init(int which){
    printk("%s %s line %d, led %d\n", __FILE__, __FUNCTION__, __LINE__, which);
	return 0;
}

static int board_demo_led_ctl(int which, char status){
    struct file *filp;
    char value;

    filp = filp_open(LED_FILE, O_WRONLY, 0);
    if(IS_ERR(filp)){
        printk("falied to open led0\n");
        return PTR_ERR(filp);
    }

    value = status;
    kernel_write(filp, &value, 1, 0);

    printk("%s %s line %d, led %d, %c\n", __FILE__, __FUNCTION__, __LINE__, which, status);
    filp_close(filp, NULL);
	return 0;
}

static struct led_operations board_demo_led_opr = {
	.init = board_demo_led_init,
	.ctl  = board_demo_led_ctl,
};

struct led_operations *get_board_led_opr(void)
{
	return &board_demo_led_opr;
}
```

虽说我在形参中声明了which变量, 但其实我目前只控制了LED0, 所以暂时没有使用这个变量.



通过这种硬件抽象的方法, 若用户需要将这里的LED0改为直接控制硬件上的LED, 也仅需修改

``` c
static int board_demo_led_ctl(int which, char status);
```

这个函数即可, 不需要重新编写驱动部分的代码.



**注意:**

​	内核态中无法使用用户态中的open/read/write等函数, 取而代之的是filp_open, kernel_write, filp_close等等函数



#### 函数解析: filp_open

Stack Overflow及诸多linux论坛都指出了这个函数并不安全, 建议对其进行安全封装后再使用.

> `filp_open` 函数是 Linux 内核中用于打开文件并返回对应文件结构的函数。
>
> ### 函数签名
> ```c
> struct file *filp_open(const char *filename, int flags, umode_t mode)
> ```
>
> ### 参数
> - `const char *filename`：要打开的文件的路径名。
> - `int flags`：打开文件的标志，例如 `O_RDONLY`、`O_WRONLY`、`O_RDWR` 等。
> - `umode_t mode`：文件的权限模式，例如 `S_IRUSR`、`S_IWUSR` 等。
>
> ### 返回值
> - 如果成功打开文件，则返回指向表示该文件的文件结构的指针。
> - 如果出现错误，则返回错误码，并且文件结构指针为 NULL。
>
> ### 功能
> - `filp_open` 函数用于在内核中打开一个文件，并返回对应的文件结构指针。
> - 它负责分配并初始化文件结构体，以便后续对文件进行读写操作。
>
> ### 注意事项
> - 在调用此函数之前，应该确保有足够的权限打开指定的文件。
> - 在使用完返回的文件结构指针后，应该调用相应的关闭函数进行资源释放，以避免资源泄漏。
>
> ### 注意
> - `filp_open` 函数是内核级别的函数，通常在驱动程序或者其他内核模块中使用，而不是在用户空间的应用程序中。
> - 在使用完返回的文件结构指针后，应该调用相应的关闭函数进行资源释放，以避免资源泄漏。
> - 在 Linux 内核中，文件结构表示打开的文件的抽象，而不是实际的文件描述符。因此，与 `filp_open` 函数配套使用的是内核级别的关闭函数 `filp_close`，而不是 `close` 等用户空间的函数。

#### 函数解析: filp_close

> ### 函数签名
>
> ```c
> void filp_close(struct file *file, fl_owner_t id)
> ```
>
> ### 参数
>
> - `struct file *file`：指向表示要关闭的文件的文件结构的指针。
> - `fl_owner_t id`：文件的所有者 ID。
>
> ### 功能
>
> - `filp_close` 函数用于关闭文件描述符对应的文件。
> - 它释放文件结构体和相关的数据结构，从而释放文件描述符所占用的资源。
>
> ### 注意事项
>
> - 在调用此函数之前，通常需要确保文件描述符已经被打开，否则可能会导致不可预测的行为。
> - 在关闭文件之前，通常应该确保所有相关的 I/O 操作都已经完成，以免丢失数据或者造成数据不一致。



##### 示例代码

```c
#include <linux/fs.h>

const char *filename = "/path/to/your/file.txt";
int flags = O_RDONLY; // 以只读方式打开文件
umode_t mode = S_IRUSR | S_IRGRP | S_IROTH; // 设置文件权限为只读

struct file *filp;

// 打开文件
filp = filp_open(filename, flags, mode);
if (IS_ERR(filp)) {
    // 打开文件失败
    printk(KERN_ERR "Failed to open file\n");
    return PTR_ERR(filp);
}

// 在此处进行文件操作...

// 关闭文件
filp_close(filp, current->files);
```

#### 函数解析: kernel_write

> `kernel_write` 函数是 Linux 内核中用于在内核空间中向文件写入数据的函数。它提供了一种方式，在内核中进行文件写操作而不需要通过用户空间。
>
> ### 函数签名
> ```c
> ssize_t kernel_write(struct file *file, const void *buf, size_t count, loff_t *pos)
> ```
>
> ### 参数
> - `struct file *file`：表示要写入数据的文件的文件结构指针。
> - `const void *buf`：指向包含要写入数据的缓冲区的指针。
> - `size_t count`：要写入的字节数。
> - `loff_t *pos`：指向写入位置的偏移量的指针。
>
> ### 返回值
> - 如果成功写入数据，则返回写入的字节数。
> - 如果出现错误，则返回一个负数，并且设置相应的错误码。
>
> ### 功能
> - `kernel_write` 函数用于在内核空间中向文件写入数据。
> - 它是对内核中常用的 `file->f_op->write` 操作的封装，可以直接在内核中进行文件写入操作。
>
> ### 注意事项
> - 在调用此函数之前，应该确保文件已经被成功打开，并且有足够的权限进行写操作。
> - 写入的数据会从指定的偏移量开始写入，然后偏移量会根据写入的字节数进行更新。
> - 写入的数据来源于指定的缓冲区，因此在调用该函数之前需要确保缓冲区的有效性和数据的正确性。
>
> ### 注意
> - `kernel_write` 函数是内核级别的函数，通常在驱动程序或者其他内核模块中使用，而不是在用户空间的应用程序中。
> - 在使用此函数时，应确保对文件的写入操作不会导致数据丢失或损坏，以避免系统不稳定性。
> - 由于 `kernel_write` 函数在内核空间执行，因此在写入文件时不会触发任何用户空间的权限检查或者文件系统的回调函数。

##### 示例代码

``` c
#include <linux/fs.h>

struct file *filp;
const char *data = "Hello, world!";
loff_t offset = 0;

// 假设 filp 是一个已经打开的文件描述符对应的文件结构指针

// 写入数据
ssize_t bytes_written = kernel_write(filp, data, strlen(data), &offset);
if (bytes_written < 0) {
    // 写入数据失败
    printk(KERN_ERR "Failed to write to file\n");
    return bytes_written;
}
```

### Makefile文件编写

由于我们的驱动模块的源码是由两部分组成的, 所以我在makefile中加入了中间件**my_led_driver**

``` makefile
my_led_driver-y := led_driver.o board_demo.o #将led_driver与board_demo共同编译成一个my_led_driver
obj-m := my_led_driver.o

all:
	sudo make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
	gcc -o led_test led_test.c
clean:
	sudo make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```



## 运行效果展示

编译完成后, 文件树如下所示. 关键文件在于.ko文件与led_test可执行文件

![image-20240406183134111](E:/Typora_note/photos/image-20240406183134111.png)

### 插入模块并查看其info

![image-20240406183254076](E:/Typora_note/photos/image-20240406183254076.png)

### 在/sys目录及/dev目录下找到我们模块所创立的虚拟设备

1. sys目录下查看

![image-20240406183454161](E:/Typora_note/photos/image-20240406183454161.png)

2. /dev目录下查看

![image-20240406183530679](E:/Typora_note/photos/image-20240406183530679.png)

### 测试驱动模块的功能

![image-20240406183659392](E:/Typora_note/photos/image-20240406183659392.png)

可以看到, 虽然我们**在程序中控制的是我们使用驱动模块所创建的虚拟设备**, 

但是它也已经成功由我们的驱动模块的内部函数, 将**控制指令映射到了树莓派板载的LED0上面**.



进入root模式后, 使用dmesg查看内核信息

![image-20240406183909965](E:/Typora_note/photos/image-20240406183909965.png)

可以看出, 我们的驱动模块也有在被正确地调用