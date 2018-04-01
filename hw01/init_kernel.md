# **OSH_LAB01 实验报告**
+ 姓名：段逸凡
+ 学号：PB16110818
## 实验环境
+ 操作系统：Ubuntu16.04
+ 内核版本：linux-4.15.14
+ gdb:7.11.1
+ qemu:2.5.0
## 正文
### 1.实验目的
1. 熟悉linux操作系统及其相关命令，相关工具的使用
2. 了解qemu，gdb等程序的使用
3. 通过调试内核的启动过程，了解系统启动之后的关键步骤
### 2.实验准备过程
1. 前往linux内核的[**官方网站**](https://www.kernel.org/)
下载最新版本的linux内核源码，本次下载的是linux-4.15.14
2. 安装qemu

    sudo apt-get install qemu

    成功安装qemu2.5.0版本

3. 编译内核
    进入下载了源码的目录，将压缩包解压，进入解压后的文件夹，运行
        
        make menuconfig

    进行内核编译的图形化设置。勾选compile the kernel with debug info,以确保后期调试的正常。
    
        make -j8

    以八核的模式编译内核，等待较长时间后，可见编译成功，此时内核位于/linux-4.15.14/arch/x86/boot/bzImage

### 3.实验过程
#### 1. 使用qemu加载内核

    ```shell
    qemu-system-x86_64 \
    -kernel /linux-4.15.14/arch/x86/boot/bzImage \ #选择编译好的内核
    -initrd \home\dyf0202\init_kernel/roptfs.img \ #选择制作好的根文件系统
    -s \ #启动gdbserver，效果等同于-gdb tcp::1234
    -S \ #启动内核时停止，等待调试器命令
    -append nokaslr \ #使得gdb可以加断点
    ```
    