=============
CFS Scheduler
=============
CFS调度

1.  OVERVIEW
============
概述

CFS stands for "Completely Fair Scheduler," and is the new "desktop" process
scheduler implemented by Ingo Molnar and merged in Linux 2.6.23.  It is the
replacement for the previous vanilla scheduler's SCHED_OTHER interactivity
code.

CFS指完全公平调度，这是一个由Ingo Molnar实现的“桌面”进程调度器，在2.6.23版本被合入主线。这是对之前的vanlilla调度器SCHED_OTHER的替换。

80% of CFS's design can be summed up in a single sentence: CFS basically models
an "ideal, precise multi-tasking CPU" on real hardware.

80%的CFS调度器的设计可以归纳为一句话：CFS在真实的硬件上基于“理想，精确的多任务处理器”的思想建模。

"Ideal multi-tasking CPU" is a (non-existent  :-)) CPU that has 100% physical
power and which can run each task at precise equal speed, in parallel, each at
1/nr_running speed.  For example: if there are 2 tasks running, then it runs
each at 50% physical power --- i.e., actually in parallel.

理想的多任务处理器是不存在的，这种CPU具有100%的物理能量，可以精确地以相同的速度并发执行每一个任务，每一个任务都有1/nr_running的速度。比如说，这里有两个执行的任务，则每一个任务都以50%的物理能量运行，真正的并行。

On real hardware, we can run only a single task at once, so we have to
introduce the concept of "virtual runtime."  The virtual runtime of a task
specifies when its next timeslice would start execution on the ideal
multi-tasking CPU described above.  In practice, the virtual runtime of a task
is its actual runtime normalized to the total number of running tasks.

在真正的硬件上，我们可以一次只运行一个单独的任务，因此，我们可以引出一个概念“虚拟运行时间（virtual runtime）”，当下一个时间片开始在上述理想的多任务CPU上执行时，一个任务的虚拟运行时间就产生了。在实践过程中，一个任务的虚拟运行时间是其真正的运行时间根据全部正在运行任务总数标准化后的。

2.  FEW IMPLEMENTATION DETAILS
==============================
实现细节

In CFS the virtual runtime is expressed and tracked via the per-task
p->se.vruntime (nanosec-unit) value.  This way, it's possible to accurately
timestamp and measure the "expected CPU time" a task should have gotten.

在CFS调度中，虚拟运行时间是通过p->se.vruntime （纳秒为单位）表达和跟踪的。这样，就可以准确的度量时间步长和度量一个任务已经获取的“期望CPU时间”。

[ small detail: on "ideal" hardware, at any time all tasks would have the same
  p->se.vruntime value --- i.e., tasks would execute simultaneously and no task
  would ever get "out of balance" from the "ideal" share of CPU time.  ]

小细节： 在“理想”硬件上，在任何时候，所有的任务都将会有相同的p->se.vruntime 值，因为任务会同时执行，并且没有task会从理想硬件共享的CPU时间上失去平衡。

CFS's task picking logic is based on this p->se.vruntime value and it is thus
very simple: it always tries to run the task with the smallest p->se.vruntime
value (i.e., the task which executed least so far).  CFS always tries to split
up CPU time between runnable tasks as close to "ideal multitasking hardware" as
possible.

CFS的任务选择任务的逻辑是基于这个虚拟运行时间p->se.vruntime的值，这就非常简单。调度算法会尝试运行p->se.vruntime值最小的的任务去执行（这样的任务到目前为止执行的时间最少），CFS通常会尝试在可运行的任务间切分CPU时间，尽可能地逼近“理想的多任务硬件”。

Most of the rest of CFS's design just falls out of this really simple concept,
with a few add-on embellishments like nice levels, multiprocessing and various
algorithm variants to recognize sleepers.

大多数的其余的CFS调度器的设计都出自于这个简单的概念，再附加一些装饰，比如nice级别，多任务处理和多种算法的变形来识别睡眠任务。

3.  THE RBTREE
==============
红黑树

CFS's design is quite radical: it does not use the old data structures for the
runqueues, but it uses a time-ordered rbtree to build a "timeline" of future
task execution, and thus has no "array switch" artifacts (by which both the
previous vanilla scheduler and RSDL/SD are affected).

