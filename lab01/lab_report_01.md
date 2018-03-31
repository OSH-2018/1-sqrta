# Lab01
工具 gdb，qemu<br>
环境 双系统下的Ubuntu 17

## 实验步骤：

### 1.制作根文件系统rootfs.img
先用root身份登录（很重要的一点。。。。）<br>
然后执行命令<br>
git clone https://github.com/mengning/menu.git<br>
cd menu<br>
gcc -Wall -pthread -o init linktable.c menu.c test.c -m32 -static<br>
（这一步一般会有很多编译报错，安装apt install gcc-multilib解决）<br>
cd ../rootfs <br>
(自行建立储存文件夹）<br>
cp ../menu/init ./ <br>
find . | cpio -o -Hnewc |gzip -9 > ../rootfs.img <br>
然后就得到了rootfs.img文件，已在github中上传 <br>

### 2.下载并编译内核，下载的linux内核版本为linux-3.18.23
解压后使用 make menuconfig 配置内核，将kernel hacking中的compiler设置中将compile the kernel with debug info选中
然后make 编译<br>
编译过程可能会发生缺少compiler-gcc7.h（或者其他数字）
找到compiler-gcc5.h在同一个地方复制一份改名即可<br>
还可能缺少各种库比如ncurses等，在网上找相应名字安装<br>

### 3.启动追踪内核
将rootfs.img移动到内核文件夹，安装qemu,gdb<br>
在终端内输入命令<br>
qemu -kernel arch/x86/boot/bzImage -initrd rootfs.img -s -S<br>
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

#### 对start_kernel的分析<br>
  该函数定义在init文件夹下的main.c文件内。一开始定义了两个地址指针kernel_para \__start___param， \__stop___param指向内核启动参数时相关结构在内存中的位置(虚拟内存）。然后调用lockdep_init(); lockdep是一个内核调用模块，用来检查内核互斥潜在的死锁问题。该函数即
  初始化了内核依赖关系的关系表，初始化了hash表，启动了lockdep<br>
  debug_objects_early_init() 对HASH锁和静态对象池进行初始化 <br>
  boot_init_stack_canary() 初始化栈canary值，canary值的是用于防止栈溢出攻击的堆栈的保护字<br>
  cgroup_init_early() 对数据结构和其中链表的初始化<br>
  
  tick_init初始化内核的时钟系统<br>
  
  setup_arch(&command_line)  内核构架的初始化函数,
  其中包含了处理器相关参数的初始化、内核启动参数（tagged list）的获取和前期处理、 
   内存子系统的早期的初始化（bootmem分配器）。 主要完成了4个方面的工作，先是取得MACHINE和PROCESSOR的信息然后将他们赋值 
    给kernel相应的全局变量，然或是对boot_command_line和tags接行解析，再接着就是 
    memory、cach的初始化，最后是为kernel的后续运行请求资源<br>
    
  setup_arch(&command_line);  
 初始化内核本身内 存使用的管理结构体init_mm。  
    2.ps：每一个任务都有一个mm_struct结构以管理内存空间，init_mm是内核的mm_struct，其中：  
    3.设置成员变量* mmap指向自己，意味着内核只有一个内存管理结构;  
    4.设置* pgd=swapper_pg_dir，swapper_pg_dir是内核的页目录(在arm体系结构有16k， 所以init_mm定义了整个kernel的内存空间)。  
    5.这些内容涉及到内存管理子系统<br>


### 2.rest_init()函数的执行
该函数是内核初始化的最后一步，通过kernel_thread创建1号进程init，init进程是所有其他用户进程的祖先<br>

  

rest_init();
创建了第一个用户进程init，并且定义了整形pid作为进程的序号<br>

