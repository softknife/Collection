# Request and Response Filters

除了常规的request/response handler system之外, Perfect server还提供了一套request and response filtering system. 任何一个被添加到server中的filter在客户端发起请求时都会被调用. 当这些filter轮流运转时,每一个filter都会有机会,在request被传递给handler之前去改变request 对象,或者在request被标记为complete之后,去改变response 对象. Filters还可以选择终止当前request.



Filters添加到server中时,我们可以同时设置priority. Priority levels can be either high, medium, or low. 高优先级的filters总是优先于medium和low的filter先被执行. 同理对于medium相对low的filter.



由于每个请求都会执行一遍filters系统, 所以务必保证filter中执行的任务尽量精简,以便快速执行完而不至于阻塞进程.



## Relevant Examples

- [Perfect-HTTPRequestLogging](https://github.com/PerfectExamples/Perfect-HTTPRequestLogging)



## Request Filters

Request filters 在request读取完,但是还未呈递给request handler之间会被调用. 这就给filters一个机会去修改request对象,在他被handle之前.



#### Creating

Request filters 必须遵守`HTTPRequestFilter`协议:

```swift

/// A filter which can be called to modify a HTTPRequest.
public protocol HTTPRequestFilter {
    /// Called once after the request has been read but before any handler is executed.
    func filter(request: HTTPRequest, response: HTTPResponse, callback: (HTTPRequestFilterResult) -> ())
}
```



当到filter运行的时间时, `filter`函数会被调用. 这些filter应该执行所有的行为, 然后回调callback, 说明他已经完成自己进程. 这个callback中带有一个参数, 用来指定下一步需要做什么.  这个参数可以告诉系统应该进行动作包括: 继续处理filters, 或者停止当前优先级的request filters,将request呈递给handler, 又或者完全终止request.



```swift
/// Result from one filter.
public enum HTTPRequestFilterResult {
    /// Continue with filtering.
    case `continue`(HTTPRequest, HTTPResponse)
    /// Halt and finalize the request. Handler is not run.
    case halt(HTTPRequest, HTTPResponse)
    /// Stop filtering and execute the request.
    /// No other filters at the current priority level will be executed.
    case execute(HTTPRequest, HTTPResponse)
}
```



由于filter会接收request和response, 然后将他们传递到自己的`HTTPRequestFilterResult`中,  所以在filter中,我们完全可以替换他们,如果像这样做的话.



#### Adding

Request filters 直接添加到server上, 并以`[(HTTPRequestFilter,HTTPFilterPriority)]`形式传入:

```swift
public class HTTPServer {
    public func setRequestFilters(_ request: [(HTTPRequestFilter, HTTPFilterPriority)]) -> HTTPServer
}
```



调用这个方法可以添加filter. 添加每个filter时需要设置他的优先级. filters数组中的元素可以是任意顺序, 服务器将会对他们进行再排序, 将high-priority的filter放到lower的前面,相同优先级的filter顺序不变.



#### Example

下面举一个filter相关的例子. 他说明了如何创建,添加,以及filter优先级如何交互.

```swift
var oneSet = false
var twoSet = false
var threeSet = false
 
struct Filter1: HTTPRequestFilter {
    func filter(request: HTTPRequest, response: HTTPResponse, callback: (HTTPRequestFilterResult) -> ()) {
        oneSet = true
        callback(.continue(request, response))
    }
}
struct Filter2: HTTPRequestFilter {
    func filter(request: HTTPRequest, response: HTTPResponse, callback: (HTTPRequestFilterResult) -> ()) {
        XCTAssert(oneSet)
        XCTAssert(!twoSet && !threeSet)
        twoSet = true
        callback(.execute(request, response))
    }
}
struct Filter3: HTTPRequestFilter {
    func filter(request: HTTPRequest, response: HTTPResponse, callback: (HTTPRequestFilterResult) -> ()) {
        XCTAssert(false, "This filter should be skipped")
        callback(.continue(request, response))
    }
}
struct Filter4: HTTPRequestFilter {
    func filter(request: HTTPRequest, response: HTTPResponse, callback: (HTTPRequestFilterResult) -> ()) {
        XCTAssert(oneSet && twoSet)
        XCTAssert(!threeSet)
        threeSet = true
        callback(.halt(request, response))
    }
}
 
var routes = Routes()
routes.add(method: .get, uri: "/", handler: {
        request, response in
        XCTAssert(false, "This handler should not execute")
        response.completed()
    }
)
 
let requestFilters: [(HTTPRequestFilter, HTTPFilterPriority)] = [
    (Filter1(), HTTPFilterPriority.high), 
    (Filter2(), HTTPFilterPriority.medium), 
    (Filter3(), HTTPFilterPriority.medium), 
    (Filter4(), HTTPFilterPriority.low)
]
 
let server = HTTPServer()
server.setRequestFilters(requestFilters)
server.serverPort = 8181
server.addRoutes(routes)
try server.start()
```



### Response Filters

在response header data 发送给客户端之前,每个response filter都会执行一次, 后续的body data 发送前再执行一遍. 这些过滤器可以以任何他们认为合适的方式修改输出响应对象，包括添加或删除headers或重写body数据。



#### Creating

Response filters必选遵守`HTTPResponseFilter`协议.

```swift
/// A filter which can be called to modify a HTTPResponse.
public protocol HTTPResponseFilter {
    /// Called once before headers are sent to the client.
    func filterHeaders(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ())
    /// Called zero or more times for each bit of body data which is sent to the client.
    func filterBody(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ())
}
```



当等到发送response headers时, `filterHeaders`方法会调用. 这个方法中,你可以对`HTTPResponse`对象进行任何需要的操作, 然后调用callback. 调用callback时需要传递一个参数`HTTPResponseFilterResult`值, 参数类型定义如下:

```swift
/// Response from one filter.
public enum HTTPResponseFilterResult {
    /// Continue with filtering.
    case `continue`
    /// Stop executing filters until the next push.
    case done
    /// Halt and close the request.
    case halt
}
```



这些值意思是, 系统是否应该继续处理filter,还是停止执行filters等待下一次数据进入, 又或者完全终止request.



当发送一段离散的数据块到客户端时, filters的`filterBody`函数将会被调用. 这个函数中,你可以通过`HTTPResponse.bodyBytes`属性检测输出数据, 还可以修改或者替换数据. 因为这个阶段headers已经发送出去了, 所以此时对header数据进行的任何修改都会被忽略. 一旦filter's body filtering 得出结论, 就会调用callback并传递一个`HTTPResponseFilterResult`参数值. 这里的参数值和`filterHeaders`函数中一样.



#### Adding

Response Filters 直接添加到服务器上,  并以`[(HTTPResponseFilter,HTTPFilterPriority)]`形式传入:

```swift
public class HTTPServer {
    public func setResponseFilters(_ response: [(HTTPResponseFilter, HTTPFilterPriority)]) -> HTTPServer
}
```



调用这个函数设置服务器的response filters. 每个filter伴随着自己的优先级. filters数组中的元素可以是任意顺序. 服务器将会重新排序,将等级高的的放前面. 等级相同的顺序不变.



#### Example

下面举一个filter相关的例子. 说明了response filter 优先级如何操作, response filter如何修改输出headers和body 数据.



```swift
struct Filter1: HTTPResponseFilter {
    func filterHeaders(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        response.setHeader(.custom(name: "X-Custom"), value: "Value")
        callback(.continue)
    }
    func filterBody(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        callback(.continue)
    }
}
struct Filter2: HTTPResponseFilter {
    func filterHeaders(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        callback(.continue)
    }
    func filterBody(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        var b = response.bodyBytes
        b = b.map { $0 == 65 ? 97 : $0 }
        response.bodyBytes = b
        callback(.continue)
    }
}   
struct Filter3: HTTPResponseFilter {
    func filterHeaders(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        callback(.continue)
    }
    func filterBody(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        var b = response.bodyBytes
        b = b.map { $0 == 66 ? 98 : $0 }
        response.bodyBytes = b
        callback(.done)
    }
}   
struct Filter4: HTTPResponseFilter {
    func filterHeaders(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        callback(.continue)
    }
    func filterBody(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        XCTAssert(false, "This should not execute")
        callback(.done)
    }
}
 
var routes = Routes()
routes.add(method: .get, uri: "/", handler: {
    request, response in
    response.addHeader(.contentType, value: "text/plain")
    response.isStreaming = true
    response.setBody(string: "ABZ")
    response.push {
        _ in
        response.setBody(string: "ABZ")
        response.completed()
    }
})
 
let responseFilters: [(HTTPResponseFilter, HTTPFilterPriority)] = [
    (Filter1(), HTTPFilterPriority.high),
    (Filter2(), HTTPFilterPriority.medium),
    (Filter3(), HTTPFilterPriority.low),
    (Filter4(), HTTPFilterPriority.low)
]
 
let server = HTTPServer()
server.setResponseFilters(responseFilters)
server.serverPort = port
server.addRoutes(routes)
try server.start()
```



这个例子将会给response header添加一个`X-Custom`键值对, 给body中添加一个小写的a或者b. 注意这个例子中handler函数中将response设置为streaming mode, 意味着块级encoded数据将会被使用, body data将会以两块离散的块级数据分别发送出去.



### 404 Response Filter

一个更有用的例子如下. 这段代码中将会创建并添加一个filter, 这个filter监听`404 not found ` responses, 你可以提供一个自定义的描述.



```swift
struct Filter404: HTTPResponseFilter {
    func filterBody(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        callback(.continue)
    }
 
    func filterHeaders(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        if case .notFound = response.status {
            response.setBody(string: "The file \(response.request.path) was not found.")
            response.setHeader(.contentLength, value: "\(response.bodyBytes.count)")
            callback(.done)
        } else {
            callback(.continue)
        }
    }
}
 
let server = HTTPServer()
server.setResponseFilters([(Filter404(), .high)])
server.serverPort = 8181
try server.start()
```





## Web Redirects

Perfect WebRedirects module将会过滤具体的路由(包括尾部通配符路由), 然后,如果匹配到某个路由时,就会按照构造那样执行重定向.

This can be important for maintaining SEO ranking in systems that have moved.  例如, 如果我们移动一个静态的HTML网站, 这里路由从`/about.html`替换成新的路由`/about`,但是并没有有效的重定向, 这个网站或者系统将会失去SEO 排名.



这里有一个demo展示如何使用Perfect WebRedirects module: [Perfect-WebRedirects-Demo](https://github.com/PerfectExamples/Perfect-WebRedirects-Demo).



### Including in your project

在你的项目Package.swift文件中添加依赖库Perfect-WebRedirects:

```swift
.Package(url: "https://github.com/PerfectlySoft/Perfect-WebRedirects", majorVersion: 1),
```



然后在你的`main.swift`文件中, 导入并添加该filter:

```swift
import PerfectWebRedirects

// Add to the "filters" section of the config:
[
    "type":"request",
    "priority":"high",
    "name":WebRedirectsFilter.filterAPIRequest,
]
```



如果你还添加了Request Logger filters, 如果Web Redirects 对象直接在RequestLogger filter后面添加, 那么原Request(还有他的绑定的重定向代码)和请求都会被打印.



### Configuration file

路由的配置信息包含在JSON文件中,在`/config/redirect-rules/*.json`位置,如下形式:

```swift
{

  "/test/no": {
    "code": 302,
    "destination": "/test/yes"
  },

    "/test/no301": {
        "code": 301,
        "destination": "/test/yes"
  },

    "/test/wild/*": {
        "code": 302,
        "destination": "/test/wildyes"
  },

    "/test/wilder/*": {
        "code": 302,
        "destination": "/test/wilding/*"
  }

}
```



注意,多个JSON文件可以存在于该文件夹下; 当filter被触发时,所有的JSON文件都会在第一个时间加载.



上面代码中"key"负责匹配路由(the "old" file or route), "value"包含HTTP code,和重定向的新目标路由.



