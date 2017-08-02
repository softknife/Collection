# Mustache Template Support

Mustache是一个无逻辑的模板系统. 它允许您使用带有占位符的预先写入的文本文件，这些占位符将在运行时被替换为给定请求特定的值。



For more general information on Mustache, consult the [mustache specification](https://mustache.github.io/mustache.5.html).

为了使用这个模块, 将如下工程添加到Package.swift文件中:

```swift
.Package(
    url: "https://github.com/PerfectlySoft/Perfect-Mustache.git", 
    majorVersion: 2
    )
```



然后,导入Mustache模块:

```swift
import PerfectMustache

```

Mustache 模板可以用在HTTP server handler中,或者无服务的独立系统中.



## Mustache Server Handler

为了使用Mustache模板作为HTTP response, 你需要创建一个遵守`MustachePageHandler`协议的handler对象. 这个handler对象将会为模板程序生成他所需要的值, 以便产生内容.

```swift
/// A mustache handler, which should be passed to `mustacheRequest`, generates values to fill a mustache template
/// Call `context.extendValues(with: values)` one or more times and then
/// `context.requestCompleted(withCollector collector)` to complete the request and output the resulting content to the client.
public protocol MustachePageHandler {
    /// Called by the system when the handler needs to add values for the template.
    func extendValuesForResponse(context contxt: MustacheWebEvaluationContext, collector: MustacheEvaluationOutputCollector)
}
```



你自定义的模板page handler可能展示如下:

```swift
struct TestHandler: MustachePageHandler { // all template handlers must inherit from PageHandler
    // This is the function which all handlers must impliment.
    // It is called by the system to allow the handler to return the set of values which will be used when populating the template.
    // - parameter context: The MustacheWebEvaluationContext which provides access to the HTTPRequest containing all the information pertaining to the request
    // - parameter collector: The MustacheEvaluationOutputCollector which can be used to adjust the template output. For example a `defaultEncodingFunc` could be installed to change how outgoing values are encoded.
    func extendValuesForResponse(context contxt: MustacheWebEvaluationContext, collector: MustacheEvaluationOutputCollector) {
        var values = MustacheEvaluationContext.MapType()
        values["value"] = "hello"
        /// etc.
        contxt.extendValues(with: values)
        do {
            try contxt.requestCompleted(withCollector: collector)
        } catch {
            let response = contxt.webResponse
            response.status = .internalServerError
            response.appendBody(string: "\(error)")
            response.completed()
        }
    }
}
```



为了引导web request到Mustache 模板, 调用`mustacheRequest`函数, 该函数定义如下:

```swift
public func mustacheRequest(request req: HTTPRequest, response: HTTPResponse, handler: MustachePageHandler, templatePath: String)

```



将当前Request 和 Response对象传递给这个函数, 你的`MustachePageHander`, 和你将要呈递的模板文件的路径. `mustacheRequest` 将会执行初始化的几步,例如: 创建Mustache模板解析器, 定位模板文件,以及调用你的Mustache handler去生产那些将会被用到模板中的值.

下面的代码块解释了如何在你的URL handler中使用Mustache 模板. 在这个例子中, 模板名为"test.html"的文件将会被定位到你的服务web root文件夹下:

```swift
{
    request, response in 
    let webRoot = request.documentRoot
    mustacheRequest(request: request, response: response, handler: TestHandler(), templatePath: webRoot + "/test.html")
}
```



Look at the [UploadEnumerator](https://github.com/PerfectExamples/Perfect-UploadEnumerator) example for a more concrete example.



## Standalone Usage

可以以非Web，独立的方式使用该Mustache程序。 你可以通过给该模板文件提供路径,或者通过给该模板提供字符串数据来完成这项功能. 无论哪种方式,你的模板内容都将会被解析, 你提供的值将会被填充到模板内容中.

第一个例子中使用原生模板字符串作为数据源. 第二个例子将一个文件路径传递给模板:

```swift
let templateText = "TOP {\n{{#name}}\n{{name}}{{/name}}\n}\nBOTTOM"
let d = ["name":"The name"] as [String:Any]
let context = MustacheEvaluationContext(templateContent: templateText, map: d)
let collector = MustacheEvaluationOutputCollector()
let responseString = try context.formulateResponse(withCollector: collector)
XCTAssertEqual(responseString, "TOP {\n\nThe name\n}\nBOTTOM")

```



```swift
let templatePath = "path/to/template.mustache"
let d = ["name":"The name"] as [String:Any]
let context = MustacheEvaluationContext(templatePath: templatePath, map: d)
let collector = MustacheEvaluationOutputCollector()
let responseString = try context.formulateResponse(withCollector: collector)
```



## Tag Support

这个Mustache 模板程序支持:

- {{regularTags}}
- {{{unencodedTags}}}
- {{# sections}} ... {{/sections}}
- {{^ invertedSections}} ... {{/invertedSections}}
- {{! comments}}
- {{> partials}}
- lambdas

**Partials**

所有用于partials的文件都必须与调用模板位于同一个文件夹下. 除此之外, 所有partial 文件必须有**.mustache**扩展名, 但是,这个扩展不能包含在partial tag中. 例如, 为了导入*foo.mustache*文件的内容,你需要使用tag `{{> foo}}`.



**Encoding**

默认, 所有的encoded tags(即regular tags)是HTML-encoded, 同时<&>符号将会被转义. 在你的handler中,你可以手动设置`MustacheEvaluationOutputCollector.defaultEncodingFunc`函数,去执行任何你需要的encoding. 例如，当输出JSON数据时，您需要将此功能设置为如下所示：

```swift
collector.defaultEncodingFunc = { 
    string in 
    return (try? string.jsonEncodedString()) ?? "bad string"
}
```



**Lambdas**

函数可以添加到值字典中. 这些函数将会被执行, 结果将会被添加到模板输出中. 这样的函数需要有如下的签名形式:

```swift
(tag: String, context: MustacheEvaluationContext) -> String
```



这个`tag`参数是tag name. 例如，标签{{name}}将为您提供tag参数的值“name”。





































