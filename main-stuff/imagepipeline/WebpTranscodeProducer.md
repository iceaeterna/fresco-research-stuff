##WebpTranscodeProducer
&#8195;WebpTranscodeProducer也是一种处理类型的Producer，用来对WebP格式图片进行解码，其处理业务方法transcodeLastResult()被封装在DelegatingConsumer中。   
&#8195;WebP格式图片转码任务使用StatefulRunnable框架实现。   
&#8195;1.任务的处理封装在getResult()中：
```
 protected EncodedImage getResult() throws Exception {
            PooledByteBufferOutputStream outputStream = mPooledByteBufferFactory.newOutputStream();
            try {
              doTranscode(encodedImageCopy, outputStream);
              CloseableReference<PooledByteBuffer> ref =
                  CloseableReference.of(outputStream.toByteBuffer());
              try {
                EncodedImage encodedImage = new EncodedImage(ref);
                encodedImage.copyMetaDataFrom(encodedImageCopy);
                return encodedImage;
              } finally {
                CloseableReference.closeSafely(ref);
              }
            } finally {
              outputStream.close();
            }
          }
```   
&#8195;调用doTranscode()进行转码，并将输出流构造为EncodedImage返回。   
&#8195;2.转码处理：
```
private static void doTranscode(
      final EncodedImage encodedImage,
      final PooledByteBufferOutputStream outputStream) throws Exception {
    InputStream imageInputStream = encodedImage.getInputStream();
    ImageFormat imageFormat = ImageFormatChecker.getImageFormat_WrapIOException(imageInputStream);
    switch (imageFormat) {
      case WEBP_SIMPLE:
      case WEBP_EXTENDED:
        WebpTranscoder.transcodeWebpToJpeg(imageInputStream, outputStream, DEFAULT_JPEG_QUALITY);
        break;
      case WEBP_LOSSLESS:
      case WEBP_EXTENDED_WITH_ALPHA:
        WebpTranscoder.transcodeWebpToPng(imageInputStream, outputStream);
        break;

      default:
        throw new IllegalArgumentException("Wrong image format");
    }
  }
```
&#8195;转码工作将根据不同的WebP格式调用native方法transcodeWebpToJpeg/transcodeWebpToPng将WebP格式图片转码为Jpeg/Png格式图片。   
&#8195;3.结果处理：
```
          @Override
          protected void disposeResult(EncodedImage result) {
            EncodedImage.closeSafely(result);
          }

          @Override
          protected void onSuccess(EncodedImage result) {
            EncodedImage.closeSafely(encodedImageCopy);
            super.onSuccess(result);
          }

          @Override
          protected void onFailure(Exception e) {
            EncodedImage.closeSafely(encodedImageCopy);
            super.onFailure(e);
          }

          @Override
          protected void onCancellation() {
            EncodedImage.closeSafely(encodedImageCopy);
            super.onCancellation();
          }
        };
```
&#8195;失败、成功、取消处理将关闭对WebP图片的引用，而在处理完结果的回调Consumer后，还将释放WebpTranscodeProducer对结果的引用。