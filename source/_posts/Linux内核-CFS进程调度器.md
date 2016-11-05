title: "Linux内核-CFS进程调度器"
comments: false
date: 2015-09-09 20:32:04
categories: Linux
tags: [Linux,内核]
---

Linux内核进程调度是一个极其重要的模块，实现多任务切换，同时为了兼顾系统性能，调度器的设计需要足够精巧，最近仔细回顾了一下Linux内核进程调度器CFS的相关内容，整理记录一下。
<!--more-->
# 进程调度器概述
现代操作系统设计的目的在于管理底层硬件资源并使整个计算机系统运行性能达到最优。我们可以发现实现这一目的的方法可以归结为两点：CPU虚拟化和内存虚拟化。现代操作系统一般都能提供多任务执行环境，每个进程都可以拥有自己的虚拟CPU，进程在执行过程中感觉自己是独占CPU。

类似地，内存虚拟化是通过让每个进程拥有虚拟地址空间，然后映射到物理内存。这样进程也不知道自己的虚拟地址其实并非物理内存中的相应位置，而是存在一个由虚拟地址向物理地址的映射。

CPU虚拟化是通过多个进程共享CPU实现的，具体来说，就是在一个固定周期内，每个进程获得一定比例的CPU时间。用来选择可以执行的进程的算法称为调度器，选择下一个执行的进程的过程称之为进程调度。

进程调度器可以说是现代操作系统中最重要的组件了。实现一个调度算法主要有两个挑战：一是对高优先级的进程要特殊照顾，分配更多的CPU资源，二是要保证低优先级的进程不会被饿死（迟迟得不到CPU资源）。此外调度器设计也需要十分精巧，可以保证执行调度器不会有太大开销进而影响整个系统吞吐量。

对一些交互性要求很高的进程，比如一个文字编辑进程，调度器就应该给每个进程很少的CPU时间，这样可以快速地切换进程，从而提高进程对IO的响应速度，达到更加良好的交互体验。而对一些非交互性的进程，情况则刚好相反。进程间的切换操作开销较大，因此给这种进程分配长CPU时间可以减少上下文切换，从而提高系统性能和吞吐量。显然以上两种情况存在冲突，需要进程调度器做一个tradeoff。

# CFS调度器
Linux中进程调度器已经经过很多次改进了，目前核心调度器是在CFS（Completely Fair Scheduler），从2.6.23开始被作为默认调度器。用作者Ingo Molnar的话讲，CFS在真实的硬件上模拟了完全理想的多任务处理器。也就是说CFS试图仿真CPU。理想、精确的多任务CPU是一个可以同时并行执行多个进程的硬件CPU，给每个进程分配等量的处理器功率（并非时间）。如果只有一个进程执行，那么它将获得100%的处理器功率，两个进程就是50%，依次平均分配。这样就可以实现所有进程公平执行。

显然这样的理想CPU是不存在的，CFS调度器试图通过软件方式去模拟出这样的CPU来。在真实处理器上，同一时间只能有一个进程被调度执行，因此其他进程必须等待。因此当前执行的进程将会获得100%的CPU功率，其他等待的进程无法获得CPU功率，这样的分配显然是违背之前的初衷的，没有公平可言。

CFS调度器就是为了减少系统中的这种不公平，CFS跟踪记录CPU的公平分配份额，用来分配给系统中的每个进程。因此CFS在一小部分CPU时钟速度下运行一个公平时钟。公平时钟增加速度通过用实际时钟时间除以等待进程个数计算。结果就是分配给每个进程的CPU时间。

进程等待CPU的时候，调度器会跟踪记录它将会使用理想的处理器时间。这个等待时间用每个进程等待执行时间来表示，可以用来对进程进行排序，决定其在被抢占之前可以被分配的CPU时间。等待时间最长的进程会被首先选择调度到CPU上执行，当这个进程执行时，它的等待时间会相应减少，其他进程等待时间自然就会增加了。这样马上就会有另外一个等待时间最长的进程出现，当前执行的进程就会被抢占。基于这一原理，CFS调度器试图公平对待所有进程并使每个进程等待时间为0，这样每个进程就可以拥有等量的CPU资源，达到完全公平的一种理想状态。

