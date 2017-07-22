# An HTTP and Web Services Primer

我们已知的web服务, 或者说服务端开发中经常用到的技术, 基本上都是建立在为数不多的一些主要技术之上的.

这篇文章尝试着,对使用Perfect进行服务端应用开发 与 这些主要技术之间的关联做一个概述.



## What is an API ?

API是两个系统之间的桥梁. 他一般接受标准类型和风格的输入,然后,内部转换, 最终达到和另一个系统协同工作去完成一个行为 或者返回给另一个系统一些信息的目的.



我们以GitHub的API为例: 大家都熟悉GitHub的web界面, 但是, 他们也有一套API, 这些API允许一些应用(task and issue management systems like [Jira](https://www.atlassian.com/software/jira) and continuous integration (CI) systems such as [Bamboo](https://www.atlassian.com/software/bamboo)) 直接连接到你的repo, 然后为你提供更多的服务.  这比任何单一的系统都能做的更多,不是吗?  github直接提供的功能毕竟是有限的.

这些API往往都是建立在HTTP或者HTTPS上的.



### HTTP and HTTPS

HTTP是'HyperText Transfer Protocol'的缩写. 他是一系列标准的集合, 这可以允许用户通过网络发布富文本内容以及获取网页上的消息.



`HTTPS`是安全加密的HTTP协议. 交互的每一端都同意一个信任的key,往外输出信息之前,先用这个key对信息加密, 接收到的信息也用这个key进行解密. 安全措施越复杂,三方拦截/读取/变更这些传输信息越是困难.

当我们在浏览器重访问一个网站时,可能是通过HTTP协议, 如果网址左侧有个绿色的锁,那么就是通过HTTPS协议.

当iOS或者android访问后端服务器获取天气信息时,他们也是使用HTTP或者HTTPS协议.



## The General Form of an API

### Routes

API由一系列的`routes`组成, 他们就像文件系统中的directory/folder name paths. 每一个route 指向不同行为.  "展示所有用户"和"创建一个新用户"这两条指令对应的行为路由是不同的.

在API中,这些路由通常并不代表真实的文件夹或者文件, 仅仅是对一个行为的指向.  "directory structure"指的是这些行为动作的逻辑组织结构. 例如:



```objective-c
/api/v1/users/list
/api/v1/users/detail
/api/v1/users/create
/api/v1/users/modify
/api/v1/users/delete

/api/v1/companies/list
/api/v1/companies/detail
/api/v1/companies/create
/api/v1/companies/modify
/api/v1/companies/delete
```



这个例子阐释了经典的"CRUD"系统. "CRUD"指"Create,Read,Update,Delete".  上面两组路由不同点仅仅在于`users`vs`componies`部分, 但是实际上他们分别指向完全不同的行为.

此外, "detail","modify","delete"路由必须包括一个唯一标识符.例如:

```objective-c
/api/v1/users/list
/api/v1/users/detail?id=5d096846-a000-43db-b6c5-a5883135d71d
/api/v1/users/create
/api/v1/users/modify?id=5d096846-a000-43db-b6c5-a5883135d71d
/api/v1/users/delete?id=5d096846-a000-43db-b6c5-a5883135d71d
```



### HTTP Verbs

HTTP 谓词 是一个浏览器或者手机客户端进行一次请求时提供给服务器路由的附加信息. 这些谓词可以告诉API 服务器该路由指向行为的上下文, 上下文中说明了行为特性.



常用的HTTP谓词包括:

#### GET

GET请求中参数直接放在URL中. GET请求通常是用来获取信息没有其他作用. 使用GET请求删除数据库记录技术上是可行的,但是我们并不推荐.

#### POST

POST请求通常用来在数据库中创建新的记录, POST请求通常用来提交表单. K-V对信息放到`POST body`中被提交到API服务器, 服务器解析K-V对以读取信息.

#### PATCH

PATCH请求一般被认为和POST请求,实际上它是用来更新数据的.

#### PUT

PUT请求主要用来上传文件, PUT请求body中包括文件信息.

#### DELETE

DELETE请求就是字面意思.当你发送一个DELETE请求时,就是命令API服务器删除一个具体的资源. 通常一些表单请求中唯一标识符直接放在URL中.



#### Using HTTP Verbs and Routes to Simplify an API

当把谓词和路由一起使用时,可以降低API感官上的复杂度.

我们来看下面的HTTP请求的结构, 当谓词和路由一起使用时, 整个处理过程都变得更简单,更具体:

```objective-c
GET     /api/v1/users
GET     /api/v1/users/5d096846-a000-43db-b6c5-a5883135d71d
POST    /api/v1/users
PATCH   /api/v1/users/5d096846-a000-43db-b6c5-a5883135d71d
DELETE  /api/v1/users/5d096846-a000-43db-b6c5-a5883135d71d
```



猛一看, 似乎所有的请求都指向同一个路由: `/api/v1/users`. 实际上,以上每个路由都指向不同的行为. 同样地, 头两个GET路由的区别在于第二个路由中id直接拼到URL中, 并不是像之前的例子中的那样作为参数放到body中.  这种命令的实现需要在初始化路由时设置通配符(wildcard)来完成.

下一步就是查看Handling Requests章节的内容. 这一部分将会说明如何实现指向特定函数或者方法的路由, 如何在服务端访问前端传递过来的信息, 判断是浏览器还是手机客户端.







