# Building with Swift Package Manager



[Swift Package Manager](https://swift.org/package-manager/) (SPM)是一个Swift项目的building,testing,and managing dependencies 命令行工具. 所有的Perfect组件都是使用SPM构建的. 如果你使用Perfect构建项目,那么需要使用SPM.



你可以使用 [PerfectTemplate](https://github.com/PerfectlySoft/PerfectTemplate) 模板快速构建Perfect项目. 这个模板仅仅能显示一个"Hello ,World",当然,你可以再编辑修改.



在开始前,确保你已经阅读了 [dependencies](https://github.com/PerfectlySoft/Perfect/wiki/Dependencies) document, 你有Swift3.0工具链环境.



下一步是克隆模板项目. 打开一个新的命令行,cd到新项目文件目录下,然后,克隆模板项目. 输入如下命令,开始克隆项目:

```swift
git clone https://github.com/PerfectlySoft/PerfectTemplate.git
```



在Perfect 模板下,你将会发现两个重要的文件项目:

- A *Sources* directory containing all of the Swift source files for Perfect
- An SPM manifest named “*Package.swift*” listing the dependencies this project requires

所有的SPM项目至少有Sources文件夹和Package.swift文件两个项目. 这个项目只有一个依赖项目: [Perfect-HTTPServer](https://github.com/PerfectlySoft/Perfect-HTTPServer) 



### Dependencies in Package.swift

The PerfectTemplate *Package.swift* manifest file 包含如下内容:

```swift

import PackageDescription

let versions = Version(0,0,0)..<Version(10,0,0)  
  
let package = Package(
    name: "PerfectTemplate",
    targets: [],
    dependencies: [
        .Package(url: "https://github.com/PerfectlySoft/Perfect-HTTPServer.git",
            versions: versions)
    ]
)
```



**Note**: 上面的展示的版本和你在终端中的版本不同. 我们建议你参考github上最新的版本.



`Package.swift`文件中有两个主要的元素你可能需要修改.

第一个是`name`. 他表示项目的名字,  当项目build时, 会根据这个name构建出来可执行的项目.



第二个是`dependencies`列表. 他表示你的项目依赖的所有子项目. 数组中的每个元素都由".Package"和依赖库的URL以及版本组成.

上面的例子表明,使用一个宽泛的版本, 这样这个模板项目就能总是使用最新版本的HTTPServer项目. 如果你想把你的项目依赖限制在一个固定的版本. 例如, 如果你想将你的项目依赖HTTPServer固定在version2上, 你的".Package"元素像如下样子:

`.Package(url: "https://github.com/PerfectlySoft/Perfect-HTTPServer.git", majorVersion: 2)`

随着你的项目推进, 你添加更多依赖, 你将要把这些依赖添加到依赖列表中.SPM将会自动下载适合的版本,并编译到你的项目中.所有的依赖都下载到Packages(SPM自动创建的)文件件下. 例如, 如果你想使用Mustache模板, 你的Package.swift文件可能像下面这样:

```swift
import PackageDescription
 
let package = Package(
    name: "PerfectTemplate",
    targets: [],
    dependencies: [
        .Package(url: "https://github.com/PerfectlySoft/Perfect-HTTPServer.git",
            majorVersion: 2),
        .Package(url: "https://github.com/PerfectlySoft/Perfect-Mustache.git",
            majorVersion: 2)
    ]
)
```



就像你看到的那样, the [Perfect-Mustache](https://github.com/PerfectlySoft/Perfect-Mustache) project作为一个依赖添加到你项目中. 这个依赖可以给你的项目提供Mustache 模板支持. 在你的项目代码中, 现在你就可以导入 `PerfectMustache`, 然后,你就可以使用他提供给你的工具了.

随着你的依赖表增加, 你可能想要个性化管理这个依赖表. 下面的例子中包含了所有的Perfect库. URLs列表单独管理, 然后我们可以把它map成需要的格式.



```swift

import PackageDescription
 
let versions = majorVersion: 2
let urls = [
    "https://github.com/PerfectlySoft/Perfect-HTTPServer.git",
    "https://github.com/PerfectlySoft/Perfect-FastCGI.git",
    "https://github.com/PerfectlySoft/Perfect-CURL.git",
    "https://github.com/PerfectlySoft/Perfect-PostgreSQL.git",
    "https://github.com/PerfectlySoft/Perfect-SQLite.git",
    "https://github.com/PerfectlySoft/Perfect-Redis.git",
    "https://github.com/PerfectlySoft/Perfect-MySQL.git",
    "https://github.com/PerfectlySoft/Perfect-MongoDB.git",
    "https://github.com/PerfectlySoft/Perfect-WebSockets.git",
    "https://github.com/PerfectlySoft/Perfect-Notifications.git",
    "https://github.com/PerfectlySoft/Perfect-Mustache.git"
]
 
let package = Package(
    name: "PerfectTemplate",
    targets: [],
    dependencies: urls.map { .Package(url: $0, versions) }
)
```

 

### Building

SPM可以使用如下的命令来构建你的项目, 也可以用来cleaning up 任何build artifacts:

```swift
swift build
```



这个命令将会下载任何未下载的依赖,然后构建项目(mac平台中swift package generate-xcodeproj命令也可以办到这件事). 如果build 成功, 那么可执行的结果将会被放到隐藏文件夹下(`.build/debug/`) . 当 构建Perfect模板时, 你将会看到SPM输出: `Linking .build/debug/PerfectTemplate`.



#### debug

 输入 `.build/debug/PerfectTemplate`命令,将会run起来服务器. 默认情况下, debug版本服务器将会生成.  

#### release

为了构建release版本, 你需要执行如下命令 `swift build -c release` .这样将会把构建的可执行结果放到隐藏文件夹下`.build/release/`.



```swift
swift build --clean
swift build --clean=dist  
```



上面两条命令可以用来消除中间数据,做一个refresh build.  使用`--clean`参数将会删除`.build`文件夹, 允许你刷新项目. 使用`--clean=dist`参数将会删除`.build`文件夹和`Packages`文件夹.  执行上述命令后, 将会在下次build时重新下载所有依赖,以确保获得所有的最新依赖.



### Xcode Projects



SPM可以根据你的package.swift文件生成Xcode项目. 这个项目允许你使用Xcode构建你的 debug应用. 为了生成Xcode项目,执行如下命令:

```swift
swift package generate-xcodeproj
```

这个命令将会生成Xcode文件.  例如, 在PerfectTemplate文件夹下执行上述命令,将会输出如下信息: `generated: ./PerfectTemplate.xcodeproj`.



**Note:  It is not advised to edit or add files directly to this Xcode project.**  如果你想添加其他依赖,或更新最新的依赖版本,你需要regenerate Xcode项目. 结果, 任何改动都会被覆盖.



#### Additional Information

For more information on the Swift Package Manager, visit:

[https://swift.org/package-manager/](https://swift.org/package-manager/)

