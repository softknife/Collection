# JSON Converter

Perfect 包含基础的JSON encoding 和decoding 功能. JSON encoding 是通过内置的swift 数据类型的一系列扩展来提供的. Decoding 是通过Swift String的扩展来提供的.



有一点很重要的需要注意, 尽管Perfect 提供了特殊的JSON encoding/decoding 系统, 你的应用也不一定使用它们. 你可以随意导入你自己喜欢的JSON相关的功能.



为了使用这个功能,你需要导入PerfectLib:

```swift
import PerfectLib
```



### Encoding To JSON Data

你可以将如下任何类型转换成JSON:

- String
- Int
- UInt
- Double
- Bool
- Array
- Dictionary
- Optional
- Custom classes which inherit from JSONConvertibleObject



注意只有满足上述类型的可选类型才能直接转换.  如果可选类型中包裹的是nil,转换为JSON后为null.



要编码这些值中的任何一个，请调用作为对象上的扩展名提供的`jsonEncodedString()`函数。 这个函数可能会抛出一个`JSONConversionError.notConvertible`Error.



例如:

```swift
let scoreArray: [String:Any] = ["1st Place": 300, "2nd Place": 230.45, "3rd Place": 150]
let encoded = try scoreArray.jsonEncodedString()
```



encoding的结果如下:

```swift
{"2nd Place":230.45,"1st Place":300,"3rd Place":150}
```



### Decoding JSON Data

JSON字符串可以通过`jsonDecode()`函数来decode. 如果 JSON字符串无效,那么解析时,这个函数会抛出一个`JSONConversionError.syntaxError`的错误.

```swift
let encoded = "{\"2nd Place\":230.45,\"1st Place\":300,\"3rd Place\":150}"
let decoded = try? encoded.jsonDecode() as? [String:Any]
```

上面字符串解析后结果为:

```swift
["2nd Place": 230.44999999999999, "1st Place": 300, "3rd Place": 150]
```

JSON字符串可以被解析为任何允许的值, 通常解析成JSON Objects(dictionaries)或者arrays. 你需要将结果映射成你需要的类型.



#### Using the Decoded Data

由于解析后的字典或者数组总是[String:Any]或者[Any]类型的, 相应的,你需要将其内部包裹的值也转成方便使用的类型. 例如:

```swift
var firstPlace = 0
var secondPlace = 0.0
var thirdPlace = 0
 
let encoded = "{\"2nd Place\":230.45,\"1st Place\":300,\"3rd Place\":150}"
guard let decoded = try encoded.jsonDecode() as? [String:Any] else {
    return
}
 
for (key, value) in decoded {
    switch key {
    case "1st Place":
        firstPlace = value as! Int
    case "2nd Place":
        secondPlace = value as! Double
    case "3rd Place":
        thirdPlace = value as! Int
    default:
        break
    }
}
 
print("The top scores are: \r" + "First Place: " + "\(firstPlace)" + " Points\r" + "Second Place: " + "\(secondPlace)" + " Points\r" + "Third Place: " + "\(thirdPlace)" + " Points")
```



输出结果是:

```swift
The top scores are: 
First Place: 300 Points
Second Place: 230.45 Points
Third Place: 150 Points
```



#### Decoding Empty Values from JSON Data

由于JSON null 值是非类型的, 系统将会将`JSONConvertibleNull`替换成JSON nulls.例如:

```swift
let jsonString = "{\"1st Place\":300,\"4th place\":null,\"2nd Place\":230.45,\"3rd Place\":150}"
 
if let decoded = try jsonString.jsonDecode() as? [String:Any] {
    for (key, value) in decoded {
        if let value as? JSONConvertibleNull {
            print("The key \"\(key)\" had a null value")
        }
    }
}
```



输出结果:

```swift
The key "4th place" had a null value

```



### JSON Convertible Object

Perfect JSON System提供了自定义类的encoding,decoding工具. 任何可以完成该操作的类必须继承自`JSONConvertibleObject`这个父类, 定义如下:

```swift
/// Base for a custom object which can be converted to and from JSON.
public class JSONConvertibleObject: JSONConvertible {
    /// Default initializer.
    public init() {}
    /// Get the JSON keys/value.
    public func setJSONValues(_ values:[String:Any]) {}
    /// Set the object properties based on the JSON keys/values.
    public func getJSONValues() -> [String:Any] { return [String:Any]() }
    /// Encode the object into JSON text
    public func jsonEncodedString() throws -> String {
        return try self.getJSONValues().jsonEncodedString()
    }
}
```



