##四、Fresco Native内存模型
在分析Native内存模型之前，首先提一下Fresco所使用的内存分为两种：由虚拟机维护的堆上内存，以及Fresco所维护的Native内存。对于未解码图片，Fresco将会将其图片数据保存在Native内存块中，而Native内存块创建和使用将由PoolFactory所提供的一系列模块来完成。   
此外，对于Bitmap图片，Android的Bitmap解码在匿名共享内存ashmem中，当需要对Bitmap图片进行渲染时，就会从ashmem中取出解码内容，当渲染结束时把它放回ashmem中，但是若在渲染期间对应的ashmem内存被释放，那么下次渲染时就必须对Bitmap重新进行解码，所以这里需要对对应的ashmem内存空间上锁，以避免对Bitmap解码的大量时间损耗所产生的卡顿问题。这就是Fresco中Bitmap的pinned purgeables。   
[关于PoolFactory](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/memory/PoolFactory.md)

[返回README](https://github.com/icemoonlol/fresco-research-stuff/blob/master/README.md)