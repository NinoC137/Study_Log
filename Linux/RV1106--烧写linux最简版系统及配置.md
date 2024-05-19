# RV1106 烧写linux最简版系统及配置

硬件: sololinker

芯片: RV1106 存储介质: SD卡

## 烧写linux系统

1. 下载SDK

2. 编译出linux系统

   ![image-20240505172713523](E:/Typora_note/photos/image-20240505172713523.png)

3. 下载瑞芯微驱动程序、瑞芯微SocTool镜像烧录程序，并安装其中的驱动。

![image-20240505172646941](E:/Typora_note/photos/image-20240505172646941.png)

4. 使用SocTool镜像烧录程序将linux镜像烧录进SD卡

   打开后选择RV1106芯片

![image-20240505172811035](E:/Typora_note/photos/image-20240505172811035.png)

在“SD卡工具”选项里，选中SD卡的读卡器USB磁盘，然后选择好我们刚才使用SDK编译出的linux镜像文件，最终创建SD即可。

![image-20240505172936794](E:/Typora_note/photos/image-20240505172936794.png)



需要注意的是, 我们从SD卡启动的话, 需要完全擦除芯片的flash.

![image-20240505181226491](E:/Typora_note/photos/image-20240505181226491.png)

## linux系统的基本配置

### 串口登录

​	使用串口助手连接芯片的**Debug_UART引脚**，然后使用支持串口的linux远程登录软件即可正常工作。

​	注意，直接使用串口助手来通信可能没有反应。

### WIFI连接

​	使用下列命令来连接WIFI

``` shell
nmcli device wifi connect "WiFi名称" password "密码"
```

​	编译SDK时选择wifi驱动，按指引短接4，6引脚，正常激活wifi功能，然后连接到我们的wifi。

![0ac5a566a817d6efe36aa46fcff2ab0](E:/Typora_note/photos/0ac5a566a817d6efe36aa46fcff2ab0.png)

连接成功后，即可读取出IP地址。

![image-20240505175936147](E:/Typora_note/photos/image-20240505175936147.png)

### 更新linux

​	1. 更新我们的软件包

``` shell
apt update
```

​	2. 安装必备的程序库

``` shell
apt install locales mc dialog
```

- `locate` 是一个用于快速定位文件的命令行工具，通过搜索数据库而不是实时搜索文件系统来实现快速定位。
- `mc` 是 Midnight Commander 的缩写，是一个经典的全屏文件管理器，提供了类似 Norton Commander 的用户界面。
- `dialog` 是一个在命令行界面中创建对话框的工具，可以用来编写交互式的脚本和程序。

**如果有程序包报错无法定位, 大概率是update没有成功, 换个网络重新update, 然后再下载**

**一直报错的话, 检查下名字对不对! 比如locales不是locates**

​	3. 升级程序包

``` shell
apt upgrade
```

4. 重新配置语言

``` shell
dpkg-reconfigure locales
```

输入首字母可以快速跳转(比如中文的zh_CN, 输入Z即可)

按**空格**代表选中, 选中会出现*符号.

5. 恢复成完整版的linux系统.

```shell
 unminimize
```

这里我用的是ubuntu22.04版本

![image-20240505182725794](E:/Typora_note/photos/image-20240505182725794.png)

**解包最小化系统非常重要!否则可能会出现so库缺失, 普通用户无法切换至root用户等各种各样意料之外的情况!**

重新配置语言

``` shell
nano /etc/profile
```

在最下面加入如图的两行定义即可

![image-20240505183205312](E:/Typora_note/photos/image-20240505183205312.png)

### 创建新用户

1. 创建用户

``` shell
adduser yourName
```

**注意:** 名字最好为全小写, 否则会有警告. 如果非要带大写, 可以加参数"--force-badname"

然后使用su yourName指令即可切换过去

2. 添加sudo权限

``` shell
 usermod -aG sudo yourName
```

**解决报错**

1. sudo: /usr/bin/sudo must be owned by uid 0 and have the setuid bit set

> 在root用户下
>
> ``` shell
> chown root:root /usr/bin/sudo
> chmod 4755 /usr/bin/sudo
> ```

2. su: Authentication failure

> 在普通用户下
>
> ``` shell
> sudo passwd root
> ```
>
> 重新配置密码



### 配置静态IP地址

**方法1: nmtui**

	1. 选中一个WiFi, 点击Edit
	1. 将IPv4改为Manual, 配置如下

![image-20240505184128061](E:/Typora_note/photos/image-20240505184128061.png)

reboot之后即可将IP地址分配为预期的固定IP.



## linux性能优化

### 创建swap分区(不成功)

**失败原因: 误删了/dev/mmcblk0p1, 此存储空间为linux的boot程序, 删掉后无法启动**



**这里将disk空间开辟为了swap空间, 会牺牲SD卡寿命, 且读写速度略慢**

用到的命令为 **fdisk**

**在command中, 输入p, 打印硬盘情况**

![image-20240505200247914](E:/Typora_note/photos/image-20240505200247914.png)

**输入n, 开始创建分区**

![image-20240505200559263](E:/Typora_note/photos/image-20240505200559263.png)

有default选项的, 可以直接输入回车

最后视情况来开辟创建swap的内存空间大小

创建成功后, 我们**输入t来将这个内存空间定义为swap空间, 输入Hex code 82即可完成定义**

![image-20240505200725722](E:/Typora_note/photos/image-20240505200725722.png)

重新输入p, 来查看创建的分区情况

![image-20240505202452367](E:/Typora_note/photos/image-20240505202452367.png)

**最后输入w来保存**



## 番外篇: 创建swap分区时误删SD卡中的boot引导文件, 如何在不损害原有数据的情况下重新加载linux内核的引导

**下图为恢复引导之后的效果**

![image-20240505204716926](E:/Typora_note/photos/image-20240505204716926.png)



众所周知, linux系统的boot需要引导文件, 文件从SD卡中读取.

对于没有运行起系统的起始引导过程, 它很显然是不能够先读取一遍SD卡的内存, 然后再在其中寻找引导程序的.



那么, boot文件应该是放在某个固定的地址上, 这样cpu仅需跳转到对应地址, 即可加载引导

在github上, 我们可以大概得知SD卡上, 各文件的存储路径:

![image-20240505205043688](E:/Typora_note/photos/image-20240505205043688.png)

我系统崩溃的原因在于误删了mmcblk1p1和p2, 对应上去, 就应该是idblock文件与uboot文件

所以我们仅需在SocTool中, 将这两样文件重新烧录回去即可, 这两个文件也不影响我们的用户数据

![image-20240505205237312](E:/Typora_note/photos/image-20240505205237312.png)

最终结果便是一开始的图片, 正常的加载了linux系统, 并且没有丢失数据.

