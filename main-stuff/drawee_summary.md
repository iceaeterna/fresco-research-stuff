####一、Drawee的基本框架与业务实现
&#8195;之前提到，Fresco对图片的加载和显示采用了类似MVC模式的结构，其中DraweeView、DraweeController、DraweeHierarchy分别对应于View(视图)、Controller(控制器)、Model(模型/业务逻辑)。
&#8195;接下来以常用的SimpleDraweeView为入口，分析Drawee的基本框架与业务实现。SimpleDraweeView继承过程为GenericDraweeView、DraweeView。

#####1.DraweeView
```
public class DraweeView<DH extends DraweeHierarchy> extends ImageView {

  private DraweeHolder<DH> mDraweeHolder;

  public DraweeView(Context context) {
    super(context);
    init(context);
  }

  public DraweeView(Context context, AttributeSet attrs) {
    super(context, attrs);
    init(context);
  }

  public DraweeView(Context context, AttributeSet attrs, int defStyle) {
    super(context, attrs, defStyle);
    init(context);
  }
 //关键成员，DraweeView与DraweeController和DraweeHierarchy的纽带
  private void init(Context context) {
    mDraweeHolder = DraweeHolder.create(null, context);
  }
```
&#8195;DraweeView的类型参数为< DH extends DraweeHierarchy >，即泛型边界为DraweeHierarchy或DraweeHierarchy的导出类型。而DraweeView又使用该类型参数定义了一个DraweeHolder对象，并在构造方法init()中调用DraweeHolder的create()来初始化该成员。
   
&#8195;DraweeHolder对象包括一个泛型成员mHierarchy和一个DraweeController类成员mController，分别为MVC模式中的Model和Controller。而DraweeView所做的工作只是管理其所持有的DraweeHolder的DraweeHierarchy和DraweeController，绑定和获取图像数据源，并将View事件传递给DraweeHolder进行响应处理。这样，当用户使用自定义View，甚至可以直接使用DraweeHolder来进行图像内容的管理。

&#8195;注意，DraweeView以后将直接继承自View而非ImageView，所以类似setImageXxx, setScaleType等ImageView的API最好不要使用。

#####2.GenericDraweeView
```
public GenericDraweeView(Context context, GenericDraweeHierarchy hierarchy) {
    super(context);
    setHierarchy(hierarchy);
  }

  public GenericDraweeView(Context context) {
    super(context);
    inflateHierarchy(context, null);
  }

  public GenericDraweeView(Context context, AttributeSet attrs) {
    super(context, attrs);
    inflateHierarchy(context, attrs);
  }

  public GenericDraweeView(Context context, AttributeSet attrs, int defStyle) {
    super(context, attrs, defStyle);
    inflateHierarchy(context, attrs);
  }

```

&#8195;构造方法一：
&#8195;调用传入GenericDraweeHierarchy来创建DraweeHolder对象并调用setHierarchy()完成初始化工作

&#8195;构造方法二~四：
&#8195;调用inflateHierarchy(),使用GenericDraweeHierarchyBuilder([构造者模式](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/drawee-research-stuff/Builder%20Pattern.md))来构造一个GenericDraweeHierarchy，以此完成初始化工作 ，其中方法二没有指定XML传入的属性值AttributeSet ，将直接使用默认值进行设置。
&#8195;GenericDraweeView为DraweeView创建了一个GenericDraweeHierarchy，但是没有控制器DraweeController。这是因为控制器的实现决定了图像数据获取和处理的方式，作为DraweeView的通用实现类，对DraweeController的初始化应推迟到具体的DraweeView实现类中实现。

#####3.SimpleDraweeView
```
  public SimpleDraweeView(Context context) {
    super(context);
    init();
  }
```
&#8195;SimpleDraweeView的构造方法的重点在于init()方法。
```
  private void init() {
    if (isInEditMode()) {
      return;
    }
    Preconditions.checkNotNull(
        sDraweeControllerBuilderSupplier,
        "SimpleDraweeView was not initialized!");
    mSimpleDraweeControllerBuilder = sDraweeControllerBuilderSupplier.get();
  }
  ```
&#8195;init()初始化了mSimpleDraweeControllerBuilder成员，从命名上可以看出，该成员是SimpleDraweeView所持有的DraweeController的建造者([构造者模式](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/drawee-research-stuff/Builder%20Pattern.md))。它是由Fresco系统初始化时创建的Fresco静态私有类成员sDraweeControllerBuilderSupplier所提供的建造者([Supplier模式](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/drawee-research-stuff/Supplier.md))。
######<font color = red>(1).调用SimpleDraweeView的setImageURI()设置图像来源</font>
&#8195;那么有了Builder，DraweeController是在是什么时候被创建的？我们可以在SimpleDraweeView中找到答案：
```
  public void setImageURI(Uri uri, @Nullable Object callerContext) {
    DraweeController controller = mSimpleDraweeControllerBuilder
        .setCallerContext(callerContext)
        .setUri(uri)
        .setOldController(getController())
        .build();
    setController(controller);
  }
```
&#8195;DraweeController的默认建造方式下，只是简单地设置了CallerContext和图片的Uri，如果需要对图像数据获取和设置更好地进行控制，就应该使用方式(2)，来自定义一个DraweeController。
```
  /** Sets the controller. */
  public void setController(@Nullable DraweeController draweeController) {
    mDraweeHolder.setController(draweeController);
    super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
  }
```
&#8195;在DraweeHolder的setController()中，会为DraweeController设置其DraweeHierarchy，以建立起Drawee的MVC模型，并将调用attachController()连接控制器，并触发控制器的onAttach()来发送图像请求。
随后会获取DraweeHierarchy的顶层图像作为显示图像，但由于图像请求结果还未到达，这里的getTopLevelDrawable()获取的可能只是占位图。
######<font color = red>(2).直接创建一个DraweeController，并调用setController()设置到SimpleDraweeView中</font>
&#8195;与(1)类似，不过用户可以直接定义和配置DraweeController，并直接设置到SimpleDraweeView中，来获取对图像数据的获取和加工的更多控制。

#####4.DraweeController
```
public interface DraweeController {
  @Nullable
  DraweeHierarchy getHierarchy();

  void setHierarchy(@Nullable DraweeHierarchy hierarchy);

  void onAttach();

  void onDetach();

  boolean onTouchEvent(MotionEvent event);

  Animatable getAnimatable();
}
```
&#8195;DraweeController持有对DraweeHierarchy的引用，用来对图层的图像内容进行设置。DraweeController定义了onAttach、onDetach、onTouchEvent、getAnimatable这几个需要实现的接口方法，用来实现DraweeController连接到DraweeHierarchy或断开连接的业务处理，以及对触屏事件的响应。

#####5.AbstractDraweeController
&#8195;AbstractDraweeController作为DraweeController的基础实现类，定义了DraweeController的核心业务逻辑，即负责向图像数据源获取图像内容，并设置到DraweeHierarchy中。

&#8195;AbstractDraweeController的核心业务模块有DraweeEventTracker、DeferredReleaser、Executor和可选模块RetryManager、GestureDetector、ControllerListener。
![](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/img/AbstractDraweeController_model.png)
&#8195;AbstractDraweeController的构造方法，也是对如上模块进行初始化的过程。各个模块的作用如下：
&#8195;(1).DraweeEventTracker：维护一个Event队列来记录控制器的控制操作(事件)

&#8195;(2).DeferredReleaser：管理DraweeController在断开与DraweeHierarchy连接后的资源释放工作。但值得一提的是，DraweeController对象<font color = red>使用的生命周期</font>并不是随着(DraweeController的)Detach事件到来而立刻结束的，这样会加重虚拟机对象创建和回收的开销。在DeferredReleaser在真正地将DraweeController所使用的资源释放(包括结束对所获取的图像的引用)前，若有新的DraweeController创建请求，则会重用”废弃的“DraweeController，将其以新的参数进行初始化后(Rebuild过程)返回给使用者，以避免对DraweeController繁重的创建和回收工作。

&#8195;(3).Executor：Executor框架是Java 5引入的一系列并发库中与executor相关的类，包括线程池、Executor、Executors、ExecutorService、CompletionService、Future、Callable等。并发编程把任务拆分为若干个Runnable，然后在提交给一个Executor执行。Executor在执行时使用内部的线程池完成操作。当获取到图像的请求结果时，就会由Executor完成结果的分发工作。

&#8195;(4).RetryManager：管理重试加载(重试加载使能/次数限制)

&#8195;(5).GestureDetector：用来处理派发的TouchEvent

&#8195;(6).ControllerListener：用来对AbstractDraweeController进行调试和单元测试

#####6.PipelineDraweeController
&#8195;在SimpleDraweeView的介绍中，我们看到，将会通过sDraweeControllerBuilderSupplier.get()获取一个DraweeController的建造者来创建DraweeController，sDraweeControllerBuilderSupplier是由Fresco在初始化Drawee时创建的PipelineDraweeControllerBuilderSupplier静态实例。
```
  private static void initializeDrawee(Context context) {
    sDraweeControllerBuilderSupplier = new PipelineDraweeControllerBuilderSupplier(context);
    SimpleDraweeView.initialize(sDraweeControllerBuilderSupplier);
  }
```
&#8195;PipelineDraweeControllerBuilderSupplier实现了Supplier接口，会提供一个get()方法，来返回某一类型(PipelineDraweeControllerBuilder)的实例。PipelineDraweeControllerBuilderSupplier的构造方法如下：
```
 public PipelineDraweeControllerBuilderSupplier(Context context) {
    this(context, ImagePipelineFactory.getInstance());
  }

public PipelineDraweeControllerBuilderSupplier(
      Context context,
      ImagePipelineFactory imagePipelineFactory,
      Set<ControllerListener> boundControllerListeners) {
    mContext = context;
    mImagePipeline = imagePipelineFactory.getImagePipeline();
    mPipelineDraweeControllerFactory = new PipelineDraweeControllerFactory(
        context.getResources(),
        DeferredReleaser.getInstance(),
        imagePipelineFactory.getAnimatedDrawableFactory(),
        UiThreadImmediateExecutorService.getInstance());
    mBoundControllerListeners = boundControllerListeners;
  }
```
&#8195;根据Fresco的初始化流程，我们知道，在初始化Drawee模块之前，会先初始化ImagePipeLineFactory([单例工厂模式](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/drawee-research-stuff/Factory%20Pattern.md))，而这里将获取ImagePipelineFactory的实例并调用其getImagePipeline()来获取一个ImagePipeline对象，来作为PipelineDraweeControllerBuilder的构造参数，而ImagePipeline可以看做是图像获取工作的流水线的抽象整体。

&#8195;PipelineDraweeControllerBuilderSupplier的get()方法将返回一个PipelineDraweeControllerBuilder实例作为DraweeController的建造者。
```
  public PipelineDraweeControllerBuilder get() {
    return new PipelineDraweeControllerBuilder(
        mContext,
        mPipelineDraweeControllerFactory,
        mImagePipeline,
        mBoundControllerListeners);
  }
```
&#8195;Drawee各个构件的创建和关系方式值得借鉴，<strong>DraweeController的构造采用了PipelineDraweeControllerBuilderSupplier
->PipelineDraweeControllerBuilder->PipelineDraweeControllerFactory->PipelineDraweeController的创建层次。</strong>
&#8195;那么这样设计的目的是什么？
#####(1).Supplier模式：
&#8195;Supplier可以看做是一个类似工厂模式的实现，封装了AbstractDraweeControllerBuilder的构造过程，以降低模块之间的耦合性。
根据get()方法的实现，可以想象，若不采用这种Supplier的方式，每构造一个Builder就必须像这样进行。
```
PipelineDraweeControllerBuilder builder = new PipelineDraweeControllerBuilder(
        context,
        new PipelineDraweeControllerFactory(
        	context.getResources(),
        	DeferredReleaser.getInstance(),
        	ImagePipelineFactory.getInstance().getAnimatedDrawableFactory(),
        	UiThreadImmediateExecutorService.getInstance()),
        ImagePipelineFactory.getInstance().getImagePipeline(),
        null);
  }
```
&#8195;如果需要修改一个Builder的构造参数，那么就必须对每处PipelineDraweeControllerBuilder对象的创建都进行修改，就得对代码进行非常繁琐的调整，所以就需要<font color = red>将创建实例的过程与使用实例的过程分开，将创建实例(初始化工作)的大量工作进行分离。</font>
#####(2).Builder模式：
&#8195;DraweeController的创建使用了一种简化的建造者模式。省略了Director和抽象建造者Builder，直接使用一个具体建造者PipelineDraweeController。
&#8195;那么为什么要对DraweeController使用建造者模式？
&#8195;一方面，用户可以通过xml文件配置和代码中进行设置，来为DraweeController设置很多属性对图像的数据获取、处理过程进行控制，这导致对象内部的组成构件面临着复杂的变化，所以就需要建造模式来完成不同组成构件的组合，来建造一个完整的DraweeController对象。而这种简化的建造者模式在Android的AlertDialog中的得到应用。<font color = red>如果我们想要设计一个具有丰富可配置项的对象时，这种建造者模式无疑是一个绝好的选择。</font>
&#8195;另一方面，PipelineDraweeController封装了对DraweeController重用过程实现，可以更好地对DraweeController对象进行管理。
#####(3).Factory模式：
&#8195;如之前分析，DraweeController的构造会重用之前”废弃但未丢弃“的DraweeController对象，但如果旧的DraweeController已经被销毁，那么就将使用PipelineDraweeControllerFactory生产一个新的DraweeController对象。
```
  protected PipelineDraweeController obtainController() {
    DraweeController oldController = getOldController();
    PipelineDraweeController controller;
    if (oldController instanceof PipelineDraweeController) {
      controller = (PipelineDraweeController) oldController;
      controller.initialize(
          obtainDataSourceSupplier(),
          generateUniqueControllerId(),
          getCallerContext());
    } else {
      controller = mPipelineDraweeControllerFactory.newController(
          obtainDataSourceSupplier(),
          generateUniqueControllerId(),
          getCallerContext());
    }
    return controller;
  }
```

