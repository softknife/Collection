# Request and Response Filters

除了常规的request/response handler system之外, Perfect server还提供了一套request and response filtering system. 任何一个被添加到server中的filter在客户端发起请求时都会被调用. 当这些filter轮流运转时,每一个filter都会有机会,在request被传递给handler之前去改变request 对象,或者在request被标记为complete之后,去改变response 对象. Filters还可以选择终止当前request.



Filters添加到server中时,我们可以同时设置priority. Priority levels can be either high, medium, or low. 高优先级的filters总是优先于medium和low的filter先被执行. 同理对于medium相对low的filter.



由于每个请求都会执行一遍filters系统, 所以务必保证filter中执行的任务尽量精简,以便快速执行完而不至于阻塞进程.



## Relevant Examples

- [Perfect-HTTPRequestLogging](https://github.com/PerfectExamples/Perfect-HTTPRequestLogging)



## Request Filters

Request filters 在request读取完,但是还未呈递给request handler之间会被调用. 这就给filters一个机会去修改request对象,在他被handle之前.



## Creating

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



## Adding

