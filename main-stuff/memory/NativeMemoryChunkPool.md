##NativeMemoryChunkPool
NativeMemoryChunkPool继承自BasePool<NativeMemoryChunk>，实现了Pool接口，以定长内存块为资源单位管理Native内存块资源。在分析NativeMemoryChunkPool之前我们必须明白什么是Pool？Pool是如何组织和实现的？

___
###Pool
Pool定义了资源池所必须实现的两个接口：
```
  V get(int size);
  void release(V value);
```
即从资源池中获取和释放资源。
___
###Bucket
资源池Pool通过一系列Bucket来维护空闲资源，而Bucket则代表了同等大小(同类)的所有空闲资源，可以看做资源池的每个子池。Bucket负责维护这些同类空闲资源的队列，当资源池收到一个资源请求时，会挑选最合适大小的Bucket，将资源申请交由Bucket进行处理，而Bucket就会取下一个空闲资源返回给请求者。其构造方法如下：
```
  public Bucket(int itemSize, int maxLength, int inUseLength) {
    Preconditions.checkState(itemSize > 0);
    Preconditions.checkState(maxLength >= 0);
    Preconditions.checkState(inUseLength >= 0);

    mItemSize = itemSize;
    mMaxLength = maxLength;
    mFreeList = new LinkedList();
    mInUseLength = inUseLength;
  }
```
各成员的作用如下：   
- mItemSize：所拥有资源的字节大小
- mMaxLength：资源队列的最大长度
- mFreeList：空闲资源队列
- mInUseLength：当前被使用的资源数

我们看看对Bucket作为分池是如何对同类资源进行组织和管理的：   
#####1. 空闲资源的获取   
(1).调用Bucket的get()将尝试获取空闲资源，若获取成功，则更新被使用资源计数
```
  public V get() {
    V value = pop();
    if (value != null) {
      mInUseLength++;
    }
    return value;
  }
```   
(2).获取一个空闲资源的pop()方法定义如下，即取出空闲资源链表的表头元素：
```
  public V pop() {
    return (V) mFreeList.poll();
  }
```   
#####2. 占用资源的释放   
(1).调用Bucket的release()将释放被占用资源，并更新被使用资源计数
```
  public void release(V value) {
    Preconditions.checkNotNull(value);
    Preconditions.checkState(mInUseLength > 0);
    mInUseLength--;
    addToFreeList(value);
  }
```   
(2).释放一个空闲资源的addToFreeList()方法定义如下，即将该资源添加到空闲资源链表的表尾：
```
  void addToFreeList(V value) {
    mFreeList.add(value);
  }
```
___
###BasePool
BasePool是资源或对象池的基本实现类，资源池被组织为一个Map，Map的每一项都是一个不同的固定大小的空闲资源的子池。BasePool的构造方法如下：
```
  public BasePool(
      MemoryTrimmableRegistry memoryTrimmableRegistry,
      PoolParams poolParams,
      PoolStatsTracker poolStatsTracker) {
    mMemoryTrimmableRegistry = Preconditions.checkNotNull(memoryTrimmableRegistry);
    mPoolParams = Preconditions.checkNotNull(poolParams);
    mPoolStatsTracker = Preconditions.checkNotNull(poolStatsTracker);

    // initialize the buckets
    mBuckets = new SparseArray<Bucket<V>>();
    initBuckets(new SparseIntArray(0));

    mInUseValues = Sets.newIdentityHashSet();

    mFree = new Counter();
    mUsed = new Counter();
  }
```   
其中，mBuckets为一个保存资源大小和块数的稀疏数组(SparseArray)，在initBuckets()中将根据具体的资源池构造参数，以资源块大小为键，以这些同等大小资源块集合(Bucket)为值进行初始化。   
mInUseValues为一个记录被使用的资源集合。   
mFree、mUsed分别用来记录空闲块和被使用块的块数和大小信息。   
```
  private synchronized void initBuckets(SparseIntArray inUseCounts) {
    Preconditions.checkNotNull(inUseCounts);

    // clear out all the buckets
    mBuckets.clear();

    // create the new buckets
    final SparseIntArray bucketSizes = mPoolParams.bucketSizes;
    if (bucketSizes != null) {
      for (int i = 0; i < bucketSizes.size(); ++i) {
        final int bucketSize = bucketSizes.keyAt(i);
        final int maxLength = bucketSizes.valueAt(i);
        int bucketInUseCount = inUseCounts.get(bucketSize, 0);
        mBuckets.put(
            bucketSize,
            new Bucket<V>(
                getSizeInBytes(bucketSize),
                maxLength,
                bucketInUseCount));
      }
      mAllowNewBuckets = false;
    } else {
      mAllowNewBuckets = true;
    }
  }
```
initBuckets()使用指定的资源池参数来初始化资源池所拥有的各个子池的资源大小，以传入的一个SparseIntArray来初始化对应大小资源的当前使用量。在资源池构造方法中，资源池各个子池资源的使用量均为0。   
#####资源池的使用和管理

