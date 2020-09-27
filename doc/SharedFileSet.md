# SharedFileSet


tablespaces[8]中保存各个namespace的OID

每次调用SharedFileSetCreate函数根据传入文件名，计算出hash值，使用hash值选择namespace数组中的OID
```c
	uint32		hash = hash_any((const unsigned char *) name, strlen(name));
	return fileset->tablespaces[hash % fileset->ntablespaces];
```

所有申请的SharedFileSet都保存在静态全局变量filesetlist中
每次调用SharedFileSetUnregister/SharedFileSetDeleteAll会删除filesetlist中的元素
```c
	foreach(l, filesetlist)
	{
		SharedFileSet *fileset = (SharedFileSet *) lfirst(l);

		/* Remove the entry from the list */
		if (input_fileset == fileset)
		{
			filesetlist = foreach_delete_current(filesetlist, l);
			return;
		}
	}
```


可以被多个进程共享，使用dsm_segment等实现
```c
	/* Register our cleanup callback. */
	if (seg)
		on_dsm_detach(seg, SharedFileSetOnDetach, PointerGetDatum(fileset));

```

本次新增加可以由单独进程控制的SharedFileSet,只在当前进程结束时才会清理filesetlist
```c
	else
	{
		static bool registered_cleanup = false;

		if (!registered_cleanup)
		{
			/*
			 * We must not have registered any fileset before registering the
			 * fileset clean up.
			 */
			Assert(filesetlist == NIL);
			on_proc_exit(SharedFileSetDeleteOnProcExit, 0);
			registered_cleanup = true;
		}

		filesetlist = lcons((void *) fileset, filesetlist);
	}
```