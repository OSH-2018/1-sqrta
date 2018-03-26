# Lab01
工具 gdb，qemu<br>
环境 双系统下的Ubuntu 17

## 实验步骤：

### 1.制作根文件系统rootfs.img
先用root身份登录（很重要的一点。。。。）<br>
然后执行命令<br>
git clone https://github.com/mengning/menu.git<br>
cd menu<br>
gcc -o init linktable.c menu.c test.c -m32 -static –lpthread<br>
（这一步一般会有很多编译报错，安装gcc-multillib解决）<br>
cd ../rootfs <br>
cp ../menu/init ./ <br>
find . | cpio -o -Hnewc |gzip -9 > ../rootfs.img <br>
然后就得到了rootfs.img文件，已在github中上传 <br>

### 2.下载并编译内核，下载的linux内核版本为linux-3.18.23
解压后使用 make menuconfig 配置内核，将kernel hacking中的compile the kernel with debug info选中
然后make 编译

### 3.启动追踪内核
将rootfs.img移动到内核文件夹，安装qemu,gdb<br>
在终端内输入命令<br>
qemu -kernel linux-3.18.6/arch/x86/boot/bzImage -initrd rootfs.img -s -S<br>
会得到qemu启动并且冻结在开始状态<br>
在另一个终端中输入<br>
gdb<br>
file vmlinux<br>
break start_kernel（也可设置其他断点）<br>
target remote:1234<br>
成功将gdb与qemu连接上，按下c可以使操作系统继续运行<br>
由于设置断点会在中途停下来，按c可以继续，list可以查看中间运行过的函数

## 实验结果：
追踪到的重要事件：
### 1. 函数start_kernel()的执行。
该函数是linux转换为平台无关代码阶段执行的第一个c语言函数，<br>意味着硬件平台相关的代码执行完毕，linux内核开始进入初始化阶段。

### 2.set_task_stack_end_magic(&init_task)函数
创建了0号进程，即最终的idle进程，运行在内核态，是系统创建的第一个进程，并且会通过kernel_thread创建1号进程init，init进程是所有其他用户进程的祖先

