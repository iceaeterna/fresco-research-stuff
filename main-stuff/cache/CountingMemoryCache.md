##### CountingMemoryCache
&#8195;CountingMemoryCache的构造方法如下：
```
  public CountingMemoryCache(
      ValueDescriptor<V> valueDescriptor,
      CacheTrimStrategy cacheTrimStrategy,
      Supplier<MemoryCacheParams> memoryCacheParamsSupplier) {
    mValueDescriptor = valueDescriptor;
    mExclusiveEntries = new CountingLruMap<>(wrapValueDescriptor(valueDescriptor));
    mCachedEntries = new CountingLruMap<>(wrapValueDescriptor(valueDescriptor));
    mCacheTrimStrategy = cacheTrimStrategy;
    mMemoryCacheParamsSupplier = memoryCacheParamsSupplier;
    mMemoryCacheParams = mMemoryCacheParamsSupplier.get();
    mLastCacheParamsCheck = SystemClock.elapsedRealtime();
  }

```
- CountingMemoryCache持有两个缓冲的LruMap：mExclusiveEntries和mCachedEntries，其中mExclusiveEntries保存了不再被使用的缓存项，这些缓存项可以随时被移除，而mCachedEntries则保存了所有的缓存项。
- mMemoryCacheParams通过传入的MemoryCacheParamsSupplier所提供，用来对缓存大小进行控制，而缓存也会适时地定期检查缓存配置是否发生变化，以对缓存及时进行调整。
- mCacheTrimStrategy为缓存大小超出期望值后的裁剪策略。
- mLastCacheParamsCheck为上次缓存配置变化检查的时间。
    
