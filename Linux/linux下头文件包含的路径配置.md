# linux下vscode头文件包含的路径配置

​	在开发linux应用/驱动, 或ROS时, 有时候vscode会提示找不到路径, 但自行编写makefile之后又可以正常编译运行, 此时就可以确定是vscode的头文件包含路径关系不全面导致的.

## linux系统文件的补全

在vscode中, ctrl+shift+P呼出c/c++配置, edit configuration界面, 找到"包含路径一项"

![image-20240406133120429](E:/Typora_note/photos/image-20240406133120429.png)

在linux驱动编写过程中, 如果想要查找某些函数, vscode没有找到函数定义且man指令无法找到时, 可以使用

``` bash
grep -rn "register_chrdev" /usr/src/linux-raspi-headers-5.15.0-1047/
```

引号中是函数名称(或者部分字段), 最后的参数是linux源文件的路径, 由于是递归寻找, 所以不需要彻底的精确路径.