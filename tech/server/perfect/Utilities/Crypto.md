# Crypto

Perfect-Crypto包是基于OpenSSL的通用密码库。他提供了高级对象来处理如下加密任务:

- Message digests and hashes, sign/verify 消息摘要和散列，签名/验证
- Cipher based encryption/decryption — 密码加密/解密
- Random data generation of arbitrary byte lengths —  随机数据生成任意字节长度
- HMAC key generation  —  HMAC密钥生成
- PEM format public/private key reading — PEM格式公开/私钥读取
- JWT (JSON Web Token) creation and validation — JWT（JSON Web Token）创建和验证



他还提供了编码相关的功能, 这些功能是很有用的, 但是通常和加密一起使用, 特别是当将二进制数据转换为字符可打印状态时。

使用这个包中的功能时,请确保`import PerfectCrypto`.



## Initialization

在使用相关功能之前, 需要初始化一次底层加密库. 一般在你编码执行任何实际任务之前, 初始化都会执行一次. 调用`PerfectCrypto.isInitialized`方法将会初始化库,返回true.  不止一次调用它是很好的，但是通常只有在引导您的应用程序时才会调用这一次。



## Extensions

这个包中提供的多数功能是通过Swift内建的类型的扩展来实现的, 例如:  `String`, `Array<UInt8>`, `UnsafeRawBufferPointer`, and `UnsafeMutableRawBufferPointer`. 这些扩展主要是对该类型增加了 `encode/decode`, `encrypt/decrypt`, `sign/verify` and `digest` functions.   其他提供处理原始二进制数据，生成随机数据等的便利功能。



这些扩展功能以“convenience”的顺序列出。 首先列出了较高级的功能，最后列出了更高效的“不安全”的功能。



### String

String的扩展分为两组. 

- 第一组提供了字符串的encode,decode,和digest功能.

```swift
public extension String {
    /// Decode the String into an array of bytes using the indicated encoding.
    /// The string's UTF8 characters are decoded.
    func decode(_ encoding: Encoding) -> [UInt8]?
    /// Encode the String into an array of bytes using the indicated encoding.
    /// The string's UTF8 characters are encoded.
    func encode(_ encoding: Encoding) -> [UInt8]?
    /// Perform the digest algorithm on the String's UTF8 bytes.
    func digest(_ digest: Digest) -> [UInt8]?
    /// Sign the String data into an array of bytes using the indicated algorithm and key.
    func sign(_ digest: Digest, key: Key) -> [UInt8]?
    /// Verify the signature against the String data.
    /// Returns true if the signature is verified. Returns false otherwise.
    func verify(_ digest: Digest, signature: [UInt8], key: Key) -> Bool
}
```

`encode`和`decode`功能给出一种`Encoding`类型，将会返回结果为[UInt8], 或者,如果输入数据对于该特定编码无效，返回nil . encoding类型必须是一个有效的`Encoding`枚举值. 例如`.hex`或者`.base64url`.



下面例子中展示了如何将String类型数据encode成base64, 以及如何decode成原始值.

```swift
let testStr = "Räksmörgåsen"
if let encoded = testStr.encode(.base64),
    let decoded = encoded.decode(.base64),
    let decodedStr = String(validatingUTF8: decoded) {
    print(decodedStr)
    // Räksmörgåsen
}
```



摘要功能允许对字符串的字符执行消息摘要操作。 摘要数据作为[UInt8]返回。 如果底层系统不支持指定的编码，则返回nil。



下面例子中展示了字符串的`sha256`摘要算法. 然后将结果值转换为可打印的十六进制字符串。

```swift
let testStr = "Hello, world!"
if let digestBytes = testStr.digest(.sha256),
    let hexBytes = digestBytes.encode(.hex),
    let hexBytesStr = String(validatingUTF8: hexBytes) {
    print(hexBytesStr)
    // 315f5bdb76d078c43b8ac0064e4a0164612b1fce77c869345bfc94c75894edd3
}
```

签名和验证功能允许使用指定的密钥对数据块进行加密签名，然后验证，以确保数据没有以任何方式进行修改。 可以使用HMAC，RSA或EC类型键进行签名。



- 第二组String扩展添加了从非空终止的UTF-8字符创建字符串的便利方法。这些字符可以通过`[UInt8]`或者`UnsafeRawBufferPointer`给出:

```swift
public extension String {
    /// Construct a string from a UTF8 character array.
    /// The array's count indicates how many characters are to be converted.
    /// Returns nil if the data is invalid.
    init?(validatingUTF8 a: [UInt8])
    /// Construct a string from a UTF8 character pointer.
    /// Character data does not need to be null terminated.
    /// The buffer's count indicates how many characters are to be converted.
    /// Returns nil if the data is invalid.
    init?(validatingUTF8 ptr: UnsafeRawBufferPointer?)
    /// Obtain a buffer pointer for the String's UTF8 characters.
    func withBufferPointer<Result>(_ body: (UnsafeRawBufferPointer) throws -> Result) rethrows -> Result
}
```



最后一个函数`withBufferPointer`是用来捕获包含UTF8字符串的`UnsafeRawBufferPointer`的. 这个buffer只在closure/function中有效.



### Array<UInt8>



数组的扩展只支持元素为`UInt8`类型值的数组.   这些功能提供编码/解码，加密/解密和摘要操作以及允许创建随机生成数据的数组的初始化器。

```swift
public extension Array where Element: Octal {
    /// Encode the Array into An array of bytes using the indicated encoding.
    func encode(_ encoding: Encoding) -> [UInt8]?
    /// Decode the Array into an array of bytes using the indicated encoding.
    func decode(_ encoding: Encoding) -> [UInt8]?
    /// Digest the Array data into an array of bytes using the indicated algorithm.
    func digest(_ digest: Digest) -> [UInt8]?
    /// Sign the Array data into an array of bytes using the indicated algorithm and key.
    func sign(_ digest: Digest, key: Key) -> [UInt8]?
    /// Verify the array against the signature.
    /// Returns true if the signature is verified. Returns false otherwise.
    func verify(_ digest: Digest, signature: [UInt8], key: Key) -> Bool
    /// Decrypt this buffer using the indicated cipher, key an iv (initialization vector).
    func encrypt(_ cipher: Cipher, key: [UInt8], iv: [UInt8]) -> [UInt8]?
    /// Encrypt this buffer using the indicated cipher, key an iv (initialization vector).
    func decrypt(_ cipher: Cipher, key: [UInt8], iv: [UInt8]) -> [UInt8]?
}
```

`encode`，`decode`和`digest`功能与String版本相同，只是输入数据是[UInt8]。  编码和解码功能给出一种编码类型，返回结果为[UInt8] , 如果输入数据对于该特定编码无效，则返回nil。 encoding类型必须是一个有效的`Encoding`枚举值. 例如`.hex`或者`.base64url`.



摘要功能允许对数组值执行消息摘要操作。 摘要数据作为[UInt8]返回。 如果底层系统不支持指定的摘要，则返回nil。

加密和解密功能将根据指示的密码枚举值对数组数据进行加密/解密。 这些操作需要传入密钥(key)和初始化向量(initialization vector iv)参数.

使用中key和iv数组的大小会根据cipher的不同而变化. `Cipher`枚举值为每个密码提供了  `blockSize`,`keyLength`,and `ivLength` 属性.



下面代码片段将会使用`.aes_256_cbc` cipher来加密一个随机数据数组. Key和iv也是根据cipher的size来随机生成的.

```swift
let cipher = Cipher.aes_256_cbc
    // the data which will be encrypted
let random = [UInt8](randomCount: 2048)
    // The key value for the encrypt/decrypt
let key = [UInt8](randomCount: cipher.keyLength)
    // Initialization vector
let iv = [UInt8](randomCount: cipher.ivLength)
 
if let encrypted = random.encrypt(cipher, key: key, iv: iv),
    let decrypted = encrypted.decrypt(cipher, key: key, iv: iv) else {
 
    for (a, b) in zip(decrypted, random) {
        (a == b)
    }
}
```



随机字节数组可以通过如下扩展生成.

```swift
public extension Array where Element: Octal {
    /// Creates a new array containing the specified number of a single random values.
    init(randomCount count: Int)
}
```



在这个数组中每一个字节将会随机产生. 下面例子中生成16个随机字节数据, 然后转换成可打印的base64字符串.

```swift
    // generate 16 random bytes of data
let random = [UInt8](randomCount: 16)
if let base64 = random.encode(.base64),
    let base64Str = String(validatingUTF8: base64) {
    print(base64Str)
}
```



### UnsafeXX Extensions

在UnsafeMutableRawBufferPointer和UnsafeRawBufferPointer上提供的扩展提供了, 对用于加密操作的缓冲区进行较低级别的访问。它们提供更有效的行为，但与Array扩展完全相同的操作。



#### UnsafeMutableRawBufferPointer