此外，CountingMemoryCache的缓存项构造方法如下：
```
    private Entry(K key, CloseableReference<V> valueRef) {
      this.key = Preconditions.checkNotNull(key);
      this.valueRef = Preconditions.checkNotNull(CloseableReference.cloneOrNull(valueRef));
      this.clientCount = 0;
      this.isOrphan = false;
    }
```   
需要注意的是，**Entry通过clientCount描述缓存项的使用计数，若没有任何使用者，那么Entry将会被放到可能随时被移除的mExclusiveEntries中以待观察，若该缓存项被命中，那么从观察名单mExclusiveEntries中移除该缓存项，否则，当缓存需要被裁剪时、新的缓存内容到来时，那么这个旧的、无效的缓存项就会被标记为Orphan并移除。**   
接下来来看CountingMemoryCache缓存操作的实现：   
##### 1. cache()：缓存键值对   
(1).缓存配置的定期检查
```
    Preconditions.checkNotNull(key);
    Preconditions.checkNotNull(valueRef);
    maybeUpdateCacheParams();
```
(2).对于新加入的缓存项，如果存在对应的过期缓存项，那么将LruMap中移除旧的缓存项，**(即将其标记为Orphan后准备释放。对于标记为Orphan的缓存项，如果没有任何Client引用它，那么该缓存项的引用和占用的内存将被释放）**
```
   CloseableReference<V> oldRefToClose = null;
    CloseableReference<V> clientRef = null;
    synchronized (this) {
      // remove the old item (if any) as it is stale now
      mExclusiveEntries.remove(key);
      Entry<K, V> oldEntry = mCachedEntries.remove(key);
      if (oldEntry != null) {
        makeOrphan(oldEntry);
        oldRefToClose = referenceToClose(oldEntry);
      }
```
(3).**CountingMemoryCache对缓存活跃项的项数、大小和单个缓存项大小进行了限制**，canCacheNewValue将判断是否可以保存该缓存项，如果可以则将其保存到mCachedEntries中、并返回缓存项的引用
```
      if (canCacheNewValue(valueRef.get())) {
        Entry<K, V> newEntry = Entry.of(key, valueRef);
        mCachedEntries.put(key, newEntry);
        clientRef = newClientReference(newEntry);
      }
    }
    CloseableReference.closeSafely(oldRefToClose);
```
&#8195;其中newClientReference将增加缓存项的使用者计数，并注册了一个使用者释放对缓存项引用的回调方法releaseClientReference()   
```
  private synchronized CloseableReference<V> newClientReference(final Entry<K, V> entry) {
    increaseClientCount(entry);
    return CloseableReference.of(
        entry.valueRef.get(),
        new ResourceReleaser<V>() {
          @Override
          public void release(V unused) {
            releaseClientReference(entry);
          }
        });
  }
```
&#8195;**releaseClientReference()将减小缓存项的使用者计数，如果缓存项的使用计数变为0，那么就需要把该项放到mExclusiveEntries中去**。
```
  private void releaseClientReference(final Entry<K, V> entry) {
    Preconditions.checkNotNull(entry);
    CloseableReference<V> oldRefToClose;
    synchronized (this) {
      decreaseClientCount(entry);
      maybeAddToExclusives(entry);
      oldRefToClose = referenceToClose(entry);
    }
    CloseableReference.closeSafely(oldRefToClose);
    maybeUpdateCacheParams();
    maybeEvictEntries();
  }
```
&#8195;在对缓存进行过cache()操作后，缓存达到一个检查配置的安全点，那么可能由于缓存配置的变化，导致新的缓存配置下活跃缓存项数或大小超出限制，或mExclusiveEntries缓存项数超出限制，那么就需要裁剪mExclusiveEntries。   
```
  private void maybeEvictEntries() {
    ArrayList<Entry<K, V>> oldEntries;
    synchronized (this) {
      int maxCount = Math.min(
          mMemoryCacheParams.maxEvictionQueueEntries,
          mMemoryCacheParams.maxCacheEntries - getInUseCount());
      int maxSize = Math.min(
          mMemoryCacheParams.maxEvictionQueueSize,
          mMemoryCacheParams.maxCacheSize - getInUseSizeInBytes());
      oldEntries = trimExclusivelyOwnedEntries(maxCount, maxSize);
      makeOrphans(oldEntries);
    }
    maybeClose(oldEntries);
  }
```
&#8195;trimExclusivelyOwnedEntries()将从mExclusiveEntries和mCachedEntries中移除LruMap中第一个缓存项，直至mExclusiveEntries的缓存项数和占用空间大小低于期望值，**以为活跃项的缓存腾出空间**。
```
  private synchronized ArrayList<Entry<K, V>> trimExclusivelyOwnedEntries(int count, int size) {
    count = Math.max(count, 0);
    size = Math.max(size, 0);
    // fast path without array allocation if no eviction is necessary
    if (mExclusiveEntries.getCount() <= count && mExclusiveEntries.getSizeInBytes() <= size) {
      return null;
    }
    ArrayList<Entry<K, V>> oldEntries = new ArrayList<>();
    while (mExclusiveEntries.getCount() > count || mExclusiveEntries.getSizeInBytes() > size) {
      K key = mExclusiveEntries.getFirstKey();
      mExclusiveEntries.remove(key);
      oldEntries.add(mCachedEntries.remove(key));
    }
    return oldEntries;
  }
```
&#8195;最后makeOrphans()将把待移除项标记为Orphans()后释放缓存项引用对象所占用的内存资源。
##### 2. get():获取缓存键对应的值
```
  public CloseableReference<V> get(final K key) {
    CloseableReference<V> clientRef = null;
    synchronized (this) {
      mExclusiveEntries.remove(key);
      Entry<K, V> entry = mCachedEntries.get(key);
      if (entry != null) {
        clientRef = newClientReference(entry);
      }
    }
    maybeUpdateCacheParams();
    maybeEvictEntries();
    return clientRef;
  }
```
&#8195;对缓存项的获取作为命中事件，首先会把缓存项从mExclusiveEntries中移除，同时将增加缓存项的引用计数，在对缓存进行过get()操作后，缓存达到一个检查配置的安全点，如cache()一样会检查新配置并进行可能的缓存调整工作。
##### 3. removeAll()：移除给定描述的缓存项
```
  public int removeAll(Predicate<K> predicate) {
    ArrayList<Entry<K, V>> oldEntries;
    synchronized (this) {
      mExclusiveEntries.removeAll(predicate);
      oldEntries = mCachedEntries.removeAll(predicate);
      makeOrphans(oldEntries);
    }
    maybeClose(oldEntries);
    maybeUpdateCacheParams();
    maybeEvictEntries();
    return oldEntries.size();
  }
```
##### 4. contains()：查找是否存在给定描述的缓存项
```
  public synchronized boolean contains(Predicate<K> predicate) {
    return !mCachedEntries.getMatchingEntries(predicate).isEmpty();
  }
```

[返回cache_summary](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/cache_summary.md)

