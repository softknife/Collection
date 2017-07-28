# Perfect-SPNEGO 

> SPNEGO(SPNEGO: Simple and Protected GSS-API Negotiation)是[微软](https://baike.baidu.com/item/%E5%BE%AE%E8%BD%AF)提供的一种使用GSS-API认证机制的安全协议，用于使Webserver共享Windows Credentials，它扩展了[Kerberos](https://baike.baidu.com/item/Kerberos)(一种网络认证协议)。 Network authentication protocol



这个项目提供了普通的服务器框架, 该框架提供了SPNEGO机制.



### Before Start

Perfect SPNEGO 目标在于普通的Server端应用,所以他可以作为任何服务器的插件使用, 例如HTTP/FTP/SSH等等.



尽管他支持本地的Perfect HTTP server, 他也可以用于其他任何Swift server.

在将他应用于任何真实的应用之前, 请确保你的服务器已经配置了Kerberos V5.



### Xcode Build Note

如果你想使用Xcode构建项目, 确保传递合适的linker flags 给SPM:

```swift
$ swift package -Xlinker -framework -Xlinker GSS generate-xcodeproj

```



### Linux Build Note

需要配置一个特殊的库叫做`libkrb5-dev`:

```swift
$ sudo apt-get install libkrb5-dev

```



如果你的server 是KDC(http://docs.oracle.com/cd/E24847_01/html/819-7061/setup-9.html), 那么你可以跳过这一步,否则请安装Kerberos V5工具:

```swift
$ sudo apt-get install krb5-user

```



### KDC Conifuration

将应用服务器的`/etc/krb5.conf`配置到你的KDC下面. 下面的配置例子给出了在KDC(叫做`nut.krb5.ca`)控制下, 如何将你的应用服务器连接到realm`KRB5.CA`.



```swift
[realms]
KRB5.CA = {
    kdc = nut.krb5.ca
    admin_server = nut.krb5.ca
}
[domain_realm]
.krb5.ca = KRB5.CA
krb5.ca = KRB5.CA
```



### Prepare Kerberos Keys for Server

联系您的KDC管理员，以将keytab文件分配给应用程序服务器。



例如, *SUPPOSE ALL HOSTS BELOW REGISTERED ON THE SAME DNS SERVER*:

- KDC server: nut.krb5.ca
- Application server: coco.krb5.ca
- Application server type: HTTP



在这种例子中, KDC管理员应该登录`nut.krb5.ca`, 然后执行如下操作:

```swift
kadmin.local: addprinc -randkey HTTP/coco.krb5.ca@KRB5.CA
kadmin.local: ktadd -k /tmp/krb5.keytab HTTP/coco.krb5.ca@KRB5.CA
```



然后安全的发送这个"krb5.keytab"文件,安装他到你的服务器上`coco.krb5.ca` , 移动到`/etc`文件夹下, 然后, 给予足够的权限访问你的swift应用服务.



## Quick Start

添加如下依赖到你的Package.swift文件中:

```swift
.Package(url: "https://github.com/PerfectlySoft/Perfect-SPNEGO.git", majorVersion: 1)
```



导入Perfect-SPNEGO :

```swift
import PerfectSPNEGO
```



### Connect to KDC

使用应用服务器上默认的keytab /etc/krb5.keytab文件中的密钥将应用程序服务器注册到KDC。

```swift
let spnego = try Spnego("HTTP@coco.krb5.ca")
```



注意host 名字和protocol 类型必须与keytab文件中记录的匹配.



### Response to A Spnego Challenge

一旦初始化, `spnego`可以应对挑战. 例如,如果一个用户尝试着链接服务器:

```swift
$ kinit rocky@KRB5.CA
$ curl --negotiate -u : http://coco.krb5.ca
```



在这个例子中, curl 命令可能会发送a base64 encoded challenge in the HTTP header:

```swift
> Authorization: Negotiate YIICnQYGKwYBBQUCoIICkTCCAo2gJzAlBgkqhkiG9xIBAgI ...

```



一旦收到这样的challenge , 你可以将base64的字符串应用于`spnego`对象:

```swift
let (user, response) = try spnego.accept(base64Token: "YIICnQYGKwYBBQUCoIICkTCCAo2gJzAlBgkqhkiG9xIBAgI...")
```



如果成功, 用户是"rocky@KRB5.CA". `response`变量可能是`nil` , 意味着不需要回复此类token, 否则你应该将此response发送回客户端.

至此, 你的服务端应用已经得到了用户信息和Request, 然后,服务器就可以根据你的ACL(access control list)配置文件, 决定这个用户是否可以访问对象资源了.



### Relevant Examples

A good example to demonstrate how to use SPNEGO in a Perfect HTTP Server can be found from: * [Perfect-Spnego-Demo](https://github.com/PerfectExamples/Perfect-Spnego-Demo)

