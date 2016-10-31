
《AFNetWorking是如何进行缓存的？之AFImageCache & NSURLCache 详解》
===
如果你是一个正在使用由Matt Thompson’s开发的网络库 `AFNetWorking`(如果你还没有使用，那你还在等什么？)的iOS开发者，也许你一直很好奇和困惑它的缓存机制，并且想要了解如何更好地充分利用它

` AFImagecache` : 继承于NSCache，AFNetworking的图片内存缓存的类。
` NSURLCache`   : NSURLConnection的默认缓存机制，用于存储NSURLResponse对象：一个默认缓存在内存，并且可以通过一些配置操作可以持久缓存到磁盘的类。
