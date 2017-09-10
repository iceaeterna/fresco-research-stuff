  Fresco使用了类似于MVC模式的结构来完成图片显示、图像控制和图片数据加载框架的设计。
  ![](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/resources/img/fresco_mvc.png)
  __DraweeView(视图层)：__
  负责图片显示以及交互事件的转发。DraweeView通过一个DraweeHolder对象持有对控制器DraweeController和模型DraweeHierarchy的引用，分别用来(DraweeController)设置图像数据源、交互事件以及图像数据处理，和(DraweeHierarchy)获取最终用于绘制显示的Drawable图像。
  视图层的分离，解除了与图像显示、图像数据获取的重耦合，并且当图像数据获取方式变化、图像显示效果需求变化时，不会对视图层的逻辑产生影响。

  __DraweeController(控制器)：__
  负责和图像数据源(默认为Fresco的Image Pipeline，也可以设置为Volley)交互，以及在图像数据基础上进行一些旋转、大小控制等。

  __DraweeHierarchy（业务逻辑层）:__
  负责组织和维护Drawee的图层，以设置图像的显示效果，如占位图、圆角等。

  本系列内容主要分析默认配置下，Fresco的业务逻辑实现过程，分析和探讨的重点在于实现过程中所使用的设计架构、设计模式、技术细节。
##### 说明
  在阅读本系列内容之前，应当参考[fresco-doc-cn](http://fresco-cn.org/docs/concepts.html#_)了解fresco的基本特性和基本使用方法，以对fresco的分析有着更深的理解和体会。此外，如果有兴趣，不妨可以对比[Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader)的技术实现来学习。
## 目录
##### [一、Drawee的基本框架与业务实现](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/drawee_summary.md)
##### [二、ImagePipeline工作过程](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/imagepipeline_summary.md)
##### [三、Fresco缓存体系](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/cache_summary.md)
##### [四、Fresco Native内存模型](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/memory_summary.md)
##### [五、DraweeHierarchy图层管理(暂不研究)](http://)
 

