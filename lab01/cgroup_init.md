# cgroup_init()
在init/main.c的start_kernel函数中会调用cgroup_init_early函数进行cgroup的初始化。对cgroup的初始化有两次，分别是cgroup_init_early和cgroup_init。原因是系统初始阶段需要使用一些 subsystem，先对这一部分进行初始化

了解到cgroup是一种机制，集成了各个进程并且对进程分组，通过各个subsystem完成对进程组使用的资源的分配和限制。比如说想要将
进程的cpu使用率限制在20%，那么创建一个cgroup即可对该进程控制其cpu的使用率<br>

与其相关的subsystem有以下几个：
  
    cpu subsystem：指定进程组能使用的CPU；
    memmory subsystem：指定进程组能使用的内存量，并编写内存使用量报告；
    cpuset subsystem：指定进程组能使用的各CPU和节点集；
    freezer subsystem：可停止进程组状态或启动；
    cpuacct subsystem：对进程组使用的CPU资源编写报告；
    blkio：设置对模块设备的输入输出限制；
    devices：设置进程组能使用的设备；