CFS的设计相当激进：它没有为运行队列使用老的数据结构，为了构建一个将要执行的任务“时间表”，CFS使用了时间排序的红黑树。这样就没有了“数组切换的开销”（之前的vanilla调度器和 RSDL/SD就是收到这个的影响）

CFS also maintains the rq->cfs.min_vruntime value, which is a monotonic
increasing value tracking the smallest vruntime among all tasks in the
runqueue.  The total amount of work done by the system is tracked using
min_vruntime; that value is used to place newly activated entities on the left
side of the tree as much as possible.

CFS也维护一个单调递增，所有的任务中虚拟运行时间最小的rq->cfs.min_vruntime值。系统完成的工作量用min_vruntime来跟踪。红黑树中最左边的实体的虚拟运行时间vruntime来更新这个值。

The total number of running tasks in the runqueue is accounted through the
rq->cfs.load value, which is the sum of the weights of the tasks queued on the
runqueue.

在运行队列中任务总的运行时间可以用rq->cfs.load来度量。这是所有运行队列上的任务队列的权重之和。

CFS maintains a time-ordered rbtree, where all runnable tasks are sorted by the
p->se.vruntime key. CFS picks the "leftmost" task from this tree and sticks to it.
As the system progresses forwards, the executed tasks are put into the tree
more and more to the right --- slowly but surely giving a chance for every task
to become the "leftmost task" and thus get on the CPU within a deterministic
amount of time.

CFS维护着一个时间排序的红黑树，所有可运行的任务以p->se.vruntime键来进行排序。CFS依照这个挑选树最左侧的任务。
随着系统的进度向前，执行过的任务越来越多的被放到了树的右边。慢慢地，每一个任务都确保会变成“最左任务”，这样在确定的时间内，可以在CPU上执行。

Summing up, CFS works like this: it runs a task a bit, and when the task
schedules (or a scheduler tick happens) the task's CPU usage is "accounted
for": the (small) time it just spent using the physical CPU is added to
p->se.vruntime.  Once p->se.vruntime gets high enough so that another task
becomes the "leftmost task" of the time-ordered rbtree it maintains (plus a
small amount of "granularity" distance relative to the leftmost task so that we
do not over-schedule tasks and trash the cache), then the new leftmost task is
picked and the current task is preempted.

总结一下，CFS工作原理：首先运行任务一下，当用于计量任务的CPU使用情况的任务调度（或者一个调度tick发生时）。p->se.vruntime是p代表任务执行所消耗的物理CPU时间的累加。
一旦p->se.vruntime变得足够大，其他的任务将变成红黑树中的“最左任务”（加少量的相对于最左任务的粒度距离，这样我们就不会过度调度和让cache失效），然后选择新的最左任务，抢占当前执行的任务。

4.  SOME FEATURES OF CFS
========================

CFS uses nanosecond granularity accounting and does not rely on any jiffies or
other HZ detail.  Thus the CFS scheduler has no notion of "timeslices" in the
way the previous scheduler had, and has no heuristics whatsoever.  There is
only one central tunable (you have to switch on CONFIG_SCHED_DEBUG):

   /proc/sys/kernel/sched_min_granularity_ns

CFS使用纳秒粒度的计量，不依赖任何jiffies（tick总数）或者时钟中断的频率（Hz）。这样CFS调度器就没有之前的调度器的“时间片”的概念，也没有启发式。仅有一项可以调节（你必须切换到CONFIG_SCHED_DEBUG）
/proc/sys/kernel/sched_min_granularity_ns

which can be used to tune the scheduler from "desktop" (i.e., low latencies) to
"server" (i.e., good batching) workloads.  It defaults to a setting suitable
for desktop workloads.  SCHED_BATCH is handled by the CFS scheduler module too.

这个可以用来调节调度器，桌面，低延时，服务器搞负载（很好的批处理能力）。默认设置适合桌面的负载。SCHED_BATCH也可以由CFS调度模块处理。

