# HTTP Request Logging

为了将HTTP Requests打印到文件中, 使用`Perfect-RequestLogger`模块.

## Relevant Examples

- [Perfect-HTTPRequestLogging](https://github.com/PerfectExamples/Perfect-HTTPRequestLogging)
- [Perfect-Session-Memory-Demo](https://github.com/PerfectExamples/Perfect-Session-Memory-Demo)



## Usage

添加如下内容到`Package.swift`文件中:

```swift
.Package(url: "https://github.com/PerfectlySoft/Perfect-RequestLogger.git", majorVersion: 1)

```



在你需要使用log功能的文件中, 导入模块:

```swift
import PerfectRequestLogger
```



## When using PerfectHTTP 2.1 or later

初始化`server`后, 添加如下代码到`main.swift`  文件中:

```swift
// Instantiate a logger
let httplogger = RequestLogger()
 
// Configure Server
var confData: [String:[[String:Any]]] = [
    "servers": [
        [
            "name":"localhost",
            "port":8181,
            "routes":[],
            "filters":[
                [
                    "type":"response",
                    "priority":"high",
                    "name":PerfectHTTPServer.HTTPFilter.contentCompression,
                    ],
                [
                    "type":"request",
                    "priority":"high",
                    "name":RequestLogger.filterAPIRequest,
                    ],
                [
                    "type":"response",
                    "priority":"low",
                    "name":RequestLogger.filterAPIResponse,
                    ]
            ]
        ]
    ]
]
```



配置文件中很重要的一部分是enable the Request Logger :

```swift
[
    "type":"request",
    "priority":"high",
    "name":RequestLogger.filterAPIRequest,
    ],
[
    "type":"response",
    "priority":"low",
    "name":RequestLogger.filterAPIResponse,
]
```



这些Request & response filters 添加必要的hooks 去标记 HTTP request and response 的 beginning 和 completion.



## When using PerfectHTTP 2.0

初始化`server`后,添加如下代码到`main.swift`中:

```swift
// Instantiate a logger
let httplogger = RequestLogger()
 
// Add the filters
// Request filter at high priority to be executed first
server.setRequestFilters([(httplogger, .high)])
// Response filter at low priority to be executed last
server.setResponseFilters([(httplogger, .low)])
```

These request & response filters add the required hooks to mark the beginning and the completion of the HTTP request and response.



## Setting a custom Logfile location

默认的logfile location 是`/var/log/prefectLog.log`. 为了设置自定义的logfile 路径, 配置`RequestLogFile.location` 属性:

```swift
RequestLogFile.location = "/var/log/myLog.log"
```



## Example Log Output

```swift
[INFO] [62f940aa-f204-43ed-9934-166896eda21c] [servername/WuAyNIIU-1] 2016-10-07 21:49:04 +0000 "GET /one HTTP/1.1" from 127.0.0.1 - 200 64B in 0.000436007976531982s
[INFO] [ec6a9ca5-00b1-4656-9e4c-ddecae8dde02] [servername/WuAyNIIU-2] 2016-10-07 21:49:06 +0000 "GET /two HTTP/1.1" from 127.0.0.1 - 200 64B in 0.000207006931304932s

```

This module expands on earlier work by [David Fleming](https://github.com/dabfleming).



