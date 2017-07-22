# Getting Started From Scratch



## Getting Started with Perfect

下面我们将会从头开始构建一个web 应用服务端. 

先创建一个文件夹,用来存放你的应用: 

```objective-c
mkdir MyAwesomeProject
cd MyAwesomeProject
```



为了更好的开发体验,我们添加一个`git repo`:

```objective-c
git init
touch README.html
git add README.html
git commit -m "Initial commit"
```

> It's also recommended to add a `.gitignore` similar to the contents of [this Swift .gitignore template from gitignore.io](https://www.gitignore.io/api/swift).



### Create the Swift Package

在根目录下创建一个`Package.swift`文件,并且输入如下内容,这些内容是是用来导入Perfect框架的.

```objective-c
import PackageDescription
 
let package = Package(
    name: "MyAwesomeProject",
    dependencies: [
        .Package(
        url: "https://github.com/PerfectlySoft/Perfect-HTTPServer.git",
        majorVersion: 2
        )
    ]
)
```



然后,创建一个叫做`Sources`的文件夹, 在其内部创建一个`main.swift`的文件: 

```objective-c
mkdir Sources
echo 'print("Well hi there!")' >> Sources/main.swift
```



至此, 项目准备工作已经做完, 我们可以通过如下命令将服务器跑起来:

```objective-c
swift build
.build/debug/MyAwesomeProject
```



你将会看到如下输出:

```objective-c

Well hi there!

```



### Setting up the server

现在,我们用`Swift package `构建的空项目已经运行起来了, 下一步就是去安装HTTPServer. 打开`Sources/main.swift`文件,清空内容后,换成如下内容:

```objective-c
import PerfectLib
import PerfectHTTP
import PerfectHTTPServer
 
// Create HTTP server.
let server = HTTPServer()
 
// Register your own routes and handlers
var routes = Routes()
routes.add(method: .get, uri: "/", handler: {
        request, response in
        response.setHeader(.contentType, value: "text/html")
        response.appendBody(string: "<html><title>Hello, world!</title><body>Hello, world!</body></html>")
        response.completed()
    }
)
 
// Add the routes to the server.
server.addRoutes(routes)
 
// Set a listen port of 8181
server.serverPort = 8181
 
do {
    // Launch the HTTP server.
    try server.start()
} catch PerfectError.networkError(let err, let msg) {
    print("Network error thrown: \(err) \(msg)")
}
```



重新build , run the project:

```objective-c
swift build
.build/debug/MyAwesomeProject
```



> The server is now running and waiting for connections. Access [http://localhost:8181/](http://127.0.0.1:8181/) to see the greeting. Hit "control-c" to terminate the server.



### Xcode

SPM可以用来生成xcode项目, 在终端里运行如下命令:

`swift package generate-xcodeproj`

打开生成的`MyAwesomeProject.xcodeproj`文件, add the following to the "Library Search Paths" for the project (not just the target):

`$(PROJECT_DIR) - Recursive`

Ensure that you have selected the executable target and selected it to run on "My Mac". Also ensure that the correct Swift toolchain is selected. You can now run and debug the server directly in Xcode.(新版的xcode8.2已经不需要此步骤了).











