## 流水线配置

&#8195;ImagePipeline将流水线各段的工作封装成为一个Producer。并由ProducerFactory负责生产各个流水线段，由ProducerSequenceFactory负责根据图像来源将各个流水线段组装成一个完整的流水线。Producer接口的定义如下：
```
public interface Producer<T> {
  /**
   * Start producing results for given context. Provided consumer is notified whenever progress is
   * made (new value is ready or error occurs).
   * @param consumer
   * @param context
   */
  void produceResults(Consumer<T> consumer, ProducerContext context);
}
```
&#8195;Producer < T >代表一个任务执行结果为T实例的单个任务，可以理解为流水线的一段。而Producer的实现类通过mNextProducer指向后续流水段来组装成一个完整的流水段。

### 后处理段
```
  public Producer<CloseableReference<CloseableImage>> getDecodedImageProducerSequence(
      ImageRequest imageRequest) {
    Producer<CloseableReference<CloseableImage>> pipelineSequence =
        getBasicDecodedImageSequence(imageRequest);
    if (imageRequest.getPostprocessor() != null) {
      return getPostprocessorSequence(pipelineSequence);
    } else {
      return pipelineSequence;
    }
  }
```
&#8195;后处理段的处理方式较为特殊，ImagePipeline维护了一个以前段流水为键、以后处理图像缓存为值的Map。
```
  private synchronized Producer<CloseableReference<CloseableImage>> getPostprocessorSequence(
      Producer<CloseableReference<CloseableImage>> nextProducer) {
    if (!mPostprocessorSequences.containsKey(nextProducer)) {
      PostprocessorProducer postprocessorProducer =
          mProducerFactory.newPostprocessorProducer(nextProducer);
      PostprocessedBitmapMemoryCacheProducer postprocessedBitmapMemoryCacheProducer =
          mProducerFactory.newPostprocessorBitmapMemoryCacheProducer(postprocessorProducer);
      mPostprocessorSequences.put(nextProducer, postprocessedBitmapMemoryCacheProducer);
    }
    return mPostprocessorSequences.get(nextProducer);
  }
```
&#8195;由上可知，后处理段的流水段为：

... -> PostprocessorBitmapMemoryCacheProducer -> PostprocessorProducer
### 获取与处理流水段
&#8195;getBasicDecodedImageSequence()将根据不同的Uri类型来组装成为不同结构的流水段。那么以NetworkUri为例分析其流水线构成。
```
  private synchronized Producer<CloseableReference<CloseableImage>> getNetworkFetchSequence() {
    if (mNetworkFetchSequence == null) {
      mNetworkFetchSequence =
          newBitmapCacheGetToDecodeSequence(getCommonNetworkFetchToEncodedMemorySequence());
    }
    return mNetworkFetchSequence;
  }
```
&#8195;newBitmapCacheGetToDecodeSequence在Bitmap缓存获取深度的流水处理之后加入了解码段DecoderProducer，以处理之后从未解码图片缓冲中获取的图片。
```
  private Producer<CloseableReference<CloseableImage>> newBitmapCacheGetToDecodeSequence(
      Producer<EncodedImage> nextProducer) {
    DecodeProducer decodeProducer = mProducerFactory.newDecodeProducer(nextProducer);
    return newBitmapCacheGetToBitmapCacheSequence(decodeProducer);
  }
```
&#8195;因此，流水段可以根据DecoderProducer分为Bitmap图片流水段和未解码图片流水段两部分。

#### Bitmap图片流水段
```
  private Producer<CloseableReference<CloseableImage>> newBitmapCacheGetToBitmapCacheSequence(
      Producer<CloseableReference<CloseableImage>> nextProducer) {
    BitmapMemoryCacheProducer bitmapMemoryCacheProducer =
        mProducerFactory.newBitmapMemoryCacheProducer(nextProducer);
    BitmapMemoryCacheKeyMultiplexProducer bitmapKeyMultiplexProducer =
        mProducerFactory.newBitmapMemoryCacheKeyMultiplexProducer(bitmapMemoryCacheProducer);
    ThreadHandoffProducer<CloseableReference<CloseableImage>> threadHandoffProducer =
        mProducerFactory.newBackgroundThreadHandoffProducer(bitmapKeyMultiplexProducer);
    return mProducerFactory.newBitmapMemoryCacheGetProducer(threadHandoffProducer);
  }
```
&#8195;由上可知，Bitmap缓存获取深度的流水处理为：

