---
layout: post
title: "Notes of debugging the kernel using Ftrace"
date: 2014-10-07 23:14
comments: true
categories:  [kernel, debug]
---

This page has my notes for debugging the kernel using Ftrace

Ftrace is a tracing utility built directly into the Linux kernel. Ftrace was introduced in kernel 2.6.27 by Steven Rostedy and Ingo Molnar. It comes with its own ring buffer for storing trace data, and uses the GCC profiling mechanism. Documentation for this can be found in the Linux kernel source tree at [Documentation/trace/ftrace.txt](https://www.kernel.org/doc/Documentation/trace/ftrace.txt)

#### References:  
* [Debugging the kernel using Ftrace - part 1](http://lwn.net/Articles/365835/)  
* [Debugging the kernel using Ftrace - part 2](http://lwn.net/Articles/366796/)  
* [Secrets of the Ftrace function tracer](http://lwn.net/Articles/370423/)  

### Setting up Ftrace:
When Ftrace is configured, it will create its own directory called tracing within the debugfs file system. 

	[~]# cd /sys/kernel/debug/tracing
	[tracing]#

For the purpose of debugging, the kernel configuration parameters that should be enabled are:  
*  Kernel Function Tracer (FUNCTION_TRACER)  
*  Kernel Function Graph Tracer (FUNCTION_GRAPH_TRACER)  
*  Enable/disable ftrace dynamically (DYNAMIC_FTRACE)  

The function tracer uses the *-pg* of *gcc* to have every function in the kernel call a special funciton `mcount()`.
During the compilation the mcount() call-sites are recorded, and that list is used at boot time to convert those
call-sites to NOP when CONFIG_DYNAMIC_FTRACE is set.
When the function or function graph tracer is enabled, that list is saved to convert those call-sites back to trace calls.


To find out which tracers are available, simply cat the *available_tracers* file in the tracing directory:

	root@K015:/ # cat /d/tracing/available_tracers
	blk function_graph wakeup_rt wakeup preemptirqsoff preemptoff irqsoff function nop

To enable the function tracer, just echo "function" into the *current_tracer* file.
	root@K015:/d/tracing # echo function > current_tracer
	root@K015:/d/tracing # cat> tracer
	# tracer: function
	#
	# entries-in-buffer/entries-written: 205147/172709617   #P:4
	#
	#                              _-----=> irqs-off
	#                             / _----=> need-resched
	#                            | / _---=> hardirq/softirq
	#                            || / _--=> preempt-depth
	#                            ||| /     delay
	#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
	#              | |       |   ||||       |         |
			Binder_D-4038  [003] ....    98.468143: cgroup_tasks_write <-cgroup_file_write
			Binder_D-4038  [003] ....    98.468143: attach_task_by_pid <-cgroup_tasks_write
			Binder_D-4038  [003] ....    98.468143: cgroup_lock_live_group <-attach_task_by_pid
			Binder_D-4038  [003] ....    98.468144: mutex_lock <-cgroup_lock_live_group
			Binder_D-4038  [003] ....    98.468144: __might_sleep <-mutex_lock
			Binder_D-4038  [003] ....    98.468144: __mutex_lock_slowpath <-mutex_lock
			Binder_D-4038  [003] ....    98.468144: add_preempt_count <-__mutex_lock_slowpath
			Binder_D-4038  [003] ...1    98.468145: sub_preempt_count <-__mutex_lock_slowpath
			Binder_D-4038  [003] ....    98.468145: __rcu_read_lock <-attach_task_by_pid

A header is printed with the tracer name that is represented by
the trace. In this case the tracer is "function". Then it shows the
number of events in the buffer as well as the total number of entries
that were written. The difference is the number of entries that were
lost due to the buffer filling up (172709617 - 205147 = 172504407 events
lost). #P is the number of online CPUs (#P:4).

The header explains the content of the events. Task name "Binder_D", the task
PID "4038", the CPU that it was running on "003", the latency format
(explained below), the timestamp in <secs>.<usecs> format, the
function name that was traced "cgroup_tasks_write" and the parent function that
called this function "cgroup_file_write". The timestamp is the time
at which the function was entered.

	irqs-off: 'd' interrupts are disabled. '.' otherwise.
	    Note: If the architecture does not support a way to
		  read the irq flags variable, an 'X' will always
		  be printed here.

	need-resched:
	'N' both TIF_NEED_RESCHED and PREEMPT_NEED_RESCHED is set,
	'n' only TIF_NEED_RESCHED is set,
	'p' only PREEMPT_NEED_RESCHED is set,
	'.' otherwise.

	hardirq/softirq:
	'H' - hard irq occurred inside a softirq.
	'h' - hard irq is running
	's' - soft irq is running
	'.' - normal context.

	preempt-depth: The level of preempt_disabled

The above is mostly meaningful for kernel developers.


###Tips
---
####Using *trace_prink()*
If you are debugging a high volume area such as the timer interrupt, the scheduler, or the network, *printk()* can lead to bogging down the system or can even create a live lock. 
It is also quite common to see a bug "disappear" when adding a few *printk()*s. This is due to the sheer overhead that *printk()* introduces.

Ftrace introduces a new form of *printk()* called *trace_printk()*.

For example, below code add entry/exit checkpoints to know how long system stays at standy mode
``` c arch/x86/platform/intel-mid/intel_soc_pmu.c
static int mid_suspend_enter(suspend_state_t state)
{
        int ret;

        if (state != PM_SUSPEND_MEM)
                return -EINVAL;
...<SKIP>...
        trace_printk("s3_entry\n");
        ret = standby_enter();
        trace_printk("s3_exit %d\n", ret);
...<SKIP>...
        return ret;
}
```
*trace_printk()* output will appear in any tracer, even the function and function graph tracers.
	# tracer: nop
	#
	# entries-in-buffer/entries-written: 4/4   #P:4
	#
	#                              _-----=> irqs-off
	#                             / _----=> need-resched
	#                            | / _---=> hardirq/softirq
	#                            || / _--=> preempt-depth
	#                            ||| /     delay
	#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
	#              | |       |   ||||       |         |
	    kworker/u8:5-1178  [000] d...   318.726581: mid_suspend_enter: s3_entry
	    kworker/u8:5-1178  [000] d...   377.664365: mid_suspend_enter: s3_exit 0
	    kworker/u8:5-1178  [000] d...   378.514933: mid_suspend_enter: s3_entry
	    kworker/u8:5-1178  [000] d...   569.010863: mid_suspend_enter: s3_exit 0

####Tracing a specific process
Trace a specific process, or set of processes. The file set_ftrace_pid lets you specify specific processes that you want to trace.
	[tracing]# echo $$ > set_ftrace_pid
The above will set the function tracer to only trace the bash shell that executed the echo command.

Clear the *set_ftrace_pid file* if you want to go back to generic function tracing
	[tracing]# echo -1 > set_ftrace_pid


####Capturing Ftrace to oops when kernel panic
You can capture the function calls leading up to a panic by placing the following on the kernel command line

	ftrace=function ftrace_dump_on_oops

or,  by echoing a "1" into */proc/sys/kernel/ftrace_dump_on_oops*, will enable Ftrace to dump to the console the entire trace buffer in ASCII format on oops or panic.

####Find latencies on kernel startup
Use the following on the kernel command line:
	tracing_thresh=10000 ftrace=function_graph
this traces all functions taking longer than 2000 microseconds (10 ms).


