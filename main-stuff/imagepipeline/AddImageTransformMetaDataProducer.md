## AddImageTransformMetaDataProducer
&#8195;AddImageTransformMetaDataProducer同样通过DelegatingConsumer委派处理。

```
    protected void onNewResultImpl(EncodedImage newResult, boolean isLast) {
      if (newResult == null) {
        getConsumer().onNewResult(null, isLast);
        return;
      }
      if (!EncodedImage.isMetaDataAvailable(newResult)) {
        newResult.parseMetaData();
      }
      getConsumer().onNewResult(newResult, isLast);
	}
```   
&#8195;其主要工作是对图片格式、高宽、角度元数据进行解析，以供ResizeAndRotateProducer进行处理。
```
  public void parseMetaData() {
    final ImageFormat imageFormat = ImageFormatChecker.getImageFormat_WrapIOException(
        getInputStream());
    mImageFormat = imageFormat;
    if (!ImageFormat.isWebpFormat(imageFormat)) {
      Pair<Integer, Integer> dimensions = BitmapUtil.decodeDimensions(getInputStream());
      if (dimensions != null) {
        mWidth = dimensions.first;
        mHeight = dimensions.second;

        // Load the rotation angle only if we have the dimensions
        if (imageFormat == ImageFormat.JPEG) {
          if (mRotationAngle == UNKNOWN_ROTATION_ANGLE) {
            mRotationAngle = JfifUtil.getAutoRotateAngleFromOrientation(
                JfifUtil.getOrientation(getInputStream()));
          }
        } else {
          mRotationAngle = 0;
        }
      }
    }
  }
```

[返回ProducerSequence](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline/producer_sequence.md)
