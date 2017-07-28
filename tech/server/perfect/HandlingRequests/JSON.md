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



