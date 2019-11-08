---
title: "Golang的JWT实战指南"
date: 2019-11-07T18:19:48+08:00
draft: false
toc: true
tags: ["jwt-go"]
series: ["JWT"]
categories: ["WEB安全"]
---


### 一、JWT介绍

#### 1.1 什么是JSON Web Token?

JSON Web令牌（JWT）是一个开放标准（[RFC 7519](https://tools.ietf.org/html/rfc7519)），它定义了一种协议且以自包含的方式在各方之间安全地传输JSON信息。由于此信息是经过数字签名的，因此可以被验证和信任。可以使用加密算法（如**HMAC**算法）或使用**RSA**或**ECDSA**的公钥/私钥对对JWT进行**签名**。

尽管可以对JWT进行加密以在双方之间提供保密性，但我们将重点关注已*签发的*令牌。已签发的令牌可以验证其中包含的声明的*完整性*，而加密的令牌则将这些声明*隐藏*在其他方的面前。当使用公钥/私钥对对令牌进行签名时，签名还证明只有持有私钥的一方才是对其进行签名的一方。

####  1.2 JWT的使用场景

* 授权：这是使用JWT的最常见方案。一旦用户登录，每个后续请求将包括JWT，从而允许用户访问该令牌允许的路由，服务和资源。单点登陆(SSO)是当今广泛使用JWT的一项功能，因为它的开销很小并且可以在不同的域中轻松使用。
* 信息交换：JSON Web令牌是在各方之间安全地传输信息的好方法。因为可以对JWT进行签名（例如，使用公钥/私钥对），所以您可以确定发送方的身份。另外，由于签名是使用header和playload计算的，因此您还可以验证内容是否未被篡改。

#### 1.3 JSON Web令牌结构
JSON Web令牌由三部分组成的轻量级令牌，这些部分由点（.）分隔，分别是：
* Header 头
* Payload 信息载体
* Signature 签名
因此，JWT的格式一般如下所示：
`xxxxx.yyyyy.zzzzz`

**Header**
标头通常由两部分组成：令牌的类型（即JWT）和所使用的签名算法，例如HMAC SHA256或RSA。
例如：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
然后，这个JSON会被**Base64Url**编码以放在JWT的第一部分。

**Playload**
令牌的第二部分是Playload，其中包含声明。声明是有关实体（通常是用户信息）和其他数据的声明。有以下三种类型：registered, public, and private。

* [**Registered claims**](https://tools.ietf.org/html/rfc7519#section-4.1)：这些是一组非强制性的但建议使用的预定义声明，以提供一组有用的，可互操作的权利要求。其中一些是： iss（签名签发者）， exp（到期时间）， sub（主题）， aud（受众）和 [其他](https://tools.ietf.org/html/rfc7519#section-4.1)
> 值得注意的是，由于JWT的轻量级，这些声明都是只有三个字符长度的

* [**Public Claims**](https://tools.ietf.org/html/rfc7519#section-4.2)：使用JWT的人可以随意定义这些声明。但是为避免冲突，应在[ IANA JSON Web令牌注册表](https://www.iana.org/assignments/jwt/jwt.xhtml)中定义它们，或将其定义为命名空间不冲突的URI。

  > 由于JWT的编码格式是可逆的，为了安全起见，要避免存放用户密码，手机号码，email等用户敏感信息。

* [**Private claims**](https://tools.ietf.org/html/rfc7519#section-4.3) ：这是各方都能识别的即不是用作*registered 声明* 也不是 *public* claims 声明，专门用于共享信息的自定义声明，

示例playload：

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

然后对playload进行Base64Url编码，得到JWT令牌的第二部分。

> 请注意，对于已签发的令牌，此信息尽管可以防止篡改，但任何人都可以读取。除非将其加密，否则请勿将机密信息放入JWT的playload或header元素中。



**Signature 签名**

要创建签名部分，您必须获取编码的Header，编码的Playload，使用header中指定的算法，并对其进行签名。

例如，如果要使用HMAC SHA256算法，则将通过以下方式创建签名：

```java
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```



签名的作用是验证消息在整个过程中没有更改，并且对于使用私钥进行签名的令牌，它还可以验证JWT的发送者是否信息中的指定真实身份。

> 当使用私钥进行签名时，若攻击者篡改了身份id，由于每个身份对应的公钥是不同的，会导致服务端解密失败。



**组合在一起**

最后输出的JWT，是三个由点分隔的Base64-URL字符串，可以在HTML和HTTP环境中轻松传递这些字符串，与基于XML的标准（例如SAML）相比，它更紧凑。

推荐一个JWT在线解吗，验证和生成JWT的网站：[jwt.io Debugger](http://jwt.io/)
![legacy-app-auth-5 (1)](/images/jwt-go/legacy-app-auth-5 (1).png)

**JWT VS SAML**
![legacy-app-auth-5 (1)](/images/jwt-go/comparing-jwt-vs-saml2.png)
与SAML对比，JWT的内容更加简短，http传输速度更快。


#### 1.4 JWT的工作流程

下图显示了如何获取JWT并将其用于访问API或资源：

![client-credentials-grant](/images/jwt-go/client-credentials-grant.png)

1. 应用程序或客户端向授权服务器请求授权。这里通过不同的授权流程中的一种进行展示。例如，典型的符合[OpenID Connect的](http://openid.net/connect/) Web应用程序将`/oauth/authorize`使用[授权码流程](http://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth)通过端点。
2. 授权完毕后，授权服务器会将访问令牌返回给应用程序。
3. 应用程序使用**Token**来访问受保护的资源（例如API）。

在使用JWT过程中需要注意以下几点：

* 安全问题，由于JWT时访问受限资源的凭据，获得JWT后需要防止安全的问题。1）通常JWT会设置时效性，保留规定的一段时间后自动失效。2）使用Https加密信道进行信息传输。

* 避免将JWT存放在local-storage中。如果您的JWT是存放在cookies内。cookies可以使用httpOnly标志，避免被JavaScript进行访问，减少被窃取的风险。

  > local-storage 是一种被广泛使用的离线存储，web存储。如果当前用户对计算机有本地管理员权限，则很容易会被窃取到local-storage的信息。因此，建议不要在local-storage中存储任何敏感的信息。

* 在请求过程中，所发送的JWT通常存放在使用**Bearer**模式的header中。格式如下：

  ``` javascript
  Authorization: Bearer <token>
  ```

  > 在一些架构中，JWT可以作为一种无状态的授权机制。受保护的路由将在HTTP的`Authorization` header中检查JWT的有效性，如果检查通过，则允许用户访问受保护的资源（api）。如果JWT包含一些必要的数据，则可以减少查询数据库操作，当然，这是可选的，并不要求一定要这么做。
  >
  > 如果你的令牌是在`Authorization` 的header中发送的话，那么CORS（Cross-Origin Resource Sharing  跨域访问）就不再是问题了，因为它不使用cookies

* 使用JWT会将令牌或者令牌中包含的所有信息都暴露给用户或者其他地方，即使他们无法篡改它，但也意味着不应该将机密信息放入令牌中（如身份证，电话，地址等用户敏感信息）。

  > 一些非机密信息，如果为了能够减少数据库的查询，可以使用对称/非对称加密算法，加密后再存放到JWT，后端根据预设的解密流程解密即可。

#### 1.5 为什么选择使用JWT令牌

与**简单Web令牌（SWT）**和**安全性声明标记语言令牌（SAML）**相比，**JSON Web令牌（JWT）**有以下的好处。

* 使用轻量级的JSON。由于JSON不如XML冗长，因此在编码时JSON的大小也较小，从而使JWT比SAML更轻量级。使用JWT是在HTML和HTTP环境中传递数据的不错的选择。
* 更安全。SWT只能使用共享密钥的对称加密HMAC算法。而JWT和SAML令牌可以使用X.509证书形式的公/私钥进行签名。与JWT的简单性相比，对XML进行数字签名而不引入难以发现的安全漏洞似乎是一件很困难的事情。
* 通用性。JSON解析器在大多数编程语言中都很常见，因为它们直接映射到对象。相反，XML没有自然的文档到对象映射。与SAML断言相比，JWT使用起来更加容易。
* 跨平台。考虑到JWT的用途，JWT常用于Internet。强调了在多个平台（尤其是移动端）上对JWT进行客户端处理的简便性。



### 二、[JWT Claims](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#rfc.section.4)

#### 2.1 Claim注册表

著名的*IANA*(The Internet Assigned Numbers Authority，互联网数字分配机构)给“JSON Web Token Claims”分配了一些标准的Claim。Claim注册表会记录“Claim Name”和定义它的规范。注册模板包括下面几个内容：[Claim Name、Claim Description、Change Controller、Specification Documents](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#rfc.section.10.1.1)

#### 2.2 JWT Claims

JWT Claim集合是可以被解析成JSON对象，它可以在JWT中进行传递。Claim Names在JWT Claim集合内不允许重名。

JWT Claim Names一共有三类：

* 已注册的标准Claim Names。
* 公共属性的Claim Names。
* 私有属性的Claim Names。

#### 2.3 已注册的Claim Names

一般情况下，已注册的Claim Names是可选的，不强制要求使用或者实现它，但它们提供了一组有用的，可操作的标记。使用JWT应用程序应定义使用了哪些特定的Claim，以及何时是必填的或者可选的。Claim的名称都很短（一般不超过8个字符），因为JWT的核心思想是使表示形式尽量紧凑。

下面列举了目前JWT已注册的7个Cliam。

| Claim name | Claim Description | Claim Controller | Specification Document                                       | 含义                                                         |
| ---------- | ----------------- | ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| iss        | Issuer            | IESG             | [Section 4.1.1](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#issDef) of RFC 7519 | 标示JWT发行方，iss的值是区分大小写的字符串，其中包含了StringOrURI值。*（此声明的使用是可选的）* |
| sub        | Subject           | IESG             | [Section 4.1.2](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#subDef) of RFC 7519 | sub标示JWT的主题。在issuer的上下文中，sub的值必须被限定为局部唯一或者全局唯一。sub是区分大小写的字符串，其中包含了StringOrURI值。*（此声明的使用是可选的）* |
| aud        | Audience          | IESG             | [Section 4.1.3](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#audDef) of RFC 7519 | aud标示了JWT的目标收件人。每个打算处理JWT的sub必须在sub中标示自己的值。如果cliam中有声明aud但是未赋值，则必须拒绝该JWT。aud是区分大小写的字符串，其中包含了StringOrURI值。在特殊情况下，JWT只有一个aud，那么aud是可以包含StringOrURI的值且区分大小写。*（此声明的使用是可选的）* |
| exp        | Expiration Time   | IESG             | [Section 4.1.4](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#expDef) of RFC 7519 | exp标示了JWT的到期时间。exp声明要求当前日期/时间必须早于exp。但实施者可以留一些余地，以解决时钟偏差，通常不超过几分钟。它的值必须的包含NumericDate值的数字。*（此声明的使用是可选的）* |
| nbf        | Not Before        | IESG             | [Section 4.1.5](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#nbfDef) of RFC 7519 | nbf声明标示了禁止接受JWT的处理时间。nbf要求当前日期/时间必须晚于或等于nbf的要求。但实施者可以留一些余地，以解决时钟偏差，通常不超过几分钟。它的值必须的包含NumericDate值的数字。*（此声明的使用是可选的） |
| iat        | Issued At         | IESG             | [Section 4.1.6](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#iatDef) of RFC 7519 | iat标示了JWT的发布时间。此声明可用于确定JWT的存在时长。它的值必须的包含NumericDate值的数字。*（此声明的使用是可选的） |
| jti        | JWT ID            | IESG             | [Section 4.1.7](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#jtiDef) of RFC 7519 | jti声明为JWT提供了唯一标识符。标示符的分配方式必须确保将相同的值偶然分配给不同的数据对象的可能性极小。如果应用程序使用多个发行者，则还必须防止不同发行者之间产生的值之间发生冲突。jti可以用来防止重播JWT。jti是区分大小写的字符串。*（此声明的使用是可选的）* |

#### 2.4 Public Claim Names

公共声明是可以由使用JWT的人员定义的。但是，为了防止冲突，任何新的Claim Name应该在[第10.1节](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#JWTClaimsReg)建立的IANA“ JSON Web Token Claims”注册表中注册，或者应该是Public Name：包含防冲突名称的值。在每种情况下，名称或值的定义者都需要采取合理的预防措施，以确保它们可以控制用于定义“声明名称”的名称空间的一部分。

#### 2.5 Private Claim Names

JWT的生产者和消费者可以同意使用的私有名称的Claim Name。不能与已注册的Claim Names或公共的Claim Names冲突。与公共声明的名称不同的是，私有声明名称可能会发生冲突，应谨慎使用。

### 三、JWT实战——Go语言
### 四、JWT最佳实践

### 参考资料

[IETF文档#JWT —— ISSN：2070-1721](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#JWTClaimsReg)

https://www.cnblogs.com/yibutian/p/9507866.html
