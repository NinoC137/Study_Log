# linux编译第一个驱动Hello driver进阶版

我的硬件设备:

​	树莓派4B + Ubuntu22.04

---



## 编写驱动模块的代码与测试程序的代码

### 驱动模块代码

1. hello_driverPlus.c

``` c
#include "linux/module.h"
#include "linux/kernel.h"
#include "linux/init.h"

#include "linux/fs.h"
#include "linux/errno.h"
#include "linux/miscdevice.h"
#include <linux/major.h>
#include <linux/mutex.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/stat.h>
#include <linux/device.h>
#include <linux/tty.h>
#include <linux/kmod.h>
#include <linux/gfp.h>

static int major = 0;
static char kernel_buf[1024];
static struct class *hello_class;

#define MIN(a,b) (a < b ? a : b)

static ssize_t hello_drv_read(struct file *file, char __user *buf, size_t size, loff_t *offset){
    int err;
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    err = copy_to_user(buf, kernel_buf, MIN(1024, size));
    return MIN(1024, size);
}

static ssize_t hello_driver_write(struct file *file, const char __user *buf, size_t size, loff_t *offset){
    int err;
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    err = copy_from_user(kernel_buf, buf, MIN(1024, size));
    return MIN(1024, size);
}

static int hello_drv_open(struct inode *node, struct file *file){
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    return 0;
}

static int hello_drv_close(struct inode *node, struct file *file){
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    return 0;
}

static struct file_operations hello_drv = {
    .owner = THIS_MODULE,
    .open = hello_drv_open,
    .release = hello_drv_close,
    .write = hello_driver_write,
    .read = hello_drv_read,
};

static int __init hello_init(void){
    int err;
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    major = register_chrdev(0, "hello", &hello_drv);
    hello_class = class_create(THIS_MODULE, "hello_class");
    err = PTR_ERR(hello_class);
    if(IS_ERR(hello_class)){
        printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
        unregister_chrdev(major, "hello");
        return -1;
    }

    device_create(hello_class, NULL, MKDEV(major, 0), NULL, "hello");

    return 0;
}

static void __exit hello_exit(void){
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    device_destroy(hello_class, MKDEV(major, 0));
    class_destroy(hello_class);
    unregister_chrdev(major, "hello");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("NinoC137");
MODULE_DESCRIPTION("Hello World Linux Device Driver Plus");
```

### 测试模块代码

2. hello_driverTest.c

``` c
#include "sys/types.h"
#include "sys/stat.h"
#include "fcntl.h"
#include "unistd.h"
#include "stdio.h"
#include "string.h"

int main(int argc, char **argv){
    int fd;
    char buf[1024];
    int len;

    if(argc < 2){
        printf("Usage: %s -w <string>\n", argv[0]);
        printf(" %s -r\n", argv[0]);
        return -1;
    }

    fd = open("/dev/hello", O_RDWR);
    if(fd == -1){
        printf("can not open file /dev/hello\n");
        return -1;
    }

    if((0 == strcmp(argv[1], "-w") && argc == 3)){
        len = strlen(argv[2]) + 1;
        len = len < 1024 ? len : 1024;
        write(fd, argv[2], len);
    }
    else{
        len = read(fd, buf, 1024);
        buf[1023] = '\0';
        printf("APP read : %s\n", buf);
    }

    close(fd);

    return 0;
}
```

### makefile代码

3. Makefile

```makefile
obj-m := hello_driverPlus.o

all: hello_driverPlus.c
	sudo make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
	gcc -o hello_driverTest hello_driverTest.c
clean:
	sudo make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

```

## 将模块插入内核

### **用dmesg指令, 检测到成功插入内核模块**

![image-20240405121744391](E:/Typora_note/photos/image-20240405121744391.png)

### **用lsmod指令, 可以在模块列表中查看到我们的模块**

![image-20240405121947181](E:/Typora_note/photos/image-20240405121947181.png)

### **在/dev下也可以查找到我们的字符设备**

![image-20240405122017791](E:/Typora_note/photos/image-20240405122017791.png)

### **在sys/class下也能够找到我们的设备节点链接文件**

![image-20240405122258794](E:/Typora_note/photos/image-20240405122258794.png)

查看设备节点的详细信息

![image-20240405122216738](E:/Typora_note/photos/image-20240405122216738.png)

查看该节点的主次设备号

![image-20240405122351894](E:/Typora_note/photos/image-20240405122351894.png)

---



## 测试该驱动模块功能

刚刚已经编译了我们的test程序 hello_driverTest, 运行它的可执行文件以测试模块功能

![image-20240405122728462](E:/Typora_note/photos/image-20240405122728462.png)

通过dmesg也能够看到我们创建的驱动设备正在响应测试程序

![image-20240405122923990](E:/Typora_note/photos/image-20240405122923990.png)



## 代码分析

### 字符设备的注册

本驱动注册了一个字符型的设备, 通过函数调用

``` c
static inline int register_chrdev(unsigned int major, const char *name,
				  const struct file_operations *fops)
{
	return __register_chrdev(major, 0, 256, name, fops);
}
```

> # NAME
>
> __register_chrdev - create and register a cdev occupying a range of minors
>
> # SYNOPSIS
>
> ``` c
> int __register_chrdev(unsigned int major, unsigned int baseminor, unsigned int count, const char * name, 
>                       const struct file_operations * fops);
> ```
>
> # ARGUMENTS
>
> ***major***
>
> major device number or 0 for dynamic allocation
>
> ***baseminor***
>
> first of the requested range of minor numbers
>
> ***count***
>
> the number of minor numbers required
>
> ***name***
>
> name of this range of devices
>
> ***fops***
>
> file operations associated with this devices
>
> # DESCRIPTION
>
> If *major* == 0 this functions will dynamically allocate a major and return its number.
>
> If *major* > 0 this function will attempt to reserve a device with the given major number and will return zero on success.
>
> Returns a -ve errno on failure.
>
> The name of this device has nothing to do with the name of the device in /dev. It only helps to keep track of the different owners of devices. If your module name has only one type of devices it's ok to use e.g. the name of the module here.

