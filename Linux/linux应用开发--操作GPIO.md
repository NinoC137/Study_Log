# linux应用开发--操作GPIO

型号为RV1106

LuckFox教程:

[GPIO | LUCKFOX WIKI](https://wiki.luckfox.com/zh/Luckfox-Pico/Luckfox-Pico-GPIO)

## GPIO设备树文件的路径

如果我们不知道当前固件的源SDK编译出的设备树, 可以通过上述指令来定位设备树种的基本描述

``` shell
cat /sys/kernel/debug/gpio
```

![image-20240507111909450](E:/Typora_note/photos/image-20240507111909450-1715052144070-1.png)



如果我们想了解pinctrl设备相关的信息, 也是同样的方法

``` shell
/sys/kernel/debug/pinctrl
```

![image-20240507112315802](E:/Typora_note/photos/image-20240507112315802.png)

其中, pinctrl-maps存储了各个IO的映射信息

![image-20240507112412986](E:/Typora_note/photos/image-20240507112412986.png)

## 确定好pin脚的映射关系, export一个IO口进入GPIO设备

1. 确认好GPIO在gpiochip中的位置

![image-20240507113018320](E:/Typora_note/photos/image-20240507113018320.png)

2. 根据硬件与软件的映射关系, 注册一个IO口

​	先读取出pin脚的配置属性,

![image-20240507113054376](E:/Typora_note/photos/image-20240507113054376.png)

​	可以得到下图的引脚对应关系

![image-20240507113415120](E:/Typora_note/photos/image-20240507113415120.png)

> GPIO{bank}_{X}
>
> GPIO 引脚编号计算公式：pin = bank * 32 + number
> GPIO 组内编号计算公式：number = X
> 综上：pin = bank * 32 + X

## 配置GPIO

![image-20240507115541326](E:/Typora_note/photos/image-20240507115541326.png)