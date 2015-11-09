##I/O Streams

###PooledByteStreams
PooledByteStreams是用于实现I/O流转型的辅助类，PooledByteStreams借助一个定长的临时字符数组将输入流的内容写到输出流上，默认的临时字符数组大小是16KB。   
PooledByteStreams提供了两种输入输出流的复制方法:   

#####1. 全部内容复制：
借助临时字符数组将所有输入流的内容复制到输出流上
```
  public long copy(final InputStream from, final OutputStream to) throws IOException {
    long count = 0;
    byte[] tmp = mByteArrayPool.get(mTempBufSize);
    try {
      while (true) {
        int read = from.read(tmp, 0, mTempBufSize);
        if (read == -1) {
          return count;
        }
        to.write(tmp, 0, read);
        count += read;
      }
    } finally {
      mByteArrayPool.release(tmp);
    }
  }
```

#####2. 定长内容的复制：
复制内容的长度不超过指定值。
```
  public long copy(
      final InputStream from,
      final OutputStream to,
      final long bytesToCopy) throws IOException {
    Preconditions.checkState(bytesToCopy > 0);
    long copied = 0;
    byte[] tmp = mByteArrayPool.get(mTempBufSize);
    try {
      while (copied < bytesToCopy) {
        int read = from.read(tmp, 0, (int) Math.min(mTempBufSize, bytesToCopy - copied));
        if (read == -1) {
          return copied;
        }
        to.write(tmp, 0, read);
        copied += read;
      }
      return copied;
    } finally {
      mByteArrayPool.release(tmp);
    }
  }
```
PooledByteStreams将指定的输出流内容写到指定输出流上，但不关心I/O流的具体实现，即不关心I/O流是Native内存上打开的输出输入流还是通过Java API所打开的I/O流。
___
###NativePooledByteBufferOutputStream
NativePooledByteBufferOutputStream是将指定内容写到Native内存中，而非由虚拟机所管理的Java堆内存。其构造方法如下：
```
  public NativePooledByteBufferOutputStream(NativeMemoryChunkPool pool, int initialCapacity) {
    super();
    Preconditions.checkArgument(initialCapacity > 0);
    mPool = Preconditions.checkNotNull(pool);
    mCount = 0;
    mBufRef = CloseableReference.of(mPool.get(initialCapacity), mPool);
  }
```
在构造方法中可以看到，NativePooledByteBufferOutputStream的创建会尝试获取一块Native内存，所有对NativePooledByteBufferOutputStream的操作均是在该块Native内存上进行的。   
我们需要关注的是如下方法的实现：   

(1). 把NativePooledByteBufferOutputStream所获取并打开的Native内存封装为一块NativePooledByteBuffer。
```
  public NativePooledByteBuffer toByteBuffer() {
    ensureValid();
    return new NativePooledByteBuffer(mBufRef, mCount);
  }
```

(2). 向Native内存中写入字符数组指定偏移和长度的内容
```
  public void write(byte[] buffer, int offset, int count) throws IOException {
    if (offset < 0 || count < 0 || offset + count > buffer.length) {
      throw new ArrayIndexOutOfBoundsException("length=" + buffer.length + "; regionStart=" + offset
          + "; regionLength=" + count);
    }
    ensureValid();
    realloc(mCount + count);
    mBufRef.get().write(mCount, buffer, offset, count);
    mCount += count;
  }
```
并且当Native内存大小不够时，重新申请一块合适大小的内存
```
  void realloc(int newLength) {
    ensureValid();
    /* Can the buffer handle @i more bytes, if not expand it */
    if (newLength <= mBufRef.get().getSize()) {
      return;
    }
    NativeMemoryChunk newbuf = mPool.get(newLength);
    mBufRef.get().copy(0, newbuf, 0, mCount);
    mBufRef.close();
    mBufRef = CloseableReference.of(newbuf, mPool);
  }
```
**NativePooledByteBufferOutputStream以OutputStream的形式，向使用者提供对Native内存块进行写操作的接口。**
___
###PooledByteBufferInputStream
**PooledByteBufferInputStream将Native内存块的内容打开为InputStream，用来向使用者提供对Native内存块进行读操作的接口。**其构造方法如下：
```
  public PooledByteBufferInputStream(PooledByteBuffer pooledByteBuffer) {
    super();
    Preconditions.checkArgument(!pooledByteBuffer.isClosed());
    mPooledByteBuffer = Preconditions.checkNotNull(pooledByteBuffer);
    mOffset = 0;
    mMark = 0;
  }
```
PooledByteBufferInputStream所提供的读的操作接口有：   

(1). 读取一个字节的内容
```
  public int read() {
    if (available() <= 0) {
      return -1;
    }
    return ((int) mPooledByteBuffer.read(mOffset++))  & 0xFF;
  }
```

(2). 读取内容到字节数组buffer中
```
  public int read(byte[] buffer) {
    return read(buffer, 0, buffer.length);
  }
```

(3). 读取从指定偏移位置开始指定大小的内容到字节数组buffer中
```
  public int read(byte[] buffer, int offset, int length) {
    if (offset < 0 || length < 0 || offset + length > buffer.length) {
      throw new ArrayIndexOutOfBoundsException(
          "length=" + buffer.length +
          "; regionStart=" + offset +
          "; regionLength=" + length);
    }

    final int available = available();
    if (available <= 0) {
      return -1;
    }

    if (length <= 0) {
      return 0;
    }

    int numToRead = Math.min(available, length);
    mPooledByteBuffer.read(mOffset, buffer, offset, numToRead);
    mOffset += numToRead;
    return numToRead;
  }
```

(4). 读指针跳过指定的字节大小
```
  public long skip(long byteCount) {
    Preconditions.checkArgument(byteCount >= 0);
    int skipped = Math.min((int) byteCount, available());
    mOffset += skipped;
    return skipped;
  }
```

(5). 设置标记位置
```
  public void mark(int readlimit) {
    mMark = mOffset;
  }
```
当调用reset()方法，读指针就会恢复到上次的标记位置
```
  public void reset() {
    mOffset = mMark;
  }
```