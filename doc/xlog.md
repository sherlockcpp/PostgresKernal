
尝试从page中读取一整个xlogrecord
page的头部时结构体XLogPageHeader
```
readOff = ReadPageInternal(state, targetPagePtr,
/* Wait for the next page to become available */
							Min(total_len - gotlen + SizeOfXLogShortPHD,
								XLOG_BLCKSZ));
pageHeader = (XLogPageHeader) state->readBuf;
```

```c
有可能没有读取全部的XLogRecord,根据现有的xlogrecord信息获取需要读取的总长度

record = (XLogRecord *) (state->readBuf + RecPtr % XLOG_BLCKSZ);
total_len = record->xl_tot_len;
```
```c
在没有读取完所有xlogrecord信息的场合 持续读取
do{
ReadPageInternal(...)
} while (gotlen < total_len);
```



xlog_redo
读取wal并开始恢复

smgr_redo

```c
XLogRegisterBuffer
将Buffer注册进registered_buffer*类型全局数组registered_buffers中
```
```c
XLogRegisterBufdata
从XLogRegisterData注册的rdatas数组中取出数据，和需要注册的数据一起注册进registered_buffers数组中
```

```c
XLogRegisterData
更新全局数组rdatas，同时将数据添加入XLogRecData *类型全局变量mainrdata_last中
mainrdata_last实际上是一个链表，每次调用XLogRegisterData数据都会创建一个新的cell保存到链表中
注册的数据从类型上属于main data
```
```c
XLogRecordAssemble
将registered_buffers中的数组全都导入一个保存XLogRecData *类型的链表中，以便之后的insertxlog操作
```
```c
XLogWrite
将之前调用xloginsert插入wal buffer的所有wal写入磁盘
```

```c
每次尝试写入xlog时，会将当前xlog的头部信息保存在全局变量hdr_scratch中
根据不同的xlog类型可能包括
XLogRecord
XLogRecordBlockHeader
XLogRecordBlockImageHeader
XLogRecordBlockCompressHeader

(backup/insert/...)

		/* Ok, copy the header to the scratch buffer */
		memcpy(scratch, &bkpb, SizeOfXLogRecordBlockHeader);
		scratch += SizeOfXLogRecordBlockHeader;
		if (include_image)
		{
			memcpy(scratch, &bimg, SizeOfXLogRecordBlockImageHeader);
			scratch += SizeOfXLogRecordBlockImageHeader;
			if (cbimg.hole_length != 0 && is_compressed)
			{
				memcpy(scratch, &cbimg,
					   SizeOfXLogRecordBlockCompressHeader);
				scratch += SizeOfXLogRecordBlockCompressHeader;
			}
		}
```

```c
执行插入时xlog信息的构建的流程
xl_heap_insert xlrec;
xl_heap_header xlhdr;

1.注册xl_heap_insert
XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);

2.注册xl_heap_header
XLogRegisterBufData(0, (char *) &xlhdr, SizeOfHeapHeader);

3.注册实际插入的数据
XLogRegisterBufData(0,
					(char *) heaptup->t_data + SizeofHeapTupleHeader,
					heaptup->t_len - SizeofHeapTupleHeader);

最终数据分布
XlogRecord : xl_heap_header : data : xl_heap_insert : main data
```

