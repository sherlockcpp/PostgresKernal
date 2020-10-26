
smgropen时创建SMgrRelation结构体
smgrclose时删除结构体

smgropen时创建的SMgrRelation结构体都会保存在静态全局的hash表SMgrRelationHash中
并且会用静态全局的链表unowned_relns保存尚未具备owner的SMgrRelation


BufferDescriptors
保存shared buffer描述信息的全局变量

BufferBlocks
保存shared buffer实际数据的全局变量

void
InitBufferPool(void)
初始化shared buffer用的pool

ReadBuff
ReadBufferExtended
ReadBufferWithoutRelcache
	ReadBuffer_common
		BufferAlloc
			StrategyGetBuffer
			从freelist中选择一个空闲的buffer
			如果freelist中没有可用的buffer，进行时钟算法，替换页面，空出新的buffer
	尝试读取的页面已经在buffer pool的场合如果不需要extend直接返回，需要extent的调用smgrextent
	页面不存在的场合调用smgrread读取，或者smgrextent扩展

