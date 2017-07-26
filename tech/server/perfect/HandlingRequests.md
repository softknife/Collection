# Handling Requests

作为一个网络服务器, Perfect的主函数是用来接受和相应客户端请求的. Perfect中有提供对象用以代表请求和相应组件, 你可以给他们添加handlers用以控制结果内容的生成.



第一件要做的事情就是生成server对象. 这个server对象是可配置,然后绑定到一个特定端口上,并且监听客户端的链接行为. 当一个链接事件发生时, 服务器可视读取request数据. 一旦request读完, 服务器将会将 [request object](http://www.perfect.org/docs/HTTPRequest.html) 传递给任何request filters.



这些filters运行收到的request被修改. 服务器将会使用request总的URI去 the [routing](http://www.perfect.org/docs/routing.html) system中查询合适的handler. 如果匹配到某个handler, 我们就可以在handler中去populate(填充) a [response object](http://www.perfect.org/docs/HTTPResponse.html).   一旦handler说明自己已经完成响应,响应对象将会被传递给response filters. 这些filters中,我们可以修改输出数据. 然后,结果数据将会被返回给客户端, 然后, the connection is either closed, or it can be reused as an HTTP persistent connection, a.k.a.(又叫做) HTTP keep-alive, for additional requests and responses.



参考下面章节,获取更多细节, 查看每一部分能完成那些事情:

- [Routing](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/Routing.md) - 描述路由系统, 展示如何添加URL handlers.  
- [HTTPRequest](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/HTTPRequest.md) -关于请求对象协议的更多细节.  
- [HTTPResponse](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/HTTPResponse.md) - 关于响应对象协议的更多细节.
- [Request & Response Filters](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/Request-ResponseFilters.md) - 展示如何添加filters,以及阐述如何使用它们. 


其他:
- [Session](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/Sessions.md)
- [Local Authentication modules](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/LocalAuthentication.md)
- [SPNEGO Security](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/SPNEGOSecurity.md)
- [JSON](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/JSON.md)


除此之外, 下面两个章节将会说明如何使用一些预制的具体handlers, 这些handlers是用来完成一些特定的任务的:

- [Static File Handler](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/StaticFileContext.md) - 描述如何呈递静态文件.
- [Mustache](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/Mustache.md) - 展示如何填充和使用Mustache模板.

- [Markdown](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/Markdown.md)
- [HTTPRequest Logging](https://github.com/EricYellow/Collection/blob/master/tech/server/perfect/HandlingRequests/HTTPRequestLogging.md)