Due to its design, the CFS scheduler is not prone to any of the "attacks" that
exist today against the heuristics of the stock scheduler: fiftyp.c, thud.c,
chew.c, ring-test.c, massive_intr.c all work fine and do not impact
interactivity and produce the expected behavior.

由于其设计，CFS调度器可以免于现存的针对启发式调度器任何攻击， fiftyp.c, thud.c,
chew.c, ring-test.c, massive_intr.c。在这些攻击下工作良好，不会影响交互和产品预期的行为。

The CFS scheduler has a much stronger handling of nice levels and SCHED_BATCH
than the previous vanilla scheduler: both types of workloads are isolated much
more aggressively.

与之前的vanlilla 调度器相比，CFS调度器更加健壮和具有SCHED_BATCH特性，工作负载的类型隔离性更好。

SMP load-balancing has been reworked/sanitized: the runqueue-walking
assumptions are gone from the load-balancing code now, and iterators of the
scheduling modules are used.  The balancing code got quite a bit simpler as a
result.

重构和加强对称多处理：运行队列遍历假设已经从负载平衡的代码中删除了，调度模块的迭代器被使用了。平衡代码变得相当简单。

5. Scheduling policies
====================== 调度策略

CFS implements three scheduling policies:

  - SCHED_NORMAL (traditionally called SCHED_OTHER): The scheduling
    policy that is used for regular tasks.

正常模式：传统上称作SCHED_OTHER ，用于普通的任务

  - SCHED_BATCH: Does not preempt nearly as often as regular tasks
    would, thereby allowing tasks to run longer and make better use of
    caches but at the cost of interactivity. This is well suited for
    batch jobs.

批调度模式：不会像普通的模式那样频繁的抢占，允许任务运行更长的时间，更好地利用缓存而不是消耗在了换入换出。这非常适合批处理任务。

  - SCHED_IDLE: This is even weaker than nice 19, but its not a true
    idle timer scheduler in order to avoid to get into priority
    inversion problems which would deadlock the machine.

空闲调度：nice值弱于19,为了避免优先级反转问题导致死锁，这不是一个真正的空闲时钟调度器

SCHED_FIFO/_RR are implemented in sched/rt.c and are as specified by
POSIX.
实现了了实时调度器，满足POSIX的标准。

The command chrt from util-linux-ng 2.13.1.1 can set all of these except
SCHED_IDLE.

chrt命令可以设置所有的除了空闲调度之外的调度策略。

6.  SCHEDULING CLASSES
======================  调度类

The new CFS scheduler has been designed in such a way to introduce "Scheduling
Classes," an extensible hierarchy of scheduler modules.  These modules
encapsulate scheduling policy details and are handled by the scheduler core
without the core code assuming too much about them.

新的CFS调度器的设计：调度模块的扩展层。这些模块封装了调度策略的细节和调度核心。

sched/fair.c implements the CFS scheduler described above.

sched/fair.c实现

sched/rt.c implements SCHED_FIFO and SCHED_RR semantics, in a simpler way than
the previous vanilla scheduler did.  It uses 100 runqueues (for all 100 RT
priority levels, instead of 140 in the previous scheduler) and it needs no
expired array.

sched/rt.c实现了SCHED_FIFO和SCHED_RR语义，比之前的vanilla调度器实现更简单。调度器使用100个运行队列（100 RT调度优先级，而不是之前调度器的140），不需要过期的数组。

Scheduling classes are implemented through the sched_class structure, which
contains hooks to functions that must be called whenever an interesting event
occurs.

调度类实现为 sched_class 结构体，结构体中包含钩子函数。

This is the (partial) list of the hooks: 部分的钩子函数列表

 - enqueue_task(...)

   Called when a task enters a runnable state.
   It puts the scheduling entity (task) into the red-black tree and
   increments the nr_running variable.

enqueue_task:当一个task进入可运行状态。
将调度实体（任务）加入到红黑树，并自增nr_running变量。

 - dequeue_task(...)

   When a task is no longer runnable, this function is called to keep the
   corresponding scheduling entity out of the red-black tree.  It decrements
   the nr_running variable.

