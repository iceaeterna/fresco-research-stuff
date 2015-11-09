##二、ImagePipeline工作过程

[TOC]


###ImagePipeline简介
> > 下面关于ImagePipeline的描述引自http://fresco-cn.org/docs/configure-image-pipeline.html#_：   
Image pipeline 负责完成加载图像，变成Android设备可呈现的形式所要做的每个事情。   
大致流程如下:   
检查内存缓存，如有，返回   
后台线程开始后续工作   
检查是否在未解码内存缓存中。如有，解码，变换，返回，然后缓存到内存缓存中。   
检查是否在文件缓存中，如果有，变换，返回。缓存到未解码缓存和内存缓存中。   
从网络或者本地加载。加载完成后，解码，变换，返回。存到各个缓存中。   
既然本身就是一个图片加载组件，那么一图胜千言。   
![](http://fresco-cn.org/static/imagepipeline.png)   
上图中，disk cache实际包含了未解码的内存缓存在内，统一在一起只是为了逻辑稍微清楚一些。   
Image pipeline 可以从本地文件加载文件，也可以从网络。支持PNG，GIF，WebP, JPEG。   

###ImagePipeline构造
&#8195;在初始化Fresco的过程中，将初始化ImagePipelineFactory和Drawee部分。Drawee部分的初始化就是创建一个静态的PipelineDraweeControllerBuilderSupplier实例，那么后面将由ImagePipelineFactory的初始化开始，揭开ImagePipeline的神秘面纱。
```
  public static void initialize(Context context) {
    ImagePipelineFactory.initialize(context);
    initializeDrawee(context);
  }
```

&#8195;ImagePipelineFactory从类的定义上可以看出使用了**单例工厂模式**。
```
public class ImagePipelineFactory { 

  private static ImagePipelineFactory sInstance = null;

  /** Gets the instance of {@link ImagePipelineFactory}. */
  public static ImagePipelineFactory getInstance() {
    return Preconditions.checkNotNull(sInstance, "ImagePipelineFactory was not initialized!");
  }

  /** Initializes {@link ImagePipelineFactory} with default config. */
  public static void initialize(Context context) {
    initialize(ImagePipelineConfig.newBuilder(context).build());
  }

  /** Initializes {@link ImagePipelineFactory} with the specified config. */
  public static void initialize(ImagePipelineConfig imagePipelineConfig) {
    sInstance = new ImagePipelineFactory(imagePipelineConfig);
  }

```
&#8195;若指定了ImagePipelineConfig，则ImagePipelineFactory将使用指定的配置进行初始化，否则将使用内部类Builder建造一个ImagePipelineConfig加以配置。之前提到的Android的AlertDialog使用就是内部类Builder的建造者模式。

&#8195;ImagePipelineConfig的各个模块如下：
```
  @Nullable private final AnimatedImageFactory mAnimatedImageFactory;
  private final Supplier<MemoryCacheParams> mBitmapMemoryCacheParamsSupplier;
  private final CacheKeyFactory mCacheKeyFactory;
  private final Context mContext;
  private final boolean mDownsampleEnabled;
  private final Supplier<MemoryCacheParams> mEncodedMemoryCacheParamsSupplier;
  private final ExecutorSupplier mExecutorSupplier;
  private final ImageCacheStatsTracker mImageCacheStatsTracker;
  @Nullable private final ImageDecoder mImageDecoder;
  private final Supplier<Boolean> mIsPrefetchEnabledSupplier;
  private final DiskCacheConfig mMainDiskCacheConfig;
  private final MemoryTrimmableRegistry mMemoryTrimmableRegistry;
  private final NetworkFetcher mNetworkFetcher;
  @Nullable private final PlatformBitmapFactory mPlatformBitmapFactory;
  private final PoolFactory mPoolFactory;
  private final ProgressiveJpegConfig mProgressiveJpegConfig;
  private final Set<RequestListener> mRequestListeners;
  private final boolean mResizeAndRotateEnabledForNetwork;
  private final DiskCacheConfig mSmallImageDiskCacheConfig;
```
&#8195;ImagePipelineConfig的所有成员均可以通过Builder进行配置并建造出来。用户通过自定义ImagePipelineConfig可以配置ImagePipeline的各个模块：
- mAnimatedImageFactory：动态图片处理工厂
- mBitmapMemoryCacheParamsSupplier：提供Bitmap缓冲的配置参数
- mCacheKeyFactory：用于产生ImagePipeline各层缓冲的缓冲键
- mDownsampleEnabled：是否降低图片采样率
- mEncodedMemoryCacheParamsSupplier:提供未解码图像缓冲的配置参数
- mExecutorSupplier：用于提供下载、解码等工作线程池
- mImageCacheStatsTracker：用于记录缓存统计信息
- mImageDecoder：解码器
- mIsPrefetchEnabledSupplier：是否预加载图片
- mMainDiskCacheConfig：磁盘主缓冲配置
- mMemoryTrimmableRegistry：用于提供ImagePipeline各层缓冲的裁剪策略
- mNetworkFetcher：用于获取网络图片
- mPlatformBitmapFactory：Bitmap图片处理工厂
- mPoolFactory：用于提供各种类型(Native内存块、Native内存缓冲块等)资源池
- mProgressiveJpegConfig：Jpeg控制策略
- mRequestListeners：图片请求监听器集合
- mResizeAndRotateEnabledForNetwork：是否允许网络图片的自动大小调整和旋转
- mSmallImageDiskCacheConfig：磁盘小图片缓冲配置

&#8195;ImagePipelineFactory的核心应该在于图片获取和加工的流水线ImagePipeline了。在PipelineDraweeControllerBuilder的构造方法中，会调用ImagePipelineFactory的getImagePipeline()获取ImagePipeline对象：
```
  public ImagePipeline getImagePipeline() {
    if (mImagePipeline == null) {
      mImagePipeline =
          new ImagePipeline(
              getProducerSequenceFactory(),
              mConfig.getRequestListeners(),
              mConfig.getIsPrefetchEnabledSupplier(),
              getBitmapMemoryCache(),
              getEncodedMemoryCache(),
              getMainBufferedDiskCache(),
              getSmallImageBufferedDiskCache(),
              mConfig.getCacheKeyFactory());
    }
    return mImagePipeline;
  }
```
&#8195;之前分析到，在DraweeController创建时，会调用ImagePipeline的fetchImageFromBitmapCache()或fetchDecodedImage()获取图像数据源DataSource，并发起图片获取工作。
```
  protected DataSource<CloseableReference<CloseableImage>> getDataSourceForRequest(
      ImageRequest imageRequest,
      Object callerContext,
      boolean bitmapCacheOnly) {
    if (bitmapCacheOnly) {
      return mImagePipeline.fetchImageFromBitmapCache(imageRequest, callerContext);
    } else {
      return mImagePipeline.fetchDecodedImage(imageRequest, callerContext);
    }
  }
```
###ImagePipeline工作过程
&#8195;fetchImageFromBitmapCache()和fetchDecodedImage()的调用过程基本相同：
```
  public DataSource<CloseableReference<CloseableImage>> fetchImageFromBitmapCache(
      ImageRequest imageRequest,
      Object callerContext) {
    try {
      Producer<CloseableReference<CloseableImage>> producerSequence =
          mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);
      return submitFetchRequest(
          producerSequence,
          imageRequest,
        // ImageRequest.RequestLevel.FULL_FETCH,
          ImageRequest.RequestLevel.BITMAP_MEMORY_CACHE,
          callerContext);
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
  }
```
&#8195;两者均为配置图像生成的获取和加工流水线之后，提交获取请求。区别在于fetchImageFromBitmapCache获取图片的获取深度为仅从BitmapCache中查找，fetchDecodedImage则会在按序在所有Cache中查找，若查找失败则根据Uri从网络或本地等来源获取。fetchImageFromBitmapCache适用于需要快速显示的应用场景，如果没有在较短的时间内获取到图片，就不进行显示。
####1.ImagePipeline流水线配置
&#8195;由于Uri来源、获取方式、处理方式的不同，并且可能设置有不同的图片的加工处理场景，所以需要根据Uri类型和用户设置来组装不同的流水线用于目标图片的获取和处理。
&#8195;那么我们看下getBasicDecodedImageSequence()：
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
&#8195;getDecodedImageProducerSequence()处理了用户设置了图片后处理器的情况，用户可以定义[后处理器](http://fresco-cn.org/docs/modifying-image.html#_)来实现一些对于图片的处理工作。

&#8195;getBasicDecodedImageSequence()则会根据Uri的类别对不同来源和类型的图像配置不同的流水线。
```
 private Producer<CloseableReference<CloseableImage>> getBasicDecodedImageSequence(
      ImageRequest imageRequest) {
    Preconditions.checkNotNull(imageRequest);

    Uri uri = imageRequest.getSourceUri();
    Preconditions.checkNotNull(uri, "Uri is null.");
    if (UriUtil.isNetworkUri(uri)) {
      return getNetworkFetchSequence();
    } else if (UriUtil.isLocalFileUri(uri)) {
      if (MediaUtils.isVideo(MediaUtils.extractMime(uri.getPath()))) {
        return getLocalVideoFileFetchSequence();
      } else {
        return getLocalImageFileFetchSequence();
      }
    } else if (UriUtil.isLocalContentUri(uri)) {
      return getLocalContentUriFetchSequence();
    } else if (UriUtil.isLocalAssetUri(uri)) {
      return getLocalAssetFetchSequence();
    } else if (UriUtil.isLocalResourceUri(uri)) {
      return getLocalResourceFetchSequence();
    } else if (UriUtil.isDataUri(uri)) {
      return getDataFetchSequence();
    } else {
      String uriString = uri.toString();
      if (uriString.length() > 30) {
        uriString = uriString.substring(0, 30) + "...";
      }
      throw new RuntimeException("Unsupported uri scheme! Uri is: " + uriString);
    }
  }
```
关于流水线配置的部分参考[流水线配置](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline/producer_sequence.md)
####2.ImagePipeline图片请求配置
&#8195;在配置完流水线后，ImagePipeline将会发起图片请求，并把图片请求和处理的工作交给流水线完成。
```
  private <T> DataSource<CloseableReference<T>> submitFetchRequest(
      Producer<CloseableReference<T>> producerSequence,
      ImageRequest imageRequest,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
      Object callerContext) {
    try {
      ImageRequest.RequestLevel lowestPermittedRequestLevel =
          ImageRequest.RequestLevel.getMax(
              imageRequest.getLowestPermittedRequestLevel(),
              lowestPermittedRequestLevelOnSubmit);
      SettableProducerContext settableProducerContext = new SettableProducerContext(
          imageRequest,
          generateUniqueFutureId(),
          mRequestListener,
          callerContext,
          lowestPermittedRequestLevel,
        /* isPrefetch */ false,
          imageRequest.getProgressiveRenderingEnabled() ||
              !UriUtil.isNetworkUri(imageRequest.getSourceUri()),
          imageRequest.getPriority());
      return CloseableProducerToDataSourceAdapter.create(
          producerSequence,
          settableProducerContext,
          mRequestListener);
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
  }
```
&#8195;其中，用户可以设置图片请求的获取深度，以控制图片的加载响应速度，而在提交请求前，也会根据用户设置的获取深度来计算本次图片获取的最大深度。最后把图片请求ImageRequest、请求会话ID、调用上下文、请求深度、请求优先级封装在ProducerContext中，并以其和配置好的流水线构造DataSource并发起工作。
####3.ImagePipeline工作发起
&#8195;紧接着上面的内容，CloseableProducerToDataSourceAdapter的create()方法实际上只是根据传入的配置好的流水线(生产者)、生产上下文、Request监听器来构造一个CloseableProducerToDataSourceAdapter对象
```
public static <T> DataSource<CloseableReference<T>> create(
      Producer<CloseableReference<T>> producer,
      SettableProducerContext settableProducerContext,
      RequestListener listener) {
    return new CloseableProducerToDataSourceAdapter<T>(
        producer, settableProducerContext, listener);
  }
```
&#8195;在AbstractProducerToDataSourceAdapter的构造方法中，创建了一个本次图像资源请求的消费者，并将其和生产上下文传递给生产者(流水线)生产结果，于是图片请求和处理的工作被提交到流水线开始执行。
```
  protected AbstractProducerToDataSourceAdapter(
      Producer<T> producer,
      SettableProducerContext settableProducerContext,
      RequestListener requestListener) {
    mSettableProducerContext = settableProducerContext;
    mRequestListener = requestListener;
    mRequestListener.onRequestStart(
        settableProducerContext.getImageRequest(),
        mSettableProducerContext.getCallerContext(),
        mSettableProducerContext.getId(),
        mSettableProducerContext.isPrefetch());
    producer.produceResults(createConsumer(), settableProducerContext);
  }

```
&#8195;那么消费者是什么？
```
public interface Consumer<T> {
  /**
   * 当有新数据产生时由生产者调用。 该方法不应抛出任何异常。
   * 注意，返回结果可能是可关闭引用，为了有效管理内容使用，生产者在onNewResult调用后会关闭该引用，
   * 如果想在onNewResult之后还想要获取返回结果，消费者就必须创建返回资源的副本。
   * @param newResult 
   * @param isLast 当这是最终结果时为true
   */
  void onNewResult(T newResult, boolean isLast);

  void onFailure(Throwable t);

  void onCancellation();

  /**
   * 当生产进度更新时调用
   * @param progress 范围为[0, 1]
   */
  void onProgressUpdate(float progress);
}
```
&#8195;消费者的实现基类为BaseConsumer。值得一提的是，BaseConsumer是线程安全的(ThreadSafe)，所有回调方法均是synchronized的，这样客户端就可以认为所有的回调都发生在同一个线程中。并且BaseConsumer对回调发生的异常会记录。不会出现Producer的多个工作线程同时调用Consumer的回调方法的情况，而避免对DataSource状态的操作冲突。

&#8195;CloseableProducerToDataSourceAdapter的createConsumer()所创建的Consumer触发了RequestListener的回调，并当有新结果或状态返回时会调用setResult()/setFailure()/setProgress()方法来设置结果的内容和状态(对于Jpeg图片来说返回的可能是中间结果)。
####4.ImagePipeline结果处理
&#8195;CloseableProducerToDataSourceAdapter作为DataSource的实现类，继承自抽象基类AbstractDataSource，并维护了图片请求的结果和状态。当新结果返回时，就会调用setResult()来设置新结果的内容和状态，而用户调用getResult()则可以获取该结果。

&#8195;由之前分析可知，Drawee和DataSource之间是通过订阅发布模型来完成图片的请求和结果的获取的。那么让我们来了解一下其订阅发布模型是如何实现的。
AbstractDataSource的成员如下：
```
  private DataSourceStatus mDataSourceStatus;
  @GuardedBy("this")
  private boolean mIsClosed;
  @GuardedBy("this")
  private @Nullable T mResult = null;
  @GuardedBy("this")
  private Throwable mFailureThrowable = null;
  @GuardedBy("this")
  private float mProgress = 0;
  private final ConcurrentLinkedQueue<Pair<DataSubscriber<T>, Executor>> mSubscribers;
```
&#8195;AbstractDataSource通过mResult维护对图片请求结果的引用，通过mProgress和mDataSourceStatus、mIsClosed维护图片请求结果的进度和状态，通过mSubscribers维护订阅者队列(使用队列可以保证订阅内容的有序发送)。
#####订阅
&#8195;在DraweeController的onAttach()方法中，将会创建一个DataSubscriber对象，并向DataSource注册订阅者：
```
  public void subscribe(final DataSubscriber<T> dataSubscriber, final Executor executor) {
    Preconditions.checkNotNull(dataSubscriber);
    Preconditions.checkNotNull(executor);
    boolean shouldNotify;

    synchronized(this) {
      if (mIsClosed) {
        return;
      }

      if (mDataSourceStatus == DataSourceStatus.IN_PROGRESS) {
        mSubscribers.add(Pair.create(dataSubscriber, executor));
      }

      shouldNotify = hasResult() || isFinished() || wasCancelled();
    }

    if (shouldNotify) {
      notifyDataSubscriber(dataSubscriber, executor, hasFailed(), wasCancelled());
    }
  }
```
&#8195;若DataSource已经获取到结果(无论成功还是失败)将直接返回不用注册
#####发布
&#8195;DataSource提供了setResult()、setFailure()、setProgress()的protected方法用于更新DataSource状态，并调用notifyDataSubscribers()通知所有订阅者。以setResult()为例：
```
    protected boolean setFailure(Throwable throwable) {
    boolean result = setFailureInternal(throwable);
    if (result) {
      notifyDataSubscribers();
    }
    return result;
    }
  
    private boolean setResultInternal(@Nullable T value, boolean isLast) {
    T resultToClose = null;
    try {
      synchronized (this) {
        if (mIsClosed || mDataSourceStatus != DataSourceStatus.IN_PROGRESS) {
          resultToClose = value;
          return false;
        } else {
          if (isLast) {
            mDataSourceStatus = DataSourceStatus.SUCCESS;
            mProgress = 1;
          }
          if (mResult != value) {
            resultToClose = mResult;
            mResult = value;
          }
          return true;
        }
      }
    } finally {
      if (resultToClose != null) {
        closeResult(resultToClose);
      }
    }
    }
```
&#8195;setResultInternal会根据返回结果来设置DataSource状态以及图片请求的内容和进度。

&#8195;notifyDataSubscribers()将遍历订阅者队列，并依次发送消息。
```
  private void notifyDataSubscribers() {
    final boolean isFailure = hasFailed();
    final boolean isCancellation = wasCancelled();
    for (Pair<DataSubscriber<T>, Executor> pair : mSubscribers) {
      notifyDataSubscriber(pair.first, pair.second, isFailure, isCancellation);
    }
  }
```
&#8195;向订阅者发送订阅内容的任务在notifyDataSubscriber()方法中实现：
```
 private void notifyDataSubscriber(
      final DataSubscriber<T> dataSubscriber,
      final Executor executor,
      final boolean isFailure,
      final boolean isCancellation) {
    executor.execute(
        new Runnable() {
          @Override
          public void run() {
            if (isFailure) {
              dataSubscriber.onFailure(AbstractDataSource.this);
            } else if (isCancellation) {
              dataSubscriber.onCancellation(AbstractDataSource.this);
            } else {
              dataSubscriber.onNewResult(AbstractDataSource.this);
            }
          }
        });
  }
```
&#8195;实际上是让工作线程触发DataSubscriber的回调以通知DataSource的最新状态。就像代理商打给订购者电话告知他”你的货到了“，”你的货订购失败“，”你的订购请求因为一些原因被取消了“。


&#8195;回顾下DraweeView请求数据的流程：
- step1.在DraweeController的创建过程中，获取DataSourceSupplier对象，DataSourceSupplier的生产方法get()将调用ImagePipeline成员的fetchDecodedImage/fetchImageFromBitmapCache成员方法

- step2.ImagePipeline将为对应的ImageRequest配置获取所需图像的流水线

- step3.创建一个消费者用来发起回调，并将消费者、上下文一并交由流水线(生产者)生产所需图像

- step4.生产者的工作线程通过消费者发起回调来处理生产结果

- step5.DraweeController的onAttach()会触发submitRequest向DataSource注册DataSubscriber来订阅DataSource状态，若DataSource已经获取到结果(无论成功还是失败)将直接返回不用注册。

- step6.获取结果或状态的变化将由DataSource分发给所有订阅者

- step7.DraweeController获取订阅结果后，将成功的返回结果设置到图层予以显示

注：用户可以直接使用BaseDataSubscriber向DataSource注册订阅者，所以同一图片请求可能有多个订阅者。(而控制器是和DraweeHierarchy、DraweeView一一对应的MVC模型，不会把同一个DraweeController设置到多个Model中去，这一点从setController()中也可以看出)
