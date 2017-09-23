## DecodeProducer
&#8195;DecodeProducer会根据图片的不同来源选择不同的解码器，并将解码任务封装在DelegatingConsumer中，交由后续流水段在获取到结果或状态变化时完成。
```
  public void produceResults(
      final Consumer<CloseableReference<CloseableImage>> consumer,
      final ProducerContext producerContext) {
    final ImageRequest imageRequest = producerContext.getImageRequest();
    ProgressiveDecoder progressiveDecoder;
    if (!UriUtil.isNetworkUri(imageRequest.getSourceUri())) {
      progressiveDecoder = new LocalImagesProgressiveDecoder(consumer, producerContext);
    } else {
      ProgressiveJpegParser jpegParser = new ProgressiveJpegParser(mByteArrayPool);
      progressiveDecoder = new NetworkImagesProgressiveDecoder(
          consumer,
          producerContext,
          jpegParser,
          mProgressiveJpegConfig);
    }
    mNextProducer.produceResults(progressiveDecoder, producerContext);
  }
```
&#8195;无论是对本地图片进行解码处理的LocalImagesProgressiveDecoder，还是对网络图片进行解码处理的ProgressiveJpegParser，都有着共同的父类ProgressiveDecoder。LocalImagesProgressiveDecoder和NetworkImagesProgressiveDecoder所不同的在于对于渐进式图片的处理，LocalImages不会保存任何中间结果，而NetworkImagesProgressiveDecoder则会对jpeg的扫描次数进行判断和处理。   
### ProgressiveDecoder
1.ProgressiveDecoder通过[JobScheduler的作业调度框架](https://github.com/icemoonlol/fresco-research-stuff/tree/master/main-stuff/imagepipeline_research-stuff/JobScheduler.md)来完成解码任务。   
&#8195;ProgressiveDecoder向JobScheduler提交的作业如下：
```
      JobRunnable job = new JobRunnable() {
        @Override
        public void run(EncodedImage encodedImage, boolean isLast) {
          if (encodedImage != null) {
            if (mDownsampleEnabled) {
              ImageRequest request = producerContext.getImageRequest();
              if (mDownsampleEnabledForNetwork ||
                  !UriUtil.isNetworkUri(request.getSourceUri())) {
                encodedImage.setSampleSize(DownsampleUtil.determineSampleSize(
                    request, encodedImage));
              }
            }
            doDecode(encodedImage, isLast);
          }
        }
      };
```
&#8195;该作业主要的工作方法是调用doDecode()方法进行解码。而ProgressiveDecoder以上述作业为参数构造了一个JobScheduler，当有新结果到来时，就会向JobScheduler提交该作业执行。
```
mJobScheduler = new JobScheduler(mExecutor, job, mImageDecodeOptions.minDecodeIntervalMs);
```
2.JobScheduler作业的提交   
&#8195;当有新结果到来时，首先会更新待解码图像(当然若新结果与上次结果相同则直接返回不需要进行处理)，随后将调用JobScheduler的scheduleJob()方法提交作业。
```
    public void onNewResultImpl(EncodedImage newResult, boolean isLast) {
      if (isLast && !EncodedImage.isValid(newResult)) {
        handleError(new NullPointerException("Encoded image is not valid."));
        return;
      }
      if (!updateDecodeJob(newResult, isLast)) {
        return;
      }
      if (isLast || mProducerContext.isIntermediateResultExpected()) {
        mJobScheduler.scheduleJob();
      }
    }
```
而后续流水段的失败或取消都会清除已经存在的解码结果。   
3. 解码处理   
&#8195;当网络状况较好时，对于渐进式加载图片，可能在很短的时间内有多次中间结果的返回，那么就没有必要对每个返回结果进行解码，Imagepipeline采用的是JobScheduler来管理解码作业，**JobScheduler对于提交的任务同时只有一个任务被执行，并且至少距离上次任务的执行间隔一定时间。这样的话，对于渐进式加载图片，对于频繁的中间图像获取进行解码的任务就可以以合理的方式进行。**
```
    private void doDecode(EncodedImage encodedImage, boolean isLast) {
      if (isFinished() || !EncodedImage.isValid(encodedImage)) {
        return;
      }

      try {
        long queueTime = mJobScheduler.getQueuedTime();
        int length = isLast ?
            encodedImage.getSize() : getIntermediateImageEndOffset(encodedImage);
        QualityInfo quality = isLast ? ImmutableQualityInfo.FULL_QUALITY : getQualityInfo();

        mProducerListener.onProducerStart(mProducerContext.getId(), PRODUCER_NAME);
        CloseableImage image = null;
        try {
          image = mImageDecoder.decodeImage(encodedImage, length, quality, mImageDecodeOptions);
        } catch (Exception e) {
          //...
          handleError(e);
          return;
        }
        //...
        handleResult(image, isLast);
      } finally {
        EncodedImage.closeSafely(encodedImage);
      }
    }
```
doDecode()调用解码器的decodeImage()方法进行解码，当解码成功时调用handleResult()方法进行来通知Consumer。

### 关于JobScheduler
&#8195;JobScheduler是一个作业管理器，用来管理对图片进行解码的作业，对于提交的任务同时只有一个任务被执行，并且至少距离上次任务的执行间隔一定时间。   
&#8195;调度器具有以下状态：
```
  enum JobState { IDLE, QUEUED, RUNNING, RUNNING_AND_PENDING }   
```
- IDLE：初始状态，标志调度器当前空闲，没有任何作业到来或执行。
- QUEUED：已入队/就绪态，标志当前只有一个待执行的新任务
- RUNNING：运行态，标志当前只有一个正在执行的新任务
- RUNNING_AND_PENDING：阻塞态，标志当前任务运行过程中有新任务到来并等待执行   
1.作业的提交(scheduleJob()):   
对于初始状态，作业的提交将计算作业的提交延迟(如果距离上次作业的执行时间小于mMinimumJobIntervalMs，则让作业在提交之前进行等待)，调度器也进入就绪状态准备执行任务。而对于RUNNING状态，新作业的到来会让调度器进入RUNNING_AND_PENDING状态。
```
  public boolean scheduleJob() {
    long now = SystemClock.uptimeMillis();
    long when = 0;
    boolean shouldEnqueue = false;
    synchronized (this) {
      if (!shouldProcess(mEncodedImage, mIsLast)) {
        return false;
      }
      switch (mJobState) {
        case IDLE:
          when = Math.max(mJobStartTime + mMinimumJobIntervalMs, now);
          shouldEnqueue = true;
          mJobSubmitTime = now;
          mJobState = JobState.QUEUED;
          break;
        case QUEUED:
          // do nothing, the job is already queued
          break;
        case RUNNING:
          mJobState = JobState.RUNNING_AND_PENDING;
          break;
        case RUNNING_AND_PENDING:
          // do nothing, the next job is already pending
          break;
      }
    }
    if (shouldEnqueue) {
      enqueueJob(when - now);
    }
    return true;
  }
```
在等待一定时间直至距离上次作业的启动时间大于阀值后，将会运行JobScheduler的mSubmitJobRunnable来执行提交操作。
```
  private void enqueueJob(long delay) {
    if (delay > 0) {
      JobStartExecutorSupplier.get().schedule(mSubmitJobRunnable, delay, TimeUnit.MILLISECONDS);
    } else {
      mSubmitJobRunnable.run();
    }
  }
```
mSubmitJobRunnable的执行内容只有一个submitJob()方法，JobScheduler对于submitJob()的实现只是简单地运行mDoJobRunnable来执行提交作业。   
2.作业的执行(doJob()):   
在对未解码图片简单地进行检查后，就会执行提交的作业来对图片进行解码，并在解码结束后回调onJobFinished()进行处理。
```
 private void doJob() {
    long now = SystemClock.uptimeMillis();
    EncodedImage input;
    boolean isLast;
    synchronized (this) {
      input = mEncodedImage;
      isLast = mIsLast;
      mEncodedImage = null;
      mIsLast = false;
      mJobState = JobState.RUNNING;
      mJobStartTime = now;
    }
    try {
      // we need to do a check in case the job got cleared in the meantime
      if (shouldProcess(input, isLast)) {
        mJobRunnable.run(input, isLast);
      }
    } finally {
      EncodedImage.closeSafely(input);
      onJobFinished();
    }
  }
```
(1).没有任何新的作业到来(提交)，那么当前的调度状态为空闲(IDLE)，并等待下一次新的作业到来。   
(2).在执行过程中，有新的作业到来，则新的作业的提交请求(并非提交操作)会将调度状态设置为RUNNING_AND_PENDING，那么这里当前任务完成后，就应该更新这个新作业的提交时间和调度状态，并将该任务提交运行。
```
  private void onJobFinished() {
    long now = SystemClock.uptimeMillis();
    long when = 0;
    boolean shouldEnqueue = false;
    synchronized (this) {
      if (mJobState == JobState.RUNNING_AND_PENDING) {
        when = Math.max(mJobStartTime + mMinimumJobIntervalMs, now);
        shouldEnqueue = true;
        mJobSubmitTime = now;
        mJobState = JobState.QUEUED;
      } else {
        mJobState = JobState.IDLE;
      }
    }
    if (shouldEnqueue) {
      enqueueJob(when - now);
    }
  }
```

[返回ProducerSequence](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline/producer_sequence.md)
