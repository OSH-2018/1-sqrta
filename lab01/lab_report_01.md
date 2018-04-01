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
成功过一次之够再次用qemu仿真是有时候会卡住，重新做一次rootfs.img就可以了（我也不知道为什么。。。。）<br>

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

**成功在qemu中启动內核**
![](https://github.com/OSH-2018/1-sqrta/blob/master/lab01/%E9%80%89%E5%8C%BA_001.png)

## 实验结果：
追踪到的重要事件：
### 1. 函数start_kernel()的执行。
该函数是linux转换为平台无关代码阶段执行的第一个c语言函数，<br>意味着硬件平台相关的代码执行完毕，linux内核开始进入初始化阶段。

#### 对start_kernel内部分函数的分析<br>
  该函数定义在init文件夹下的main.c文件内。一开始定义了两个地址指针kernel_para \__start___param， \__stop___param指向内核启动参数时相关结构在内存中的位置(虚拟内存）。然后调用lockdep_init(); lockdep是一个内核调用模块，用来检查内核互斥潜在的死锁问题。该函数即
  初始化了内核依赖关系的关系表，初始化了hash表，启动了lockdep<br>
  
  **debug_objects_early_init()** 对HASH锁和静态对象池进行初始化 <br>
  
  **boot_init_stack_canary()** 初始化栈canary值，canary值的是用于防止栈溢出攻击的堆栈的保护字<br>
  
  **cgroup_init_early()** 对cgroup子系统的数据结构和其中链表的初始化<br>
  
  **tick_init** 初始化内核的时钟系统<br>
  
  **setup_arch(&command_line)**  内核构架的初始化函数,
  其中包含了处理器相关参数的初始化、内核启动参数（tagged list）的获取和前期处理、 
   内存子系统的早期的初始化（bootmem分配器）。 主要完成了4个方面的工作，先是取得MACHINE和PROCESSOR的信息然后将他们赋值 
    给kernel相应的全局变量，然或是对boot_command_line和tags接行解析，再接着就是 
    memory、cach的初始化，最后是为kernel的后续运行请求资源<br>
    
  **mm_init_owner(&init_mm, &init_task)** 
 初始化内核本身内存使用的管理结构体init_mm <br>
 
 **build_all_zonelists(NULL, NULL),page_alloc_init()** 建立系统内存页区链表，然后将内存页初始化<br>
 
**pidhash_init()** 初始化hash表，以便于从进程的PID获得对应的进程描述指针<br>

**mm_init()** 建立了内核的内存分配器<br>

**rcu_init();** 初始化互斥访问机制<br>

**init_IRQ();** 初始化中断向量初始化<br>

**init_timers();** 定时器初始化<br>

**hrtimers_init();** 高精度时钟初始化<br>

**softirq_init();** 软中断初始化<br>

**timekeeping_init();**   初始化资源和普通计时器<br> 

**local_irq_enable();** 设置使能中断<br>

**console_init();** 初始化控制台，在这之后显示printk的内容<br>

start_kernel主要包括了 获取内核启动时的参数并进行处理，内核构架的初始化函数setup_arch(&command_line)，对内存管理功能的各种初始化，以及内核体系的初始化，最后是rest_init函数，这个函数内也包括了许多功能。<br>

### 2.rest_init()函数的执行
该函数是内核初始化的最后一步，通过kernel_thread创建1号进程init，init进程是所有其他用户进程的祖先<br>

#### 对rest_init()函数的分析

源代码<br>
	
	static noinline void __init_refok rest_init(void)
	{

	int pid;
	rcu_scheduler_starting();
	/*
	 * We need to spawn init first so that it obtains pid 1, however
	 * the init task will end up wanting to create kthreads, which, if
	 * we schedule it before we create kthreadd, will OOPS.
	 */
	kernel_thread(kernel_init, NULL, CLONE_FS);
	numa_default_policy();
	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
	rcu_read_lock();
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
	rcu_read_unlock();
	complete(&kthreadd_done);

	/*
	 * The boot idle thread must execute schedule()
	 * at least once to get things moving:
	 */
	init_idle_bootup_task(current);
	schedule_preempt_disabled();
	/* Call into cpu_idle with preempt disabled */
	cpu_startup_entry(CPUHP_ONLINE);
	}

对于kernel_thread(kernel_init, NULL, CLONE_FS)和cpu_idle(); 

kernel_thread中传入的函数kernel_init截取部分代码：

	if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init") ||
	    !try_to_run_init_process("/bin/init") ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;
  	panic("No working init found.  Try passing init= option to kernel. "
	      "See Linux Documentation/init.txt for guidance.");
  
  会尝试四种init方式，该函数定义为：

 	static int try_to_run_init_process(const char *init_filename)
	{	
	int ret;
 
	ret = run_init_process(init_filename);

	if (ret && ret != -ENOENT) {
		pr_err("Starting init: %s exists but couldn't execute it (error %d)\n",
		       init_filename, ret);
	}

	return ret;
	}

  执行了四种init文件，均失败时会给出报错信息。run_init_process就是通过 do_execve()来运行 init 程序，其参数就是要执行的可执行文件名，也就
是这里的 init process 在磁盘上的文件。<br>

然后cpu_idle();  将0号进程设置idle

Linux在start_kernel执行之前的主要执行的都是汇编代码，在他执行后，各种环境初始化后，执行c代码。0号进程是作者手工创建的，它的任务就是在CPU的队列中没有进程的时候一直执行，在有进程的时候切换到新进程，而后被设置为空闲状态。start_kernel最后一部分是第一个用户态进程PID=1的正式生成，就是rest_init(),这个进程是系统的1号进程，这个时候0号进程会被设置成idle进程。1号进程执行，生成系统所需的所有进程，其实就是调用了run_init_process()函数加载文件，生成进程
  


