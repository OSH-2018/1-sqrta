# cgroup_init()
在init/main.c的start_kernel函数中会调用cgroup_init_early函数进行cgroup的初始化。对cgroup的初始化有两次，分别是cgroup_init_early和cgroup_init。原因是系统初始阶段需要使用一些 subsystem，先对这一部分进行初始化

了解到cgroup是一种机制，集成了各个进程并且对进程分组，通过各个subsystem完成对进程组使用的资源的分配和限制。比如说想要将
进程的cpu使用率限制在20%，那么创建一个cgroup即可对该进程控制其cpu的使用率<br>
ls /sys/fs/cgroup/ 指令可以查看cgroup subsystem，
与其相关的subsystem有以下几个：
  
    cpu subsystem：指定进程组能使用的CPU；
    memmory subsystem：指定进程组能使用的内存量，并编写内存使用量报告；
    cpuset subsystem：指定进程组能使用的各CPU和节点集；
    freezer subsystem：可停止进程组状态或启动；
    cpuacct subsystem：对进程组使用的CPU资源编写报告；
    blkio：设置对模块设备的输入输出限制；
    devices：设置进程组能使用的设备；

整个cgroup呈现为一个树状结构，以树状分层的形式组织在一起，树根的顶部为一个top_cgroup，所有的cgroup
通过双向链表和top_group组成一棵树，类似与cpu使用率，内存管理情况等subsystem直接与top_group相连，
以便管理进程组，控制进程组的资源。<br>

其定义的数据结构代码为：


    struct task_struct {
    ......

    #ifdef CONFIG_CGROUPS
    /* Control Group info protected by css_set_lock */
    struct css_set __rcu *cgroups;
    /* cg_list protected by css_set_lock and tsk->alloc_lock */
    struct list_head cg_list;
    #endif

    ......
    }
    
进程需要与cgroup以及subsystem建立联系，关键在于css_set结构体。 
首先，每个进程都可以对应一个css_set结构体： 
进程对应一个task_struct结构体，每个task_struct结构体中有一个css_set类型的成员指针cgroups

cgroups指针指向进程对应的css_set结构体，css_set结构体定义如下

    /*
    * A css_set is a structure holding pointers to a set of
    * cgroup_subsys_state objects. This saves space in the task struct
    * object and speeds up fork()/exit(), since a single inc/dec and a
    * list_add()/del() can bump the reference count on the entire cgroup
    * set for a task.
    */

    struct css_set {

    /* Reference count */
    atomic_t refcount;        //引用计数

    /*
     * List running through all cgroup groups in the same hash
     * slot. Protected by css_set_lock
     */
    struct hlist_node hlist;      //hash表相关

    /*
     * List running through all tasks using this cgroup
     * group. Protected by css_set_lock
     */
    struct list_head tasks;  //与该css_set关联的task链表

    /*
     * List of cgrp_cset_links pointing at cgroups referenced from this
     * css_set.  Protected by css_set_lock.
     */
    struct list_head cgrp_links;  //与该css_set关联的cgrp_cset_links的链表，后面会讲到

    /*
     * Set of subsystem states, one for each subsystem. This array
     * is immutable after creation apart from the init_css_set
     * during subsystem registration (at boot time) and modular subsystem
     * loading/unloading.
     */
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];

    /* For RCU-protected deletion */
    struct rcu_head rcu_head;
    };
    
css_set结构体中struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];<br>
指针数组，保存了对应进程所属于（或者使用）的subsystem的设置信息，可快速浏览已设置的 subsystem。
cgroup_subsys_state的定义如下：

    struct cgroup_subsys_state {
      struct cgroup *cgroup;
      struct cgroup_subsys *ss;
      struct percpu_ref refcnt;
      struct cgroup_subsys_state *parent;
      unsigned long flags;
      struct rcu_head rcu_head;
      struct work_struct destroy_work;
      };
  每个cgroup_subsys对应一个cgroup_subsys_state，通过cgroup_subsys_state的ss指针可以找到该css对应的cgroup_subsys。 
所以每个进程可以有自己的css_set结构体（由进程的.cgroups指向），css_set结构体中有对应的cgroup_subsys_state指针数组，通过cgroup_subsys_state可以找到对应的cgroup_subsys。 
于是就成功建立进程与cgroup_subsys的联系了。<br>

接着内核定义了一个结构体叫做cgrp_cset_link用来让进程与cgroup建立联系。

      struct cgrp_cset_link {
    /* the cgroup and css_set this link associates */
    struct cgroup       *cgrp;
    struct css_set      *cset;

    /* list of cgrp_cset_links anchored at cgrp->cset_links */
    struct list_head    cset_link;

    /* list of cgrp_cset_links anchored at css_set->cgrp_links */
    struct list_head    cgrp_link;
    };

同一个cgroup可能关联了多个css_sets，因为不同进程可能属于不同的cgroup。 
另一方面，一个css_set通常关联多个cgroup（即对一个进程会使用多个控制行为）。 
cgroup和css_set的这种多对多的对应关系可以用struct cgrp_cset_link结构体表示，该结构体用于连接css_set与cgroup。
    
