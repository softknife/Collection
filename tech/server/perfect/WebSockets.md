# WebSockets

WebSockets通过单个TCP连接设计为全双工通信通道(full-duplex communication channels)。 WebSocket protocol 便于从服务器进行实时数据传输. 这可以通过提供标准化的方式使服务器将内容发送到浏览器而不被客户端请求，并允许在保持连接打开的同时传递消息。  以这种方式，可以在浏览器和服务器之间进行双向（双向）对话。 这种会话通过典型的TCP 端口,例如80 or 443完成.



WebSocket protocol 当前被大多数主流浏览器支持,包括Google Chrome , Microsoft Edge, Internet Explorer, Firefox, Safari 和Opera. WebSockets 还需要服务端的应用支持.

## Relevant Examples

- [Perfect-Chat-Demo](https://github.com/PerfectExamples/Perfect-Chat-Demo)
- [Perfect-WebSocketsServer](https://github.com/PerfectExamples/Perfect-WebSocketsServer)



## Getting Started

添加WebSocket 依赖到你的Package.swift文件中:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-WebSockets.git", majorVersion: 2)
```



在你的代码中导入WebSocket library:

```swift
import PerfectWebSockets
```

一个典型的场景是网页之间的通信，如聊天室，多个用户正在近距离交互。



这个例子中,添加了一个与`/echo`路由可以交互的WebSocket service handler: 

```swift
var routes = Routes()
 
routes.add(method: .get, uri: "/echo", handler: {
        request, response in
    // Provide your closure which will return the service handler.
    WebSocketHandler(handlerProducer: {
        (request: HTTPRequest, protocols: [String]) -> WebSocketSessionHandler? in
 
        // Check to make sure the client is requesting our "echo" service.
        guard protocols.contains("echo") else {
            return nil
        }
 
        // Return our service handler.
        return EchoHandler()
    }).handleRequest(request: request, response: response)
    }
)
```



## Handling WebSocket Sessions

WebSocket service handler 必须实现`WebSocketSessionHandler`协议.

这个协议需要`handleSession(request:HTTPRequest, socket:WebSocket)` 函数. 一旦WebSocket 连接被建立,这个函数会被调用一次,  此时开始读取和写入messages是安全的. 

启动的会话提供了初始化HTTPRequest对象作为参考.

消息通过提供的WebSocket对象传输。

- Call `WebSocket.sendStringMessage` or `WebSocket.sendBinaryMessage` to send data to the client.
- Call `WebSocket.readStringMessage` or `WebSocket.readBinaryMessage` to read data from the client.

默认情况下，读取将无限期阻止，直到消息到达或发生网络错误。

读取超时可以通过`WebSocket.readTimeoutSeconds` 属性设置.

使用`WebSocket.close()`方法可以关闭session.

例子中的`EchoHandler`组成如下:

```swift
class EchoHandler: WebSocketSessionHandler {
 
    // The name of the super-protocol we implement.
    // This is optional, but it should match whatever the client-side WebSocket is initialized with.
    let socketProtocol: String? = "echo"
 
    // This function is called by the WebSocketHandler once the connection has been established.
    func handleSession(request: HTTPRequest, socket: WebSocket) {
 
        // Read a message from the client as a String.
        // Alternatively we could call `WebSocket.readBytesMessage` to get the data as an array of bytes.
        socket.readStringMessage {
            // This callback is provided:
            //  the received data
            //  the message's op-code
            //  a boolean indicating if the message is complete
            // (as opposed to fragmented)
            string, op, fin in
 
            // The data parameter might be nil here if either a timeout
            // or a network error, such as the client disconnecting, occurred.
            // By default there is no timeout.
            guard let string = string else {
                // This block will be executed if, for example, the browser window is closed.
                socket.close()
                return
            }
 
            // Print some information to the console for informational purposes.
            print("Read msg: \(string) op: \(op) fin: \(fin)")
 
            // Echo the data received back to the client.
            // Pass true for final. This will usually be the case, but WebSockets has
            // the concept of fragmented messages.
            // For example, if one were streaming a large file such as a video,
            // one would pass false for final.
            // This indicates to the receiver that there is more data to come in
            // subsequent messages but that all the data is part of the same logical message.
            // In such a scenario one would pass true for final only on the last bit of the video.
            socket.sendStringMessage(string, final: true) {
 
                // This callback is called once the message has been sent.
                // Recurse to read and echo new message.
                self.handleSession(request, socket: socket)
            }
        }
    }
}
```



## FastCGI Caveat

WebSockets服务仅支持独立的HTTP服务器。此时, WebSocket server 不能和Perfect FastCGI 连接器一起运行.



## WebSocket Class

### enum OpcodeType

WebSocket 消息可以是不同的类型: continuation, text, binary, close, ping or pong, or invalid types.

### var readTimeoutSeconds

当从当前socket中读取消息时, 这个属性可以帮助socket读取消息在超时之前. 如果这个属性设置为 NetEvent.noTimeout (-1), 他将会无限期等待下去.

### read message

读取消息有两种方式：文本或二进制，它们仅在返回的数据类型（即String和[UInt8]）中有所不同。

#### read text message:

```swift
public func readStringMessage(continuation: @escaping (String?, _ opcode: OpcodeType, _ final: Bool) -> ())
```



#### read binary message:

```swift
public func readBytesMessage(continuation: @escaping ([UInt8]?, _ opcode: OpcodeType, _ final: Bool) -> ())
```

callback中有三个参数.

- String / [UInt8]

readMessage 将会通过这个参数 呈递 从客户端发送到closure的文本/二进制数据.

- opcode

如果你想在会话交流中有更多的控制,使用opcode.

- final

这个参数指示消息是完整的还是碎片的.



### send message

​发送消息有两种方式: 文本或者二进制 , 它们仅在返回的数据类型（即String和[UInt8]）中有所不同。.

#### send text message:

```swift
public func sendStringMessage(string: String, final: Bool, completion: @escaping () -> ())
```



#### send binary message:

```swift
public func sendBinaryMessage(bytes: [UInt8], final: Bool, completion: @escaping () -> ())
```

这个参数final指示消息是完整的还是碎片的.



### ping & pong

Perfect WebSocket 也提供了便利的方式去测试链接情况. ping方法开始, 期望获得pong 回应.

Check out these two methods:

```swift
/// Send a "pong" message to the client.
public func sendPong(completion: @escaping () -> ())
 
/// Send a "ping" message to the client.
    /// Expect a "pong" message to follow.
public func sendPing(completion: @escaping () -> ())
```



### func close()

To close the WebSocket connection:

```swift

socket.close()

```



For a Perfect WebSockets server example, visit the [Perfect-WebSocketsServer](https://github.com/PerfectExamples/Perfect-WebSocketsServer) demo.