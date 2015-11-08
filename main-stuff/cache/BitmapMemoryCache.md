##BitmapMemoryCache
#####BitmapMemoryCacheFactory
&#8195;BitmapMemoryCache是通过工厂模式创建的。**++BitmapMemoryCacheFactory将实际的Cache向传入的ImageCacheStatsTracker注册后，创建了一个MemoryStatsTracker用来追踪缓存行为，当缓存发生了命中、丢失、新缓存项设置事件，将触发ImageCacheStatsTracker的回调++**。
```
  public static MemoryCache<CacheKey, CloseableImage> get(
    final CountingMemoryCache<CacheKey, CloseableImage> bitmapCountingMemoryCache,
    final ImageCacheStatsTracker imageCacheStatsTracker) {

    imageCacheStatsTracker.registerBitmapMemoryCache(bitmapCountingMemoryCache);

    MemoryCacheTracker memoryCacheTracker = new MemoryCacheTracker() {
      @Override
      public void onCacheHit() {
        imageCacheStatsTracker.onBitmapCacheHit();
      }

      @Override
      public void onCacheMiss() {
        imageCacheStatsTracker.onBitmapCacheMiss();
      }

      @Override
      public void onCachePut() {
        imageCacheStatsTracker.onBitmapCachePut();
      }
    };

    return new InstrumentedMemoryCache<>(bitmapCountingMemoryCache, memoryCacheTracker);
  }
```

#####InstrumentedMemoryCache
&#8195;InstrumentedMemoryCache实现了MemoryCache，它包括一个物理意义上的缓存和一个用来追踪缓存行为(如缓存命中)的Tracker。
```
  private final MemoryCache<K, V> mDelegate;
  private final MemoryCacheTracker mTracker;

  public InstrumentedMemoryCache(MemoryCache<K, V> delegate, MemoryCacheTracker tracker) {
    mDelegate = delegate;
    mTracker = tracker;
  }
```
&#8195;InstrumentedMemoryCache实现了MemoryCache的对缓存的操作接口：   
1. 获取缓存键对应的值，并记录命中或丢失事件
```
  @Override
  public CloseableReference<V> get(K key) {
    CloseableReference<V> result = mDelegate.get(key);
    if (result == null) {
      mTracker.onCacheMiss();
    } else {
      mTracker.onCacheHit();
    }
    return result;
  }
```   
2. 缓存键值对，并记录缓存事件
```
  @Override
  public CloseableReference<V> cache(K key, CloseableReference<V> value) {
    mTracker.onCachePut();
    return mDelegate.cache(key, value);
  }
```   
3. 移除给定描述的所有缓存项
```
  @Override
  public int removeAll(Predicate<K> predicate) {
    return mDelegate.removeAll(predicate);
  }
```   
4. 查找是否存在给定描述的缓存项
```
  @Override
  public boolean contains(Predicate<K> predicate) {
    return mDelegate.contains(predicate);
  }
```   

