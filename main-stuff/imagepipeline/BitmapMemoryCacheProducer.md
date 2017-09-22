## BitmapMemoryCacheProducer
&#8195;BitmapMemoryCacheProducer作为Bitmap缓存，将缓存已获取并解码的Bitmap图片。其生产方法produceResults()的实现如下：   
&#8195;1.首先从Bitmap缓存中查找对应ImageRequest的Bitmap图像是否存在。若存在则还需要查看该图像的分辨率是否完全(渐进式加载可能返回不完整的图像内容)，若完全则表示获取图像的全部内容，将回调Consumer的onProgressUpdate()来通知图像加载完毕。   
&#8195;考虑到Jpeg图片的渐进式加载过程，无论图片是否加载完全，都将回调onNewResult()来通知新的图像内容获取。在触发本段流水线的Consumer(实际上是之前层层封装的多级Consumer代理)的回调后，将关闭这里对新结果的引用。
```
    final ProducerListener listener = producerContext.getListener();
    final String requestId = producerContext.getId();
    listener.onProducerStart(requestId, getProducerName());
    final ImageRequest imageRequest = producerContext.getImageRequest();
    final CacheKey cacheKey = mCacheKeyFactory.getBitmapCacheKey(imageRequest);
    CloseableReference<CloseableImage> cachedReference = mMemoryCache.get(cacheKey);
    if (cachedReference != null) {
      boolean isFinal = cachedReference.get().getQualityInfo().isOfFullQuality();
      if (isFinal) {
        listener.onProducerFinishWithSuccess(
            requestId,
            getProducerName(),
            listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "true") : null);
        consumer.onProgressUpdate(1f);
      }
      consumer.onNewResult(cachedReference, isFinal);
      cachedReference.close();
      if (isFinal) {
        return;
      }
    }
```    
2.若缓存中没有结果，那么需要对获取深度进行检查，若获取深度小于Bitmap则返回null并直接结束获取。
```
    if (producerContext.getLowestPermittedRequestLevel().getValue() >=
        ImageRequest.RequestLevel.BITMAP_MEMORY_CACHE.getValue()) {
      listener.onProducerFinishWithSuccess(
          requestId,
          getProducerName(),
          listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "false") : null);
      consumer.onNewResult(null, true);
      return;
    }
```
3.BitmapMemoryCacheProducer把委托任务封装在DelegatingConsumer中，并传递给下一个生产者(流水线的下一段)进行处理。
```
Consumer<CloseableReference<CloseableImage>> wrappedConsumer = wrapConsumer(consumer, cacheKey);
    listener.onProducerFinishWithSuccess(
        requestId,
        getProducerName(),
        listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "false") : null);
    mNextProducer.produceResults(wrappedConsumer, producerContext);
```
可以预见，这部分工作无疑就是将新结果缓存在BitmapCache中。   
4.新结果的缓存可能有以下几张情况：   
(1).获取结果为空，或者结果为包含状态的图像(如包含当前帧的GIF图像)，就不会将其放到缓存中，而是将直接发起Consumer回调。
```
        if (newResult == null) {
          if (isLast) {
            getConsumer().onNewResult(null, true);
          }
          return;
        }
        // stateful results cannot be cached and are just forwarded
        if (newResult.get().isStateful()) {
          getConsumer().onNewResult(newResult, isLast);
          return;
        }
```
(2).对于渐进式加载图像，如果当前新获得的图像分辨率要小于当前缓存的图像，那么会将当前缓存的图像作为新的结果转发，而新获得的图像就是无效的，其引用将会被关闭，占用的内存资源也会被释放。
```
        if (!isLast) {
          CloseableReference<CloseableImage> currentCachedResult = mMemoryCache.get(cacheKey); 
          if (currentCachedResult != null) {
            try {
              QualityInfo newInfo = newResult.get().getQualityInfo();
              QualityInfo cachedInfo = currentCachedResult.get().getQualityInfo();
              if (cachedInfo.isOfFullQuality() || cachedInfo.getQuality() >= newInfo.getQuality()) {
                getConsumer().onNewResult(currentCachedResult, false);
                return;
              }
            } finally {
              CloseableReference.closeSafely(currentCachedResult);
            }
          }
        }
```
(3).否则，新获得的图像将作为图像的最新版本缓存在Bitmap缓存中，并发起onNewResult()回调。
```
        CloseableReference<CloseableImage> newCachedResult =
            mMemoryCache.cache(cacheKey, newResult);
        try {
          if (isLast) {
            getConsumer().onProgressUpdate(1f);
          }
          getConsumer().onNewResult(
              (newCachedResult != null) ? newCachedResult : newResult, isLast);
        } finally {
          CloseableReference.closeSafely(newCachedResult);
        }
```

[返回ProducerSequence](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline/producer_sequence.md)
