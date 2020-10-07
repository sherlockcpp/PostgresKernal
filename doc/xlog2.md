```c
pg_replication_slot_advance
	pg_logical_replication_slot_advance
		XLogBeginRead
		while (ctx->reader->EndRecPtr < moveto)
		{
			record = XLogReadRecord(ctx->reader, &errm);
			LogicalDecodingProcessRecord(ctx, ctx->reader);
		}
```

Xloginsert
对于REGBUF_FORCE_IMAGE标志的注册数据，需要将整个页面写入wal中，这个时候不需要再写入更新的数据
对于整个页面写入的情况，可以通过wal_compress选项控制是否对整个页面进行压缩
```c
/* Determine if this block needs to be backed up */
if (regbuf->flags & REGBUF_FORCE_IMAGE)
	needs_backup = true;
...
needs_data = !needs_backup;

if (include_image)
{
	rdt_datas_last->data = page;
	rdt_datas_last->len = BLCKSZ;
}
```

```c
ReadCheckpointRecord
	ReadRecord
		XLogReadRecord
			ReadPageInternal
				XLogPageRead
					WaitForWALToBecomeAvailable
					具备5中状态保存，wal数据的来源保存在currentSource
	/*
	 * 1. Read from either archive or pg_wal (XLOG_FROM_ARCHIVE), or just
	 *	  pg_wal (XLOG_FROM_PG_WAL)
	 * 2. Check trigger file
	 * 3. Read from primary server via walreceiver (XLOG_FROM_STREAM)
	 * 4. Rescan timelines
	 * 5. Sleep wal_retrieve_retry_interval milliseconds, and loop back to 1.
	 * /
```

recoveryStopsBefore
```c
检查是否可以停止point in time recovery
```

注册XLogPageRead的代码
```c
	/* Set up XLOG reader facility */
	MemSet(&private, 0, sizeof(XLogPageReadPrivate));
	xlogreader =
		XLogReaderAllocate(wal_segment_size, NULL,
						   XL_ROUTINE(.page_read = &XLogPageRead,
									  .segment_open = NULL,
									  .segment_close = wal_segment_close),
						   &private);
```


CheckForStandbyTrigger
```c
检查用户创建的升格主从的文件是否存在，或者是否接受到升格指令
```

MarkBufferDirtyHint
```c
在一些情况下，指向对buffer做一些bit位的简单修改，对于这些修改，即使写入失败也可以简单恢复不需要写wal，
但是在checksum启动的情况下，即使是bit位的修改也可能会造成不一致，
所以在这种情况下如果full_page启动的场合会进行 full page image write。
```

RemoveOldXlogFiles
```c
删除或复用已经落盘的wal文件
	RemoveXlogFile
	wal_recycle参数支持的情况下，可以将文件复用位新的wal文件

```
