##DiskCacheProducer
#####DiskCacheProducer的业务流程
1.因为对磁盘的读写速度要慢的多，所以磁盘缓冲是可选的，当没有设置磁盘缓冲时，将根据获取深度交由下一段流水线进行处理。
```
    ImageRequest imageRequest = producerContext.getImageRequest();
    if (!imageRequest.isDiskCacheEnabled()) {
      maybeStartNextProducer(consumer, consumer, producerContext);
      return;
    }
```
2.如果设置了磁盘缓冲，那么有一个问题需要考虑到，就是磁盘的读写操作比较耗时，对磁盘的操作必须是异步任务，同时需要对磁盘读写进行同步控制，以避免读写冲突。对Cache项的查找操作是通过Bolts框架的Task/Continuation完成的:
```
    AtomicBoolean isCancelled = new AtomicBoolean(false);
    final Task<EncodedImage> diskCacheLookupTask =
        cache.get(cacheKey, isCancelled);
    diskCacheLookupTask.continueWith(continuation);
    subscribeTaskForRequestCancellation(isCancelled, producerContext);
```
3.结果处理：
```
 Continuation<EncodedImage, Void> continuation = new Continuation<EncodedImage, Void>() {
          @Override
          public Void then(Task<EncodedImage> task)
              throws Exception {
          //(1).任务的取消将会转发给Consumer触发onCancellation回调
            if (task.isCancelled() ||
                (task.isFaulted() && task.getError() instanceof CancellationException)) {
              //...
              consumer.onCancellation();
		  //(2).对于其他类型的错误，会被认为磁盘缓存中没有该图像，将交由下一段流水线进行处理。
            } else if (task.isFaulted()) {
              //...
              maybeStartNextProducer(
                  consumer,
                  new DiskCacheConsumer(consumer, cache, cacheKey),
                  producerContext);
		  //(3).当磁盘缓冲中有该图像则返回该图像，若没有该图像则交由下一段流水线处理
            } else {
              EncodedImage cachedReference = task.getResult();
              if (cachedReference != null) {
                //...
                consumer.onProgressUpdate(1);
                consumer.onNewResult(cachedReference, true);
                cachedReference.close();
              } else {
                //...
                maybeStartNextProducer(
                    consumer,
                    new DiskCacheConsumer(consumer, cache, cacheKey),
                    producerContext);
              }
            }
            return null;
          }
        };
```
DiskCacheConsumer的onNewResultImpl与其他缓存一样，当新结果返回时，需要将结果缓存在磁盘Cache中,并转发新结果。
```
    public void onNewResultImpl(EncodedImage newResult, boolean isLast) {
      if (newResult != null && isLast) {
        mCache.put(mCacheKey, newResult);
      }
      getConsumer().onNewResult(newResult, isLast);
    }
```