### file_operation 字符设备功能的定义

设备是供给于用户使用的, 所以我们至少要定义其read/write/open/close基本功能.

我们可以在linux源码中找到**fs.h文件**, 其中定义了**结构体file_operation**

``` c
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iopoll)(struct kiocb *kiocb, bool spin);
	int (*iterate) (struct file *, struct dir_context *);
	int (*iterate_shared) (struct file *, struct dir_context *);
	__poll_t (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	unsigned long mmap_supported_flags;
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*setfl)(struct file *, unsigned long);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
	ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
			loff_t, size_t, unsigned int);
	loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
				   struct file *file_out, loff_t pos_out,
				   loff_t len, unsigned int remap_flags);
	int (*fadvise)(struct file *, loff_t, loff_t, int);
} __randomize_layout;
```

其中的成员很多, 在本次简单驱动中, 我们必须要用到的成员如下:

``` c
	struct module *owner;
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	int (*open) (struct inode *, struct file *);
	int (*release) (struct inode *, struct file *);
```

可以看出, 其中共有的形参有 **struct file ***

在用户使用open(), read(), write()等函数时, 对应设备名就会传入这个形参之中, 以供我们操作, 如下所示.

``` c
    int fd;
    fd = open("/dev/hello", O_RDWR);
    len = read(fd, buf, 1024);
    write(fd, argv[2], len);
```

### file_operation 字符设备功能的实现

在了解完毕file_operation的功能之后, 我们就可以着手实现其功能了

``` c
static struct file_operations hello_drv = {
    .owner = THIS_MODULE,
    .open = hello_drv_open,
    .release = hello_drv_close,
    .write = hello_driver_write,
    .read = hello_drv_read,
};

static ssize_t hello_drv_read(struct file *file, char __user *buf, size_t size, loff_t *offset){
    int err;
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    err = copy_to_user(buf, kernel_buf, MIN(1024, size));
    return MIN(1024, size);
}

static ssize_t hello_driver_write(struct file *file, const char __user *buf, size_t size, loff_t *offset){
    int err;
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    err = copy_from_user(kernel_buf, buf, MIN(1024, size));
    return MIN(1024, size);
}

static int hello_drv_open(struct inode *node, struct file *file){
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    return 0;
}

static int hello_drv_close(struct inode *node, struct file *file){
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    return 0;
}
```

可以看到, 我们用到了**copy_to_user与copy_from_user**两个函数

这两个函数是在**内核态与用户态之间传递数据**所需要的函数, 即我们的**驱动程序与应用程序之间传递数据**所需要的函数.

以**copy_to_user**举例 两函数的形参格式相同

> ## SYNOPSIS
>
> ``` c
> unsigned long copy_to_user(void __user *to, const void *from, unsigned long *n);
> ```
>
> ## ARGUMENTS
>
> ***to***
>
> ​	Destination address, in user space.
>
> ***from***
>
> ​	Source address, in kernel space.
>
> ***n***
>
> ​	Number of bytes to copy.
>
> ## CONTEXT
>
> User context only. This function may sleep.
>
> ## DESCRIPTION
>
> Copy data from kernel space to user space.
>
> Returns number of bytes that could not be copied. On success, this will be zero.



### class设备的创建与销毁

在我们的代码中, 在init之中为驱动模块创建了一个class设备, 并在exit中将其销毁

创建class之后, 就会在/sys目录下创建一些目录与文件, 这样linux系统中的app(比如udev, mdev)就可以根据这些目录或文件来创造设备节点.

``` c
static int __init hello_init(void){
    int err;
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    major = register_chrdev(0, "hello", &hello_drv);
    hello_class = class_create(THIS_MODULE, "hello_class");
    err = PTR_ERR(hello_class);
    if(IS_ERR(hello_class)){
        printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
        unregister_chrdev(major, "hello");
        return -1;
    }
    device_create(hello_class, NULL, MKDEV(major, 0), NULL, "hello");
    return 0;
}

static void __exit hello_exit(void){
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    device_destroy(hello_class, MKDEV(major, 0));
    class_destroy(hello_class);
    unregister_chrdev(major, "hello");
}
```

以下代码将会在**/sys/class**目录下创建一个子目录"hello_class"

``` c
hello_class = class_create(THIS_MODULE, "hello_class");
```

以下代码将会在**/sys/class/hello_class**目录下创建一个文件"hello"

``` c
device_create(hello_class, NULL, MKDEV(major, 0), NULL, "hello");
```

并且它的主设备号为major的值



其中的函数主要有两个

1. class_create

``` c
/**
 * class_create - create a struct class structure
 * @owner: pointer to the module that is to "own" this struct class
 * @name: pointer to a string for the name of this class.
 *
 * This is used to create a struct class pointer that can then be used
 * in calls to device_create().
 *
 * Returns &struct class pointer on success, or ERR_PTR() on error.
 *
 * Note, the pointer created here is to be destroyed when finished by
 * making a call to class_destroy().
 */
#define class_create(owner, name)		\
({						\
	static struct lock_class_key __key;	\
	__class_create(owner, name, &__key);	\
})
```

2. class_destroy

``` c
//Note, the pointer to be destroyed must have been created with a call to class_create.
//@cls: pointer to the struct class that is to be destroyed
void class_destroy(struct class * cls);
```



