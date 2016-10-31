
AFNetWorking是如何进行缓存的？之AFImageCache & NSURLCache 详解
===
[原文－Tim Brandt《How Does Caching Work in AFNetworking? : AFImageCache & NSUrlCache Explained》](http://blog.originate.com/blog/2014/02/20/afimagecache-vs-nsurlcache/)
   
   如果你是一个正在使用由[Matt Thompson](https://github.com/mattt)开发的网络库 `AFNetWorking`(如果你还没有使用，那你还在等什么？)的iOS开发者，也许你一直很好奇和困惑它的缓存机制，并且想要了解如何更好地充分利用它?
   
`AFNetworking`实际上利用了两套单独的缓存机制:
---

1. `AFImagecache` : 继承于`NSCache`，`AFNetworking`的图片内存缓存的类。
2. `NSURLCache`   : `NSURLConnection`的默认缓存机制，用于存储`NSURLResponse`对象：一个默认缓存在内存，并且可以通过一些配置操作可以持久缓存到磁盘的类。

`AFImageCache`是如何工作的？
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
`AFImageCache`是`NSCache`的私有实现，它把所有可访问的`UIImage`对象存入`NSCache`中，并控制着`UIImage`对象应该在何时释放，如果`UIImage`对象释放的时候你希望去做一些监听操作，你可以实现`NSCacheDelegate`的 `cache:willEvictObject` 代理方法。
   Matt Thompson’s已经谦虚的告诉我在AFNetworking2.1版本中可通过`setSharedImageCache`方法来配置`AFImageCache`，这里是 AFN2.2.1中的`UIImageView+AFNetworking`文档。

`NSURLCache`
---
`AFNetworking`使用了`NSURLConnection`，它利用了iOS原生的缓存机制，并且`NSURLCache`缓存了服务器返回的`NSURLRespone`对象。`NSURLCache` 的`shareCache`方法是默认开启的，你可以利用它来获取每一个`NSURLConnection`对象的URL内容。
   让人很难过的是，它的默认配置是缓存到内存而且并没有写入到磁盘。为了tame the beast(驯服野兽？不太懂)，增加可持续性，你可以在`AppDelegate`中简单地声明一个共享的`NSURLCache`对象，像这样：
```objective-c
NSURLCache *sharedCache = [[NSURLCache alloc] initWithMemoryCapacity:2 * 1024 * 1024
                                              diskCapacity:100 * 1024 * 1024
                                              diskPath:nil];
[NSURLCache setSharedURLCache:sharedCache];
```

设置`NSURLRequest`对象的缓存策略
---
 `NSURLCache` 将对每一个`NSURLRequest`对象遵守缓存策略(`NSURLRequestCachePolicy`)，策略如下所示：

```objective-c
- NSURLRequestUseProtocolCachePolicy                默认的缓存策略，对特定的URL请求使用网络协议中实现的缓存逻辑
- NSURLRequestReloadIgnoringLocalCacheData          忽略本地缓存，重新请请求
- NSURLRequestReloadIgnoringLocalAndRemoteCacheData 忽略本地和远程缓存，重新请求
- NSURLRequestReturnCacheDataElseLoad               有缓存则从中加载，如果没有则去请求
- NSURLRequestReturnCacheDataDontLoad               无网络状态下不去请求，一直加载本地缓存数据无论其是否存在
- NSURLRequestReloadRevalidatingCacheData           默从原始地址确认缓存数据的合法性之后，缓存数据才可使用，否则请求原始地址
```
用`NSURLCache`缓存数据到磁盘
---
###Cache-Control HTTP Header
  `Cache-Controlheader`或`Expires header`存在于服务器返回的`HTTP response header`中，来用于客户端的缓存工作(前者优先级要高于后者)，这里面有很多地方需要注意,`Cache-Control`可以拥有被定义为类似`max-age`的参数（在更新响应之前要缓存多长时间), public/private 访问或者是`non－cache`(不缓存响应数据),[这里](http://blog.csdn.net/zhaokaiqiang1992)对`HTTP cache headers`进行了很好的介绍
  
###继承并控制`NSURLCache`
  如果你想跳过`Cache-Control`,并且想要自己来制定规则读写一个带有`NSURLResponse`对象的`NSURLCache`,你可以继承`NSURLCache`。下面有个例子,使用 `CACHE_EXPIRES`来判断在获取源数据之前对缓存数据保留多长时间.(感谢 Mattt Thompson的[回复](https://twitter.com/mattt/status/444538735838134272))
 ```objective-c
 @interface CustomURLCache : NSURLCache

static NSString * const CustomURLCacheExpirationKey = @"CustomURLCacheExpiration";
static NSTimeInterval const CustomURLCacheExpirationInterval = 600;

@implementation CustomURLCache

+ (instancetype)standardURLCache {
    static CustomURLCache *_standardURLCache = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _standardURLCache = [[CustomURLCache alloc]
                                 initWithMemoryCapacity:(2 * 1024 * 1024)
                                 diskCapacity:(100 * 1024 * 1024)
                                 diskPath:nil];
    }

    return _standardURLCache;
}

#pragma mark - NSURLCache

- (NSCachedURLResponse *)cachedResponseForRequest:(NSURLRequest *)request {
    NSCachedURLResponse *cachedResponse = [super cachedResponseForRequest:request];

    if (cachedResponse) {
        NSDate* cacheDate = cachedResponse.userInfo[CustomURLCacheExpirationKey];
        NSDate* cacheExpirationDate = [cacheDate dateByAddingTimeInterval:CustomURLCacheExpirationInterval];
        if ([cacheExpirationDate compare:[NSDate date]] == NSOrderedAscending) {
            [self removeCachedResponseForRequest:request];
            return nil;
        }
    }
}

    return cachedResponse;
}

- (void)storeCachedResponse:(NSCachedURLResponse *)cachedResponse
                 forRequest:(NSURLRequest *)request
{
    NSMutableDictionary *userInfo = [NSMutableDictionary dictionaryWithDictionary:cachedResponse.userInfo];
    userInfo[CustomURLCacheExpirationKey] = [NSDate date];

    NSCachedURLResponse *modifiedCachedResponse = [[NSCachedURLResponse alloc] initWithResponse:cachedResponse.response data:cachedResponse.data userInfo:userInfo storagePolicy:cachedResponse.storagePolicy];

    [super storeCachedResponse:modifiedCachedResponse forRequest:request];
}
@end
```
现在你有了属于自己的`NSURLCache`的子类，不要忘了在`AppDelegate`中初始化并且使用它。

###在缓存之前重写`NSURLResponse`
  `-connection:willCacheResponse` 代理方法是在被缓存之前用于截断和编辑由`NSURLConnection`创建的`NSURLCacheResponse`的地方。
对`NSURLCacheResponse`进行处理并返回一个可变的拷贝对象(代码来自[NSHipster blog](http://nshipster.com/nsurlcache/))
```objective-c
- (NSCachedURLResponse *)connection:(NSURLConnection *)connection
                  willCacheResponse:(NSCachedURLResponse *)cachedResponse {
    NSMutableDictionary *mutableUserInfo = [[cachedResponse userInfo] mutableCopy];
    NSMutableData *mutableData = [[cachedResponse data] mutableCopy];
    NSURLCacheStoragePolicy storagePolicy = NSURLCacheStorageAllowedInMemoryOnly;

    // ...

    return [[NSCachedURLResponse alloc] initWithResponse:[cachedResponse response]
                                                    data:mutableData
                                                userInfo:mutableUserInfo
                                           storagePolicy:storagePolicy];
}

// If you do not wish to cache the NSURLCachedResponse, just return nil from the delegate function:

- (NSCachedURLResponse *)connection:(NSURLConnection *)connection
                  willCacheResponse:(NSCachedURLResponse *)cachedResponse {
    return nil;
}
```
###禁用`NSURLCache`
   不想使用`NSURLCache`?不为所动？好吧，你可以禁用`NSURLCache`，只需要将内存和磁盘空间设置为0就行了.
```objective-c
NSURLCache *sharedCache = [[NSURLCache alloc] initWithMemoryCapacity:0
                                              diskCapacity:0
                                              diskPath:nil];
[NSURLCache setSharedURLCache:sharedCache];
```
总结
---
  我写这篇博客的目的是为iOS社区贡献绵薄之力，并总结了我是如何来处理关于`AFNetworking`缓存问题的。我们有个内部App在加载了大量图片后，出现了内存和性能问题，而我的主要职责是诊断这个App的缓存行为，在研究过程中，我在网上搜索了很多资料并且做了很多调试，在我总结之后就写到了这篇博客中,我希望这篇文章可以为开发者使用`AFNetworking`时提供一些帮助，真心希望对你们有用！


