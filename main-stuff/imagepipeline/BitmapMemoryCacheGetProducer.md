## BitmapMemoryCacheGetProducer

&#8195;BitmapMemoryCacheGetProducer继承自[BitmapMemoryCacheProducer](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline/BitmapMemoryCacheProducer.md)。   
&#8195;BitmapMemoryCacheGetProducer仅仅是重写了wrapConsumer()和getProducerName()方法，BitmapMemoryCacheGetProducer仅仅是尝试从Bitmap缓存中获取，并确定其后续的Producer会负责处理Bitmap缓存的相关工作，其wrapConsumer()将不做任何额外工作。   
&#8195;那么为什么要这样安排？这里需要提前解释以下ThreadHandoffProducer和BitmapMemoryCacheKeyMultiplexProducer的作用:   
- ThreadHandoffProducer把图像获取工作由前台UI线程转到后台线程池中获取的工作线程
- BitmapMemoryCacheKeyMultiplexProducer则对同一数据源的图像请求进行合并。  

&#8195;所以，**当需要快速地完成图像获取工作，那么在UI线程中就会先尝试从Bitmap缓存中获取图片，后续流水线的处理可能是非常耗时的，所以在尝试从更大的获取深度获取图像时，首先要把工作由前台转移到后台，再对图像请求进行合并以减少重复的请求工作。这种对于复杂任务的安排方式值得借鉴。**
