##PostprocessorProducer
&#8195;PostprocessorProducer将使用用户自定义的处理方法对获得的Bitmap图像进行后处理，但这部分工作是封装在DelegatingConsumer(PostprocessorConsumer)中交由后续流水线在获得新结果时调用进行处理。
```
  public void produceResults(
      final Consumer<CloseableReference<CloseableImage>> consumer,
      ProducerContext context) {
    final ProducerListener listener = context.getListener();
    final Postprocessor postprocessor = context.getImageRequest().getPostprocessor();
    final PostprocessorConsumer basePostprocessorConsumer =
        new PostprocessorConsumer(consumer, listener, context.getId(), postprocessor, context);
    final Consumer<CloseableReference<CloseableImage>> postprocessorConsumer;
    if (postprocessor instanceof RepeatedPostprocessor) {
      postprocessorConsumer = new RepeatedPostprocessorConsumer(
          basePostprocessorConsumer,
          (RepeatedPostprocessor) postprocessor,
          context);
    } else {
      postprocessorConsumer = new SingleUsePostprocessorConsumer(basePostprocessorConsumer);
    }
    mNextProducer.produceResults(postprocessorConsumer, context);
  }
```
&#8195;PostprocessorConsumer的onNewResultImpl实现如下：
```
    protected void onNewResultImpl(CloseableReference<CloseableImage> newResult, boolean isLast) {
      if (!CloseableReference.isValid(newResult)) {
        // try to propagate if the last result is invalid
        if (isLast) {
          maybeNotifyOnNewResult(null, true);
        }
        // ignore if invalid
        return;
      }
      updateSourceImageRef(newResult, isLast);
    }
```
&#8195;对于无效的最终结果将适时通知Consumer，否则将对Bitmap进行处理，但考虑到对Bitmap的处理是非常耗时的，所以后处理的工作应该是异步进行的，并且为了避免对Bitmap后处理的冲突操作，需要对后处理过程进行同步控制：
```
mExecutor.execute(
          new Runnable() {
            @Override
            public void run() {
              CloseableReference<CloseableImage> closeableImageRef;
              boolean isLast;
              synchronized (PostprocessorConsumer.this) {
                // instead of cloning and closing the reference, we do a more efficient move.
                closeableImageRef = mSourceImageRef;
                isLast = mIsLast;
                mSourceImageRef = null;
                mIsDirty = false;
              }
              if (CloseableReference.isValid(closeableImageRef)) {
                try {
                  doPostprocessing(closeableImageRef, isLast);
                } finally {
                  CloseableReference.closeSafely(closeableImageRef);
                }
              }
              clearRunningAndStartIfDirty();
            }
          });
```
&#8195;doPostprocessing调用postprocessInternal()进行开始后处理工作。
```
    private void doPostprocessing(
        CloseableReference<CloseableImage> sourceImageRef,
        boolean isLast) {
      //...
      CloseableReference<CloseableImage> destImageRef = null;
      try {
        try {
          destImageRef = postprocessInternal(sourceImageRef.get());
        } //...
    }
```
&#8195;真正的处理将交由指定的Postprocessor完成
```
    private CloseableReference<CloseableImage> postprocessInternal(CloseableImage sourceImage) {
      CloseableStaticBitmap staticBitmap = (CloseableStaticBitmap) sourceImage;
      Bitmap sourceBitmap = staticBitmap.getUnderlyingBitmap();
      CloseableReference<Bitmap> bitmapRef = mPostprocessor.process(sourceBitmap, mBitmapFactory);
      int rotationAngle = staticBitmap.getRotationAngle();
      try {
        return CloseableReference.<CloseableImage>of(
            new CloseableStaticBitmap(bitmapRef, sourceImage.getQualityInfo(), rotationAngle));
      } finally {
        CloseableReference.closeSafely(bitmapRef);
      }
    }
```
&#8195;在Postprocessor的基础实现类BasePostprocessor中，process()方法将使用PlatformBitmapFactory创建一个与源图像副本，并在副本上调用用户自定义的处理方法进行处理。
```
  public CloseableReference<Bitmap> process(
      Bitmap sourceBitmap,
      PlatformBitmapFactory bitmapFactory) {
    CloseableReference<Bitmap> destBitmapRef =
        bitmapFactory.createBitmap(sourceBitmap.getWidth(), sourceBitmap.getHeight());
    try {
      process(destBitmapRef.get(), sourceBitmap);
      return CloseableReference.cloneOrNull(destBitmapRef);
    } finally {
      CloseableReference.closeSafely(destBitmapRef);
    }
  }
  
  public void process(Bitmap destBitmap, Bitmap sourceBitmap) {
    Bitmaps.copyBitmap(destBitmap, sourceBitmap);
    process(destBitmap);
    }
```

[返回ProducerSequence](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline/producer_sequence.md)