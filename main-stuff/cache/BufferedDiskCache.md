##BufferedDiskCache
&#8195;BufferedDiskCache实现了对磁盘缓冲的有序读写操作，以内存缓冲的形式将EncodedImage返回给使用者。其构造方法如下：
```
  public BufferedDiskCache(
      FileCache fileCache,
      PooledByteBufferFactory pooledByteBufferFactory,
      PooledByteStreams pooledByteStreams,
      Executor readExecutor,
      Executor writeExecutor,
      ImageCacheStatsTracker imageCacheStatsTracker) {
    mFileCache = fileCache;
    mPooledByteBufferFactory = pooledByteBufferFactory;
    mPooledByteStreams = pooledByteStreams;
    mReadExecutor = readExecutor;
    mWriteExecutor = writeExecutor;
    mImageCacheStatsTracker = imageCacheStatsTracker;
    mStagingArea = StagingArea.getInstance();
  }
```
&#8195;其中mFileCache为磁盘文件缓冲。而StagingArea使用一个缓存键和未解码图像缓冲池引用的HashMap，作为缓冲文件与EncodedImage、磁盘文件与内存的交换区，提供对缓存在磁盘的EncodedImage图像进行存取的接口。下面分析对BufferedDiskCache的操作实现：   
1. put()：
(1). 把键值对保存在mStagingArea中
```
    // Store encodedImage in staging area
    mStagingArea.put(key, encodedImage);
```
(2). 将EncodedImage写到磁盘缓存，该任务将交由写线程池执行。一旦发生失败，则必须回滚删除mStagingArea中的键值对
```
    final EncodedImage finalEncodedImage = EncodedImage.cloneOrNull(encodedImage);
    try {
      mWriteExecutor.execute(
          new Runnable() {
            @Override
            public void run() {
              try {
                writeToDiskCache(key, finalEncodedImage);
              } finally {
                mStagingArea.remove(key, finalEncodedImage);
                EncodedImage.closeSafely(finalEncodedImage);
              }
            }
          });
    } catch (Exception exception) {
      //...
      mStagingArea.remove(key, encodedImage);
      EncodedImage.closeSafely(finalEncodedImage);
    }
```  
2. contains():
(1). 在交换区mStagingArea内查找该项是否存在(交换区获取的EncodedImage引用为不可指向其他EncodedImage的final类型)
```
    final EncodedImage pinnedImage = mStagingArea.get(key);
    if (pinnedImage != null) {
      pinnedImage.close();
      //...
      mImageCacheStatsTracker.onStagingAreaHit();
      return Task.forResult(true);
    }
```
(2).若交换区中没有该缓存项，则从FileCache中查找并返回结果，若FileCache中没有该缓存项对应的文件信息，那么磁盘文件缓冲中就没有该项，这里就会触发一次丢失。
```
 try {
      return Task.call(
          new Callable<Boolean>() {
            @Override
            public Boolean call() throws Exception {
              EncodedImage result = mStagingArea.get(key);
              if (result != null) {
                result.close();
                //...
                mImageCacheStatsTracker.onStagingAreaHit();
                return true;
              } else {
               //...
                mImageCacheStatsTracker.onStagingAreaMiss();
                try {
                  return mFileCache.hasKey(key);
                } catch (Exception exception) {
                  return false;
                }
              }
            }
          },
          mReadExecutor);
    } catch (Exception exception) {
      //...
      return Task.forError(exception);
    }
```   
3. get()：
&#8195;与contains类似，并且当StagingArea中没有该缓存项时，将调用readFromDiskCache()从磁盘文件中读取。
```
try {
      final PooledByteBuffer buffer = readFromDiskCache(key);
      CloseableReference<PooledByteBuffer> ref = CloseableReference.of(buffer);
      try {
        result = new EncodedImage(ref);
      } finally {
        CloseableReference.closeSafely(ref);
      }
    } catch (Exception exception) {
      return null;
    }
```
4. remove():将从StagingArea和FileCache中移除缓存项
5. readFromDiskCache()：   
&#8195;图片所对应的文件的信息作为BinaryResource保存在FileCache中，从磁盘中读取EncodedImage的操作就是打开对应磁盘文件的输入流，将其写到Native内存字节缓冲块PooledByteBuffer中，并返回指向该缓冲块的引用。
```
  private PooledByteBuffer readFromDiskCache(final CacheKey key) throws IOException {
    try {
      final BinaryResource diskCacheResource = mFileCache.getResource(key);
      if (diskCacheResource == null) {
        mImageCacheStatsTracker.onDiskCacheMiss();
        return null;
      } else {
        mImageCacheStatsTracker.onDiskCacheHit();
      }

      PooledByteBuffer byteBuffer;
      final InputStream is = diskCacheResource.openStream();
      try {
        byteBuffer = mPooledByteBufferFactory.newByteBuffer(is, (int) diskCacheResource.size());
      } finally {
        is.close();
      }

      return byteBuffer;
    } catch (IOException ioe) {

      mImageCacheStatsTracker.onDiskCacheGetFail();
      throw ioe;
    }
  }
```   
6. writeToDiskCache()：写操作有所不同，这里调用了mFileCache的insert()，并传入一个写回调方法作为参数，该方法将把EncodedImage的数据写到指定的资源文件的输出流上。
```
  private void writeToDiskCache(
      final CacheKey key,
      final EncodedImage encodedImage) {
    FLog.v(TAG, "About to write to disk-cache for key %s", key.toString());
    try {
      mFileCache.insert(
          key, new WriterCallback() {
            @Override
            public void write(OutputStream os) throws IOException {
              mPooledByteStreams.copy(encodedImage.getInputStream(), os);
            }
          }
      );
   //...
  }
```