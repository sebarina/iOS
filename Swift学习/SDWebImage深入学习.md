# SDWebImage源码分析

SDWebImage是很有名的图片下载和缓存的第三方开源框架，想必很多App都使用了它吧，最近决定静下心来好好学习一下这个开源框架。

## 简介
SDWebImage是个什么样的东西呢？下面是来自官方的介绍：
> This library provides a category for UIImageView with support for remote images coming from the web.

翻译过来就是： ***一个异步下载且支持缓存的UIImageView的分类。***SDWebImage不仅支持普通的图片格式，还支持GIF动画和WebP格式的图片。
我们在程序中最常使用的SDWebImage的方法就是：
<pre><code>
images[i].sd_setImageWithURL(NSURL(string: urls[i]), placeholderImage: UIImage(named: "Placeholder"))</code></pre>

**另外，SDWebImage不仅支持UIImageView控件的图片加载，还支持UIButton的图片加载。**
<pre>userImage?.sd_setImageWithURL(NSURL(string: info.avatar), forState: UIControlState.Normal, placeholderImage: UIImage(named: "User_Placeholder"))</pre>
## SDWebImage架构
架构图如下所示：

![1](http://7xos21.com1.z0.glb.clouddn.com/SDWebImage.png)

最上层为UIImageView和UIButton的分类，扩展图片加载的上层接口
SDWebImageManager是整个图片加载过程中的核心，是管理者，它管理下载操作和缓存操作。接下来我们看看一个图片加载的过程吧。

### 实现原理
首先，我们来看看UIImageView加载图片的一个流程：

![2](http://7xos21.com1.z0.glb.clouddn.com/SDWebImage_set.png)

在UIImageView加载图片的过程中，主要包括缓存和下载2大模块，这2个模块是通过SDWebImageManager来管理的，接下来我会对SDWebImage中的几大模块做个分析。

## SDWebImage几大模块分析
### SDWebImageManager
在 SDWebImageManager.h 中你可以看到关于 SDWebImageManager 的描述:

>The SDWebImageManager is the class behind the UIImageView+WebCache category and likes. It ties the asynchronous downloader (SDWebImageDownloader) with the image cache store (SDImageCache). You can use this class directly to benefit from web image downloading with caching in another context than a UIView.

这个类就是隐藏在 UIImageView+WebCache 背后, 用于处理异步下载和图片缓存的类, 当然你也可以直接使用 SDWebImageManager 的上述方法 downloadImageWithURL:options:progress:completed: 来直接下载图片.

可以看到, 这个类的主要作用就是为 UIImageView+WebCache 和 SDWebImageDownloader, SDImageCache 之间构建一个桥梁, 使它们能够更好的协同工作, 我们在这里分析这个核心方法的源代码, 它是如何协调异步下载和图片缓存的.
>- 首先处理传入的url字符串是否为有效的请求链接
>- 然后通过SDWebImageCombinedOperation 对象来管理Cache和Downloader的工作的

### SDWebImageCombinedOperation
我们首先来看看这个类的定义吧：
<pre>@interface SDWebImageCombinedOperation : NSObject <SDWebImageOperation></pre>
这里仅仅是将这个 SDWebImageOperation 类包装成一个看着像 NSOperation 其实并不是 NSOperation 的类, 而这个类唯一与 NSOperation 的相同之处就是它们都可以响应 cancel 方法. 这个cancel方法就是用来取消掉正在运行的缓存操作和下载操作。调用这个类的存在实际是为了使代码更加的简洁。
<pre>- (void)cancel {
    self.cancelled = YES;
    if (self.cacheOperation) {
        [self.cacheOperation cancel];
        self.cacheOperation = nil;
    }
    if (self.cancelBlock) {
        self.cancelBlock();
        _cancelBlock = nil;
    }
}</pre>

在cancel方法里主要做2件事，一个是取消掉cacheOperation（NSOperation的子类），另一个是调用cancelBlock方法（其实这个方法里就是取消掉了SDWebImageDownloaderOperation）。

首先检查是否存在缓存图片：
<pre>
operation.cacheOperation = [self.imageCache queryDiskCacheForKey:key done:
^(UIImage *image, SDImageCacheType cacheType) {
    ...
}];
</pre>
这里调用 SDImageCache 的实例方法 queryDiskCacheForKey:done: 来尝试在缓存中获取图片的数据. 而这个方法返回的就是货真价实的 NSOperation.

如果我们在缓存中查找到了对应的图片, 那么我们直接调用 completedBlock 回调块结束这一次的图片下载操作.

<pre>
    dispatch_main_sync_safe(^{
        // If image was found in the cache bug SDWebImageRefreshCached is provided, notify about the cached image
        // AND try to re-download it in order to let a chance to NSURLCache to refresh it from server.
        completedBlock(image, nil, cacheType, YES, url);
    });
</pre>

如果我们没有找到图片, 那么就会调用 SDWebImageDownloader 的实例方法:

<pre>
    id <SDWebImageOperation> subOperation = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *data, NSError *error, BOOL finished) {
        ...
    }];
