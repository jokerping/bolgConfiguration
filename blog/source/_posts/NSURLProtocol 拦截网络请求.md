---
title: NSURLProtocol 拦截网络请求
categories: iOS
tags: 基础知识
date: 2016-09-01 17:38:10

---
# 写在前面的话
这篇文章是看来[敬哥](http://awhisper.github.io/)的[黑盒测试库](https://github.com/Awhisper/VKDevTool)想到的，这个测试库有个显示网络请求的模块，但是在自己项目里怎么也不行，上网查了各种资料总结了这篇文章。


# NSURLProtocol

NSURLProtocol可以让我们拦截程序中的一切网络请求，主要进行如下拦截处理:
- 自定义请求 和 响应
- 过滤掉某些请求不让其发起、以及修改
- 提供 自定义的全局缓存 逻辑
- 重定向 网络请求
- 提供 HTTP Mocking (方便前期测试)

<!--more-->

# 如何拦截网络请求?

## 编写NSURLProtocol子类

### 重写方法一 请求的过滤，筛选出需要进行处理的请求
```objectivec
+ (BOOL)canInitWithRequest:(NSURLRequest *)request
{
#ifdef VKDevMode
   
    //只处理http和https请求
    NSString *scheme = [[request URL] scheme];
    if ( ([scheme caseInsensitiveCompare:@"http"] == NSOrderedSame ||
          [scheme caseInsensitiveCompare:@"https"] == NSOrderedSame))
    {
            //看看是否已经处理过了，防止无限循环
         if ([NSURLProtocol propertyForKey:VKURLProtocolHandledKey inRequest:request])
           {
               return NO;
           }
        
        return YES;
    }
#endif
    return NO;
}
```

这个方法指明你是否要处理request，如果不打算处理的话返回NO就行，系统会使用默认的URL加载系统处理request，如果返回YES,URL Loading System会将request的处理操作交给你的urlProtocol。上面的代码我们只处理了http和https请求(你也可以拦截ftp等请求),通常我们的接口中都会包含特定的字符串，将它设置为你项目中对应的接口字段即可。还有一点需要注意，我们不能总是返回YES，这样会导致无限循环，(因为一个请求在被拦截处理过程中，也会发起一个请求) 我们可以通过+ (void)setProperty:(id)value forKey:(NSString *)key inRequest:(NSMutableURLRequest *)request方法来标记request已经被处理过，如果通过+ (nullable id)propertyForKey:(NSString *)key inRequest:(NSURLRequest *)request方法获取到已经处理过该request，则返回NO。

### 重写方法二、请求重定向、修改请求头
```objectivec
+ (NSURLRequest *) canonicalRequestForRequest:(NSURLRequest *)request {
    return request;
}
```
### 重写方法三、让被拦截的请求执行
```objectivec
-(void)startLoading{
    //打标签，防止无限循环
    
    NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
    
    [NSURLProtocol setProperty:@YES forKey:VKURLProtocolHandledKey inRequest:mutableReqeust];
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
    self.connection = [NSURLConnection connectionWithRequest:mutableReqeust delegate:self];
#pragma clang diagnostic pop
}
```

### 重写方法四、取消执行请求
```objectivec
-(void)stopLoading{
    [self.connection cancel];
}
```

### 重写方法五、可用来使用缓存数据结束此次网络请求
如果不考虑自定义缓存逻辑，可以不重写此方法
```objectivec
+(BOOL)requestIsCacheEquivalent:(NSURLRequest *)a toRequest:(NSURLRequest *)b
{
    return [super requestIsCacheEquivalent:a toRequest:b];
}
```


### demo1 自定义NSURLProtol拦截请求后进行处理，然后使用NSURLSession继续执行网络请求
```objectivec
#import <Foundation/Foundation.h>

@interface MySessionURLProtocol : NSURLProtocol

@end
```

```objectivec
#import "MySessionURLProtocol.h"

//防止循环拦截的标识
#define protocolKey @"SessionProtocolKey"

@interface MySessionURLProtocol ()<NSURLSessionDataDelegate>

@property (nonatomic, strong) NSURLSession * session;

@end
```

```objectivec
@implementation MySessionURLProtocol

/**
 *  是否拦截处理指定的请求
 *
 *  @param request 指定的请求
 *
 *  @return 返回YES表示要拦截处理，返回NO表示不拦截处理
 */
+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    
    /*
     防止无限循环，因为一个请求在被拦截处理过程中，也会发起一个请求，这样又会走到这里，如果不进行处理，就会造成无限循环
     如果已经拦截处理了，就不再进行拦截了，直接让其发起网络请求.
     */
    if ([NSURLProtocol propertyForKey:protocolKey inRequest:request]) {
        return NO;
    }
    
    // URL的全路径字符串
    NSString * url = request.URL.absoluteString;
    
  //只处理http和https请求
  NSString *scheme = [[request URL] scheme];
  if ( ([scheme caseInsensitiveCompare:@"http"] == NSOrderedSame ||
     [scheme caseInsensitiveCompare:@"https"] == NSOrderedSame))
    {
      return YES;
    }
    
    return NO;
}

/**
 *  如果需要对请求进行重定向，添加指定头部等操作，可以在该方法中进行
 *
 *  @param request 原请求
 *
 *  @return 修改后的请求
 */
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
    
    // 修改了请求的头部信息
    NSMutableURLRequest * mutableReq = [request mutableCopy];
    
    NSMutableDictionary * headers = [mutableReq.allHTTPHeaderFields mutableCopy];
    [headers setObject:@"BBBB" forKey:@"Key2"];
    
    mutableReq.allHTTPHeaderFields = headers;
    
    //返回修改请求头后的Request
    return [mutableReq copy];
    
    //还可以修改Request.url进行重定向
    //...
}

/**
 *  开始加载，在该方法中，加载一个请求
 */
- (void)startLoading {

  
    NSMutableURLRequest * request = [self.request mutableCopy];
    
    // 标记当前传入的Request已经被拦截处理过，
    //防止在最开始又继续拦截处理
    [NSURLProtocol setProperty:@(YES) forKey:protocolKey inRequest:request];
    
    //如下使用NSURLSession让拦截的请求进行请求网络
    NSURLSessionConfiguration * config = [NSURLSessionConfiguration defaultSessionConfiguration];
    self.session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:[NSOperationQueue mainQueue]];
    NSURLSessionDataTask * task = [self.session dataTaskWithRequest:request];
    [task resume];
}

/**
 *  取消请求
 */
- (void)stopLoading {
    [self.session invalidateAndCancel];
    self.session = nil;
}

#pragma mark - NSURLSessionDataDelegate

-(void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error
{
    if (error) {
        [self.client URLProtocol:self didFailWithError:error];
    } else {
        [self.client URLProtocolDidFinishLoading:self];
    }
}

-(void)URLSession:(NSURLSession *)session
         dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler
{
    [self.client URLProtocol:self
          didReceiveResponse:response
          cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    
    completionHandler(NSURLSessionResponseAllow);
}

-(void)URLSession:(NSURLSession *)session
         dataTask:(NSURLSessionDataTask *)dataTask
   didReceiveData:(NSData *)data
{
    [self.client URLProtocol:self didLoadData:data];
}

- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler
{
    completionHandler(proposedResponse);
}

@end
```

### demo2、简单的重定向处理
```objectivec
@implementation MySessionURLProtocol

...

+ (NSURLRequest *) canonicalRequestForRequest:(NSURLRequest *)request {
    NSMutableURLRequest *mutableReqeust = [request mutableCopy];
    mutableReqeust = [self redirectHostInRequset:mutableReqeust];
    return mutableReqeust;
}

...

+(NSMutableURLRequest*)redirectHostInRequset:(NSMutableURLRequest*)request
{
    if ([request.URL host].length == 0) {
        return request;
    }
    
    //原始url全路径
    NSString *originUrlString = [request.URL absoluteString];
    
    //原始url的host
    NSString *originHostString = [request.URL host];
    
    //获取host的range
    NSRange hostRange = [originUrlString rangeOfString:originHostString];
    
    if (hostRange.location == NSNotFound) {
        return request;
    }
    
    //重定向到bing搜索主页
    NSString *ip = @"cn.bing.com";
    
    // 替换开始的host
    NSString *urlString = [originUrlString stringByReplacingCharactersInRange:hostRange withString:ip];
    
    //得到重定向后的url
    NSURL *url = [NSURL URLWithString:urlString];
    
    //将新的url赋值给Request
    request.URL = url;

    //返回修改host重定向的request
    return request;
}

...

@end
```

### demo3自定义NSURLProtol拦截请求后进行处理，然后使用NSURLConnection继续执行网络请求

```objectivec
#import "MyConnectionURLProtocol.h"

#define protocolKey @"ConnectionProtocolKey"

@interface MyConnectionURLProtocol () <NSURLConnectionDataDelegate>

@property (nonatomic, strong) NSURLConnection * connection;

@end

@implementation MyConnectionURLProtocol

/**
 *  是否拦截处理指定的请求
 *
 *  @param request 指定的请求
 *
 *  @return 返回YES表示要拦截处理，返回NO表示不拦截处理
 */
+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    
    /* 
        防止无限循环，因为一个请求在被拦截处理过程中，也会发起一个请求，这样又会走到这里，如果不进行处理，就会造成无限循环
     */
    if ([NSURLProtocol propertyForKey:protocolKey inRequest:request]) {
        return NO;
    }
    
    NSString * url = request.URL.absoluteString;
    
    // 如果url已http或https开头，则进行拦截处理，否则不处理
    if ([url hasPrefix:@"http"] || [url hasPrefix:@"https"]) {
        return YES;
    }
    return NO;
}

/**
 *  如果需要对请求进行重定向，添加指定头部等操作，可以在该方法中进行
 *
 *  @param request 原请求
 *
 *  @return 修改后的请求
 */
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
    
    // 修改了请求的头部信息
    NSMutableURLRequest * mutableReq = [request mutableCopy];
    NSMutableDictionary * headers = [mutableReq.allHTTPHeaderFields mutableCopy];
    [headers setObject:@"AAAA" forKey:@"Key1"];
    mutableReq.allHTTPHeaderFields = headers;
    NSLog(@"connection reset header");
    return [mutableReq copy];
}

/**
 *  开始加载，在该方法中，加载一个请求
 */
- (void)startLoading {
    NSMutableURLRequest * request = [self.request mutableCopy];
    // 表示该请求已经被处理，防止无限循环
    [NSURLProtocol setProperty:@(YES) forKey:protocolKey inRequest:request];
    
    self.connection = [NSURLConnection connectionWithRequest:request delegate:self];
}

/**
 *  取消请求
 */
- (void)stopLoading {
    [self.connection cancel];
}

#pragma mark - NSURLConnectionDelegate

- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    [self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
}

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    [self.client URLProtocol:self didLoadData:data];
}

- (void)connectionDidFinishLoading:(NSURLConnection *)connection {
    [self.client URLProtocolDidFinishLoading:self];
}

- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error {
    [self.client URLProtocol:self didFailWithError:error];
}

@end
```

## 注册NSURLProtocol子类

当app准备发起网络请求时会遍历所有已注册的NSURLProtocol，询问它们能否处理当前请求，所以你需要尽早注册这个Protocol。在遍历子类的过程是反向遍历的，最后注册的子类会先被遍历，这样自定义的子类将能提前截获请求，当一个子类截获请求后，将终止遍历。

```objectivec
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions 
{
    [NSURLProtocol registerClass:[MyURLProtocol class]];
    return YES;
}
```

## 发送网络请求

NSURLProtocol 可以拦截 NSURLConnection、NSURLSession、UIwebview 的请求，但是不能拦截WKwebview 和CFNetwork的请求

```objectivec
   NSError *error =[NSError errorWithDomain:@"woshige erro" code:1 userInfo:nil];
    
    NSURLSession *session = [NSURLSession sharedSession];
    NSURL *url = [NSURL URLWithString:@"http://appwk.baidu.com/naapi/iap/userbankinfo?uid=bd_0&from=ios_appstore&app_ua=Simulator&ua=bd_1334_750_Simulator_3.4.9_9.2&fr=2&pid=1&bid=2&Bdi_bear=wifi&app_ver=3.4.9&sys_ver=9.2&cuid=50c78ca9f3c39a34c963de578bef1d8c7aecc087&sessid=1471498926&screen=750_1334&opid=wk_na&ydvendor=84942C9A-E479-4856-945A-D55FBCDF4D57"];
    // 通过URL初始化task,在block内部可以直接对返回的数据进行处理
    NSURLSessionTask *task = [session dataTaskWithURL:url
                                    completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
                                        if (data) {
                                            NSLog(@"%@", [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:nil]);
                                        }
                                    }];
    [task resume];
}
```

# AFNetworking 3.0 以后版本出现的问题

NSURLProtocol 确实可以拦截NSURLSession的请求，但是敬哥DEMO中的NSURLSession 是这样创建的
      NSURLSession *session = [NSURLSession sharedSession];
这样创建的session会使用全局的cooike和其他设置，所以默认是会被hook掉。但是AFN并不是这样创建的session。所以默认是不起作用的。
解决办法一是
```objectivec
    
    NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSArray *protocolArray = @[[VKURLProtocol class]];
    configuration.protocolClasses = protocolArray;

    
    AFHTTPSessionManager *manager = [[AFHTTPSessionManager alloc] initWithSessionConfiguration:configuration];

```
像这样在每个请求之前添加`protocolClasses`或者子类和AFN然后进行设置。

简单的办法是。修改AFN源文件`AFURLSessionManager`
```objectivec
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
....

    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
        NSArray *protocolArray = @[NSClassFromString(@"VKURLProtocol")];
        configuration.protocolClasses = protocolArray;
    }
 ....
```
在这个if中对`NSURLSessionConfiguration`进行设置。或者采用method swizzing的方法进行。