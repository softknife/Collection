# Static File Content



就像在Routing 章节看到的那样, Prefect 擅长处理复杂的动态routing. 同时,他还擅长处理静态的内容,例如: HTML,images,CSS,以及JavaScript.



Static file content 服务的处理是通过`StaticFileHandler`对象来完成的. 一旦一个Request 对象传递给这个对象, 他将会找到指定的文件或者找不到时返回404. `StaticFileHandler` also handles caching through use of the ETag header as well as byte range serving for very large files.



`StaticFileHandler`对象通过`a document root path parameter`来完成初始化. 这个root path将会作为所有文件prefix添加到文件路径头部. 当前HTTPRequest对象的`path`属性被用来指定文件的路径,该文件将会被读取和返回.



`StaticFileHandler`可以在你的web handler中直接创建,然后调用他的`handleRequest`方法.



例如, 返回一个Request file的handler展示如下:

```swift
{
    request, response in
    StaticFileHandler(documentRoot: request.documentRoot).handleRequest(request: request, response: response)
}
```



然而, 除非需要自定义行为, 否则不必这样手动处理. 设置server的`documentRoot`属性将会自动添加一个handler, 这个handler将会为指定文件夹下的所有文件提供伺服服务.  设置server的document root 等同于如下代码片段:

```swift
let dir = Dir(documentRoot)
if !dir.exists {
    try Dir(documentRoot).create()
}
routes.add(method: .get, uri: "/**", handler: {
    request, response in
    StaticFileHandler(documentRoot: request.documentRoot).handleRequest(request: request, response: response)
})
```



添加的handler将会给所有root目录下/子目录下的文件提供伺服.



documentRoot 属性的使用例子可以在PerfectTemplate中找到:

```swift
import PerfectLib
import PerfectHTTP
import PerfectHTTPServer
 
// Create HTTP server.
let server = HTTPServer()
 
// Register your own routes and handlers
var routes = Routes()
routes.add(method: .get, uri: "/", handler: {
        request, response in
        response.appendBody(string: "<html>...</html>")
        response.completed()
    }
)
 
// Add the routes to the server.
server.addRoutes(routes)
 
// Set a listen port of 8181
server.serverPort = 8181
 
// Set a document root.
// This is optional. If you do not want to serve 
// static content then do not set this.
// Setting the document root will automatically add a 
// static file handler for the route
server.documentRoot = "./webroot"
 
// Gather command line options and further configure the server.
// Run the server with --help to see the list of supported arguments.
// Command line arguments will supplant any of the values set above.
configureServer(server)
 
do {
    // Launch the HTTP server.
    try server.start()
} catch PerfectError.networkError(let err, let msg) {
    print("Network error thrown: \(err) \(msg)")
}
```



注意`server.documentRoot = "./webroot"` 这行代码. 他意思是如果在webroot文件夹下有个style.css 文件, 那么URI为"/style.css"的Request发起时, 将会返回这个文件给浏览器.



下面的例子中创建了一个虚拟的文件路径, 从物理目录"/var/www/htdocs"提供以"/files"未开头的所有URI:



```swift
routes.add(method: .get, uri: "/files/**", handler: {
    request, response in
 
    // get the portion of the request path which was matched by the wildcard
    request.path = request.urlVariables[routeTrailingWildcardKey]
 
    // Initialize the StaticFileHandler with a documentRoot
    let handler = StaticFileHandler(documentRoot: "/var/www/htdocs")
 
    // trigger the handling of the request, 
    // with our documentRoot and modified path set
    handler.handleRequest(request: request, response: response)
    }
)
```



在上述例子中, URI为"/files/foo.html"的一个Request将会返回对应的文件"/var/www/htdocs/foo.html".

















