####二、ImagePipeline工作过程
#####1.ImagePipeline简介
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
上图中，disk cache实际包含了未解码的内存缓存在内，统一在一起只是为了逻辑稍微清楚一些。关于缓存，更多细节可以参考这里。
Image pipeline 可以从本地文件加载文件，也可以从网络。支持PNG，GIF，WebP, JPEG。

#####2.ImagePipeline构造
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
- ~~mDownsampleEnabled：是否下载样例图片~~
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
#####3.ImagePipeline工作过程
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
#####(1).ImagePipeline流水线配置
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
关于流水线配置的部分参考