#####1. 当有新的资源请求到来时，将调用get(int size)方法尝试获取空闲资源。   
(1).首先要保证资源池的总大小小于软上限，随后根据申请的资源大小，由具体的资源池实现类来计算适合该资源的整块资源大小(资源块的对齐只能由具体资源池来决定)。
```
    ensurePoolSizeInvariant();
    int bucketedSize = getBucketedSize(size);
    int sizeInBytes = -1;
```   
(2).找到对应资源大小的子池(Bucket)，并尝试获取一个空闲资源，若获取成功，那么将其保存到正在被使用的集合mInUseValues中，并更新mUsed和mFree计数
```
    synchronized (this) {
      Bucket<V> bucket = getBucket(bucketedSize);
      if (bucket != null) {
        V value = bucket.get();
        if (value != null) {
          Preconditions.checkState(mInUseValues.add(value));
          bucketedSize = getBucketedSizeForValue(value);
          sizeInBytes = getSizeInBytes(bucketedSize);
          mUsed.increment(sizeInBytes);
          mFree.decrement(sizeInBytes);
          mPoolStatsTracker.onValueReuse(sizeInBytes);
          logStats();
          return value;
        }
      }
```   
(3).在Bucket的初始化过程中，并没有立即初始化所有资源，因为这会造成内存资源的极大浪费，只有在需要使用的时候，才会初始化资源并返回给申请者使用。若无法从空闲资源队列中获取空闲资源，那么就会在资源上限的限制下尝试资源的扩充，如果可以扩充资源，那么就会更新mUsed和Bucket的资源计数，并准备初始化新的资源。这里乐观估计资源初始化不会失败，所以先更新计数，后执行资源的初始化。
```
      sizeInBytes = getSizeInBytes(bucketedSize);
      if (!canAllocate(sizeInBytes)) {
        throw new PoolSizeViolationException(
            mPoolParams.maxSizeHardCap,
            mUsed.mNumBytes,
            mFree.mNumBytes,
            sizeInBytes);
      }
      // Optimistically assume that allocation succeeds - if it fails, we need to undo those changes
      mUsed.increment(sizeInBytes);
      if (bucket != null) {
        bucket.incrementInUseCount();
      }
    }//end synchronized
```   
(4).由资源池的具体实现类来进行对应资源的初始化。若失败则进行计数的回滚。注意，**这里的alloc操作时在同步块之外的，因为内存的申请工作可能开销非常大，从而会阻塞其他线程对资源池的操作**。当两个线程都希望获取该资源时，都会尝试初始化新的资源，而新资源的申请不会破坏资源池的管理。但是可能因为这样，导致多个线程都初始化了一份资源导致资源池过大，于是有了下面对资源池大小的检查。
```
 V value = null;
    try {
      value = alloc(bucketedSize);
    } catch (Throwable e) {
      synchronized (this) {
        mUsed.decrement(sizeInBytes);
        Bucket<V> bucket = getBucket(bucketedSize);
        if (bucket != null) {
          bucket.decrementInUseCount();
        }
      }
      Throwables.propagateIfPossible(e);
    }
```   
(5).若新资源的初始化使得资源池大小超过软上限，那么就需要对资源池进行裁剪。关于裁剪资源池的部分后面再进行分析。
```
    synchronized(this) {
      Preconditions.checkState(mInUseValues.add(value));
      // If we're over the pool's max size, try to trim the pool appropriately
      trimToSoftCap();
      mPoolStatsTracker.onAlloc(sizeInBytes);
      logStats();
    }
    return value;
```
#####2. 当应用程序使用完资源后，将调用release()释放资源   
之前分析到，Bucket资源队列设置了空闲资源的个数上限，若多个应用程序创建并使用同一种资源，那么使用完后，会尝试将该资源放到空闲资源队列中，但是若资源个数超出上限，那么将立刻释放该资源。   
release()首先将获取该资源对应的Bucket:
```
    final int bucketedSize = getBucketedSizeForValue(value);
    final int sizeInBytes = getSizeInBytes(bucketedSize);
    synchronized (this) {
      final Bucket<V> bucket = getBucket(bucketedSize);
```
接下来，有以下几种情况：   
(1).从mInUseValues中移除并释放该资源失败，则直接释放该资源(这个资源并不是该资源池所初始化的，不受其管理)
```
      if (!mInUseValues.remove(value)) {
        FLog.e(
            TAG,
            "release (free, value unrecognized) (object, size) = (%x, %s)",
            System.identityHashCode(value),
            bucketedSize);
        free(value);
        mPoolStatsTracker.onFree(sizeInBytes);
      }
```   
(2).若找不到对应的Bucket(可能过大)，或Bucket的资源块数过多(若已使用的资源由用户释放后全部放到空闲资源队列中则超出资源个数限制)，或资源池大小超过软限制，或该资源不可以被重新利用，**那么只会更新Bucket和mUsed的资源计数并直接释放该资源，而不会将其加入到资源池/空闲资源队列中，即不再会使用该资源**。
```
        if (bucket == null ||
            bucket.isMaxLengthExceeded() ||
            isMaxSizeSoftCapExceeded() ||
            !isReusable(value)) {
          if (bucket != null) {
            bucket.decrementInUseCount();
          }
          free(value);
          mUsed.decrement(sizeInBytes);
          mPoolStatsTracker.onFree(sizeInBytes);
        }
```   
(3).否则，将该资源加入到空闲资源队列中，并在之前的计数基础上增加mFree的统计计数
```
        else {
          bucket.release(value);
          mFree.increment(sizeInBytes);
          mUsed.decrement(sizeInBytes);
          mPoolStatsTracker.onValueRelease(sizeInBytes);
        }
```

