>Linux 里面，无论是进程，还是线程，到了内核里面，我们统一都叫任务（Task），由一个统一的结构 task_struct 进行管理。Linux中用一个链表把task串联起来。
task_struct结构比较复杂，我们对于task_struct中结构挨个学习，便于我们了解进程的概念。具体结构如下：
![task_struct](../../picture/Linux/task_struct.jpeg)

### 任务ID
任务ID的结构如下：
```
pid_t pid; //进程的唯一性ID
pid_t tgid; //进程的主线程的pid
struct task_struct *group_leader;   //线程的主线程
```

>任何一个进程，如果只有主线程，那 pid 是自己，tgid 是自己，group_leader 指向的还是自己。但是，如果一个进程创建了其他线程，那就会有所变化了。线程有自己的 pid，tgid 就是进程的主线程的pid，group_leader 指向的就是进程的主线程。  
>有了 tgid，我们就知道 tast_struct 代表的是一个进程还是代表一个线程了。

### 信号处理
后续学习到进程间通信再会头学习
```
/* Signal handlers: */
struct signal_struct    *signal;
struct sighand_struct    *sighand;
sigset_t      blocked;
sigset_t      real_blocked;
sigset_t      saved_sigmask;
struct sigpending    pending;
unsigned long      sas_ss_sp;
size_t        sas_ss_size;
unsigned int      sas_ss_flags;
```

