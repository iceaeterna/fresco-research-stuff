##PostprocessedBitmapMemoryCacheProducer
&#8195;后处理器对Bitmap的修改应该是在Bitmap的副本上进行的，否则会影响Bitmap的其他使用者，ImagePipeline使用了一个缓存用来保存不同图片以及不同后处理器的修改结果。   
&#8195;下面分析PostprocessedBitmapMemoryCacheProducer的业务(produceResults()方法)实现：   
1.若没有设置后处理器，那么直接获取下一段结果就可以了
```
    final Postprocessor postprocessor = imageRequest.getPostprocessor();
    if (postprocessor == null) {
      mNextProducer.produceResults(consumer, producerContext);
      return;
    }
```
2.从缓存中尝试获取结果，若存在则作为最终结果直接返回
```
    final CacheKey postprocessorCacheKey = postprocessor.getPostprocessorCacheKey();
    final CacheKey cacheKey;
    CloseableReference<CloseableImage> cachedReference = null;
    if (postprocessorCacheKey != null) {
      cacheKey = mCacheKeyFactory.getPostprocessedBitmapCacheKey(imageRequest);
      cachedReference = mMemoryCache.get(cacheKey);
    } else {
      cacheKey = null;
    }
    if (cachedReference != null) {
      listener.onProducerFinishWithSuccess(
          requestId,
          getProducerName(),
          listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "true") : null);
      consumer.onProgressUpdate(1.0f);
      consumer.onNewResult(cachedReference, true);
      cachedReference.close();
    }
```
3.若在缓存中查找不到，则构造一个CachedPostprocessConsumer把后续获得Bitmap、进行后处理的结果、保存到后处理缓存的工作委托给后续流水线工作。
```
else {
      final boolean isRepeatedProcessor = postprocessor instanceof RepeatedPostprocessor;
      final String processorName = postprocessor.getClass().getName();
      Consumer<CloseableReference<CloseableImage>> cachedConsumer = new CachedPostprocessorConsumer(
          consumer,
          cacheKey,
          isRepeatedProcessor,
          processorName,
          mMemoryCache);
      listener.onProducerFinishWithSuccess(
          requestId,
          getProducerName(),
          listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "false") : null);
      mNextProducer.produceResults(cachedConsumer, producerContext);
    }
```
4.CachedPostprocessConsumer继承自DelegatingConsumer，使用了[代理模式](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/researching_stuffs/imagepipeline_research_stuff/DelegatingPattern.md)将对后续流水线结果的处理交由代理完成。
```
  private final Consumer<O> mConsumer;

  public DelegatingConsumer(Consumer<O> consumer) {
    mConsumer = consumer;
  }

  public Consumer<O> getConsumer() {
    return mConsumer;
  }

  @Override
  protected void onFailureImpl(Throwable t) {
    mConsumer.onFailure(t);
  }

  @Override
  protected void onCancellationImpl() {
    mConsumer.onCancellation();
  }

  @Override
  protected void onProgressUpdateImpl(float progress) {
    mConsumer.onProgressUpdate(progress);
  }
```
DelegatingConsumer可对Consumer进行层层代理，由mConsumer指向被代理对象。可以看出，DelegatingConsumer的实现类在完成指定的处理工作后，会把代理工作继续交由下一层代理继续处理。   
CachedPostprocessConsumer的onNewResultImpl实现如下：   
(1).空结果的返回
```
      if (!isLast && !mIsRepeatedProcessor) {
        return;
      }
      // Given a null result, we just pass it on.
      if (newResult == null) {
        getConsumer().onNewResult(null, isLast);
        return;
      }
```
(2).移除原有的具有相同后处理器名的缓存结果，并缓存新结果
```
      final CloseableReference<CloseableImage> newCachedResult;
      if (mCacheKey != null) {
        mMemoryCache.removeAll(
            new Predicate<CacheKey>() {
              @Override
              public boolean apply(CacheKey cacheKey) {
                if (cacheKey instanceof BitmapMemoryCacheKey) {
                  return mProcessorName.equals(
                      ((BitmapMemoryCacheKey) cacheKey).getPostprocessorName());
                }
                return false;
              }
            });
        newCachedResult = mMemoryCache.cache(mCacheKey, newResult);
      } else {
        newCachedResult = newResult;
      }
```
(3).转发新结果
```
      try {
        getConsumer().onProgressUpdate(1f);
        getConsumer().onNewResult(
            (newCachedResult != null) ? newCachedResult : newResult, isLast);
      } finally {
        CloseableReference.closeSafely(newCachedResult);
      }
```

[返回ProducerSequence](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline/producer_sequence.md)
