##NativePooledByteBuffer

###NativePooledByteBufferFactory
NativePooledByteBuffer是使用工厂模式构造的，NativePooledByteBufferFactory中对于NativePooledByteBuffer的获取有以下几种形式：
#####按大小构造
构造一块固定大小的Native内存字节缓冲。先从Native内存池中获取一个内存块，再用该内存块构造一个NativePooledByteBuffer。按大小构造的Native内存将直接使用NativePooledByteBuffer的构造方法创建即可。
```
  public NativePooledByteBuffer newByteBuffer(int size) {
    Preconditions.checkArgument(size > 0);
    CloseableReference<NativeMemoryChunk> chunkRef = CloseableReference.of(mPool.get(size), mPool);
    try {
      return new NativePooledByteBuffer(chunkRef, size);
    } finally {
      chunkRef.close();
    }
  }
```
#####按输入流构造
构造一块包含输入流所有数据内容的Native内存字节缓冲。这也是ImagePipeline常用的一种方式，即将用户打开的输入流如FileInputStream等保存到Native内存中。对于用户打开的InputStream，将其写入到Native内存中需要借助NativePooledByteBufferOutputStream来构造，并调用newByteBuf(InputStream inputStream,NativePooledByteBufferOutputStream outputStream)来写入数据。
```
  public NativePooledByteBuffer newByteBuffer(InputStream inputStream) throws IOException {
    NativePooledByteBufferOutputStream outputStream = new NativePooledByteBufferOutputStream(mPool);
    try {
      return newByteBuf(inputStream, outputStream);
    } finally {
      outputStream.close();
    }
  }
```
该方法根据借助PooledByteStreams将用户打开的InputStream内容写入到Native内存中
```
  NativePooledByteBuffer newByteBuf(
      InputStream inputStream,
      NativePooledByteBufferOutputStream outputStream)
      throws IOException {
    mPooledByteStreams.copy(inputStream, outputStream);
    return outputStream.toByteBuffer();
  }
```
#####按输入流+大小构造
构造一块给定初始大小，包含输入流所有数据内容的Native内存字节缓冲
```
  public NativePooledByteBuffer newByteBuffer(InputStream inputStream, int initialCapacity)
      throws IOException {
    NativePooledByteBufferOutputStream outputStream =
        new NativePooledByteBufferOutputStream(mPool, initialCapacity);
    try {
      return newByteBuf(inputStream, outputStream);
    } finally {
      outputStream.close();
    }
  }

```
#####按字节数组构造
与按输入流构造类似，不过由于写入数据已经在字节数组中，所以不需要借助PooledByteStreams，直接使用outputStream的写接口就可以完成。
```
  public NativePooledByteBuffer newByteBuffer(byte[] bytes) {
    NativePooledByteBufferOutputStream outputStream =
        new NativePooledByteBufferOutputStream(mPool, bytes.length);
    try {
      outputStream.write(bytes, 0, bytes.length);
      return outputStream.toByteBuffer();
    } catch (IOException ioe) {
      throw Throwables.propagate(ioe);
    } finally {
      outputStream.close();
    }
  }
```

联系BufferedDiskCache中读磁盘文件的操作，BufferedDiskCache以文件输入流为内容申请了一块Native内存来存放EncodedImage，而EncodedImage的引用将指向这片Native内存。其过程如下：   
Native OutputStream会通过NativeMemoryChunkPool申请一块Native内存，而把Java InputStream的内容借助PooledByteStreams通过字符数组写到Native OutputStream中。   
这里通过下图简单地描述Fresco的Native内存体系：   
![](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/resources/img/native_mem_hierarchy.png)
###NativePooledByteBuffer
NativePooledByteBuffer实现了PooledByteBuffer，仅提供了读接口，对NativePooledByteBuffer的写工作将由NativePooledByteBufferOutputStream向Native内存写入，以实现对Native内容的保护，防止用户对Native内容的冲突读写操作。

#####1. 读取指定偏移处字节
```
  @Override
  public synchronized byte read(int offset) {
    ensureValid();
    Preconditions.checkArgument(offset >= 0);
    Preconditions.checkArgument(offset < mSize);
    return mBufRef.get().read(offset);
  }
```

#####2. 读取指定偏移和长度的内容到字节缓冲中
```
  @Override
  public synchronized void read(int offset, byte[] buffer, int bufferOffset, int length) {
    ensureValid();
    // We need to make sure that PooledByteBuffer's length is preserved.
    // Al the other bounds checks will be performed by NativeMemoryChunk.read method.
    Preconditions.checkArgument(offset + length <= mSize);
    mBufRef.get().read(offset, buffer, bufferOffset, length);
  }
```

[返回PoolFactory](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/memory/PoolFactory.md)