### 任务状态
在task_struct中任务状态是以下几个字段：
```
 volatile long state;    /* -1 unrunnable, 0 runnable, >0 stopped */
 int exit_state;
 unsigned int flags;
```
Linux中线程具体存在state，如下代码实例。可以看出state是通过bitset的方式设置的。
```
/* Used in tsk->state: */
#define TASK_RUNNING                    0
#define TASK_INTERRUPTIBLE              1 //等待io的睡眠状态，可被中断
#define TASK_UNINTERRUPTIBLE            2 //等待io的睡眠状态，不可被中断
#define __TASK_STOPPED                  4
#define __TASK_TRACED                   8
/* Used in tsk->exit_state: */
#define EXIT_DEAD                       16
#define EXIT_ZOMBIE                     32
#define EXIT_TRACE                      (EXIT_ZOMBIE | EXIT_DEAD)
/* Used in tsk->state again: */
#define TASK_DEAD                       64
#define TASK_WAKEKILL                   128
#define TASK_WAKING                     256
#define TASK_PARKED                     512
#define TASK_NOLOAD                     1024
#define TASK_NEW                        2048
#define TASK_STATE_MAX                  4096
```
进程的状态可以大致分为以下几类（通过ps可以看到）[参考](https://www.cnblogs.com/klb561/p/11945157.html)：
* R    正在运行，或在队列中的进程
* S    处于休眠状态
* T    停止或被追踪
* Z    退出进程
* W    进入内存交换（从内核2.6开始无效）
* X    死掉的进程

##### R状态
TASK_RUNNING
##### S状态
TASK_INTERRUPTIBLE，TASK_UNINTERRUPTIBLE，TASK_KILLABLE
> 处于这个状态的进程因为等待某某事件的发生（比如等待socket连接、等待信号量），而被挂起。是否可以被中断指的是否可以被信号处理唤醒，例如：I/O操作未完成，信号函数通知放弃这个I/O等待。    

> UNINTERRUPTIBLE是一个比较危险的事情，除非程序员极其有把握，不然还是不要设置成 TASK_UNINTERRUPTIBLE。UNINTERRUPTIBLE状态存在的意义就在于，内核的某些处理流程是不能被打断的。  

> TASK_KILLABLE，可以终止的新睡眠状态。进程处于这种状态中，它的运行原理类似 TASK_UNINTERRUPTIBLE，只不过可以响应致命信号。具体需要两个状态位`#define TASK_KILLABLE (TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)`。可以理解为功能处于他们中间的功能。

##### T状态
(TASK_STOPPED or TASK_TRACED)，暂停状态或跟踪状态。
> 向进程发送一个SIGSTOP信号，它就会因响应该信号而进入TASK_STOPPED状态;向进程发送一个SIGCONT信号，可以让其从TASK_STOPPED状态恢复到TASK_RUNNING状态。

> 当进程正在被跟踪时，它处于TASK_TRACED这个特殊的状态。“正在被跟踪”指的是进程暂停下来，等待跟踪它的进程对它进行操作。

##### Z退出状态
(EXIT_DEAD - EXIT_ZOMBIE)
> 一旦一个进程要结束，先进入的是 EXIT_ZOMBIE 状态，但是这个时候它的父进程还没有使用 wait() 等系统调用来获知它的终止信息，此时进程就成了僵尸进程。
> EXIT_DEAD 是进程的最终状态。

### 调度相关
在进程调度环节单独分析
```
//是否在运行队列上
int        on_rq;
//优先级
int        prio;
int        static_prio;
int        normal_prio;
unsigned int      rt_priority;
//调度器类
const struct sched_class  *sched_class;
//调度实体
struct sched_entity    se;
struct sched_rt_entity    rt;
struct sched_dl_entity    dl;
//调度策略
unsigned int      policy;
//可以使用哪些CPU
int        nr_cpus_allowed;
cpumask_t      cpus_allowed;
struct sched_info    sched_info;
```

### 运行统计信息
在进程的运行过程中，会有一些统计量，具体你可以看下面的列表。这里面有进程在用户态和内核态消耗的时间、上下文切换的次数等等。
```
u64        utime;//用户态消耗的CPU时间
u64        stime;//内核态消耗的CPU时间
unsigned long      nvcsw;//自愿(voluntary)上下文切换计数
unsigned long      nivcsw;//非自愿(involuntary)上下文切换计数
u64        start_time;//进程启动时间，不包含睡眠时间
u64        real_start_time;//进程启动时间，包含睡眠时间
```

### 进程亲缘关系
从创建进程的过程，可以看出，任何一个进程都有父进程。所以，整个进程其实就是一棵进程树。而拥有同一父进程的所有进程都具有兄弟关系。
```
struct task_struct __rcu *real_parent; /* real parent process */
struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */ 
struct list_head children;      /* list of my children */
struct list_head sibling;       /* linkage in my parent's children list */
```
> parent 指向其父进程。当它终止时，必须向它的父进程发送信号。children 表示链表的头部。链表中的所有元素都是它的子进程。sibling 用于把当前进程插入到兄弟链表中。

> 通常情况下，real_parent 和 parent 是一样的，但是也会有另外的情况存在。例如，bash 创建一个进程，那进程的 parent 和 real_parent 就都是 bash。如果在 bash 上使用 GDB 来 debug 一个进程，这个时候 GDB 是 parent，bash 是这个进程的 real_parent。

### 进程权限
```
/* Objective and real subjective task credentials (COW): 被哪个操纵*/
const struct cred __rcu         *real_cred;
/* Effective (overridable) subjective task credentials (COW): 操纵哪个*/
const struct cred __rcu         *cred;
```
cred的结构体如下：
```
struct cred {
......
        kuid_t          uid;            /* real UID of the task */
        kgid_t          gid;            /* real GID of the task */
        kuid_t          suid;           /* saved UID of the task */
        kgid_t          sgid;           /* saved GID of the task */
        kuid_t          euid;           /* effective UID of the task */
        kgid_t          egid;           /* effective GID of the task */
        kuid_t          fsuid;          /* UID for VFS ops */
        kgid_t          fsgid;          /* GID for VFS ops */
......
        kernel_cap_t    cap_inheritable; /* caps our children can inherit */
        kernel_cap_t    cap_permitted;  /* caps we're permitted */
        kernel_cap_t    cap_effective;  /* caps we can actually use */
        kernel_cap_t    cap_bset;       /* capability bounding set */
        kernel_cap_t    cap_ambient;    /* Ambient capability set */
......
} __randomize_layout;
```

##### 用户用户组

> 第一个是 uid 和 gid，注释是 real user/group id。一般情况下，谁启动的进程，就是谁的 ID。但是权限审核的时候，往往不比较这两个，也就是说不大起作用。一般情况下就是用户登录时的 user id。子进程的 real user id 从父进继承。

> 第二个是 euid 和 egid，注释是 effective user/group id。一看这个名字，就知道这个是起“作用”的。当这个进程要操作消息队列、共享内存、信号量等对象的时候，其实就是在比较这个用户和组是否有权限。

> 第三个是 fsuid 和 fsgid，也就是 filesystem user/group id。这个是对文件操作会审核的权限。

> suid 和 sgid指的是，指定该进程的用户/用户组权限，通过该字段保存，也就是 saved uid 和 save gid。这样就可以很方便地使用 setuid，通过设置 uid 或者 suid 来改变权限

通过以上字段，可以理解，Linux对于资源的操纵权限是基于用户的，我们可以通过给进程指定操纵用户来获取权限。Linux 系统通过进程的有效用户 ID(effective user id) 和有效用户组 ID(effective group id) 来决定进程对系统资源的访问权限。

##### capabilities
除了以用户和用户组控制权限，Linux 还有另一个机制就是 capabilities，用位图表示权限。
* cap_permitted 表示进程能够使用的权限。但是真正起作用的是 cap_effective。cap_permitted 中可以包含 cap_effective 中没有的权限。
* cap_inheritable表示当可执行文件的扩展属性设置了inheritable位时，调用 exec（启动新进程），新继承会继承调用者的 inheritable 集合，并将其加入到 permitted 集合。
* cap_bset，也就是 capability bounding set，是系统中所有进程允许保留的权限。
* cap_ambient 是比较新加入内核的，就是为了解决 cap_inheritable 鸡肋的状况，也就是，非 root 用户进程使用 exec 执行一个程序的时候，如何保留权限的问题。当执行 exec 的时候，cap_ambient 会被添加到 cap_permitted 中，同时设置到 cap_effective 中。

capabilities位图示例如下：
```
#define CAP_CHOWN            0
#define CAP_KILL             5
#define CAP_NET_BIND_SERVICE 10
#define CAP_NET_RAW          13
#define CAP_SYS_MODULE       16
#define CAP_SYS_RAWIO        17
#define CAP_SYS_BOOT         22
#define CAP_SYS_TIME         25
#define CAP_AUDIT_READ          37
#define CAP_LAST_CAP         CAP_AUDIT_READ
```

### 内存管理
每个进程都有自己独立的虚拟内存空间，这需要有一个数据结构来表示，就是 mm_struct。这个我们在内存管理那一节详细学习。
```
struct mm_struct                *mm;
struct mm_struct                *active_mm;
```

### 文件与文件系统
每个进程有一个文件系统的数据结构，还有一个打开文件的数据结构。这个我们放到文件系统那一节详细学习。
```
/* Filesystem information: */
struct fs_struct                *fs;
/* Open file information: */
struct files_struct             *files;
```