任何想要具有JSON encoded/decoded的对象必须首先将自己注册到系统中. 一旦应用启动,就要马上注册. 调用`JSONDecoding,registerJSONDecodable`方法去注册. 函数定义如下:

```swift
public class JSONDecoding {
    /// Function which returns a new instance of a custom object which will have its members set based on the JSON data.
    public typealias JSONConvertibleObjectCreator = () -> JSONConvertibleObject
    static public func registerJSONDecodable(name: String, creator: JSONConvertibleObjectCreator)
}
```



注册一个对象时,你需要提供一个唯一的字符串.  还需要一个`creator`函数,这个函数将会返回一个新的对象实例.



当系统encodes一个`JSONConvertibleObject`时, 他会调用对象的`getJSONValues`函数. 这个函数将会返回一个`[String:Any]`字典,其中包含对象的所有属性转化的k-v对. 这个字典中必须还要包含一个唯一标识该对象类型的值. 这个值必须匹配我们上述注册时填入的值. 这个字典中的key由`JSONDecoding.objectIdentifierKey`标识.



当系统decodes这样一个对象时, 将会查找`JSONDecoding.objectIdentifierKey`对应的value,并且查找开始时注册的object creator. 系统将会创建这个类型的一个实例,然后调用这个新对象的`setJSONValue(_ values:[String:Any])`方法.给这个方法传入一个字典,这个字典中包含所有解析过的值. 这些值将会匹配那些对象第一次转换时,通过调用`getJSONValues`返回的值.  通过`setJSONValues`函数, 对象将会获得所有他想恢复的属性.



下面例子中定义了一个自定义的`JSONConvertibleObject` 对象, 然后将他转化为一个JSON String. 然后,解析这个对象,并把它和原来的对比. 注意这个例子中国对象调用`getJSONValue`便利方法时, 这个方法将会从字典中获取命名的value,并且,如果字典不能匹配到指定的key时,允许提供默认值.



这个例子被分割成几个部分.

定义Class

```swift
class User: JSONConvertibleObject {
    static let registerName = "user"
    var firstName = ""
    var lastName = ""
    var age = 0
    override func setJSONValues(_ values: [String : Any]) {
        self.firstName = getJSONValue(named: "firstName", from: values, defaultValue: "")
        self.lastName = getJSONValue(named: "lastName", from: values, defaultValue: "")
        self.age = getJSONValue(named: "age", from: values, defaultValue: 0)
    }
    override func getJSONValues() -> [String : Any] {
        return [
            JSONDecoding.objectIdentifierKey:User.registerName,
            "firstName":firstName,
            "lastName":lastName,
            "age":age
        ]
    }
}
```



注册类:

```swift
// do this once
JSONDecoding.registerJSONDecodable(name: User.registerName, creator: { return User() })
```

Encode 对象:

```swift
let user = User()
user.firstName = "Donnie"
user.lastName = "Darko"
user.age = 17
 
let encoded = try user.jsonEncodedString()
```



encoded值将会展示如下:

```swift
{"lastName":"Darko","age":17,"_jsonobjid":"user","firstName":"Donnie"}

```



Decode the object:

```swift
guard let user2 = try encoded.jsonDecode() as? User else {
    return // error
}
 
// check the values
XCTAssert(user.firstName == user2.firstName)
XCTAssert(user.lastName == user2.lastName)
XCTAssert(user.age == user2.age)
```



注意该`JSONDecoding.objectIdentifierKey`(他的值是"_jsonobjid")键值对标示着,将要被解析的Swift Object对象必须包含在JSON字符串中. 如果没有出现, 那么JSON字符串解析的结果将会是正常的Dictionary ,而不是Swift 对象.



### JSON Conversion Error

当一个对象转化成一个JSON字符串或者字符串转化成一个对象时, 处理过程可能会抛出一个异常`JSONConversionError`. 定义如下:

```swift
/// An error occurring during JSON conversion.
public enum JSONConversionError: ErrorProtocol {
    /// The object did not suppport JSON conversion.
    case notConvertible(Any)
    /// A provided key was not a String.
    case invalidKey(Any)
    /// The JSON text contained a syntax error.
    case syntaxError
}
```

