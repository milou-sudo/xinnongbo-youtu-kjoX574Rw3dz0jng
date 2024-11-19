
在开发者便利度角度，我们很轻松地使用HttpClient对象发出HTTP请求，只需要关注应用层协议的BaseAddr、Url、ReqHeader、timeout。


实际在HttpClient在源码级别是由 HttpMessageHandler实例发出的请求。


![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/e429121c92494bcb955f60ee4158ff25~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5o6Y6YeR56CB55Sy5ZOl:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDQ4MjU2NDc2NzI3NjYyIn0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1732526810&x-orig-sign=ZFG0E7VNazzJdgQfcDxlbty0bI8%3D)


### 1\. 早期.NET HttpClient遇到的Socket滥用/DNS解析问题


[早期.NET的HttpClient使用HttpClientHandler](https://github.com "早期.NET的HttpClient使用HttpClientHandler"), 该handler具备完整的async、proxy、dns、Connection pool 请求一条龙能力。


![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/f2f4169690ba473d8c2b6a84d105170f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5o6Y6YeR56CB55Sy5ZOl:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDQ4MjU2NDc2NzI3NjYyIn0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1732526810&x-orig-sign=BKo9HgrCzQ2DVGhznkFDdriI1aE%3D)


底层Handler又会构建tcp连接池 ，开发者不注意使用场景和底层原理容易造成Socket滥用、主机端口耗尽（参考资料是：tcp4次挥手，主动端开方不会立即释放端口，存在2min的time\_wait状态）。


一般实践会采用单例模式，重用HttpClient对象（也即重用HttpClientHandler）， 但此时又会遇到DNS解析问题的尴尬（HttpClient仅在创建时为连接解析DNS，不跟踪DNS服务器指令的TTL）。




---


意识到重用httpClient带上的dns解析副作用之后， .NET团队和.ASP.NETCore团队分别给出了技术路线来尝试解决这个问题，


前者在.NETCore 2\.1 引入了具备对连接池中连接做生命周期管理能力的 SocketsHttpHandler；


后者基于ASP.NETCore框架随处可见的DI能力，实现了针对HttpClientHandler实例的缓存工厂。




---


### 2\. .NET Core2\.1\+ HttpClient 改造HttpClientHandler证明自己


新版本的思路是哪里有问题， 我就改造哪里。


.NET Core 2\.1改造了HttpClient原始的HttpClientHandler源码, 让其`underlyingHandler=SocketsHttpHandler`，也就是说在.NETCore2\.1起HttpClient的核心Handler实质就是[SocketsHttpHandler](https://github.com "SocketsHttpHandler")， HttpClientHandler只是一个套壳。


![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/0ed8e2f46b974712a3bd9971d3990156~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5o6Y6YeR56CB55Sy5ZOl:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDQ4MjU2NDc2NzI3NjYyIn0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1732526810&x-orig-sign=wk9XbqIUXJULBTcHCQ90SryQFe4%3D)


看上面的UML图，被改造后的套壳HttpClientHandler内置了一个默认的`SocketsHttpHandler`来完成一条龙HTTP服务 （Dispose工作也全权交给了SocketsHttpHandler）， 当然开发者也可以在构建HttpClient实例时指定handler。


SocketsHttpHandler中与连接生命周期相关的三个关键属性：



```


|  | var handler = new SocketsHttpHandler |
| --- | --- |
|  | { |
|  | PooledConnectionLifetime = TimeSpan.FromMinutes(15), // 限制连接的生命周期,默认无限  Recreate every 15 minutes， 这个配置可用于缓解DNS解析问题 |
|  | PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),   // 空闲连接在连接池中的存活时间， <=NET5默认2min, >NET6 1min |
|  | MaxConnectionsPerServer =  100,                  // 定义到每个目标服务节点能建立的最大连接数  未设置 = int.MaxValue |
|  |  |
|  | }; |
|  | var sharedClient = new HttpClient(handler); |


```

都聊到此了，在打算重用HttpClient实例时，插入SocketsHttpHandler并调整`PooledConnectionLifetime`，可缓解DNS解析问题。


### 3\. ASP.NETCore IHttpClientFactory缓存工厂 曲线救国


IHttpClientFactory 充分体现了“计算机领域的任何问题都可以通过增加一个间接的中间层来解决” 这一方法论。


为解决重用HttpClient引起的DNS解析副作用，IHttpClientFactory对实际使用的核心HttpClienthandler开启了缓存工厂模式，在外侧尝试跟踪并控制Handler的存活周期。


![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/b5a27c60840a424db9719ab11ef12cc6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5o6Y6YeR56CB55Sy5ZOl:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDQ4MjU2NDc2NzI3NjYyIn0%3D&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1732526810&x-orig-sign=QZxsR1Z%2ByKUfiyePi0grHMUvk%2FE%3D)


① 通过IHttpClientFactory注入的命名的/类型化的HttpClient实例，底层核心的Handler来自缓存字典；


② 缓存字典中的缓存项默认2min，意味着2min时间内产生的命名HttpClient实例都是引用同一个核心HttpMessageHandler实例（`LifeTimeTrackingHttpMessageHandler`）；



```


|  | public HttpClient CreateClient(string name) |
| --- | --- |
|  | { |
|  | ThrowHelper.ThrowIfNull(name); |
|  |  |
|  | HttpMessageHandler handler = CreateHandler(name); |
|  | var client = new HttpClient(handler, disposeHandler: false); |
|  |  |
|  | HttpClientFactoryOptions options = _optionsMonitor.Get(name); |
|  | for (int i = 0; i < options.HttpClientActions.Count; i++) |
|  | { |
|  | options.HttpClientActions[i](client "i"); |
|  | } |
|  |  |
|  | return client; |
|  | } |
|  |  |
|  | public HttpMessageHandler CreateHandler(string name) |
|  | { |
|  | ThrowHelper.ThrowIfNull(name); |
|  |  |
|  | ActiveHandlerTrackingEntry entry = _activeHandlers.GetOrAdd(name, _entryFactory).Value; // 工厂模式，惰性取值 |
|  |  |
|  | StartHandlerEntryTimer(entry);      // 跟踪缓存项的过期时间 |
|  |  |
|  | return entry.Handler; |
|  |  |
|  | } |


```

缓存是用线程安全的字典`ConcurrentDictionary`以惰性生成的方式实现：



```


|  | _activeHandlers  =new ConcurrentDictionary<string, Lazy<ActiveHandlerTrackingEntry>>(StringComparer.Ordinal); |
| --- | --- |
|  |  |
|  | _entryFactory = (name) => { |
|  | return new Lazy<ActiveHandlerTrackingEntry>(() => |
|  | { |
|  | return CreateHandlerEntry(name); |
|  | }, LazyThreadSafetyMode.ExecutionAndPublication); |
|  | }; |


```

缓存的是[LifeTimeTrackingHttpMessageHandler](https://github.com "LifeTimeTrackingHttpMessageHandler"):[悠兔机场](https://xinnongbo.com)对象，这是一个托管资源。


③ 每个活跃的核心handler上外挂了存活时间， 一旦到期便从活跃字典中移出， 并移动到[过期handler队列](https://github.com "过期handler队列")；



```


|  | internal sealed class ExpiredHandlerTrackingEntry |
| --- | --- |
|  | { |
|  | private readonly WeakReference _livenessTracker; |
|  |  |
|  | // IMPORTANT: don't cache a reference to `other` or `other.Handler` here. |
|  | // We need to allow it to be GC'ed. |
|  | public ExpiredHandlerTrackingEntry(ActiveHandlerTrackingEntry other) |
|  | { |
|  | Name = other.Name; |
|  | Scope = other.Scope; |
|  |  |
|  | _livenessTracker = new WeakReference(other.Handler);    // 跟踪LifeTimeTrackingHttpMessageHandler 托管资源 |
|  | InnerHandler = other.Handler.InnerHandler!;        // InnerHandler 是托管资源底层引用的非托管资源 |
|  | } |
|  |  |
|  | public bool CanDispose => !_livenessTracker.IsAlive; |
|  |  |
|  | public HttpMessageHandler InnerHandler { get; } |
|  |  |
|  | public string Name { get; } |
|  |  |
|  | public IServiceScope? Scope { get; } |
|  | } |


```

托管资源LifeTimeTrackingHttpMessageHandler 不接受dispose(httpclient)的指引，而是由gc跟踪再无HttpClient引用而被清理。


Q： 此时就出现了一个问题， 托管资源已经被gc清理， 那依赖的底层非托管资源什么时候清理的？ 这个不清理可是有大问题。


A ：这里使用了一个C\#高级的用法：弱引用[WeakReference](https://github.com "WeakReference")：能够在不影响gc的情况下，获得对象的“弱引用”， 并据此知道该实例是不是已经被gc清理了；本文是`弱引用_livenessTracker`跟踪了托管资源LifeTimeTrackingHttpMessageHandler， 该托管资源被gc清理后\_livenessTracker会得到感知。


btw，关于弱引用，我会开一新篇章来讲述。


④ 最后由程序内置的定时清理程序来清理底层非托管资源。



```


|  | if (entry.CanDispose)       //跟踪到托管对象已经被gc |
| --- | --- |
|  | { |
|  | try |
|  | { |
|  | entry.InnerHandler.Dispose(); |
|  | entry.Scope?.Dispose(); |
|  | disposedCount++; |
|  | } |
|  | catch (Exception ex) |
|  | { |
|  | Log.CleanupItemFailed(_logger, entry.Name, ex); |
|  | } |
|  | } |
|  | //注意：InnerHandler并不是托管对象LifeTimeTrackingHttpMessageHandler |


```

具体是通过弱引用`entry.CanDispose`得知引用被gc之后，再去清理底层的非托管资源:`InnerHandler.Dispose()`。


在使用层面， IHttpClientFactory并非直接管控连接池连接，而是在外层对Handler做存活缓存，故工厂对外只提供了SetHandlerLifetime(TimeSpan.FromMinutes(5\)) 这一个配置函数。


## 总结


本文从早期的HttpClient带来的尴尬（重用HttpClient带来的DNS解析问题）， 扩展到.NET团队尝试解决该问题的两个思路。


.NET Core 2\.1的思路是增强HttpClient库底层的连接池能力，提供了SocketsHttpHandler来控制连接的生命周期，


IHttpClientFactory的思路是绕过HttpClient本身的问题，在上层用存活缓存的思路来控制HttpClientHandler实例， 充分贯彻了“计算机领域的任何问题都可以通过增加一个间接的中间层来解决”的思想。


