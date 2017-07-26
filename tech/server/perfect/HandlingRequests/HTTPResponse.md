# HTTPResponse

当我们处理一个request时, 所有的客户端交互都是通过HTTPRequest 和HTTPResponse对象来完成的.



HTTPResponse对象中包括所有的输出响应数据. 他的组成为: the HTTP status code and message, the HTTP headers, and any response body data. HTTPResponse还包含向客户端传送(stream)或推送(push)响应数据块的功能, 以及完成和终止请求的能力.



在下面文章内容里, 除非特别说明, 否则所有的属性和函数都是HTTPResponse 协议的一部分.



## Relevant Examples

- [Perfect-Cookie-Demo](https://github.com/PerfectExamples/Perfect-Cookie-Demo)
- [Perfect-HTTPRequestLogging](https://github.com/PerfectExamples/Perfect-HTTPRequestLogging)



### HTTP Status

这个HTTP Status是指发送给客户端的请求是否成功. 如果有错误, 或者请求还要采取其他任何行为, 否则, 默认情况下HTTPResponse 对象状态参数为200 OK状态. 如果需要status可以赋予任意值. HTTP status 代码展示在`HTTPResponseStatus`枚举值中. 这个枚举中包含了所有正式的状态码 和一个`.custom(code:Int,message:String)`自定义枚举值.



Response的状态值通过如下属性设置:

```swift
/// The HTTP response status.
var status: HTTPResponseStatus { get set }
```



### Response Headers

Response headers可以获取,设置和迭代. 正式的或者常用的headers names 通过`HTTPResponse.Name`枚举展示. 枚举中同样包含一个自定的header names:`.custom(name:String)`:

```swift
/// Returns the requested outgoing header value.
func header(_ named: HTTPResponseHeader.Name) -> String?
/// Add a header to the outgoing response.
/// No check for duplicate or repeated headers will be made.
func addHeader(_ named: HTTPResponseHeader.Name, value: String)
/// Set the indicated header value. 
/// If the header already exists then the existing value will be replaced.
func setHeader(_ named: HTTPResponseHeader.Name, value: String)
/// Provide access to all current header values.
var headers: AnyIterator<(HTTPResponseHeader.Name, String)> { get }

```



HTTPResponse提供一个优先支持HTTP cookies. cookies的完成需要通过创建cookie对象,然后把它添加到response对象中.



```swift
/// This bundles together the values which will be used to set a cookie in the outgoing response
public struct HTTPCookie {
    /// Cookie public initializer
    public init(name: String,
                value: String,
                domain: String?,
                expires: Expiration?,
                path: String?,
                secure: Bool?,
                httpOnly: Bool?)
}
```



Cookies通过如下函数绑定到HTTPResponse:

```swift
/// Add a cookie to the outgoing response.
func addCookie(_ cookie: HTTPCookie)
```

 当cookie被添加后, 他被格式化, 并通过"Set-Cookie"这个key绑定到response header上.



### Body Data

response当前body data 通过如下属性可以访问到:

```swift
/// Body data waiting to be sent to the client.
/// This will be emptied after each chunk is sent.
var bodyBytes: [UInt8] { get set }
```



数据可以直接添加到这个数组中, 或者通过如下任何一个方便的方法添加进去. 这些函数或者直接将数据set进去,或者直接将raw UInt8 bytes/String data append 到body bytes数组. String data将会转化为UTF-8格式. 最后一个函数云讯将一个[String:Any]字典装换JSON String.



```swift
/// Append data to the bodyBytes member.
func appendBody(bytes: [UInt8])
/// Append String data to the outgoing response.
/// All such data will be converted to a UTF-8 encoded [UInt8]
func appendBody(string: String)
/// Set the bodyBytes member, clearing out any existing data.
func setBody(bytes: [UInt8])
/// Set the String data of the outgoing response, clearing out any existing data.
/// All such data will be converted to a UTF-8 encoded [UInt8]
func setBody(string: String)
/// Encodes the Dictionary as a JSON string and converts that to a UTF-8 encoded [UInt8]
func setBody(json: [String:Any]) throws
```



当我们准备将数据返回给客户端时, 一定要记得在header中设置content-length.  当HTTPResponse对象开始发送累积的数据到客户端时, 将会检查content-length是否被设置. 如果没有, 会自动根据`bodyBytes`数组的大小自动给header设置该属性值.



下面这个函数将会推送所有当前response headers 和 任何body data:

```swift
/// Push all currently available headers and body data to the client.
/// May be called multiple times.
func push(callback: (Bool) -> ())
```

大多数请求中, 没必要调用这个方法,因为当请求结束时,系统会自动更新所有有待输出的数据.  然而, 在一些情况下, 我们还是希望对这个过程给予更多的直接控制.

 例如, 如果你准备呈递一个非常大的文件时, 读取整个文件到内存,并且直接设置到body bytes中是很不现实的. 取而代之的方法是, 你应该设置response的content-length为文件大小,然后, 块级读取文件数据并放到body bytes中, 然后多次调用这个`push`函数, 直到所有的文件发送完成.  这个`push`函数将会回调这个callback,并传递一个bool值作为参数. 如果内容成功发送给客户端,那这个bool值为true. 如果bool值为false,那么认为请求失败, 没有后续动作.



### Streaming

在一些情况下, response的content-length并不能容易确定. 例如, 如果你正在传送直播视频或者音乐内容, 那么 不可能设置content-length header. 在这种情况下, 将HTTPResponse 对象设置为streaming 模式. 当使用streaming  mode时, response将会发送HTTP-chunked encoding.  当streaming时,不需要设置content-length. 取而代之的通常做法是将内容添加到body中, 然后调用`push`函数. 如果push成功, body data将会清空,然后读取更多数据.  不断的调用`push`方法直到request结束, 或者直到push函数回调callback并传递一个`false`的参数.



```swift
/// Indicate that the response should attempt to stream all outgoing data.
/// This is primarily used when the resulting content length can not be known.
var isStreaming: Bool { get set }
```



如果使用streaming , 在数据push给客户端之前,需要把`isStreaming`属性设置为true.



### Request Completion

**重要**: 当请求结束时,一定要调用`completed()`函数. 这样可以保证所有有待输出的数据被发送给客户端, 底层的TCP 链接才可以或者被关闭,或者保持在HTTP keep-alive状态, 新的request将会被读取,处理.

```swift
/// Indicate that the request has completed.
/// Any currently available headers and body data will be pushed to the client.
/// No further request related activities should be performed after calling this.
func completed()
```

