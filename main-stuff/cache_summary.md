我们回顾一下图片获取流程：   
>检查内存缓存，如有，返回   
后台线程开始后续工作   
检查是否在未解码内存缓存中。如有，解码，变换，返回，然后缓存到内存缓存中。   
检查是否在文件缓存中，如果有，变换，返回。缓存到未解码缓存和内存缓存中。   

其中内存缓存BitmapMemoryCache和未解码内存缓存EncodedMemoryCache具有基本相同的组织方式，都采用了引用计数式的方式来维护缓存项。而磁盘缓存BufferedDiskCache则有着不同的处理方式：需要实现对图片资源文件的读写，以实现磁盘文件缓冲的效果。   

接下来，来看Bitmap(Encoded)MemoryCache和BufferedDiskCache的实现：   
- [Bitmap(Encoded)MemoryCache](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/cache/Bitmap(Encoded)MemoryCache.md)
- [BufferedDiskCache](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/cache/BufferedDiskCache.md)

[返回README](https://github.com/icemoonlol/fresco-research-stuff/blob/master/README.md)