... -> BitmapMemoryCacheGetProducer -> ThreadHandoffProducer-> BitmapMemoryCacheKeyMultiplexProducer -> BitmapMemoryCacheProducer
#### 未解码图片流水段
(从未解码图片缓存到NetFetch的流水处理)
```
  private synchronized Producer<EncodedImage> getCommonNetworkFetchToEncodedMemorySequence() {
    if (mCommonNetworkFetchToEncodedMemorySequence == null) {
      Producer<EncodedImage> nextProducer =
          newEncodedCacheMultiplexToTranscodeSequence(
              mProducerFactory.newNetworkFetchProducer(mNetworkFetcher));
      mCommonNetworkFetchToEncodedMemorySequence =
          ProducerFactory.newAddImageTransformMetaDataProducer(nextProducer);

      if (mResizeAndRotateEnabledForNetwork && !mDownsampleEnabled) {
        mCommonNetworkFetchToEncodedMemorySequence =
            mProducerFactory.newResizeAndRotateProducer(
                mCommonNetworkFetchToEncodedMemorySequence);
      }
    }
    return mCommonNetworkFetchToEncodedMemorySequence;
  }
```
&#8195;由上可知，从Bitmap缓存到NetFetch的流水处理可以分解为两部分：一部分是包含NetworkFetchProducer的从未解码图片缓冲混流到网络图片转码部分的流水段，一部分是为Jpeg图片解析元数据的AddImageTransformMetaDataProducer流水段和可选的大小调整和旋转的ResizeAndRotateProducer流水段。
后者的流水段为：

... -> ResizeAndRotateProducer -> AddImageTransformMetaDataProducer
##### 未解码图片缓冲混流到网络图片转码部分流水段
```
  private Producer<EncodedImage> newEncodedCacheMultiplexToTranscodeSequence(
          Producer<EncodedImage> nextProducer) {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR2) {
      nextProducer = mProducerFactory.newWebpTranscodeProducer(nextProducer);
    }
    nextProducer = mProducerFactory.newDiskCacheProducer(nextProducer);
    EncodedMemoryCacheProducer encodedMemoryCacheProducer =
        mProducerFactory.newEncodedMemoryCacheProducer(nextProducer);
    return mProducerFactory.newEncodedCacheKeyMultiplexProducer(encodedMemoryCacheProducer);
  }
```
&#8195;由上可知，包含NetworkFetchProducer的从未解码图片缓冲混流到网络图片转码部分的流水段为：

...->EncodedCacheKeyMultiplexProducer -> EncodedMemoryCacheProducer -> DiskCacheProducer -> WebpTranscodeProducer -> (NetworkFetchProducer)
___

最后NetUri被组装为：

-> PostprocessorBitmapMemoryCacheProducer  
-> PostprocessorProducer  
-> BitmapMemoryCacheGetProducer -> ThreadHandoffProducer-> BitmapMemoryCacheKeyMultiplexProducer  
-> BitmapMemoryCacheProducer  
-> DecoderProducer  
-> ResizeAndRotateProducer -> AddImageTransformMetaDataProducer  
-> EncodedCacheKeyMultiplexProducer -> EncodedMemoryCacheProducer -> DiskCacheProducer -> WebpTranscodeProducer  
-> (NetworkFetchProducer)
___
接下来，将仍以NetUri为例，分析各个流水段的业务实现。
##### &#8195;Producer目录
- [PostprocessorBitmapMemoryCacheProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/PostprocessorBitmapMemoryCacheProducer.md)
- [PostprocessorProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/PostprocessorProducer.md)
- [BitmapMemoryCacheGetProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/BitmapMemoryCacheGetProducer.md)
- [ThreadHandoffProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/ThreadHandoffProducer.md)
- [BitmapMemoryCacheKeyMultiplexProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/BitmapMemoryCacheKeyMultiplexProducer.md)
- [BitmapMemoryCacheProducer](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline/BitmapMemoryCacheProducer.md)
- [DecoderProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/DecoderProducer.md)
- [ResizeAndRotateProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/ResizeAndRotateProducer.md)
- [AddImageTransformMetaDataProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/AddImageTransformMetaDataProducer.md)
- [EncodedCacheKeyMultiplexProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/EncodedCacheKeyMultiplexProducer.md)
- [EncodedMemoryCacheProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/EncodedMemoryCacheProducer.md)
- [DiskCacheProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/DiskCacheProducer.md)
- [WebpTranscodeProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/WebpTranscodeProducer.md)
- [NetworkFetchProducer](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline/NetworkFetchProducer.md)


