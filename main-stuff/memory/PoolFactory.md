##PoolFactory
PoolFactory提供了由Fresco管理的不同类型的内存池。
```
  private BitmapPool mBitmapPool;
  private FlexByteArrayPool mFlexByteArrayPool;
  private NativeMemoryChunkPool mNativeMemoryChunkPool;
  private PooledByteBufferFactory mPooledByteBufferFactory;
  private PooledByteStreams mPooledByteStreams;
  private SharedByteArray mSharedByteArray;
  private ByteArrayPool mSmallByteArrayPool;

```
ImagePipeline中，未解码图片EncodedImage将保存在不受虚拟机管理的Native内存块中。而接下来，我们将以PooledByteBuffer为例，来认识Native内存块是如何管理和使用的：  
___
首先了解PooledByteBuffer的管理和使用需要哪些模块：
1. PooledByteBuffer使用的是Native内存块，从而使得图像占用内存不受虚拟机管理，而减小GC大体积对象的压力。而PooledByteBuffer将由NativePooledByteBufferFactory透明地提供给使用者。   
```
  public PooledByteBufferFactory getPooledByteBufferFactory() {
    if (mPooledByteBufferFactory == null) {
      mPooledByteBufferFactory = new NativePooledByteBufferFactory(
          getNativeMemoryChunkPool(),
          getPooledByteStreams());
    }
    return mPooledByteBufferFactory;
  }
```
2. PooledByteBuffer对Native内存块的获取依赖于Native内存池，NativeMemoryChunkPool的获取如下：
```
  public NativeMemoryChunkPool getNativeMemoryChunkPool() {
    if (mNativeMemoryChunkPool == null) {
      mNativeMemoryChunkPool = new NativeMemoryChunkPool(
          mConfig.getMemoryTrimmableRegistry(),
          mConfig.getNativeMemoryChunkPoolParams(),
          mConfig.getNativeMemoryChunkPoolStatsTracker());
    }
    return mNativeMemoryChunkPool;
  }
```
3. PooledByteBuffer同时还需要和Java IO流进行交互，所以还依赖于PooledByteStreams，PooledByteStreams的获取如下:
```
  public PooledByteStreams getPooledByteStreams() {
    if (mPooledByteStreams == null) {
      mPooledByteStreams = new PooledByteStreams(getSmallByteArrayPool());
    }
    return mPooledByteStreams;
  }
```
4. PooledByteStreams需要一块ByteArrayPool来进行辅助Native IO流与Java IO流之间的交互操作。
```
  public SharedByteArray getSharedByteArray() {
    if (mSharedByteArray == null) {
      mSharedByteArray = new SharedByteArray(
          mConfig.getMemoryTrimmableRegistry(),
          mConfig.getFlexByteArrayPoolParams());
    }
    return mSharedByteArray;
  }
```
___
PooledByteBuffer分别依赖NativeMemoryChunkPool来获取Native内存块，依赖PooledByteStreams和Native I/O流来完成与用户打开的IO流的读写交互。后面我们分别将探讨NativeMemoryChunkPool和PooledByteStreams的实现。
- [NativeMemoryChunkPool](http://)
- [I/O Streams](http://)

在了解完Fresco是如何获取和管理Native内存，以及是如何对Native内存进行读写后，我们来看NativePooledByteBuffer的创建和使用：    
- [NativePooledByteBuffer](http://)