#####资源池内存用量的裁剪
资源池裁剪与其他缓冲类似，分为一般情况下的trimToSize()，即裁剪到合适大小，以及极端情况下的trimToNothing()，即完全清空。   
1.定额裁剪(trimToSize)：   
(1).裁剪量的计算，若已使用的空间大小大于目标大小，那么，应裁剪到已使用空间的大小，否则，裁剪到目标大小。
```
    int bytesToFree = Math.min(mUsed.mNumBytes + mFree.mNumBytes - targetSize, mFree.mNumBytes);
    if (bytesToFree <= 0) {
      return;
    }
```
(2).从小尺寸资源到大尺寸资源，按空闲资源队列顺序进行裁剪，直至到达合适的大小。
```
    for (int i = 0; i < mBuckets.size(); ++i) {
      if (bytesToFree <= 0) {
        break;
      }
      Bucket<V> bucket = mBuckets.valueAt(i);
      while (bytesToFree > 0) {
        V value = bucket.pop();
        if (value == null) {
          break;
        }
        free(value);
        bytesToFree -= bucket.mItemSize;
        mFree.decrement(bucket.mItemSize);
      }
    }
```
2.全额裁剪(trimToNothing)：   
(1).需要进行裁剪的Bucket是空闲资源队列不为空的Bucket，并且将其资源大小和正在被使用的资源个数保存在inUseCounts以进行资源池的重置(空闲资源全额裁剪的不变信息需要保留下来)。
```
    final List<Bucket<V>> bucketsToTrim = new ArrayList<>(mBuckets.size());
    final SparseIntArray inUseCounts = new SparseIntArray();
    synchronized (this) {
      for (int i = 0; i < mBuckets.size(); ++i) {
        final Bucket<V> bucket = mBuckets.valueAt(i);
        if (bucket.getFreeListSize() > 0) {
          bucketsToTrim.add(bucket);
        }
        inUseCounts.put(mBuckets.keyAt(i), bucket.getInUseCount());
      }
      // reinitialize the buckets
      initBuckets(inUseCounts);
      // free up the stats
      mFree.reset();
      logStats();
    }
```
(2).清空空闲资源
```
    onParamsChanged();
    for (int i = 0; i < bucketsToTrim.size(); ++i) {
      final Bucket<V> bucket = bucketsToTrim.get(i);
      while (true) {
        V item = bucket.pop();
        if (item == null) {
          break;
        }
        free(item);
      }
    }
```
___
###NativeMemoryChunkPool
NativeMemoryChunkPool的构造方法如下：
```
  public NativeMemoryChunkPool(
      MemoryTrimmableRegistry memoryTrimmableRegistry,
      PoolParams poolParams,
      PoolStatsTracker nativeMemoryChunkPoolStatsTracker) {
    super(memoryTrimmableRegistry, poolParams, nativeMemoryChunkPoolStatsTracker);
    SparseIntArray bucketSizes = poolParams.bucketSizes;
    mBucketSizes = new int[bucketSizes.size()];
    for (int i = 0; i < mBucketSizes.length; ++i) {
      mBucketSizes[i] = bucketSizes.keyAt(i);
    }
    initialize();
  }
```
bucketSizes是由创建参数DefaultNativeMemoryChunkPoolParams所指定的一个关于缓冲块大小和块数的稀疏数组。(注：虽然可以用HashMap< Interger, Integer >来保存缓冲块大小和块数关系，但是使用SparseArray具有更高的性能(其实这里没有几项))。根据默认配置对于1KB、2KB、...、128KB大小的缓冲块都各有5块，对于256KB、512KB、1024KB的缓冲块各有2块。   