# CFS实现细节
## 三个重要问题
### 选择下一个task
正如前文所说，CFS调度器试图保证完全公平调度，CFS将CPU时间按照运行队列里所有的se（sched_entity）的权重分配。se的调度顺序由每个task使用CPU的虚拟时间（vruntime）决定，vruntime越少，对应在队列中的位置就越靠前，就会被尽快调度执行。CFS采用红黑树这一数据结构来维护task的队列，红黑树的时间复杂度可以达到O(log n)。
### tick中断
tick中断更新调度信息，调整当前进程在红黑树中的位置，调整完成后如果当前进程不是最左边的叶子节点（vruntime最小），就标记为需要调度。中断返回时就会调用schedule()来完成进程切换。否则当前进程继续执行。
### 红黑树
红黑树是内核中一种极其重要的数据结构，CFS用来维护进程队列，CFS中红黑树节点就是se，键值就是vruntime，vruntime由进程已经使用CPU时间、nice值和当前CPU负载共同计算而来。关于计算细节下面介绍。
## 实现细节
### 调度实体sched_entity
进程调度实际上不是以进程为实际操作对象的，进程结构体定义在"linux/sched.h"中
```c
struct task_struct {
	volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
	void *stack;
	atomic_t usage;
	unsigned int flags;	/* per process flags, defined below */
	unsigned int ptrace;

	int lock_depth;		/* BKL lock depth */
/*Some unimportant members are not presented.*/
	int prio, static_prio, normal_prio;
	unsigned int rt_priority;
	const struct sched_class *sched_class;
	struct sched_entity se;
	struct sched_rt_entity rt;
/*Some unimportant members are not presented.*/
}
```
其中struct sched_entity se便是进程调度器操作的实际对象，我们称作调度实体，其定义在"linux/sched.h"中
```c
struct sched_entity {
	struct load_weight	load;		/* for load-balancing */
	struct rb_node		run_node;
	struct list_head	group_node;
	unsigned int		on_rq;

	u64			exec_start;
	u64			sum_exec_runtime;
	u64			vruntime;              //Look!
	u64			prev_sum_exec_runtime;

	u64			last_wakeup;
	u64			avg_overlap;

	u64			nr_migrations;

	u64			start_runtime;
	u64			avg_wakeup;
/*Some unimportant members are not presented.*/
};
```
可以看到vruntime是在se中维护的，还有很多和执行时间分配有关的变量，这里没有一一列出。se被嵌入到task_struct中，这样做的目的是避免频繁操作task_struct，可以降低调度器的设计复杂性。
### 虚拟运行时间vruntime
CFS中已经不再使用时间片了，vruntime表示进程虚拟运行时间，顾名思义就不是进程实际运行时间。CFS用vruntime来记录一个进程执行了多长时间以及还应该再执行多长时间。下面我们通过源码来追踪一下。

