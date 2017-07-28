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