同时DefaultNativeMemoryChunkPoolParams还定义了Native内存使用量的软上线和硬上线：   

```
  private static int getMaxSizeSoftCap() {
    final int maxMemory = (int)Math.min(Runtime.getRuntime().maxMemory(), Integer.MAX_VALUE);
    if (maxMemory < 16 * ByteConstants.MB) {
      return 3 * ByteConstants.MB;
    } else if (maxMemory < 32 * ByteConstants.MB) {
      return 6 * ByteConstants.MB;
    } else {
      return 12 * ByteConstants.MB;
    }
  }
```   
maxMemory为虚拟机能从系统中获取内存最大大小。硬上线则比软上线要大一些：
```
  private static int getMaxSizeHardCap() {
    final int maxMemory = (int) Math.min(Runtime.getRuntime().maxMemory(), Integer.MAX_VALUE); 
    if (maxMemory < 16 * ByteConstants.MB) {
      return maxMemory / 2;
    } else {
      return maxMemory / 4 * 3;
    }
  }
```   
下面关注NativeMemoryChunkPool的关于内存块的一些业务实现：   

#####1. 寻找最小的可以满足申请内存大小的内存块大小
```
  protected int getBucketedSize(int requestSize) {
    int intRequestSize = requestSize;
    if (intRequestSize <= 0) {
      throw new InvalidSizeException(requestSize);
    }
    // find the smallest bucketed size that is larger than the requested size
    for (int bucketedSize : mBucketSizes) {
      if (bucketedSize >= intRequestSize) {
        return bucketedSize;
      }
    }
    // requested size doesn't match our existing buckets - just return the requested size
    // this will eventually translate into a plain alloc/free paradigm
    return requestSize;
  }
```

#####2. Native内存块分配将创建一个新的NativeMemoryChunk对象
```
  protected NativeMemoryChunk alloc(int bucketedSize) {
    return new NativeMemoryChunk(bucketedSize);
  }
```

#####3. Native内存块释放
```
  protected void free(NativeMemoryChunk value) {
    Preconditions.checkNotNull(value);
    value.close();
  }
```
关于NativeMemoryChunk的创建和其他操作，参考[NativeMemoryChunk](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/memory/NativeMemoryChunk.md)