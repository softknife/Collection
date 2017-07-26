# Routing

Routing 决定了哪个handler会接收到具体的请求.  一个handler是一个例程,函数或者方法, 他能决定接收和响应一个什么具体类型的请求或者信号. 请求的路由基于两点信息: the HTTP request method, and the request path. 一个路由完整意义上是指 an HTTP method, path, and handler 三者的组合. 路由的创建和添加是在服务器启动监听端口之前完成的. 例如:

```objective-c

var routes = Routes()
routes.add(method: .get, uri: "/path/one", handler: { request, response in
    response.setBody(string: "Handler was called")
    response.completed()
})
server.addRoutes(routes)
```

 

一旦服务器收到一个请求, 他将会把这个请求传递给注册的request filters.  这些filters将会有机会通过某种方式去修改请求对象, 这将会影响整个路由进程, 例如修改request path. 服务器将会搜索匹配当前request method和path的路由. 如果成功找到该路由, 服务器将会传递request和response给对应的handler. 如果未找到对应的路由, 服务器将会返回"404"或者"Not Found"错误给客户端.



## Creating Routes

Routing API 是 the [PerfectHTTP](https://github.com/PerfectlySoft/Perfect-HTTP) project 的一部分. 在和routing系统交互之前,你需要先`import PerfectHTTP`.



在添加任何路由之前,你需要合适的handler function.  他接收一个request和一个response对象, 并且需要生产内容,放到response中, 他还需要在完成当前任务时做出声明. 一个request handler的简写方式如下:

```objective-c

/// Function which receives request and response objects and generates content.
public typealias RequestHandler = (HTTPRequest, HTTPResponse) -> ()
```



Requests在声明自己已经得出结论之前,我们始终认为他是激活状态. 当我们需要声明一个请求结束时,我们可以调用`HTTPResponse.completed()`函数. Request Handling 完全是异步的,所以handler函数可以返回, 分拆到一个新线程中去,  或者执行其他一系列的异步活动. 除非`HTTPResponse.completed()`被调用, 否则request一直被认为是激活状态.  一旦response被标记为完成,输出的headers和body数据将会发送给客户端.



在路由对象添加到服务器之前,所有的路由事件需要添加到路由对象中. 路由对象创建后, 一个或者多个路由事件可以使用`add`函数添加到路由对象中.  路由体用如下函数:

```objective-c
public struct Routes {
    /// Initialize with no baseUri.
    public init()
    // Initialize with a baseUri.
    public init(baseUri: String)
    /// Add all the routes in the Routes object to this one.
    public mutating func add(routes: Routes)
    /// Add the given method, uri and handler as a route.
    public mutating func add(method: HTTPMethod, uri: String, handler: RequestHandler)
    /// Add the given method, uris and handler as a route.
    public mutating func add(method: HTTPMethod, uris: [String], handler: RequestHandler)
    /// Add the given uri and handler as a route.
    /// This will add the route got both GET and POST methods.
    public mutating func add(uri: String, handler: RequestHandler)
    /// Add the given method, uris and handler as a route.
    /// This will add the route got both GET and POST methods.
    public mutating func add(uris: [String], handler: RequestHandler)
    /// Add one Route to this object.
    public mutating func add(_ route: Route)
}
```

路由对象可以初始化一个baseURI. 在任何一个路由事件被添加到路由对象中之前,baseURI会被添加到路由前面. 例如, 你可以初始化路由对象到指定的API版本, 比如给他指定baseURI为"/v1", 这样所有路由都会以"/v1"作为前缀. 路由对象也可以被添加到其他路由对象中, 这样每一个内部的路由就会有相同的前缀了. 下面例子中展示了如何创建两个API版本的路由集合. 第二个版本和第一个版本的路由,即使路径最后一段相同,他们的行为也不同:

```objective-c
var routes = Routes()
// Create routes for version 1 API
var api = Routes()
api.add(method: .get, uri: "/call1", handler: { _, response in
    response.setBody(string: "API CALL 1")
    response.completed()
})
api.add(method: .get, uri: "/call2", handler: { _, response in
    response.setBody(string: "API CALL 2")
    response.completed()
})
 
// API version 1
var api1Routes = Routes(baseUri: "/v1")
// API version 2
var api2Routes = Routes(baseUri: "/v2")
 
// Add the main API calls to version 1
api1Routes.add(routes: api)
// Add the main API calls to version 2
api2Routes.add(routes: api)
// Update the call2 API
api2Routes.add(method: .get, uri: "/call2", handler: { _, response in
    response.setBody(string: "API v2 CALL 2")
    response.completed()
})
 
// Add both versions to the main server routes
routes.add(routes: api1Routes)
routes.add(routes: api2Routes)
```



## Adding Server Routes

HTTP1.1 和FastCGI perfect服务器都支持路由. 调用`addRoutes`函数就可以把路由添加到服务器. `addRoutes`函数可以被多次调用,以便添加多个路由. 一旦服务器启动后, 路由不可以再被添加和修改.

```objective-c
// Create server object
let server = HTTPServer()
// Add our routes
let routes = Routes()
...
// Add routes to server
server.addRoutes(routes)
```



## Variables

Routes URIs可以包含变量. 变量需要使用`{}`括起来. 在大括号内部是变量标识. 变量标志符可以由任何字母组成,除了`{}`大括号. 变量的工作机制有点像匹配单个字符的通配符, 它是用来匹配单个字符路径值的. 任何被变量匹配到的URL组件真实值会被记录到`HTTPRequest.urlVariables`字典中, 并且有效. 这个字典是`[String:String]`类型的. URI变量是一个收集request动态数据的很好方式. 例如, URL可以将用户管理关联到request上, 包含user id到URL组件中.



例如, 假设有这么一个URI`/foo/{bar}/baz`, URL为`/foo/123/baz`的request将会匹配到,`["bar":"123"]`这对key-value将会被放到`HTTPRequest.urlVariables`字典中.



## Wildcards

通配符是用来代替若干个字符的. 除了完整的文字URI路径，路由可以包含通配符段。



通配符可以匹配URI中的任何一部分, 可以将一组路由导向一个handler. 通配符由若干个星号组成. 只要URI路径能标识一个完整的URI组件,single asterisk可以出现在任何地方.  A double asterisk, or trailing wildcard只能出现在URI尾部. 尾部通配符匹配URI中任何剩余的部分.



URI为`/foo/*/baz`的路由可以匹配如下两个URLs:

```objective-c
/foo/123/baz
/foo/bar/baz
```

URI为`/foo/**`的路由将会匹配如下URLs:

```objective-c
/foo/bar/baz
/foo
```



URI为`/**`的路由将会匹配任何请求.

尾部为通配符的路由当被匹配到时,将会保存URI部分.  It will place this path segment in the `HTTPRequest.urlVariables` map under the key indicated by the global variable `routeTrailingWildcardKey`. 例如, 假设一个路由的URI为`/foo/**`,请求的URI为`/foo/bar/baz`, 下面这段代码执行结果为`true`:

```objective-c
request.urlVariables[routeTrailingWildcardKey] == "/bar/baz"
```



## Priority/Ordering

由于路由URIs可能会冲突, 所以,字符/通配符/变量路径会按照一个特定的顺序检查. Path匹配按照如下顺序:

1.Variable paths

2.Literal paths

3.Wildcard paths

4.Trailing wildcard paths are checked last

## Implicit Trailing Wildcard

当服务器的`.documentRoot`属性设置时, 服务器会自动添加`/**` 尾部通配符的路由, 自动私服静态文件到指定的文件夹下. 例如, 设置document root到`./webroot`下, 服务器将会呈递该文件夹下的任意文件. 可见,我们不可以乱用通配符.



## Further Information

For more information and examples of URL routing, see the [URL Routing](https://github.com/PerfectExamples/Perfect-URLRouting) example application.