dequeue_task:当任务不再可运行，调用这个函数将相应的调度实体从红黑树取出。自减nr_running变量。

 - yield_task(...)

   This function is basically just a dequeue followed by an enqueue, unless the
   compat_yield sysctl is turned on; in that case, it places the scheduling
   entity at the right-most end of the red-black tree.

yield_task：让出任务，这个函数基于进队/出队。除非compat_yield的sysctl打开;在那种场景下，会将调度实体放到最右的红黑树。

 - check_preempt_curr(...)

   This function checks if a task that entered the runnable state should
   preempt the currently running task.

这个函数检查进入了可运行状态的任务是否应该抢占当前运行的任务

 - pick_next_task(...)

   This function chooses the most appropriate task eligible to run next.

这个函数选择下一个最合适的任务

 - set_curr_task(...)

   This function is called when a task changes its scheduling class or changes
   its task group.

当这个任务改变了其调度类或者改变了任务组，调用这个函数

 - task_tick(...)

   This function is mostly called from time tick functions; it might lead to
   process switch.  This drives the running preemption.

这个函数大多数在时间tick时调用，这可能导致进程切换，驱动运行进程的抢占。


7.  GROUP SCHEDULER EXTENSIONS TO CFS
===================================== CFS组调度扩展

Normally, the scheduler operates on individual tasks and strives to provide
fair CPU time to each task.  Sometimes, it may be desirable to group tasks and
provide fair CPU time to each such task group.  For example, it may be
desirable to first provide fair CPU time to each user on the system and then to
each task belonging to a user.

通常，调度程序会处理单个任务，并努力为每个任务提供合理的CPU时间。
有时，可能需要对任务进行分组并为每个此类任务组提供合理的CPU时间。
例如，可能希望首先为系统上的每个用户提供公平的CPU时间，然后为属于该用户的每个任务提供公平的CPU时间。

CONFIG_CGROUP_SCHED strives to achieve exactly that.  It lets tasks to be
grouped and divides CPU time fairly among such groups.

CONFIG_CGROUP_SCHED努力实现这一目标。它可以对任务进行分组，并在这些组之间公平地分配CPU时间。

CONFIG_RT_GROUP_SCHED permits to group real-time (i.e., SCHED_FIFO and
SCHED_RR) tasks.
CONFIG_RT_GROUP_SCHED允许分组实时任务

CONFIG_FAIR_GROUP_SCHED permits to group CFS (i.e., SCHED_NORMAL and
SCHED_BATCH) tasks.
CONFIG_FAIR_GROUP_SCHED 允许分组CFS任务

   These options need CONFIG_CGROUPS to be defined, and let the administrator
   create arbitrary groups of tasks, using the "cgroup" pseudo filesystem.  See
   Documentation/admin-guide/cgroup-v1/cgroups.rst for more information about this filesystem.
这些选项需要定义CONFIG_CGROUPS，并允许管理员使用“ cgroup”伪文件系统创建任意组的任务。有关此文件系统的更多信息，请参见Documentation / admin-guide / cgroup-v1 / cgroups.rst。

When CONFIG_FAIR_GROUP_SCHED is defined, a "cpu.shares" file is created for each
group created using the pseudo filesystem.  See example steps below to create
task groups and modify their CPU share using the "cgroups" pseudo filesystem::

CONFIG_FAIR_GROUP_SCHED定义之后，为每一个组创建"cpu.shares"文件来使用伪文件系统。

	# mount -t tmpfs cgroup_root /sys/fs/cgroup
	# mkdir /sys/fs/cgroup/cpu
	# mount -t cgroup -ocpu none /sys/fs/cgroup/cpu
	# cd /sys/fs/cgroup/cpu

	# mkdir multimedia	# create "multimedia" group of tasks
	# mkdir browser		# create "browser" group of tasks

	# #Configure the multimedia group to receive twice the CPU bandwidth
	# #that of browser group

	# echo 2048 > multimedia/cpu.shares
	# echo 1024 > browser/cpu.shares

	# firefox &	# Launch firefox and move it to "browser" group
	# echo <firefox_pid> > browser/tasks

	# #Launch gmplayer (or your favourite movie player)
	# echo <movie_player_pid> > multimedia/tasks
