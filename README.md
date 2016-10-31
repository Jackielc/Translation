
《AFNetWorking是如何进行缓存的？之AFImageCache & NSURLCache 详解》
===
如果你是一个正在使用由Matt Thompson’s开发的网络库 `AFNetWorking`(如果你还没有使用，那你还在等什么？)的iOS开发者，也许你一直很好奇和困惑它的缓存机制，并且想要了解如何更好地充分利用它

1. ` AFImagecache` : 继承于NSCache，AFNetworking的图片内存缓存的类。
2. ` NSURLCache`   : NSURLConnection的默认缓存机制，用于存储NSURLResponse对象：一个默认缓存在内存，并且可以通过一些配置操作可以持久缓存到磁盘的类。

AFImageCache是如何工作的？
---

`AFImageCache`属于`UIImageView+AFNetworking`的一部分,继承于`NSCache`,以URL(从`NSURLRequest`对象中获取)字符串作为key值来存储`UIImage`对象。
`AFImageCache`的定义如下：(这里我们声明了一个2M内存、100M磁盘空间的`NSURLCache`对象。)

```objective-c
@interface AFImageCache : NSCache <AFImageCache>

// singleton instantiation :

+ (id <AFImageCache>)sharedImageCache {
    static AFImageCache *_af_defaultImageCache = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _af_defaultImageCache = [[AFImageCache alloc] init];

// clears out cache on memory warning :

    [[NSNotificationCenter defaultCenter] addObserverForName:UIApplicationDidReceiveMemoryWarningNotification object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification * __unused notification) {
        [_af_defaultImageCache removeAllObjects];
    }];
});

// key from [[NSURLRequest URL] absoluteString] :

static inline NSString * AFImageCacheKeyFromURLRequest(NSURLRequest *request) {
    return [[request URL] absoluteString];
}

@implementation AFImageCache

// write to cache if proper policy on NSURLRequest :

- (UIImage *)cachedImageForRequest:(NSURLRequest *)request {
    switch ([request cachePolicy]) {
        case NSURLRequestReloadIgnoringCacheData:
        case NSURLRequestReloadIgnoringLocalAndRemoteCacheData:
            return nil;
        default:
            break;
    }

    return [self objectForKey:AFImageCacheKeyFromURLRequest(request)];
}

// read from cache :

- (void)cacheImage:(UIImage *)image
        forRequest:(NSURLRequest *)request {
    if (image && request) {
        [self setObject:image forKey:AFImageCacheKeyFromURLRequest(request)];
    }
}
```
