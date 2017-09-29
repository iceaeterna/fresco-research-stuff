## DiskStorageCache
&#8195;DiskStorageCache实现了FileCache，为BufferedDiskCache提供对磁盘文件的缓存管理服务。DiskStorageCache也是通过工厂模式创建的，以实现构造与使用的分离：   
```
  public static DiskStorageCache newDiskStorageCache(DiskCacheConfig diskCacheConfig) {
    DiskStorageSupplier diskStorageSupplier = newDiskStorageSupplier(diskCacheConfig);
    DiskStorageCache.Params params = new DiskStorageCache.Params(
        diskCacheConfig.getMinimumSizeLimit(),
        diskCacheConfig.getLowDiskSpaceSizeLimit(),
        diskCacheConfig.getDefaultSizeLimit());
    return new DiskStorageCache(
        diskStorageSupplier,
        params,
        diskCacheConfig.getCacheEventListener(),
        diskCacheConfig.getCacheErrorLogger(),
        diskCacheConfig.getDiskTrimmableRegistry());
  }
```
&#8195;其中，DiskStorageSupplier以Supplier的方式为DiskStorageCache提供磁盘资源文件的操作接口DiskStorage。接下来就分析DiskStorageCache的业务实现：
##### 1. insert()：插入缓存键值对
(1).根据缓存键获取对应的资源id，资源id的生成使用的是SHA-1算法
```
 final String resourceId = getResourceId(key);
```
(2).创建临时资源文件
```
BinaryResource temporary = createTemporaryResource(resourceId, key);
```
(3).创建临时文件，把图像写到临时文件中，将临时文件保存为资源文件(重命名)，并更新资源文件的统计计数以供磁盘缓冲大小调整
```
      try {
        mStorageSupplier.get().updateResource(resourceId, temporary, callback, key);
        return commitResource(resourceId, key, temporary);
      } finally {
        deleteTemporaryResource(temporary);
      }
```

##### 2. hasKey()：指定CacheKey是否存在对应的资源文件
```
  public boolean hasKey(final CacheKey key) {
    try {
      return mStorageSupplier.get().contains(getResourceId(key), key);
    } catch (IOException e) {
      return false;
    }
  }
```

##### 3. getResource()：获取指定CacheKey对应的资源文件
```
  public BinaryResource getResource(final CacheKey key) {
    try {
      synchronized (mLock) {
        BinaryResource resource = mStorageSupplier.get().getResource(getResourceId(key), key);
        if (resource == null) {
          mCacheEventListener.onMiss();
        } else {
          mCacheEventListener.onHit();
        }
        return resource;
      }
    }
```

##### 4. remove()：删除指定CacheKey对应的资源文件
```
  public void remove(CacheKey key) {
    synchronized (mLock) {
      try {
        mStorageSupplier.get().remove(getResourceId(key));
      }//...
  }
```
___
##### 缓存空间用量的裁剪
&#8195;DiskStorageCache实现了DiskTrimmable，通过该接口可以实现对缓存的大小用量进行缩减。
&#8195;DiskTrimmable提供了两个操作接口：trimToMinimum()和trimToNothing()分别应用于内存空间较小和内存空间耗尽的情况。   
1. trimToMinimum()将把缓冲内容裁剪到缓存最小限制值
```
  public void trimToMinimum() {
    synchronized (mLock) {
      maybeUpdateFileCacheSize();
      long cacheSize = mCacheStats.getSize();
      if (mCacheSizeLimitMinimum <= 0 || cacheSize <= 0 || cacheSize < mCacheSizeLimitMinimum) {
        return;
      }
      double trimRatio = 1 - (double) mCacheSizeLimitMinimum / (double) cacheSize;
      if (trimRatio > TRIMMING_LOWER_BOUND) {
        trimBy(trimRatio);
      }
    }
  }
```
trimBy的实现如下：
```
  private void trimBy(final double trimRatio) {
    synchronized (mLock) {
      try {
        // Force update the ground truth if we are about to evict
        mCacheStats.reset();
        maybeUpdateFileCacheSize();
        long cacheSize = mCacheStats.getSize();
        long newMaxBytesInFiles = cacheSize - (long) (trimRatio * cacheSize);
        evictAboveSize(
            newMaxBytesInFiles,
            CacheEventListener.EvictionReason.CACHE_MANAGER_TRIMMED);
      }
```   
&#8195;其中newMaxBytesInFiles为裁剪缓存内容后可占用的最大字节数。而evictAboveSize()将对缓存项按时间排序后不断尝试删除最早被使用的缓存项内容，直至占用空间缩减到理想值。   
2. trimToNothing()做法较为极端，将会清除所有缓存内容
```
  public void trimToNothing() {
    clearAll();
  }
```

[返回cache_summary](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/cache_summary.md)
