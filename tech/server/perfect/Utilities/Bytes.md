# Bytes

> http://www.perfect.org/docs/bytes.html

字节对象提供与UInt8数组的常用Swift值的简单流式转换。它支持导入和导出UInt8，UInt16，UInt32和UInt64值。当导入这些值时, 他们将会被拼接到数组最后. 当导出时, 保持可重新定位的标记，指示当前的出口位置。字节对象是PerfectLib的一部分. 如果你想使用Bytes object,确保`import PerfectLib`.

字节对象背后的主要目的是使二进制网络有效载荷容易地组装和分解。 所得到的UInt8数组可通过`data`属性获得。

下面例子中阐释了如何导入不同大小的值, 以及导出和校验这些值.  它还显示了如何重新定位标记，以及如何确定要导出多少字节。



```swift
let i8 = 254 as UInt8
let i16 = 54045 as UInt16
let i32 = 4160745471 as UInt32
let i64 = 17293541094125989887 as UInt64
 
let bytes = Bytes()
 
bytes.import64Bits(from: i64)
    .import32Bits(from: i32)
    .import16Bits(from: i16)
    .import8Bits(from: i8)
 
let bytes2 = Bytes()
bytes2.importBytes(from: bytes)
 
XCTAssert(i64 == bytes2.export64Bits())
XCTAssert(i32 == bytes2.export32Bits())
XCTAssert(i16 == bytes2.export16Bits())
bytes2.position -= sizeof(UInt16.self)
XCTAssert(i16 == bytes2.export16Bits())
XCTAssert(bytes2.availableExportBytes == 1)
XCTAssert(i8 == bytes2.export8Bits())
```