&#8195;可以看到，PipelineDraweeController构造参数的<font color = red>获取过程比较复杂</font>，PipelineDraweeControllerFactory封装了PipelineDraweeController的构造参数，来将创建实例(初始化工作)的工作分离。
这里先说一下obtainDataSourceSupplier()将调用getDataSourceForRequest()，并把图像请求交给ImagePipeline尝试获取图像。也就是说，图像获取工作从DraweeController的构造就开始了。
```
  public PipelineDraweeController newController(
      Supplier<DataSource<CloseableReference<CloseableImage>>> dataSourceSupplier,
      String id,
      Object callerContext) {
    return new PipelineDraweeController(
        mResources,
        mDeferredReleaser,
        mAnimatedDrawableFactory,
        mUiThreadExecutor,
        dataSourceSupplier,
        id,
        callerContext);
  } 
```
<font color = red><strong>
&#8195;以上内容可以看做是一个系统常用的、用户可配置的复杂组件的实用构造方式：
&#8195;由工厂Supplier/ProductFactory生产组件的建造者，由用户进行配置后，经工厂处理配置项来设定Product各个模块，最后生产组件Product。</strong>
</font>

&#8195;最后我们需要关注的是PipelineDraweeController的核心成员：一个提供请求图像数据源DataSource的Supplier，该成员将作为Drawee与ImagePipeline交互的接口。
```
  private Supplier<DataSource<CloseableReference<CloseableImage>>> mDataSourceSupplier;
```

#####7.图片获取(与设置)过程
&#8195;在View附加到窗体Window(PhoneWindow)时，会回调onAttachToWindow，并派发给AbstractDraweeController调用其onAttach()方法进行处理。并且当DraweeController连接到DraweeHierarchy时也会触发onAttach()方法：
```
  public void onAttach() {
    //...ignore Log 
    mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
    Preconditions.checkNotNull(mSettableDraweeHierarchy);
    mDeferredReleaser.cancelDeferredRelease(this);
    mIsAttached = true;
    if (!mIsRequestSubmitted) {
      submitRequest();
    }
  }
```
onAttach()将调用submitRequest()请求图像资源：
```
protected void submitRequest() {
    mEventTracker.recordEvent(Event.ON_DATASOURCE_SUBMIT);
    //ControllerListener的请求资源事件监听
    getControllerListener().onSubmit(mId, mCallerContext);
    //加载进度条的同步
    mSettableDraweeHierarchy.setProgress(0, true);
    mIsRequestSubmitted = true;
    mHasFetchFailed = false;
    //图像数据源
    mDataSource = getDataSource();
   //...ignore LOG
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
   //数据订购者DataSubscriber
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
        //不同结果的回调方法：
        //获取到新的返回值
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            // isFinished must be obtained before image, otherwise we might set intermediate result
            // as final image.
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            if (image != null) {
              onNewResultInternal(id, dataSource, image, progress, isFinished, wasImmediate);
            } else if (isFinished) {
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }
        //失败
          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
          }
        //进度更新
          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            onProgressUpdateInternal(id, dataSource, progress, isFinished);
          }
        };
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
  }
```
&#8195;对图像内容的获取过程可能是缓慢的，这个过程可以想象成你向代理商订购一件货物，并留下电话号码，代理商从工厂拿到货物后会打电话通知你。这个代理商就是DataSource，电话号码就是DataSubscriber。这就是Drawee所使用的[订阅发布模型](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/drawee-research-stuff/Observer%20Pattern.md)。
&#8195;当获取到新的返回值时，若返回结果不为空，即可获取订阅的图像数据，否则，若本次订阅结束仍没有获取订阅内容，则本次请求失败。
&#8195;而返回请求失败时，将调用onFailureImpl()并返回失败原因。
&#8195;当加载进度更新时，会调用onProgressUpdate()进行进度条的更新。

下面看一下Drawee收到货物后是如何处理的：
(1).onNewResultInternal()：
```
// ...
// create drawable
    Drawable drawable;
    try {
      //根据返回的图像构造一个Drawable(数据源可能是不同的图像类型)
      drawable = createDrawable(image);
    } catch (Exception exception) {
      logMessageAndImage("drawable_failed @ onNewResult", image);
      releaseImage(image);
      onFailureInternal(id, dataSource, exception, isFinished);
      return;
    }
    //设置新的图像并释放掉旧的图像资源，以此实现图像资源的自动管理
    T previousImage = mFetchedImage;
    Drawable previousDrawable = mDrawable;
    mFetchedImage = image;
    mDrawable = drawable;
    try {
      // set the new image
      if (isFinished) {
      	//释放掉持有的DataSource引用，更新DraweeHierarchy图层，通知ControllerListener
        logMessageAndImage("set_final_result @ onNewResult", image);
        mDataSource = null;
        mSettableDraweeHierarchy.setImage(drawable, 1f, wasImmediate);
        getControllerListener().onFinalImageSet(id, getImageInfo(image), getAnimatable());
        // IMPORTANT: do not execute any instance-specific code after this point
      } else {
      	//设置中间结果作为图像显示内容
        logMessageAndImage("set_intermediate_result @ onNewResult", image);
        mSettableDraweeHierarchy.setImage(drawable, progress, wasImmediate);
        getControllerListener().onIntermediateImageSet(id, getImageInfo(image));
        // IMPORTANT: do not execute any instance-specific code after this point
      }
    } finally {
      //释放掉之前占用的图像资源
      if (previousDrawable != null && previousDrawable != drawable) {
        releaseDrawable(previousDrawable);
      }
      if (previousImage != null && previousImage != image) {
        logMessageAndImage("release_previous_result @ onNewResult", previousImage);
        releaseImage(previousImage);
      }
    }
    
```
以上是获取到新的返回值时的处理，其中isFinished为false的情况支持了渐进式图片加载。
(2).onFailureInternal()：
```
 if (isFinished) {
      logMessageAndFailure("final_failed @ onFailure", throwable);
      mDataSource = null;
      mHasFetchFailed = true;
      // Set the previously available image if available.
      if (mRetainImageOnFailure && mDrawable != null) {
        mSettableDraweeHierarchy.setImage(mDrawable, 1f, true);
      } else if (shouldRetryOnTap()) {
        mSettableDraweeHierarchy.setRetry(throwable);
      } else {
        mSettableDraweeHierarchy.setFailure(throwable);
      }
      getControllerListener().onFailure(mId, throwable);
      // IMPORTANT: do not execute any instance-specific code after this point
    } else {
      logMessageAndFailure("intermediate_failed @ onFailure", throwable);
      getControllerListener().onIntermediateImageFailed(mId, throwable);
      // IMPORTANT: do not execute any instance-specific code after this point
    }
```
失败的处理根据设置有不同的处理方式：
	a.维持上次图片内容显示
    b.重试处理
    c.加载失败处理
    
此外AbstractDraweeController实现了GestureDetector.ClickListener的onClick接口，在当用户点击图片时，会请求重试加载图片。
```
  public boolean onClick() {
    if (shouldRetryOnTap()) {
      mRetryManager.notifyTapToRetry();
      mSettableDraweeHierarchy.reset();
      submitRequest();
      return true;
    }
    return false;
  }
```