</pre>

如果这个方法返回了正确的 downloadedImage, 那么我们就会在全局的缓存中存储这个图片的数据:

<pre>[self.imageCache storeImage:downloadedImage recalculateFromImage:NO imageData:data forKey:key toDisk:cacheOnDisk];</pre>

并调用 completedBlock 对 UIImageView 或者 UIButton 添加图片, 或者进行其它的操作.

<pre>completedBlock(downloadedImage, nil, SDImageCacheTypeNone, finished, url);</pre>

最后, 我们将这个 subOperation 的 cancel 操作添加到 operation.cancelBlock 中. 方便操作的取消.

<pre>
    operation.cancelBlock = ^{
        [subOperation cancel];
        @synchronized (self.runningOperations) {
            [self.runningOperations removeObject:weakOperation];
        }
    };
</pre>

### SDWebImageCache
首先来看看这个类的描述吧：

> SDImageCache maintains a memory cache and an optional disk cache.

@property (strong, nonatomic) NSCache *memCache;
@property (strong, nonatomic) NSString *diskCachePath;


它维护了一个内存缓存和一个可选的磁盘缓存, 我们先来看一下在上一阶段中没有解读的两个方法, 首先是:
<pre>- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock</pre>

在这个方法里，首先会去查询内存缓存是否含有该图片：
<pre>UIImage *image = [self imageFromMemoryCacheForKey:key];</pre>
内存缓存是通过memCache对象来管理的（NSCache对象）

如果内存缓存中没有该图片，则会查询硬盘图片：

<pre>
    dispatch_async(self.ioQueue, ^{
        if (operation.isCancelled) {
            return;
        }

        @autoreleasepool {
            UIImage *diskImage = [self diskImageForKey:key];
            if (diskImage) {
                NSUInteger cost = SDCacheCostForImage(diskImage);
                [self.memCache setObject:diskImage forKey:key cost:cost];
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                doneBlock(diskImage, SDImageCacheTypeDisk);
            });
        }
    });
</pre>
硬盘查询是一个异步操作的过程，成功获取到硬盘图片后，会将改图片对象存储到内存缓存中去。

### SDWebImageDownloader
首先来看看这个类的描述吧：
> Asynchronous downloader dedicated and optimized for image loading.

专用的并且优化的图片异步下载器.

这个类的核心功能就是下载图片, 而核心方法就是上面提到的:

<pre>
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url options:(SDWebImageDownloaderOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageDownloaderCompletedBlock)completedBlock
</pre>

这个方法实现里调用了一个很关键的方法：

 <pre>[self addProgressCallback:progressBlock andCompletedBlock:completedBlock forURL:url createCallback:^{
    ...
}];</pre>

它为这个下载的操作添加回调的块, 在下载进行时, 或者在下载结束时执行一些操作, 先来阅读一下这个方法的源代码:

<pre>
    dispatch_barrier_sync(self.barrierQueue, ^{
        BOOL first = NO;
        if (!self.URLCallbacks[url]) {
            self.URLCallbacks[url] = [NSMutableArray new];
            first = YES;
        }

        // Handle single download of simultaneous download request for the same URL
        NSMutableArray *callbacksForURL = self.URLCallbacks[url];
        NSMutableDictionary *callbacks = [NSMutableDictionary new];
        if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
        if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
        [callbacksForURL addObject:callbacks];
        self.URLCallbacks[url] = callbacksForURL;

        if (first) {
            createCallback();
        }
    });
</pre>
可以看到，只有在第一次下载这个url的图片时才会去回调createCallback，上一个方法会在这个回调中去创建下载请求NSMutableURLRequest，所有可以有效的做到了每个图片只创建一个下载请求。

<pre> NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:(options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData) timeoutInterval:timeoutInterval];</pre>

在初始化了这个 request 之后, 又初始化了一个 SDWebImageDownloaderOperation 的实例, 这个实例, 就是用于请求网络资源的操作. 它是一个 NSOperation 的子类,

