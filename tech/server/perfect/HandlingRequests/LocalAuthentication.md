# Perfect Local Authentication

对于那些需要本地化存储和处理身份验证的项目,这个包可以提供基于本地的身份验证库.



## Relevant Templates & Examples

模板app 链接在这: [https://github.com/PerfectlySoft/Perfect-Local-Auth-PostgreSQL-Template](https://github.com/PerfectlySoft/Perfect-Local-Auth-PostgreSQL-Template), 提供了一个全功能的起点, 和演示系统的使用.



## Adding to your project

将这个项目作为依赖添加到你的Package.swift文件中.

PostgreSQL Driver:

```swift
.Package(url: "https://github.com/PerfectlySoft/Perfect-LocalAuthentication-PostgreSQL.git", majorVersion: 1)
```



MySQL Driver:

```swift
.Package(url: "https://github.com/PerfectlySoft/Perfect-LocalAuthentication-MySQL.git", majorVersion: 1)
```



项目中使用该模块:

```swift
import LocalAuthentication
```



## Configuration

为了安装database和session configuration,在main.swift文件中进行如下配置是很重要的:



导入需要的模块:

```swift
import PerfectSession
import PerfectSessionPostgreSQL // Or PerfectSessionMySQL
import PerfectCrypto
import LocalAuthentication
```

初始化PerfectCrypto:

```swift
let _ = PerfectCrypto.isInitialized
```



现在设置一些默认值:

```swift
// Used in email communications
// The Base link to your system, such as http://www.example.com/
var baseURL = ""
 
// Configuration of Session
SessionConfig.name = "perfectSession" // <-- change
SessionConfig.idle = 86400
SessionConfig.cookieDomain = "localhost" //<-- change
SessionConfig.IPAddressLock = false
SessionConfig.userAgentLock = false
SessionConfig.CSRF.checkState = true
SessionConfig.CORS.enabled = true
SessionConfig.cookieSameSite = .lax
```



Session 配置的详情文章请参考:[https://www.perfect.org/docs/sessions.html](https://www.perfect.org/docs/sessions.html) .



Database 和 email 配置应该设置如下(如果使用JSON文件进行配置):

```swift
let opts = initializeSchema("./config/ApplicationConfiguration.json") // <-- loads base config like db and email configuration
httpPort = opts["httpPort"] as? Int ?? httpPort
baseURL = opts["baseURL"] as? String ?? baseURL
```



否则, 需要设置等同如下功能之一:

- PostgreSQL: [https://github.com/PerfectlySoft/Perfect-LocalAuthentication-PostgreSQL/blob/master/Sources/LocalAuthentication/Schema/InitializeSchema.swift](https://github.com/PerfectlySoft/Perfect-LocalAuthentication-PostgreSQL/blob/master/Sources/LocalAuthentication/Schema/InitializeSchema.swift)
- MySQL: [https://github.com/PerfectlySoft/Perfect-LocalAuthentication-MySQL/blob/master/Sources/LocalAuthentication/Schema/InitializeSchema.swift](https://github.com/PerfectlySoft/Perfect-LocalAuthentication-MySQL/blob/master/Sources/LocalAuthentication/Schema/InitializeSchema.swift)



### Set the session driver

PostgreSQL:

```swift
let sessionDriver = SessionPostgresDriver()
```



MySQL:

```swift
let sessionDriver = SessionMySQLDriver()
```



### Request & Response Filters

下面这两个session filters需要添加到server配置中:

PostgreSQL Driver:

```swift
// (where filter is a [[String: Any]] object)
filters.append(["type":"request","priority":"high","name":SessionPostgresFilter.filterAPIRequest])
filters.append(["type":"response","priority":"high","name":SessionPostgresFilter.filterAPIResponse])
```



MySQL Driver:

```swift
// (where filter is a [[String: Any]] object)
filters.append(["type":"request","priority":"high","name":SessionMySQLFilter.filterAPIRequest])
filters.append(["type":"response","priority":"high","name":SessionMySQLFilter.filterAPIResponse])
```



例子可以参考: [https://github.com/PerfectlySoft/Perfect-Local-Auth-PostgreSQL-Template/blob/master/Sources/PerfectLocalAuthPostgreSQLTemplate/configuration/Filters.swift](https://github.com/PerfectlySoft/Perfect-Local-Auth-PostgreSQL-Template/blob/master/Sources/PerfectLocalAuthPostgreSQLTemplate/configuration/Filters.swift)



### Add routes for login, register etc

可以根据需要或者定制添加下面的路由, 以登录,登出,注册为例:

```swift
// Login
routes.append(["method":"get", "uri":"/login", "handler":Handlers.login]) // simply a serving of the login GET
routes.append(["method":"post", "uri":"/login", "handler":LocalAuthWebHandlers.login])
routes.append(["method":"get", "uri":"/logout", "handler":LocalAuthWebHandlers.logout])
 
// Register
routes.append(["method":"get", "uri":"/register", "handler":LocalAuthWebHandlers.register])
routes.append(["method":"post", "uri":"/register", "handler":LocalAuthWebHandlers.registerPost])
routes.append(["method":"get", "uri":"/verifyAccount/{passvalidation}", "handler":LocalAuthWebHandlers.registerVerify])
routes.append(["method":"post", "uri":"/registrationCompletion", "handler":LocalAuthWebHandlers.registerCompletion])
 
// JSON
routes.append(["method":"get", "uri":"/api/v1/session", "handler":LocalAuthJSONHandlers.session])
routes.append(["method":"get", "uri":"/api/v1/logout", "handler":LocalAuthJSONHandlers.logout])
routes.append(["method":"post", "uri":"/api/v1/register", "handler":LocalAuthJSONHandlers.register])
routes.append(["method":"login", "uri":"/api/v1/login", "handler":LocalAuthJSONHandlers.login])
```



例子可以在 [https://github.com/PerfectlySoft/Perfect-Local-Auth-PostgreSQL-Template/blob/master/Sources/PerfectLocalAuthPostgreSQLTemplate/configuration/Routes.swift](https://github.com/PerfectlySoft/Perfect-Local-Auth-PostgreSQL-Template/blob/master/Sources/PerfectLocalAuthPostgreSQLTemplate/configuration/Routes.swift) 地址找到.



## Testing for authentication:

user id 可以通过如下方式访问:

```swift
request.session?.userid ?? ""
```



如果访问一页内容时,user id(登录状态下)是必须的, 如下代码可以用来检测和重定向:

```swift
let contextAuthenticated = !(request.session?.userid ?? "").isEmpty
if !contextAuthenticated { response.redirect(path: "/login") }
​```
```

