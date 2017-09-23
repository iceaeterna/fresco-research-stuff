## NetworkFetchProducer
&#8195;NetworkFetchProducer将调用NetworkFetcher的fetch()方法获取图片
```
  public void produceResults(Consumer<EncodedImage> consumer, ProducerContext context) {
    context.getListener()
        .onProducerStart(context.getId(), PRODUCER_NAME);
    final FetchState fetchState = mNetworkFetcher.createFetchState(consumer, context);
    mNetworkFetcher.fetch(
        fetchState, new NetworkFetcher.Callback() {
          @Override
          public void onResponse(InputStream response, int responseLength) throws IOException {
            NetworkFetchProducer.this.onResponse(fetchState, response, responseLength);
          }

          @Override
          public void onFailure(Throwable throwable) {
            NetworkFetchProducer.this.onFailure(fetchState, throwable);
          }

          @Override
          public void onCancellation() {
            NetworkFetchProducer.this.onCancellation(fetchState);
          }
        });
  }
```
##### 网络获取结果的处理
&#8195;从返回结果的Java输入流读取图像数据并写到Native输出流中，并适时地返回中间结果，最后当获取到所有的返回结果时，调用handleFinalResult()处理输出流结果。
```
  private void onResponse(
      FetchState fetchState,
      InputStream responseData,
      int responseContentLength)
      throws IOException {
    final PooledByteBufferOutputStream pooledOutputStream;
    if (responseContentLength > 0) {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream(responseContentLength);
    } else {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream();
    }
    final byte[] ioArray = mByteArrayPool.get(READ_SIZE);
    try {
      int length;
      while ((length = responseData.read(ioArray)) >= 0) {
        if (length > 0) {
          pooledOutputStream.write(ioArray, 0, length);
          maybeHandleIntermediateResult(pooledOutputStream, fetchState);
          float progress = calculateProgress(pooledOutputStream.size(), responseContentLength);
          fetchState.getConsumer().onProgressUpdate(progress);
        }
      }
      mNetworkFetcher.onFetchCompletion(fetchState, pooledOutputStream.size());
      handleFinalResult(pooledOutputStream, fetchState);
    } finally {
      mByteArrayPool.release(ioArray);
      pooledOutputStream.close();
    }
  }
```   
&#8195;handleFinalResult()的主要是调用notifyConsumer()通知Consumer：
```
  private void notifyConsumer(
      PooledByteBufferOutputStream pooledOutputStream,
      boolean isFinal,
      Consumer<EncodedImage> consumer) {
    CloseableReference<PooledByteBuffer> result =
        CloseableReference.of(pooledOutputStream.toByteBuffer());
    EncodedImage encodedImage = null;
    try {
      encodedImage = new EncodedImage(result);
      encodedImage.parseMetaData();
      consumer.onNewResult(encodedImage, isFinal);
    } finally {
      EncodedImage.closeSafely(encodedImage);
      CloseableReference.closeSafely(result);
    }
  }
```
&#8195;notifyConsumer()将**根据Native输出流构造的Native内存字节缓冲块构造一个EncodedImage返回给Consumer**

[返回ProducerSequence](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline/producer_sequence.md)
