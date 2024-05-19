# 制作自己的Linux系统img文件

1. 创建存储镜像文件的文件夹

``` shell
sudo mkdir rootfs-img
sudo mkdir rootfs-img/rootfs
```



2. 使用dd指令, 创建空白镜像文件

``` shell
dd if=/dev/zero of=linux-rootfs.img bs=1M count=1024
/dev/zero：为虚拟盘的名字。
linux-rootfs.img为你创建的镜像文件。
bs=1M
count=1024为此镜像的大小。一般1G的根文件系统很大了，如果担心不够用，也可以直接2048.
```



3. 格式化镜像文件

``` shell
sudo mkfs.ext4 linux-rootfs.img
```



4. 挂载镜像, 并向镜像中拷入我们的文件系统

``` shell
sudo mount linux-rootfs.img rootfs
sudo cp -rfp /* rootfs/
```

此步骤未成功, 报错无法挂载, 搁置一下 -- 2024.5.6 20:36
