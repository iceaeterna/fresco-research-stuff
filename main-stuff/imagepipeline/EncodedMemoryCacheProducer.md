##EncodedMemoryCacheProducer
&#8195;EncodedMemoryCacheProducer的业务流程与BitmapMemoryCacheProducer基本一致，先查询CacheKey对应的缓存项(未解码图像)是否存在，若存在则将该图像作为最后结果返回，否则需要判断获取深度并交由下一段流水线进行获取。   
&#8195;EncodedMemoryCacheProducer对于新结果的处理有所不同:   
&#8195;对于中间结果，将直接转发结果而不会对结果进行缓存，这点不同于BitmapMemoryCache会缓存最近一次最优分辨率的图像，因为对中间结果的缓存实际上只需要一次就够了，否则由于中间结果的频繁获取将加剧内存的消耗。
```
 if (!isLast || newResult == null) {
            getConsumer().onNewResult(newResult, isLast);
            return;
 }
```
对于不存在的缓存项将缓存到mMemoryCache中，最后转发新的结果(以指向图像数据所在缓冲区的引用和图像元数据构造一个EncodedImage返回给Consumer)。
```
public void onNewResultImpl(EncodedImage newResult, boolean isLast) {
          // intermediate or null results are not cached, so we just forward them
          if (!isLast || newResult == null) {
            getConsumer().onNewResult(newResult, isLast);
            return;
          }
          // cache and forward the last result
          CloseableReference<PooledByteBuffer> ref = newResult.getByteBufferRef();
          if (ref != null) {
            CloseableReference<PooledByteBuffer> cachedResult;
            try {
              cachedResult = mMemoryCache.cache(cacheKey, ref);
            } finally {
              CloseableReference.closeSafely(ref);
            }
            if (cachedResult != null) {
              EncodedImage cachedEncodedImage;
              try {
                cachedEncodedImage = new EncodedImage(cachedResult);
                cachedEncodedImage.copyMetaDataFrom(newResult);
              } finally {
                CloseableReference.closeSafely(cachedResult);
              }
              try {
                getConsumer().onProgressUpdate(1f);
                getConsumer().onNewResult(cachedEncodedImage, true);
                return;
              } finally {
                EncodedImage.closeSafely(cachedEncodedImage);
              }
            }
          }
          getConsumer().onNewResult(newResult, true);
        }
```