<pre>
operation = [[wself.operationClass alloc] initWithRequest:request
            options:options
            progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                    SDWebImageDownloader *sself = wself;
                    if (!sself) return;
                    __block NSArray *callbacksForURL;
                    dispatch_sync(sself.barrierQueue, ^{
                        callbacksForURL = [sself.URLCallbacks[url] copy];
                    });
                    for (NSDictionary *callbacks in callbacksForURL) {
                        SDWebImageDownloaderProgressBlock callback = callbacks[kProgressCallbackKey];
                        if (callback) callback(receivedSize, expectedSize);
                    }
            }
            completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
                SDWebImageDownloader *sself = wself;
                if (!sself) return;
                __block NSArray *callbacksForURL;
                dispatch_barrier_sync(sself.barrierQueue, ^{
                    callbacksForURL = [sself.URLCallbacks[url] copy];
                    if (finished) {
                        [sself.URLCallbacks removeObjectForKey:url];
                    }
                });
                for (NSDictionary *callbacks in callbacksForURL) {
                        SDWebImageDownloaderCompletedBlock callback = callbacks[kCompletedCallbackKey];
                        if (callback) callback(image, data, error, finished);
                }
            }
            cancelled:^{
                SDWebImageDownloader *sself = wself;
                if (!sself) return;
                dispatch_barrier_async(sself.barrierQueue, ^{
                    [sself.URLCallbacks removeObjectForKey:url];
                });
            }];
</pre>

但是在初始化之后, 这个操作并不会开始(NSOperation 实例只有在调用 start 方法或者加入 NSOperationQueue 才会执行), 我们需要将这个操作加入到一个 NSOperationQueue 中.

<pre>[wself.downloadQueue addOperation:operation];</pre>

只有将它加入到这个下载队列中, 这个操作才会执行.

注意： 如果这个操作被取消掉了，该拖的下载请求也会被清除： [sself.URLCallbacks removeObjectForKey:url];

### SDWebImageDownloaderOperation
这个类就是处理 HTTP 请求, URL 连接的类, 当这个类的实例被加入队列之后, start 方法就会被调用, 而 start 方法首先就会产生一个 NSURLConnection.

<pre>@synchronized (self) {
    if (self.isCancelled) {
        self.finished = YES;
        [self reset];
        return;
    }
    self.executing = YES;
    self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];
    self.thread = [NSThread currentThread];
}</pre>

而接下来这个 connection 就会开始运行:
<pre>[self.connection start];</pre>

它会发出一个 SDWebImageDownloadStartNotification 通知:
<pre>[[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStartNotification object:self];</pre>

在 start 方法调用之后, 就是 NSURLConnectionDataDelegate 中代理方法的调用.
<pre>
- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response;
- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response;
- (void)connectionDidFinishLoading:(NSURLConnection *)aConnection;
</pre>

在这三个代理方法中的前两个会不停回调 progressBlock 来提示下载的进度.

而最后一个代理方法会在图片下载完成之后调用 completionBlock 来完成最后 UIImageView.image 的更新.

而这里调用的 progressBlock completionBlock cancelBlock 都是在之前存储在 URLCallbacks 字典中的.

## 总结

最后用一个树状图来回顾SDWebImage的工作过程：

- 查看缓存
    - 缓存命中
        - 返回图片
        - 更新 UIImageView
    - 缓存未命中
        - 异步下载图片
        - 加入缓存
        - 更新 UIImageView

### SDWebImage 如何为 UIImageView 添加图片(面试回答)
SDWebImage 中为 UIView 提供了一个分类叫做 WebCache, 这个分类中有一个最常用的接口, sd_setImageWithURL:placeholderImage:, 这个分类同时提供了很多类似的方法, 这些方法最终会调用一个同时具有 option progressBlock completionBlock 的方法, 而在这个类最终被调用的方法首先会检查是否传入了 placeholderImage 以及对应的参数, 并设置 placeholderImage.

然后会获取 SDWebImageManager 中的单例调用一个 downloadImageWithURL:... 的方法来获取图片, 而这个 manager 获取图片的过程有大体上分为两部分, 它首先会在 SDWebImageCache 中寻找图片是否有对应的缓存, 它会以 url 作为数据的索引先在内存中寻找是否有对应的缓存, 如果缓存未命中就会在磁盘中利用 MD5 处理过的 key 来继续查询对应的数据, 如果找到了, 就会把磁盘中的缓存备份到内存中.

然而, 假设我们在内存和磁盘缓存中都没有命中, 那么 manager 就会调用它持有的一个 SDWebImageDownloader 对象的方法 downloadImageWithURL:... 来下载图片, 这个方法会在执行的过程中调用另一个方法 addProgressCallback:andCompletedBlock:fotURL:createCallback: 来存储下载过程中和下载完成的回调, 当回调块是第一次添加的时候, 方法会实例化一个 NSMutableURLRequest 和 SDWebImageDownloaderOperation, 并将后者加入 downloader 持有的下载队列开始图片的异步下载.

而在图片下载完成之后, 就会在主线程设置 image 属性, 完成整个图像的异步下载和配置.

