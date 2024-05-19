# linux驱动开发 --- 导入并控制一个GPIO

## 导入对应GPIO设备

在树莓派SDK中，GPIO外设被分为了两个chip，即**gpiochip0**与**gpiochip504**

**在/sys/class/gpio目录下可以看到**

![image-20240409161708983](E:/Typora_note/photos/image-20240409161708983.png)

其中，gpiochip0对应的是主板上的通用GPIO引脚，而gpiochip504对应的是SPI总线。

## 利于第三方库来查看/控制GPIO

这里我们需要**用到wiringpi第三方库**

如果直接在系统中下载wiringPi，则会报错找不到目标板，因为官方的wiringPi已经太久没有更新，树莓派4B是无法直接使用这个库的

### 下载wiringPi三方库

> ```bash
> git clone https://github.com/WiringPi/WiringPi.git
> cd ~/wiringPi
> ./build
> ```

### 使用三方库查看引脚状态

通过**gpio readall**指令可以查看到，树莓派目前各个引脚的状态

![image-20240409161945366](E:/Typora_note/photos/image-20240409161945366.png)

## 控制GPIO引脚的前置条件

如果我们想要直接在驱动中对一个GPIO进行控制, 那么我们至少先要导入(export)它, 这里以GPIO25为例.**注意:这里的GPIO Index是BCM的编号**

### 导入一个引脚

(在root用户模式下, 否则权限不足)

``` bash
echo 25 > /sys/class/gpio/export 
```

同理, 当我们完成操作后, 重复此步骤于unexport即可解除对应引脚.

![image-20240409162446617](E:/Typora_note/photos/image-20240409162446617.png)

### 查看GPIO状态

![image-20240409162537293](E:/Typora_note/photos/image-20240409162537293.png)

在导入完成之后，我们便可以打开对应文件夹来查看信息

对于我们GPIO操作而言，最关键的部分在于下列几个参数

1. active_low	---	电平有效值. 如果设置为1, 则GPIO为低电平有效, 设置为0则高电平有效
2. direction --- 引脚方向. 写入"in"则代表输入模式, 写入"out"则代表输出模式
3. edge --- 当引脚在输入模式时, 我们可以通过此选项来配置"引脚状态变化事件"的触发条件. 可以写入"none", "rising", "falling"或"both"
4. value --- 引脚为输出模式时, 可以写入"0"或"1"来配置信号电平; 引脚为输入模式时, 此文件可以读取出当前引脚电平
5. uevent --- 此文件包含GPIO设备的事件信息, 如设备的名称, 子系统和设备类型.

我们可以通过**cat指令**来获取其内部参数

![image-20240409162719180](E:/Typora_note/photos/image-20240409162719180.png)

### 通过终端来配置、控制GPIO

此处以配置GPIO25为输出模式, 并输出高电平1为例.

1. 设置引脚方向

   ``` bash
   echo out > direction
   ```

2. 为引脚赋值

   ``` bash
   echo 1 > value
   ```

效果如下所示:

![image-20240409163412452](E:/Typora_note/photos/image-20240409163412452.png)

![image-20240409163459156](E:/Typora_note/photos/image-20240409163459156.png)

可以看出，我们已经成功配置了这个GPIO的引脚方向与电平值。

## 通过linux驱动程序来控制GPIO

