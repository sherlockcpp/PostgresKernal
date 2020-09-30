# Disk-based Hash Aggregation



底层提供四种创建使用共享内存的方式,用户可以通过dynamic_shared_memory_type选择
posix/sysv/windows/mmap

### dsm_impl.c
```C
#ifdef USE_DSM_POSIX
static bool dsm_impl_posix(dsm_op op, dsm_handle handle, Size request_size,
						   void **impl_private, void **mapped_address,
						   Size *mapped_size, int elevel);
#endif
#ifdef USE_DSM_SYSV
static bool dsm_impl_sysv(dsm_op op, dsm_handle handle, Size request_size,
						  void **impl_private, void **mapped_address,
						  Size *mapped_size, int elevel);
#endif
#ifdef USE_DSM_WINDOWS
static bool dsm_impl_windows(dsm_op op, dsm_handle handle, Size request_size,
							 void **impl_private, void **mapped_address,
							 Size *mapped_size, int elevel);
#endif
#ifdef USE_DSM_MMAP
static bool dsm_impl_mmap(dsm_op op, dsm_handle handle, Size request_size,
						  void **impl_private, void **mapped_address,
						  Size *mapped_size, int elevel);
#endif
```

函数前置堆栈
```C
BaseInit
	InitCommunication
		CreateSharedMemoryAndSemaphores
			dsm_postmaster_startup
```


在数据库启动时首先通过各个模块所需内存，计算需要分配内存的总数
```C
		size = add_size(size, ProcGlobalShmemSize());
		...
		size = add_size(size, BackgroundWorkerShmemSize());
```

申请系统用的共享内存
PGSharedMemoryCreate
尝试申请共享内存shmget，如果申请失败尝试attatch到现有shmat
```C
		/* Try to create new segment */
		memAddress = InternalIpcMemoryCreate(NextShmemSegID, sysvsize);
		if (memAddress)
			break;				/* successful create and attach */
		/* Check shared memory and possibly remove and recreate */

		/*
		 * shmget() failure is typically EACCES, hence SHMSTATE_FOREIGN.
		 * ENOENT, a narrow possibility, implies SHMSTATE_ENOENT, but one can
		 * safely treat SHMSTATE_ENOENT like SHMSTATE_FOREIGN.
		 */
		shmid = shmget(NextShmemSegID, sizeof(PGShmemHeader), 0);
		if (shmid < 0)
		{
			oldhdr = NULL;
			state = SHMSTATE_FOREIGN;
		}
		else
			state = PGSharedMemoryAttach(shmid, NULL, &oldhdr);
```


dsm_impl_op
第一次申请映射内存时，按照指定的size映射，
进行attatch操作时，按照现有的文件size映射，而不是用户指定的size
```C
	if (op == DSM_OP_ATTACH)
	{
		struct stat st;
	}
	fstat(fd, &st);
	request_size = st.st_size;
```


内存映射保存在dsm_control_header结构体中
内存映射块的数量由 backend进程的数量有关
```C
	/* Determine size for new control segment. */
	maxitems = PG_DYNSHMEM_FIXED_SLOTS
		+ PG_DYNSHMEM_SLOTS_PER_BACKEND * MaxBackends;

typedef struct dsm_control_header
{
	uint32		magic;
	uint32		nitems;
	uint32		maxitems;
	dsm_control_item item[FLEXIBLE_ARRAY_MEMBER];
} dsm_control_header;
```

映射后保存在静态全局变量中
```C
static dsm_control_header *dsm_control;
```

## RELATED COMMITS
- https://github.com/postgres/postgres/commit/84b1c63ad41872792d47e523363fce1f0e230022
									