计时器每隔一个周期产生一个tick中断，会调用一个schedluer_tick函数，定义在"linux/sched.h"中
```c
void scheduler_tick(void)
{
/*Some unimportant code is not presented.*/
	raw_spin_lock(&rq->lock);
	update_rq_clock(rq);//更新运行队列维护的时钟
	update_cpu_load(rq);//更新运行队列维护的CPU负载
	curr->sched_class->task_tick(rq, curr, 0);//触发调度器，这里我们看CFS的
	raw_spin_unlock(&rq->lock);
/*Some unimportant code is not presented.*/
}
```
暂时不关注其他的，直接看CFS的task_tick，对应"kernel/sched_fair.c"中的task_tick_fair函数
```c
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &curr->se;

	for_each_sched_entity(se) {
		cfs_rq = cfs_rq_of(se);
		entity_tick(cfs_rq, se, queued);//这里更新每个se的执行时间
	}
}
```
调用entity_tick更新每个进程的运行状态
```c
static void
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
	/*
	 * Update run-time statistics of the 'current'.
	 */
	update_curr(cfs_rq);//更新当前运行队列，主要是vruntime了

/*Some unimportant code are not presented.*/

	if (cfs_rq->nr_running > 1 || !sched_feat(WAKEUP_PREEMPT))
		check_preempt_tick(cfs_rq, curr);//如果可运行的进程多于1个，就
}
```
update_curr函数更新队列里每个进程的vruntime，并重新进行排序。
```c
static void update_curr(struct cfs_rq *cfs_rq)
{
	struct sched_entity *curr = cfs_rq->curr;
	u64 now = rq_of(cfs_rq)->clock;
	unsigned long delta_exec;

	if (unlikely(!curr))
		return;

	/*获取最新一次更新负载后当前进程所使用的CPU总时间 */
	delta_exec = (unsigned long)(now - curr->exec_start);
	if (!delta_exec)
		return;

	__update_curr(cfs_rq, curr, delta_exec);//真正计算vruntime在这里
	curr->exec_start = now;

	if (entity_is_task(curr)) {
		struct task_struct *curtask = task_of(curr);

		trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
		cpuacct_charge(curtask, delta_exec);
		account_group_exec_runtime(curtask, delta_exec);
	}
}
```
update_curr计算当前进程执行时间，然后传递给__update_curr
```c
static inline void
__update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr,
	      unsigned long delta_exec)
{
	unsigned long delta_exec_weighted;

	schedstat_set(curr->exec_max, max((u64)delta_exec, curr->exec_max));
	
	//当前进程执行时间加上传递过来的delta_exec
	curr->sum_exec_runtime += delta_exec;
	schedstat_add(cfs_rq, exec_clock, delta_exec);
	//计算vruntime
	delta_exec_weighted = calc_delta_fair(delta_exec, curr);
	curr->vruntime += delta_exec_weighted;
	update_min_vruntime(cfs_rq);
}
```
这里调用了calc_delta_fair来计算权重用来更新vruntime
```c
/*
delta/=w
*/
static inline unsigned long
calc_delta_fair(unsigned long delta, struct sched_entity *se)
{
	if (unlikely(se->load.weight != NICE_0_LOAD))
		delta = calc_delta_mine(delta, NICE_0_LOAD, &se->load);
	return delta;
}
```
进一步看calc_delta_mine函数
```c

/*
 * delta *= weight / lw
 */
static unsigned long
calc_delta_mine(unsigned long delta_exec, unsigned long weight,
		struct load_weight *lw)
{
	u64 tmp;

	if (!lw->inv_weight) {
		if (BITS_PER_LONG > 32 && unlikely(lw->weight >= WMULT_CONST))
			lw->inv_weight = 1;
		else
			lw->inv_weight = 1 + (WMULT_CONST-lw->weight/2)
				/ (lw->weight+1);
	}

	tmp = (u64)delta_exec * weight;
	/*
	 * Check whether we'd overflow the 64-bit multiplication:
	 */
	if (unlikely(tmp > WMULT_CONST))
		tmp = SRR(SRR(tmp, WMULT_SHIFT/2) * lw->inv_weight,
			WMULT_SHIFT/2);
	else
		tmp = SRR(tmp * lw->inv_weight, WMULT_SHIFT);

	return (unsigned long)min(tmp, (u64)(unsigned long)LONG_MAX);
}
```
我们可以看出，
**一个调度周期vruntime=实际运行时间*（NICE_0_LOAD/weight）**

### 进程的切换
上面的周期调度器scheduler_tick只对进程的虚拟运行时间进行更新并重新进行了排序，供主调度器来调度执行。主调度器在"kernel/sched.c"中schedule()函数实现。其中最重要的一步就是选择下一个要调度执行的进程，在函数pick_next_task中将会进一步调用调度类的pick_next_task，对应就是pick_next_task_fair了。
```c
static struct task_struct *pick_next_task_fair(struct rq *rq)
{
/*some code is not presented*/
	do {
		se = pick_next_entity(cfs_rq);//选择下一个进程调度实体
		set_next_entity(cfs_rq, se);//设置为当点调度队列下一个调度实体
		cfs_rq = group_cfs_rq(se);
	} while (cfs_rq);

	p = task_of(se);
	hrtick_start_fair(rq, p);

	return p;
}
```
这里调用了pick_next_entity来获取下一个进程调度实体，进一步看其调用的__pick_next_entity函数
```c
static struct sched_entity *__pick_next_entity(struct cfs_rq *cfs_rq)
{
	struct rb_node *left = cfs_rq->rb_leftmost;

	if (!left)
		return NULL;

	return rb_entry(left, struct sched_entity, run_node);
}
```
终于看到胜利的曙光了，这里就是选择红黑树最左边的子节点了。完成选择下一个调度执行进程后，接下来就是上下文切换，进程调度执行了。关于上下文切换这里不作详细讨论。
# 小结
进程调度器作为多任务操作系统中最重要的一部分，在Linux中进行了很多次革新，目前CFS是作为内核默认的进程调度器。调度过程的实际代码实现远远不止前文所描述的那般简单，需要对代码进行更加深入的跟踪学习，这里仅仅从原理和基本实现过程进行简单介绍。
# 参考资料
[1]Robert Love,《Linux Kernel Development》,3rd edition
[2][Completely Fair Scheduler](http://www.linuxjournal.com/magazine/completely-fair-scheduler)