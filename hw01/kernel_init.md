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
    ![menuconfig.png](https://github.com/OSH-2018/1-yjsx/blob/master/hw01/pic/menuconfig.png?raw=true)
    
        make -j8

    以八核的模式编译内核，等待较长时间后，可见编译成功，此时内核位于/linux-4.15.14/arch/x86/boot/bzImage

### 3.实验过程
#### 1. 使用qemu加载内核

    
    qemu-system-x86_64 \
    -kernel /linux-4.15.14/arch/x86/boot/bzImage \ #选择编译好的内核
    -initrd rootfs.img \ #选择制作好的根文件系统
    -s \ #启动gdbserver，效果等同于-gdb tcp::1234
    -S \ #启动内核时停止，等待调试器命令
    -append nokaslr \ #使得gdb可以加断点
    
**注**：
1. 起初并没有制作根文件系统，而是直接在内存中运行内核，后来遇到了 ***VFS:Unable to mount root fs on Unknown-block(0,0)*** 的问题,使得内核无法继续运行，后来通过指定根文件系统解决
2. 最开始的时候自作聪明，将bzImage与rootfs.img拷贝到一个文件夹内，以图操作省事，结果后期调试的时候，gdb遇到了 ***no source available*** 的问题，无法读取内核的源代码，很坑。。。

#### 2. gdb启动
最初时通过常规命令

    target remote:1234
    file vmlinux
    b start_kernel
    c

来进行调试，结果遇到了 ***Remote 'g' packet reply is too long*** 的问题，经过查阅资料，在stackoverflow的[这个问题](https://stackoverflow.com/questions/48620622/how-to-solve-qemu-gdb-debug-error-remote-g-packet-reply-is-too-long/49348616#49348616)中找到了解决方法.

    gdb \
    -ex "add-auto-load-safe-path $(pwd)" \
    -ex "file vmlinux" \
    -ex 'set arch i386:x86-64:intel' \
    -ex 'target remote localhost:1234' \
    -ex 'break start_kernel' \
    -ex 'continue' \
    -ex 'disconnect' \
    -ex 'set arch i386:x86-64' \
    -ex 'target remote localhost:1234'

运行上述代码后，内核运行到了start_kernel函数处.

#### 3.追踪内核启动步骤
在查阅资料的时候，碰巧看到了这张描述了内核初始化的过程图
![kernel_init.png](http://images.51cto.com/files/uploadimg/20100723/103833643.jpg)
我便以这个为基础，分析一下linux内核初始化的过程。

1. 此时内核处于start_kernel处
![start_kernel.png](https://github.com/OSH-2018/1-yjsx/blob/master/hw01/pic/start_kernel.png?raw=true)
通过查看源码，凭直觉选择了几个看起来重要的函数，一一查阅其功能得到下表

|函数名称    |功能        |
|:----------:|--------|
|boot_cpu_init   |激活当前cpu|
|group_init_early|数据结构及其链表的初始化|
|tick_init|时钟初始化|
|boot_cpu_init|激活cpu|
|trap_init|中断初始化|
|mm_init|内存初始化|
|sched_init|调度初始化|
|console_init|控制台初始化|
|rest_init|剩余的初始化，进入下一个大函数|


本着眼见为实的原则，我对printk（）函数产生了好奇，所以我给prink设置了断点，可发觉并没有输出出现在屏幕上，经过查阅[资料](https://blog.csdn.net/groundhappy/article/details/54666800)，得知consle_init的具体作用。其功能为通过调用函数，初始化univ8250_console设备的端口,并注册该设备,然后通过printk->vprintk_emit->console_unlock->call_console_drivers->con->write(con, text, len)才实现了输出的功能。 
但是经过测试，我在初始化univ8250_console的函数univ8250_console_init前增加断点，可以看见，在并没有调用这个函数的时候，屏幕上已有输出。为探究，究竟是哪个函数最终导致了屏幕的输出，我在与之相关的函数前增设断点：
+ console_init
+ univ8250_console_init
+ register_console
不过经过了长达好久的查找，并没有找到这个关键的函数。。。
通过查看堆栈，估计是在某个函数调用register_console之前完成的这项任务。
![stack.png](https://github.com/OSH-2018/1-yjsx/blob/master/hw01/pic/stack.png?raw=true)

2. rest_init
名称叫做rest_init,意为剩下的初始化，但实际上其最重要的使命时启动kernel_init

    kernel_thread(kernel_init, NULL, CLONE_FS)

其通过调用如上的函数，启动了kernel_init的第一个线程,为内核一号进程.

3. kernel_init
这个函数将完成所有驱动程序的初始化

    do_basic_setup

上方的函数为其最关键的函数，此时与体系结构相关的部分已经初始化完成，现在开始调用do_basic_setup()函数初始化设备，完成外设及其驱动程序（直接编译进内核的模块）的加载和初始化

4. init_post
在这初始化的最后一个函数中，其完成了用户空间的初始化，并将系统状态由内核态进入用户态
其中的run_init_process函数内部完成了内核态到用户态的切换
查看虚拟中的时间，在1.89s的时候，内核开始调用run_init_process函数，而在3.27s的时候系统才完全启动完成，可见系统在内核态与用户态之间切换的时间开销之大。
![time.png](https://github.com/OSH-2018/1-yjsx/blob/master/hw01/pic/time.png?raw=true)


##结语
通过本次实验，我大概熟悉了linux下的一些命令，也大概知道了系统内核在启动时的一些关键步骤，通过阅览源代码，我不由得感触到了计算机科学的博大精深，也认识到了要想完成一个成型的大型代码是多么的不容易，最后希望自己以后的实验不要赶ddl。
