# Getting Started

你渴望使用swift和Perfect编程吗? 这份指南将会让你知道如何运行Perfect,创建你的第一个APP.



通过这份指南,你将要学到的知识包括:

- How to create and run an HTTP/HTTPS server and get Perfect up and running
- The prerequisite components you must install to run Perfect on either OS X or Ubuntu Linux
- How to build, test, and manage dependencies for Swift projects
- How to deploy Perfect in additional environments including Heroku, Amazon Web Services, Docker, Microsoft Azure, Google Cloud, IBM Bluemix CloudFoundry, and IBM Bluemix Docker



## 一.准备工作:

### swift3.0

当你安装过3.0工具链之后, 打开终端,输入`swift --version`

将会看到类似如下的一段话:

`Apple Swift version 3.0.1 (swiftlang-800.0.58.6 clang-800.0.42.1)Target: x86_64-apple-macosx10.9`



确保你运行的是swift3.0.1发布版. 如果你运行的版本低于3.0.1,那么Perfect不能成功编译.

> You can find out which version of Swift you will need by looking in [the README of the main Perfect repo](https://github.com/PerfectlySoft/Perfect#compatibility-with-swift).



### OS X

不再需要安装其他组件.

### Ubuntu Linux

Perfect runs in Ubuntu Linux 14.04, 15.10 and 16.04 environments. Perfect relies on OpenSSL, libssl-dev, and uuid-dev. To install these, in the terminal, type:

`sudo apt-get install openssl libssl-dev uuid-dev`

When building on Linux, OpenSSL 1.0.2+ is required for this package. On Ubuntu 14 or some Debian distributions you will need to update your OpenSSL before this package will build.



********

## 二.Getting Started with Perfect

现在可以开始构建你的第一个web app项目了.

### Build Starter Project

在终端中执行如下语句,将会clone并创建一个空的项目. 将会在你的电脑上启动一个本地服务器,监听8181端口.



```objective-c
git clone https://github.com/PerfectlySoft/PerfectTemplate.git
cd PerfectTemplate
swift build
.build/debug/PerfectTemplate
```



终端上,你将会看到输出:

`Starting HTTP server on 0.0.0.0:8181 with document root ./webroot`



服务器现在运行中,等待接入. 你可以访问`http://localhost:8181/` 地址查看欢迎语. 按`control-c` 可以终止服务.



### Xcode

SPM可以生成一个Xcode项目,这样你就可以在xcode中进行代码编辑与调试了.输入如下内容生成xcode项目:

`swift package generate-xcodeproj`

打开你刚刚生成的`PerfectTemplate.xcodeproj`文件. Ensure that you have selected the executable target and selected it to run on "My Mac". 你就可以直接在xcode上调试这个server项目了.





***********

## 三.Next Steps

下面一段代码将会向你展示在开发一款web服务器或者REST服务器时,如何处理一些经常会遇到的任务. 下面所有例子中的`request`和`response`变量都是指代`HTTPRequest`和`HTTPResponse`对象.



细节可以参考 [API reference](https://perfect.org/docs/api.html) .



### Get a Client Request Header

```objective-c
if let acceptEncoding = request.header(.acceptEncoding) {
    ...
}
```



### Client GET and POST Parameters

```objective-c
if let foo = request.param(name: "foo") {
    ...
}
if let foo = request.param(name: "foo", defaultValue: "default foo") {
    ...
}
let foos: [String] = request.params(named: "foo")
```



### Get the Current Request Path

```objective-c
let path = request.path
```



### Accessing the Server's Document Directory and Returning an Image File to the Client

> 如果是开发API 服务器,此项可以忽略

```objective-c
let docRoot = request.documentRoot
do {
    let mrPebbles = File("\(docRoot)/mr_pebbles.jpg")
    let imageSize = mrPebbles.size
    let imageBytes = try mrPebbles.readSomeBytes(count: imageSize)
    response.setHeader(.contentType, value: MimeType.forExtension("jpg"))
    response.setHeader(.contentLength, value: "\(imageBytes.count)")
    response.setBody(bytes: imageBytes)
} catch {
    response.status = .internalServerError
    response.setBody(string: "Error handling request: \(error)")
}
response.completed()
```



### Getting Client Cookies

> 如果是开发API服务器,此项可以忽略

```objective-c
for (cookieName, cookieValue) in request.cookies {
    ...
}

```



### Setting Client Cookies

> 如果是开发API服务器,此项可以忽略

```objective-c
let cookie = HTTPCookie(name: "cookie-name", value: "the value", domain: nil,
                    expires: .session, path: "/",
                    secure: false, httpOnly: false)
response.addCookie(cookie)	
```



### Returning JSON Data to Clien

```objective-c
response.setHeader(.contentType, value: "application/json")
let d: [String:Any] = ["a":1, "b":0.1, "c": true, "d":[2, 4, 5, 7, 8]]
 
do {
    try response.setBody(json: d)
} catch {
    //...
}
response.completed()
```

> *This snippet uses built-in JSON encoding. Feel free to use the JSON encoder/decoder you prefer.*



### Redirecting the Client

> 如果是开发API服务器,此项可以忽略

```objective-c
response.status = .movedPermanently
response.setHeader(.location, value: "http://www.perfect.org/")
response.completed()
```



### Filtering and Handling 404 Errors in a Custom Manner

```objective-c
struct Filter404: HTTPResponseFilter {
    func filterBody(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        callback(.continue)
    }
 
    func filterHeaders(response: HTTPResponse, callback: (HTTPResponseFilterResult) -> ()) {
        if case .notFound = response.status {
            response.bodyBytes.removeAll()
            response.setBody(string: "The file \(response.request.path) was not found.")
            response.setHeader(.contentLength, value: "\(response.bodyBytes.count)")
            callback(.done)
        } else {
            callback(.continue)
        }
    }
}
 
try HTTPServer(documentRoot: webRoot)
    .setResponseFilters([(Filter404(), .high)])
    .start(port: 8181)
```



