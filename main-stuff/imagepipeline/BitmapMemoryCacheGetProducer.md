## BitmapMemoryCacheGetProducer

&#8195;BitmapMemoryCacheGetProducer继承自[BitmapMemoryCacheProducer](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline/BitmapMemoryCacheProducer.md)。   
&#8195;BitmapMemoryCacheGetProducer仅仅是重写了wrapConsumer()和getProducerName()方法，BitmapMemoryCacheGetProducer仅仅是尝试从Bitmap缓存中获取，并确定其后续的Producer会负责处理Bitmap缓存的相关工作，其wrapConsumer()将不做任何额外工作，也就是说这是一个只读缓存。   

