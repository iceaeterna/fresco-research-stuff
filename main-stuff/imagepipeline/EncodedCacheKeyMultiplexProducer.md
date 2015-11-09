##EncodedCacheKeyMultiplexProducer
&#8195;EncodedCacheKeyMultiplexProducer继承自MultiplexProducer，区别于BitmapMemoryCacheKeyMultiplexProducer，EncodedCacheKeyMultiplexProducer所使用的Multiplexer键如下：
```
  protected Pair<CacheKey, ImageRequest.RequestLevel> getKey(ProducerContext producerContext) {
    return Pair.create(
        mCacheKeyFactory.getEncodedCacheKey(producerContext.getImageRequest()),
        producerContext.getLowestPermittedRequestLevel());
  }
```
&#8195;EncodedCacheKey为SimpleCacheKey，仅仅标志同一SourceUri的图像:
```
  public boolean equals(Object o) {
    if (o == this) {
      return true;
    }
    if (o instanceof SimpleCacheKey) {
      final SimpleCacheKey otherKey = (SimpleCacheKey) o;
      return mKey.equals(otherKey.mKey);
    }
    return false;
  }
```   
&#8195;而BitmapCacheKey还混合了大小调整、自动旋转、解码选项信息。
```
  public boolean equals(Object o) {
    if (!(o instanceof BitmapMemoryCacheKey)) {
      return false;
    }
    BitmapMemoryCacheKey otherKey = (BitmapMemoryCacheKey) o;
    return mHash == otherKey.mHash &&
        mSourceString.equals(otherKey.mSourceString) &&
        Objects.equal(this.mResizeOptions, otherKey.mResizeOptions) &&
        mAutoRotated == otherKey.mAutoRotated &&
        Objects.equal(mImageDecodeOptions, otherKey.mImageDecodeOptions) &&
        Objects.equal(mPostprocessorCacheKey, otherKey.mPostprocessorCacheKey) &&
        Objects.equal(mPostprocessorName, otherKey.mPostprocessorName);
  }

```   
&#8195;可以看出，BitmapMemoryCacheKeyMultiplexProducer在应用层面上，对“同一Bitmap请求”进行合并，而EncodedCacheKeyMultiplexProducer在数据层面上，对“同一图片来源请求”进行合并。
