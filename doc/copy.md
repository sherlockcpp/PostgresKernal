

```c
/*
 * List of internal parallel worker entry points.  We need this for
 * reasons explained in LookupParallelWorkerFunction(), below.
 */
static const struct
{
	const char *fn_name;
	parallel_worker_main_type fn_addr;
}			InternalParallelWorkers[] =

{
	{
		"ParallelQueryMain", ParallelQueryMain
	},
	{
		"_bt_parallel_build_main", _bt_parallel_build_main
	},
	{
		"parallel_vacuum_main", parallel_vacuum_main
	}
};
```
具体并行执行的每个并行进程执行的内部函数


```c
/*
 * List of internal background worker entry points.  We need this for
 * reasons explained in LookupBackgroundWorkerFunction(), below.
 */
static const struct
{
	const char *fn_name;
	bgworker_main_type fn_addr;
}			InternalBGWorkers[] =

{
	{
		"ParallelWorkerMain", ParallelWorkerMain
	},
	{
		"ApplyLauncherMain", ApplyLauncherMain
	},
	{
		"ApplyWorkerMain", ApplyWorkerMain
	}
};
```
后台进程的入口函数

开启后台进程的时机
1.PostmasterMain 在进入postmaster的主循环 ServerLoop之前
2.在ServerLoop的每次循环中，因为有可能HaveCrashedWorker，即有进程崩溃需要重新开启
3.在startup 用来启动别的进程的临时进程结束时，也会尝试开启后台进程
4.sigusr1_handler 在接收到子进程的信号信息时，会尝试开启后台进程

maybe_start_bgworkers
遍历所有注册了需要开启的后台进程，启动
每一次调用有一定的数量限制
do_start_bgworker
StartBackgroundWorker
	ParallelWorkerMain
		InternalParallelWorkers[xx]


InitializeParallelDSM
开启方式通过在共享内存中保存并行信息
shm_toc_insert(pcxt->toc, PARALLEL_KEY_ENTRYPOINT, entrypointstate);
然后在ParallelWorkerMain读取后启动相应的进程
entrypointstate = shm_toc_lookup(toc, PARALLEL_KEY_ENTRYPOINT, false);


根据并行数量，实际注册多个进程
LaunchParallelWorkers
RegisterDynamicBackgroundWorker
注册进全局数组中BackgroundWorkerData

发送信号个postmaster
SendPostmasterSignal(PMSIGNAL_BACKGROUND_WORKER_CHANGE);

根据注册的进程，添加信息到BackgroundWorkerList全局链表中
BackgroundWorkerStateChange

FixedParallelExecutorState *fpes;
保存每个并行任务分配的数据结构



```c
/*
 *	Represents the end-of-line terminator type of the input
 */
typedef enum EolType
{
	EOL_UNKNOWN,
	EOL_NL,
	EOL_CR,
	EOL_CRNL
} EolType;


```
行尾符类型(windows上和linux上可能不一样)
