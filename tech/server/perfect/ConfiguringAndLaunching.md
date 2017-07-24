# HTTPServer

这篇文章提供了三种方式,用来启动Perfect HTTPServer. 这些方式在复杂性上各不相同, 每一种都能满足不同的使用场景.



- 第一种方式是data driven, 这个数据是你提供的Swift Dictionary 或者 JSON文件 ,用来描述服务器的.    
- 第二种方式构建的服务器,完全使用Swift语言, 通过Swift的类型检查和编译时限制来完成构建. 
- 第三种方法允许你实例化一个HTTPServer对象, 在start服务器之前,你需要逐步地配置每一个属性.

你可以使用HTTPServer 命名空间里的函数配置并启动服务器. HTTPServer由以下内容组成: at least a name and a listen port, one or more handlers, and zero or more request or response filters.  除此之外, HTTPS服务器还需要TLS相关的配置信息,例如: a certificate or key file path.



当启动服务器时,你可以选择等待直到终止(这种情况一般不会发生直到进程终止),或者收到对应服务器的`LaunchContext`对象, 这个对象允许你单独地控制终止或者等待.



## HTTPServer Configuration Data

我们可以使用structured configuration data来配置一个或者多个HTTPServer,并且启动他们.  这些配置数据包括设置参数,例如: the listen port and bind address but also permits pointing handlers to specific fuctions by name.  如果我们从JSON文件中加载这些配置数据,那么这些特性是必须的. 为了在Linux上驱动这些函数, 你必须构建你的SPM可执行程序,使用如下额外的标记:

```objective-c
swift build -Xlinker --export-dynamic
```



上述命令只是在Linux上需要, 只是当你想使用JSON文件来描述服务器配置数据时才需要.



调用一个静态函数(`HTTPServer.launch`), 带有一个指向JSON配置文件的路径, 指向配置文件的文件对象, 或者 Swift字典 的参数.



配置数据将会被用来启动一个或者多个HTTPServer.

```objective-c
public extension HTTPServer {
    public static func launch(wait: Bool = true, configurationPath path: String) throws -> [LaunchContext]
    public static func launch(wait: Bool = true, configurationFile file: File) throws -> [LaunchContext]
    public static func launch(wait: Bool = true, configurationData data: [String:Any]) throws -> [LaunchContext]
}
```



参数`wait`的默认值true意味着该launch函数不会结束, 一直block,直到所有的服务器终止或者进程被杀.  如果`wait`参数值为false,那么`LaunchContext`数组返回值可以用来监听或者终止某个服务.  大多数的应用都希望函数一直处在等待状态, 所以调用launch函数时,可以忽略`wait`参数.



```objective-c
do {
    try HTTPServer.launch(configurationPath: "/path/to/perfecthttp.json")
} catch {
    // handle critical failure
}
```

注意: 配置文件可以本地化或者重命名, 但是必须有`.json`扩展名. 我们将来会支持其他格式的文件, 以确保你含有重要内容的配置文件都被识别.

从JSON解析出来后, 配置文件中,在顶层应该包括"servers"key, 对应的value是一个字典数组, 这些字典用来描述那些将会被启动的服务器.

```objective-c
[
    "servers":[
        […],
        […],
        […]
    ]
]
```



一个简单的单一服务器配置字典可能像下面格式.  注意例子中的keys/values将会在随后的文件中解释.

```objective-c
[
    "servers":[
        [
            "name":"localhost",
            "port":8080,
            "routes":[
                [
                    "method":"get",
                    "uri":"/**",
                    "handler":"PerfectHTTPServer.HTTPHandler.staticFiles",
                    "documentRoot":"/path/to/webroot"
                ],
                [
                    "methods":["get", "post"],
                    "uri":"/api/**",
                    "handler":"PerfectHTTPServer.HTTPHandler.redirect",
                    "base":"http://other.server.ca"
                ]
            ]
        ]
    ]
]
```



参数如下:



**name**:

这个属性必须是字符串数值, 它对于唯一标识服务器是首要的, 一般和服务器的名字一致. 应用可以使用服务器名字去构建URLs指向服务器本身.



多个服务器可以使用同一个名字. 例如, 你可能在同一个host上通过监听多个不同的端口创建三个服务, 这三个服务可以有相同的名字.

相应的HTTPServer属性: `HTTPServer.serverName`.

**port**:

这个属性要求是整数类型的值,  他指定了服务器监听的端口. TCP 端口范围从0-65535. 0-1024范围要求root权限. 如果我们不注意将端口绑定到这样的端口上, 请参考 `runAs`参数, 这个参数说明了进程被自动切换到那个端口上了.

对应的HTTPServer属性: `HTTPServer.serverPort`.



**address**:

这个属性是可选的字符串类型值,  应该是IP地址.  服务器会绑定到这个属性指定的本地地址上. 如果不给定, this value defaults to "0.0.0.0" which indicates that the server should bind on all available local IP addresses.   Using "::" for this value will enable listening on all local IPv6 & IPv4 addresses.

对应的HTTPServer属性: `HTTPServer.serverAddress`.



**routes**:

这个属性是可选参数, 是字典数组类型. 数组中的每个元素指定一个URI路由, 这个路由maps an incoming HTTP request to a handler. 关于Perfect框架中URI路由系统细节请参考 [Routing](https://www.perfect.org/docs/routing.html) .



每个路由由一个或者多个HTTP方法, 一个URI和一个返回`RequestHandler`的函数名组成.



关键字是"method/methods","uri","handler".  他们的值类型除了methods之外,都是字符串类型, methods为字符数组类型. method参数值没有值, 任意HTTP method都可能被触发.

当命名函数被调用时, 任何额外的键值对都可以提供作为参考. 这些键是用来配置handler的行为的. 例如, `staticFiles` handler需要一个`documentRoot`键去配置说明包含本地静态文件的文件夹.

Perfect带有request handlers , 用以处理各种常用的任务: redirecting clients or serving static, on-disk files.   下面的例子中定义了一个监听8080端口的服务器, 他有两个handlers,一个用来提供静态文件服务, 另一个用来引导客户端重定向到一个新的URL上.



```objective-c

[
    "servers":[
        [
            "name":"localhost",
            "port":8080,
            "routes":[
                [
                    "method":"get",
                    "uri":"/**",
                    "handler":"PerfectHTTPServer.HTTPHandler.staticFiles",
                    "documentRoot":"/path/to/webroot"
                ],
                [
                    "methods":["get", "post"],
                    "uri":"/api/**",
                    "handler":"PerfectHTTPServer.HTTPHandler.redirect",
                    "base":"http://other.server.ca"
                ]
            ]
        ]
    ]
]
```



相应的HTTPServer属性: `HTTPServer.addRoutes`.



**Adding Custom Request Handlers**

尽管Perfect内置的request handlers 唾手可得, 但是大多用户还是希望能添加一些自定义的处理. "handler"键值可以指向我们自己的函数, 当路由uri 匹配到一个客户端请求时, 这个自定义的函数将会返回一个`RequestHandler`.



需要注意的是你写入配置数据中的函数名是静态函数, 这些函数返回值是`RequestHandler`, 他们将会在后边被使用.  为了解析配置数据中的参数例如`staticFiles` "documentRoot"等,这些函数接受当前配置数据中的k-v对, 对应到每一个特定的路由上.

同样重要的是你写的名字必选完全有效. 也就是说他必须包含你的swift 模块名, 任何内置的结构名字,例如struct,enum, 还有函数名本身. 这些需要通过"."分割开. 例如, 正如你所看到的静态文件handler展示为"PerfectHTTPServer.HTTPHandler.staticFiles". 它存在于"PerfectHTTPServer"内部的一个"HTTPServer"扩展中, 名字叫做"staticFiles".



注意如果直接通过字典的方式创建你的配置到代码中, 那么你不必用双引号引起来函数名. "handler"的值可以直接引用.



下面例子中是一个`RequestHandler`生成器:

```objective-c

public extension HTTPHandler {
    public static func staticFiles(data: [String:Any]) throws -> RequestHandler {
        let documentRoot = data["documentRoot"] as? String ?? "./webroot"
        let allowResponseFilters = data["allowResponseFilters"] as? Bool ?? false
        return {
            req, resp in
            StaticFileHandler(documentRoot: documentRoot, allowResponseFilters: allowResponseFilters)
                .handleRequest(request: req, response: resp)
        }
    }
}
```



注意: 结构体`HTTPHandler`是一个定义在PerfectHTTPServer中的抽象命名空间. 他由静态request handler generator组成,就像上面例子中的那样.

Request handler generators 需要的配置数据用户可能没有提供,或者提供的数据无效,所以,建议采用`throw`写法. 注意确保你抛出异常的描述尽量浅显易懂些. 如果生成器不能返回一个有效的`RequestHandler`,那么他应该抛出一个异常.



**filters**:







