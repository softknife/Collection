# HTTPRequest

当我们处理请求时,所有与客户端的交互都是通过HTTPReques和HTTPResponse实现的.

HTTPRequest 对象中会将客户端请求的信息转换成变量, 例如: headers, query parameters, POST body data, and other relevant information such as the client IP address and URL variables.



HTTPRequest 对象会解析和反编码所有的`application/x-www-form-urlencoded`类型和`multipart/form-data`类型的请求.   It will make the data for any other content types available in a raw, unparsed form.  当处理multipart form数据时,HTTPRequest会自动解码数据,然后为上传上来的数据创建临时文件. 在请求结束之前这些临时文件将会一直存在,请求结束后,他们将会被自动删除.  下面列出了HTTPRequest协议中的所有属性和函数.



## Relevant Examples

- [Perfect-HTTPRequestLogging](https://github.com/PerfectExamples/Perfect-HTTPRequestLogging)



## Metadata

HTTPRequest 提供一些数据,这些数据并不是客户端发送过来的. 这些信息包括: the client and server IP addresses, TCP ports, and document root fit into this category.



Client and server addresses 以元祖(a finite有限的 ordered list of elements)形式存在, 其中包括IP addresses 和对应的端口.

```swift
/// The IP address and connecting port of the client.
var remoteAddress: (host: String, port: UInt16) { get }
/// The IP address and listening port for the server.
var serverAddress: (host: String, port: UInt16) { get }
```



当server创建时,你可以设置他的canonial name,或者CNAME. 这在各种情况下都是有用的，例如创建到服务器的完整链接时。当一个HTTPRequest创建时, 服务器将会应用CNAME到HTTPRequest上. 通过如下属性我们可以使canonial name有效:

```swift
/// The canonical name for the server.
var serverName: String { get }
```



静态内容一般放置到服务器的document root文件夹下, 达到私服的功能. 如果你不私服静态内容, 这个document root可能不存在. 在服务器开始接收requests之前, document root需要被配置好. 在准备访问静态内容(例如Mustache 模板)之前, 你一般需要给所有文件加上前缀document root:

```swift
/// The server's document root from which static file content will generally be served.
var documentRoot: String { get }
```



## Request Line

一个请求路线由如下组成: a method, path, query parameters, and an HTTP protocol identifier.  例如:

```swift
GET /path?q1=v1&q2=v2 HTTP/1.1
```



HTTPRequest 解析request line 到如下属性里.  `queryParams`这个参数是一个数组,其中每个元素是一个k-v元祖, 其中的每个key-value都是URL decoded的:



```swift
/// The HTTP request method.
var method: HTTPMethod { get set }
/// The request path.
var path: String { get set }
/// The parsed and decoded query/search arguments.
var queryParams: [(String, String)] { get }
/// The HTTP protocol version. For example (1, 0), (1, 1), (2, 0)
var protocolVersion: (Int, Int) { get }
```



在路由过程中, 路由URI可能含有URL variables, 这些变量已经被解析过,并放到字典中:

```swift
/// Any URL variables acquired during routing the path to the request handler.
var urlVariables: [String:String] { get set }
```



HTTPRequest 也可以获得full request URI . 它包括request path 和 URL encoded query parameters:

```swift
/// Returns the full request URI.
var uri: String { get }
```



## Client Headers

客户端请求headers 可以通过名字单个访问,或者通过迭代器访问所有值.  HTTPRequest将会自动解析所有HTTP cookie的键值对, 并且使其可以被访问. 所有的request header name展示在枚举类型中`HTTPRequestHeader.Name`,  其中还包含用来自定义header.name的枚举值`.custom(name:String)` .



 请求读取后,我们才可以设置client headers. 这将会很有用,例如,在  HTTPRequest filters 中这就很有用, 因为filters方法中我们可能需要重写或者添加某个的headers:

```swift
/// Returns the requested incoming header value.
func header(_ named: HTTPRequestHeader.Name) -> String?
/// Add a header to the response.
/// No check for duplicate or repeated headers will be made.
func addHeader(_ named: HTTPRequestHeader.Name, value: String)
/// Set the indicated header value.
/// If the header already exists then the existing value will be replaced.
func setHeader(_ named: HTTPRequestHeader.Name, value: String)
/// Provide access to all current header values.
var headers: AnyIterator<(HTTPRequestHeader.Name, String)> { get }
```



我们可通过一个以元祖为元素的数组访问cookie:

```swift
/// Returns all the cookie name/value pairs parsed from the request.
var cookies: [(String, String)] 
```



## GET and POST Parameters & Body Data



对于`application/x-www-form-urlencoded` and `multipart/form-data` content-types, HTTPRequest会自动将请求中的内容解析并放置到`postParams`或者`postFileUploads`属性中.



Request body data如果是其他content-type, 那么他们将不会被解析,直接以raw bytes或者String data类型. 例如, 客户端提交一个`JSON data`, 服务器以String的方式去访问body data , 那么body data 将会被decoded into 有意义的值.



HTTPRequest 通过如下属性将body data 转换成可访问的数据:

```swift

/// POST body data as raw bytes.
/// If the POST content type is multipart/form-data then this will be nil.
var postBodyBytes: [UInt8]? { get set }
/// POST body data treated as UTF-8 bytes and decoded into a String, if possible.
/// If the POST content type is multipart/form-data then this will be nil.
var postBodyString: String? { get }
```



值得注意的是, 如果客户端request 中content-type为`multipart/form-data`类型, 那么`postBodyBytes`值为空.  但是, 无论content-type是什么类型,request中都会有body data.



`postBodyString`属性将会尝试着将body data 从UTF-8类型转化成String类型.  如果body data为空或者无法将body data从UTF-8转化成功, 那么`postBodyString`属性为空.



### Using Form Data

在REST 应用中,有许多常用HTTP 谓词. 最常用是`POST`和`GET`谓词.

*什么时候使用什么谓词的最佳分配实践在每个方法中千变万化,没有定式, 这方面的知识已经超出了本文讨论范畴*



`GET`方法将所有参数放到URL中:

```swift
http://www.example.com/page.html?message=Hello,%20World!
```



上面例子中的"query parameters"可以使用`.queryParams`方法访问:

```swift
let params = request.queryParams
```

尽管上面的例子仅仅是拿GET请求举例,但是 `.queryParams`方法适用于所有请求,因为所有请求都包含query parameters.



#### POST Parameters

在浏览器或者其他端与APIs之间, POST parameters是传递复杂数据的标准做法, 一般用来创建或者修改内容.

Perfect's HTTP 库 使得访问POST params 数组或者某一个params变得容易.

返回所有params(Query or POST),以`[(String,String)]`数组形式:

```swift
let params = request.params()
```



返回仅仅POST params ,以`[(String,String)]`数组形式:

```swift
let params = request.postParams()
```

返回具有特定名称（如多个复选框）的所有参数:

```swift
let params = request.postParams(name: <String>)
```

这个返回值为`[String]`.



返回一个特定的参数,例如一个可选的`String?`:

```swift
let param = request.param(name: <String>)
```



当给定的一个`request`对象中的POST parameter是可选的, 如果这个参数没有给定值时,我们给他一个默认值是很有用的,使用下面的语法返回一个可选的字符串:

```swift
let param = request.param(name: <String>, defaultValue: <string>)
```



### File Uploads

**using form data**的一个特别的例子是处理上传文件请求.

有两个主要的form encoding types:

- application/x-www-form-urlencoded (the default)
- multipart/form-data

当你想要包含文件上传功能时, 你必须选择multipart/form-data格式作为你的form's `enctype`(encoding)type.



下面代码可以被看做[https://github.com/iamjono/perfect-file-uploads](https://github.com/iamjono/perfect-file-uploads)例子中的一个请求行为.

HTML表单的例子—— 包含正确的encoding和file input 元素, 应该展示为:

```swift
<form 
    method="POST" 
    enctype="multipart/form-data" 
    action="/upload">
    <input type="file" name="filetoupload">
    <br>
    <input type="submit">
</form>
```



#### Receiving the File on the Server Side

因为表单时POST 方法, 我们将会处理带有`method:.post`的路由:

```swift
var routes = Routes()
routes.add( method: .post, uri: "/upload", handler: handler)
server.addRoutes(routes)
```



一旦请求已经offload到`handler`中,我们可以:

```swift
// Grab the fileUploads array and see what's there
// If this POST was not multi-part, then this array will be empty
 
if let uploads = request.postFileUploads, uploads.count > 0 {
    // Create an array of dictionaries which will show what was uploaded
    var ary = [[String:Any]]()
 
    for upload in uploads {
        ary.append([
            "fieldName": upload.fieldName,
            "contentType": upload.contentType,
            "fileName": upload.fileName,
            "fileSize": upload.fileSize,
            "tmpFileName": upload.tmpFileName
            ])
    }
    values["files"] = ary
    values["count"] = ary.count
}
```



就像上面展示的那样, 上传上来的文件被展示在`request.postFileUploads`数组中, 各个属性,例如:`fileName`,`fileSize`and`tmpFileName`在数组元素中访问到.

> 注意: 上传的文件会被放到temp文件夹中. 你需要将他们转移到目标位置.



所以,我们来创建一个文件夹去保持上传文件. 为了安全考虑, 我们需要把该文件放在webroot文件夹之外:

```swift

// create uploads dir to store files
let fileDir = Dir(Dir.workingDir.path + "files")
do {
    try fileDir.create()
} catch {
    print(error)
}
```



下一步, 在`for upload in uploads`这块代码中, 我们需要添加移动文件的代码:

```swift
// move file
let thisFile = File(upload.tmpFileName)
do {
    let _ = try thisFile.moveTo(path: fileDir.path + upload.fileName, overWrite: true)
} catch {
    print(error)
}
```



现在,上传的文件将会被移动到具体的文件夹中, 原来的文件名依然保留.



更多的文件系统操作,请参考the [Directory Operations](http://www.perfect.org/docs/dir.html) and [File Operations](http://www.perfect.org/docs/file.html) chapters.





