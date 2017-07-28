# Sessions

Perfect 涵盖了Redis, PostgreSQL, MySQL, SQLite3 and CouchDB 服务的session 驱动, 还有开发期的in-memory session storage. 支持MongoDB和Redis正在计划中.



Session 管理对于web或者app环境是一项基础功能,并且可以提供与认证，过渡偏好存储和事务数据（例如传统购物车）的链接。



一般原则是当用户使用浏览器访问一个网站或者系统时, 服务器会给用户分配一个"token"或者"session id", 并且以cookie或者JSON data的形式将这个value传递给客户端或者浏览器. 这个token将会以cookie或者Bearer token的形式伴随与每一个随后的请求.



Sessions有一个失效时间, 通常是以"idle timeout"的形式存在. 意思是如果session 几秒钟还没有被激活, 那么这个session被认为是过期无效的 .

Perfect sessions 记录创建日期和时间,以及最后一次"touched"的时间, 还有idle time. 每次客户端或者浏览器访问时, session 被"touched",idle time重置.  如果"last touched" 加上 "idle time"小于当前时间,那么我们认为session 过期.



每一个session有如下属性:

- **token** - the session id
- **userid** - an optionally stored user id string
- **created** - an integer representing the date/time created, in seconds
- **updated** - an integer representing the date/time last touched, in seconds
- **idle** - an integer representing the number of seconds the session can be idle before being considered expired
- **data** - a [String:Any] Array that is converted to JSON for storage. This is intended for storage of simple preference values.
- **ipaddress** - the IP Address (v4 or v6) that the session was first used on. Used for optional session verification.
- **useragent** - the User Agent string that the session was first used with. Used for optional session verification.
- **CSRF** - the [CSRF (Cross Site Request Forgery)](http://www.perfect.org/docs/csrf.html) security configuration.
- **CORS** - the [CORS (Cross Origin Resource Sharing)](http://www.perfect.org/docs/cors.html) security configuration.



## Example

每一个模块有自己的demo. 本文档中描述的大部分功能可以在每个示例中进行观察。



- [In-Memory Sessions](https://github.com/PerfectExamples/Perfect-Session-Memory-Demo)
- [Redis Sessions](https://github.com/PerfectExamples/Perfect-Session-Redis-Demo)
- [PostgreSQL Sessions](https://github.com/PerfectExamples/Perfect-Session-PostgreSQL-Demo)
- [MySQL Sessions](https://github.com/PerfectExamples/Perfect-Session-MySQL-Demo)
- [SQLite Sessions](https://github.com/PerfectExamples/Perfect-Session-SQLite-Demo)
- [CouchDB Sessions](https://github.com/PerfectExamples/Perfect-Session-CouchDB-Demo)
- [MongoDB Sessions](https://github.com/PerfectExamples/Perfect-Session-MongoDB-Demo)



## Installation

如果使用in-memory 驱动, 导入这个基础模块通过在Package.swift中引入如下代码:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session.git", majorVersion: 1)

```



### Database-Specific Drivers

Redis:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-Redis.git", majorVersion: 1)
```



PostgreSQL:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-PostgreSQL.git", majorVersion: 1)
```



MySQL:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-MySQL.git", majorVersion: 1)

```



SQLite3:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-SQLite.git", majorVersion: 1)
```



CouchDB:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-CouchDB.git", majorVersion: 1)

```



MongoDB:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-MongoDB.git", majorVersion: 1)

```



## Configuration

`SessionConfig`这个结构体中包含可以根据你的喜好进行自定义的设置:

```swift
// The name of the session. 
// This will also be the name of the cookie set in a browser.
SessionConfig.name = "PerfectSession"
 
// The "Idle" time for the session, in seconds.
// 86400 is one day.
SessionConfig.idle = 86400
 
// Optional cookie domain setting
SessionConfig.cookieDomain = "localhost"
 
// Optional setting to lock session to the initiating IP address. Default is false
SessionConfig.IPAddressLock = true
 
// Optional setting to lock session to the initiating user agent string. Default is false
SessionConfig.userAgentLock = true
 
// The interval at which stale sessions are purged from the database
SessionConfig.purgeInterval = 3600 // in seconds. Default is 1 hour.
 
// CouchDB-Specific
// The CouchDB database used to store sessions
SessionConfig.couchDatabase = "sessions"
 
// MongoDB-Specific
// The MongoDB collection used to store sessions
SessionConfig.mongoCollection = "sessions"
```



如果你想改变SessionConfig的值, 必须在Session Driver定义前设置这些值.



The [**CSRF** (Cross Site Request Forgery)](http://www.perfect.org/docs/csrf.html) security configuration and [**CORS** (Cross Origin Resource Sharing)](http://www.perfect.org/docs/cors.html) security configuration are discussed separately.



注意: 对于Redis驱动的session, 并没有通过`SessionConfig.purgeInterval`设置进行的`timed event`, 因为此时的过期机制可以直接通过`Expires`值(在session 创建或者更新时会被添加)来控制.



### IP Address and User Agent Locks

如果`SessionConfig.IPAddressLock`或者`SessionConfig.userAgentLock`设置为true, 那么,如果当session初始化后,每次用户请求的信息不能匹配session中的ipAddress或者userAgent, 这个session将会被强制终止.



这种安全措施有助于阻止中间人攻击 —  man-in-the-middle / session hijacking attacks.



注意, 当我们将`SessionConfig.IPAddressLock`设置为`true`时, 如果用户通过wifi网络登录,然后又切换到蜂窝网或者有线网络时, 将会退出session登录.



### Defining the Session Driver

每一种Session Driver都有自己的实现,这是针对存储选项的优化. 因此, 你必须在`server.start()`之前,设置好session driver 和 HTTP filters.



在main.swift文件中, SessionConfig改变后, 进行如下设置:

```swift
// Instantiate the HTTPServer
let server = HTTPServer()
 
// Define the Session Driver
let sessionDriver = SessionMemoryDriver()
 
// Add the filters so the pre- and post- route actions are executed.
server.setRequestFilters([sessionDriver.requestFilter])
server.setResponseFilters([sessionDriver.responseFilter])
```

在Perfect中, filters在某种形式上类似于其他框架中的中间件的概念. `request`filter将会截获到来的request ,解析其中的session token, 尝试着从storage中加载session, 将session data 应用于application. 在Response发送给客户端或者浏览器之前, 他将会被执行, 保存session,并且重新发送cookies 给客户端或者浏览器.



有关存储适当的设置和驱动程序语法，请参阅下面的“数据库特定选项”。



### Accessing the Session Data



Session的`token`,`userid`和`data`属性暴露在`request`对象中.我们可以在handler中通过request来读取&写入 `userid`,`data`属性, 并且自动保存到response filter的storage中.



在列子中,我们可以看到一个简单的handler样例.

```swift
// Defining an "Index" handler
open static func indexHandlerGet(request: HTTPRequest, _ response: HTTPResponse) {
    // Random generator from TurnstileCrypto
    let rand = URandom()
 
    // Adding some random data to the session for demo purposes
    request.session.data[rand.secureToken] = rand.secureToken
 
    // For demo purposes, dumping all current session data as a JSON string.
    let dump = try? request.session.data.jsonEncodedString()
 
    // Some simple HTML that displays the session token/id, and the current data as JSON.
    let body = "<p>Your Session ID is: <code>\(request.session.token)</code></p><p>Session data: <code>\(dump)</code></p>"
 
    // Send the response back.
    response.setBody(string: body)
    response.completed()
}
```



读取当前session id:

```swift
request.session.token
```

访问userid:

```swift
// read:
request.session.userid
 
// write:
request.session.userid = "MyString"
```

访问存储在session中的数据:

```swift
// read:
request.session.data
 
// write:
request.session.data["keyString"] = "Value"
request.session.data["keyInteger"] = 1
request.session.data["keyBool"] = true
 
// reading a specific value
if let val = request.session.data["keyString"] as? String {
    let keyString = val
}
```



### Database-Specific options

#### Redis

Importing the module, in Package.swift:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-Redis.git", majorVersion: 1)
```



Defining the connection to the PostgreSQL server:

```swift
RedisSessionConnector.host = "localhost"
RedisSessionConnector.port = 5432
RedisSessionConnector.password = "secret"
```



Defining the Session Driver:

```swift
let sessionDriver = SessionRedisDriver()
```



#### PostgreSQL

Importing the module, in Package.swift:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-PostgreSQL.git", majorVersion: 1)
```

Defining the connection to the PostgreSQL server:

```swift
PostgresSessionConnector.host = "localhost"
PostgresSessionConnector.port = 5432
PostgresSessionConnector.username = "username"
PostgresSessionConnector.password = "secret"
PostgresSessionConnector.database = "mydatabase"
PostgresSessionConnector.table = "sessions"
```



Defining the Session Driver:

```swift

let sessionDriver = SessionPostgresDriver()

```



#### MySQL

Importing the module, in Package.swift:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-MySQL.git", majorVersion: 1)
```



Defining the connection to the MySQL server:

```swift
MySQLSessionConnector.host = "localhost"
MySQLSessionConnector.port = 3306
MySQLSessionConnector.username = "username"
MySQLSessionConnector.password = "secret"
MySQLSessionConnector.database = "mydatabase"
MySQLSessionConnector.table = "sessions"
```



Defining the Session Driver:

```swift
let sessionDriver = SessionMySQLDriver()
```



#### SQLite

Importing the module, in Package.swift:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-SQLite.git", majorVersion: 1)
```



Defining the connection to the SQLite server:

```swift
SQLiteConnector.db = "./SessionDB"
```



Defining the Session Driver:

```swift
let sessionDriver = SessionSQLiteDriver()
```



#### CouchDB

Importing the module, in Package.swift:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-CouchDB.git", majorVersion: 1)
```



Defining the CouchDB database to use for session storage:

```swift
SessionConfig.couchDatabase = "perfectsessions"
```



Defining the connection to the CouchDB server:

```swift
CouchDBConnection.host = "localhost"
CouchDBConnection.username = "username"
CouchDBConnection.password = "secret"
```



Defining the Session Driver:

```swift
let sessionDriver = SessionCouchDBDriver()
```



#### MongoDB

Importing the module, in Package.swift:

```swift
.Package(url:"https://github.com/PerfectlySoft/Perfect-Session-MongoDB.git", majorVersion: 1)
```



Defining the CouchDB database to use for session storage:

```swift
SessionConfig.mongoCollection = "perfectsessions"
```



Defining the connection to the CouchDB server:

```swift
MongoDBConnection.host = "localhost"
MongoDBConnection.database = "perfect_testing"
```



Defining the Session Driver:

```swift
let sessionDriver = SessionMongoDBDriver()
```





*****



## CSRF (Cross Site Request Forgery) Security

>  跨站点请求伪造



CSRF是一种攻击行为, 他会使web应用的终端用户(他们已经被授权了)强制执行unwanted actions. **CSRF attacks specially target state-changing requests, not theft of data, since the attacker has no way to see the response to the forged request**(CSRF攻击专门针对状态改变的请求，而不是窃取数据，因为攻击者无法看到对伪造请求的响应。).

社交工程师通过一点辅助技巧(例如通过邮箱或者聊天发送一个链接), 攻击者就可以哄骗web app用户去执行他所希望的行为.  如果受害者是普通用户, 成功的CSRF攻击可以迫使用户执行状态改变请求,例如转移资金,修改email地址, 等等诸如此类行为. 如果受害者是管理员账号, CSRF攻击可以损坏整个web app(OWASP).



CSRF as an attack vector is often overlooked, and represents a significant "chaos" factor unless the validation is handled at the highest level: the framework.(CSRF作为攻击媒介往往被忽视，除非验证在最高级别处理：框架, 否则CSRF将会成为一个重大的“混乱”因素。)  这样可以允许web app 和API 作者对安全层有一个强有力的控制.



The [Perfect Sessions](http://www.perfect.org/docs/sessions.html) 模块支持 CSRF 配置.

If you have included Perfect Sessions or any of its datasource-specific implementations in your Packages.swift file, you already have CSRF support.



### Relevant Examples

- [Perfect-Session-Memory-Demo](https://github.com/PerfectExamples/Perfect-Session-Memory-Demo)



### Configuration

配置CSRF的例子可能像如下形式:

```swift
SessionConfig.CSRF.checkState = true
SessionConfig.CSRF.failAction = .fail
SessionConfig.CSRF.checkHeaders = true
SessionConfig.CSRF.acceptableHostnames.append("http://www.example.com")
SessionConfig.CSRF.requireToken = true
```



#### SessionConfig.CSRF.checkState

这个是最主要的开关,如果设置为true,CSRF 将会对所有routes起作用.

#### SessionConfig.CSRF.failAction

这个参数是用来说明如果CSRF验证失败后,后续采取什么动作的. 可能的选择如下:

- `.fail` - Execute an immediate halt. No further processing will be done, and an HTTP Status `406 Not Acceptable` is generated.
- `.log` - Processing will continue, however the event will be recorded in the log.
- `.none` - Processing will continue, no action is taken.

#### SessionConfig.CSRF.acceptableHostnames

这个Host names 数组将会在后续使用, 用来对`origin`匹配验收进行比较.



#### SessionConfig.CSRF.checkHeaders

如果`CSRF.checkheader`配置为`true`, origin和host headers 将会验证有效性.

- The `Origin`, `Referrer` or `X-Forwarded-For` headers must be populated ("origin").
- If the "origin" is specified in `SessionConfig.CSRF.acceptableHostnames`, the CSRF check will continue to the next phase and the following checks are skipped.
- The `Host` or `X-Forwarded-Host` header must be present ("host").
- The "host" and "origin" values must match exactly.



#### SessionConfig.CSRF.requireToken

当设置为true时, 将会强制所有的POST请求都带有"_csrf"参数, 或者, 如果content-type 是"application/json",那么request的header中必须绑定"X-CSRF-Token". header中的内容或者参数应该匹配`request.session.data["csrf"]`的值. 这个值会在session start时自动设置好.



#### Session state

虽然不是配置参数，但值得注意的是，如果`SessionConfig.CSRF.checkState`为true，如果会话为“new”，则不接受POST请求。 这是安全建议支持的故意立场。





OWASP, Cross-Site Request Forgery (CSRF): [https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))





*****

## CORS (Cross Origin Resource Sharing) Security

> 跨原始资源共享

CORS是"open web"中的一个很重要的部分,  因此, 如果不支持CORS,框架就不是完整的.

Monsur Hossain from [html5rocks.com introduces CORS very effectively](https://www.html5rocks.com/en/tutorials/cors/):

> APIs are the threads that let you stitch together a rich web experience. But this experience has a hard time translating to the browser, where the options for cross-domain requests are limited to techniques like JSON-P (which has limited use due to security concerns) or setting up a custom proxy (which can be a pain to set up and maintain).
>
> Cross-Origin Resource Sharing (CORS) is a W3C spec that allows cross-domain communication from the browser. By building on top of the XMLHttpRequest object, CORS allows developers to work with the same idioms as same-domain requests.
>
> The use-case for CORS is simple. Imagine the site alice.com has some data that the site bob.com wants to access. This type of request traditionally wouldn’t be allowed under the browser’s same origin policy. However, by supporting CORS requests, alice.com can add a few special response headers that allows bob.com to access the data.
>
> As you can see from this example, CORS support requires coordination between both the server and client. Luckily, if you are a client-side developer you are shielded from most of these details. The rest of this article shows how clients can make cross-origin requests, and how servers can configure themselves to support CORS.



The [Perfect Sessions](http://www.perfect.org/docs/sessions.html) 模块支持  CORS configuration, 这样你就可以随心的让你的API和assets可用或者被包含起来.

如果你已经导入了Sessions模块或者任何数据库实现的Sessions, 你已经支持CORS; 然而,默认他是关闭的 .



### Relevant Examples

- [Perfect-Session-Memory-Demo](https://github.com/PerfectExamples/Perfect-Session-Memory-Demo)

### Configuration



```swift
// Enabled, true or false.
// Default is false.
SessionConfig.CORS.enabled = true
 
// Array of acceptable hostnames for incoming requests
// To enable CORS on all, have a single entry, *
SessionConfig.CORS.acceptableHostnames = ["*"]
 
// However if you wish to enable specific domains:
SessionConfig.CORS.acceptableHostnames.append("http://www.test-cors.org")
 
// Wildcards can also be used at the start or end of hosts
SessionConfig.CORS.acceptableHostnames.append("*.example.com")
SessionConfig.CORS.acceptableHostnames.append("http://www.domain.*")
 
// Array of acceptable methods
public var methods: [HTTPMethod] = [.get, .post, .put]
 
// An array of custom headers allowed
public var customHeaders = [String]()
 
// Access-Control-Allow-Credentials true/false.
// Standard CORS requests do not send or set any cookies by default. 
// In order to include cookies as part of the request enable the client to do so by setting to true
public var withCredentials = false
 
// Max Age (seconds) of request / OPTION caching.
// Set to 0 for no caching (default)
public var maxAge = 3600
```



当一个CORS请求提交到服务器时, 如果没有匹配到,那么CORS headers不会在OPTIONs response中产生,  这将指示浏览器不能接受该资源.

```swift
// An array of allowable HTTP Methods.
// In the case of the above configuration example:
Access-Control-Allow-Methods: GET, POST, PUT
 
// If the origin is acceptable the origin will be echoed back to the requester 
// (even if configured with *)
Access-Control-Allow-Origin: http://www.test-cors.org
 
// If the server wishes cookies to be sent along with requests, 
// this should return true
Access-Control-Allow-Credentials: true
 
// The length of time the OPTIONS request can be cached for.
Access-Control-Max-Age: 3600
```



An excellent resource for testing CORS entitlements and responses is available at [http://www.test-cors.org](http://www.test-cors.org/)

