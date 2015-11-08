##ResizeAndRotateProducer
&#8195;ResizeAndRotateProducer根据EXIF元数据对JPEG图片进行大小调整和旋转。   
&#8195;先了解一下EXIF的概念：   
&#8195;EXIF（Exchangeable Image File）是“可交换图像文件”的缩写，当中包含了专门为数码相机的照片而定制的元数据，可以记录数码照片的拍摄参数、缩略图及其他属性信息。Exif 信息就是由数码相机在拍摄过程中采集一系列的信息，然后把信息放置在我们熟知的 JPEG/TIFF 文件的头部。Exif 所记录的元数据信息非常丰富，主要包含了以下几类信息：   
- 拍摄日期
- 拍摄器材（机身、镜头、闪光灯等）
- 拍摄参数（快门速度、光圈F值、ISO速度、焦距、测光模式等）
- 图像处理参数（锐化、对比度、饱和度、白平衡等）
- 图像描述及版权信息
- GPS定位数据
- 缩略图  

&#8195;很多图像编辑器会自动读取Exif数据来对图像进行优化，最常见的便是从 Exif中读取出相机姿态信息，从而自动识别出竖拍甚至是颠倒拍摄的照片并对其进行旋转校正。也有一些软件可以根据 Exif中的机内处理信息对图像进行针对性优化，从而保证图像不会因为过度处理而失真。  
&#8195;ResizeAndRotateProducer的处理与DecodeProducer类似，都是使用JobScheduler作业执行框架，将处理业务封装在DelegatingConsumer中，在下个流水段获取结果后触发回调进行处理。  
#####什么样的图片需要进行大小调整和旋转？
```
  private static TriState shouldTransform(
      ImageRequest request,
      EncodedImage encodedImage) {
    if (encodedImage == null || encodedImage.getImageFormat() == ImageFormat.UNKNOWN) {
      return TriState.UNSET;
    }
    if (encodedImage.getImageFormat() != ImageFormat.JPEG) {
      return TriState.NO;
    }
    return TriState.valueOf(
        getRotationAngle(request, encodedImage) != 0 ||
            shouldResize(getScaleNumerator(request, encodedImage)));
  }
```
1. 基本要求就是该图片必须是JPEG格式。   
2. 在拍照过程中，由于拍照姿势不同，可能图片发生了旋转。当图片携带的旋转角度不为0时，说明发生了旋转(可能是90°、180°、270°)，那么这里就有必要进行旋转调整
```
  private static int getRotationAngle(ImageRequest imageRequest, EncodedImage encodedImage) {
    if (!imageRequest.getAutoRotateEnabled()) {
      return 0;
    }
    int rotationAngle = encodedImage.getRotationAngle();
    Preconditions.checkArgument(
        rotationAngle == 0 || rotationAngle == 90 || rotationAngle == 180 || rotationAngle == 270);
    return rotationAngle;
  }
```   
3.用户可通过ResizeOptions对图像进行大小调整的处理。但在大小调整之前需要计算调整比例以低于大小限制。   
(1). 比例因子的计算
```
private static int getScaleNumerator(
      ImageRequest imageRequest,
      EncodedImage encodedImage) {
    final ResizeOptions resizeOptions = imageRequest.getResizeOptions();
    if (resizeOptions == null) {
      return JpegTranscoder.SCALE_DENOMINATOR;
    }
//如果图片发生了方向旋转(90度或270度)，那么，需要交换图像的高宽数据，
    final int rotationAngle = getRotationAngle(imageRequest, encodedImage);
    final boolean swapDimensions = rotationAngle == 90 || rotationAngle == 270;
    final int widthAfterRotation = swapDimensions ? encodedImage.getHeight() :
            encodedImage.getWidth();
    final int heightAfterRotation = swapDimensions ? encodedImage.getWidth() :
            encodedImage.getHeight();
//确定图像比例，并计算比例因子，比例因子的最小为1，最大为MAX_JPEG_SCALE_NUMERATOR
    float ratio = determineResizeRatio(resizeOptions, widthAfterRotation, heightAfterRotation);
    int numerator = roundNumerator(ratio);
    if (numerator > MAX_JPEG_SCALE_NUMERATOR) {
      return MAX_JPEG_SCALE_NUMERATOR;
    }
    return (numerator < 1) ? 1 : numerator;
  }
```   
比例因子的计算如下：
```
static int roundNumerator(float maxRatio) {
    return (int) (ROUNDUP_FRACTION + maxRatio * JpegTranscoder.SCALE_DENOMINATOR);
  }
```   
比例因子为(int)(0.67 + 缩放比例 * 8)，最大比例因子为16，即最多可以放大约2倍大小
(2).比例因子的调整   
&#8195;用户可以通过ResizeOptions来设置所希望得到图像的大小，但是对于图像大小的调整需要锁定高宽比，使得图像大小在可接受的范围内(分辨率小于2048，但这个值实际上应该通过Canvas.getMaximumBitmapWidth/Height调用进行计算)。
```
static float determineResizeRatio(
      ResizeOptions resizeOptions,
      int width,
      int height) {
    if (resizeOptions == null) {
      return 1.0f;
    }
    final float widthRatio = ((float) resizeOptions.width) / width;
    final float heightRatio = ((float) resizeOptions.height) / height;
    float ratio = Math.max(widthRatio, heightRatio);
    if (width * ratio > MAX_BITMAP_SIZE) {
      ratio = MAX_BITMAP_SIZE / width;
    }
    if (height * ratio > MAX_BITMAP_SIZE) {
      ratio = MAX_BITMAP_SIZE / height;
    }
    return ratio;
  }
```   
#####大小和方向调整的实现：
doTransform()：调用了JpegTranscoder类的native方法transcodeJpeg()，对图像输入流根据旋转角度和比例因子进行旋转和大小调整处理，并将转码输出解析为新的JPEG图片。
```
    private void doTransform(EncodedImage encodedImage, boolean isLast) {
     //...
      try {
        int numerator = getScaleNumerator(imageRequest, encodedImage);
        extraMap = getExtraMap(encodedImage, imageRequest, numerator);
        is = encodedImage.getInputStream();
        JpegTranscoder.transcodeJpeg(
            is,
            outputStream,
            getRotationAngle(imageRequest, encodedImage),
            numerator,
            DEFAULT_JPEG_QUALITY);
        CloseableReference<PooledByteBuffer> ref =
            CloseableReference.of(outputStream.toByteBuffer());
        try {
          ret = new EncodedImage(ref);
          ret.setImageFormat(ImageFormat.JPEG);
          try {
            ret.parseMetaData();
           //...
    }
```   
关于图片处理的Native实现将在以后进行。
