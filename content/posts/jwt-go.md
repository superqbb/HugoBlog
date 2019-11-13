---
title: "Golang的JWT实战指南"
date: 2019-11-07T18:19:48+08:00
draft: false
toc: true
tags: ["jwt-go"]
series: ["JWT"]
categories: ["WEB安全"]
---

[TOC]

### 一、JWT介绍

#### 1.1 什么是JSON Web Token?

JSON Web令牌（JWT）是一个开放标准（[RFC 7519](https://tools.ietf.org/html/rfc7519)），它定义了一种协议且以自包含的方式在各方之间安全地传输JSON信息。由于此信息是经过数字签名的，因此可以被验证和信任。可以使用加密算法（如**HMAC**算法）或使用**RSA**或**ECDSA**的公钥/私钥对对JWT进行**签名**。

尽管可以对JWT进行加密以在双方之间提供保密性，但我们将重点关注已 *签发的* 令牌。已签发的令牌可以验证其中包含的声明的 *完整性* ，而加密的令牌则将这些声明 *隐藏* 在其他方的面前。当使用公钥/私钥对对令牌进行签名时，签名还证明只有持有私钥的一方才是对其进行签名的一方。

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

##### Header
标头通常由两部分组成：令牌的类型（即JWT）和所使用的签名算法，例如HMAC SHA256或RSA。
例如：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
然后，这个JSON会被**Base64Url**编码以放在JWT的第一部分。

#####  Playload
令牌的第二部分是Playload，其中包含声明。声明是有关实体（通常是用户信息）和其他数据的声明。有以下三种类型：registered, public, and private。

* [**Registered claims**](https://tools.ietf.org/html/rfc7519#section-4.1)：这些是一组非强制性的但建议使用的预定义声明，以提供一组有用的，可互操作的权利要求。其中一些是： iss（签名签发者）， exp（到期时间）， sub（主题）， aud（受众）和 [其他](https://tools.ietf.org/html/rfc7519#section-4.1)
> 值得注意的是，由于JWT的轻量级，这些声明都是只有三个字符长度的

* [**Public Claims**](https://tools.ietf.org/html/rfc7519#section-4.2)：使用JWT的人可以随意定义这些声明。但是为避免冲突，应在[ IANA JSON Web令牌注册表](https://www.iana.org/assignments/jwt/jwt.xhtml)中定义它们，或将其定义为命名空间不冲突的URI。

  > 由于JWT的编码格式是可逆的，为了安全起见，要避免存放用户密码，手机号码，email等用户敏感信息。

* [**Private claims**](https://tools.ietf.org/html/rfc7519#section-4.3) ：这是各方都能识别的即不是用作*registered 声明* 也不是 *public* claims 声明，专门用于共享信息的自定义声明。

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



#####  Signature 签名

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

推荐一个JWT在线解码，验证和生成JWT的网站：[jwt.io Debugger](http://jwt.io/)
![legacy-app-auth-5 (1)](/images/jwt-go/legacy-app-auth-5 (1).png)




#### 1.4 JWT的工作流程

下图显示了如何获取JWT并将其用于访问API或资源：

![client-credentials-grant](/images/jwt-go/qg4tew3ml3.png)

1. 应用程序或客户端向授权服务器 `Authentication Server` 请求授权。这里通过不同的授权流程中的一种进行展示。例如，典型的符合[OpenID Connect的](http://openid.net/connect/) Web应用程序将`/oauth/authorize`使用[授权码流程](http://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth)通过端点。
2. 授权完毕后，授权服务器会将访问令牌JWT返回给应用程序，客户端将凭证信息存储在本地(cookie或浏览器缓存)
3. 当用户发起新的请求时，需要在请求头中附带此凭证信息，当服务器接收到用户请求时，会先检查请求头中有无凭证，是否过期，是否有效。如果凭证有效，将放行请求；若凭证非法或者过期，服务器将回跳到认证中心，重新对用户身份进行验证，直至用户身份验证成功。


#### 1.5 JWT vs cookie-session

##### 使用JWT的优点：

**1. 更易于服务端的水平扩展**

在cookie-session方案中，cookie内仅包含一个session标识符，而诸如用户信息、授权列表等都保存在服务端的session中。如果把session中的认证信息都保存在JWT中，在服务端就没有session存在的必要了。当服务端水平扩展的时候，就不用处理session复制（session replication）/ session黏连（sticky session）或是引入外部session存储了。

**2. 无状态**

jwt不在服务端存储任何状态。RESTful API的原则之一是无状态，发出请求时，总会返回带有参数的响应，不会产生附加影响。用户的认证状态引入这种附加影响，这破坏了这一原则。另外jwt的载荷中可以存储一些常用信息，用于交换信息，有效地使用 JWT，可以降低服务器查询数据库的次数。

**3. 防篡改**

由于JWT是由服务端颁发的，带有数字签名，因此可以有效防止被攻击者篡改。

**4.单点登录SSO**

由于JWT存储在客户端，因此接口调用时，JWT可以由独立、统一的用户权限认证中心进行用户身份认证和token签发。携带JWT的请求，微服务或子系统理论上不需要再访问认证中心，即可以使用公共的密钥进行验证。

**5.更少的数据库连接**

JWT的负载中存放的数据，具有不可篡改的特性，服务端可以直接使用，不需要从数据库中再次查询，从而达到减少数据库连接的效果，获得更快的系统响应时间。

**6.对 Restful 接口友好**

Restful接口的原则是无状态(`Statelessness`)。如果使用session来保存用户状态信息，那就意味着在基于状态的Web系统中的Server无法对用户请求进行负载均衡等自由的调度(一个Client请求只能由一个指定的Server处理)。



##### 使用JWT的缺点：

**1. 安全性**

由于jwt的payload是使用base64编码的，并没有加密，因此jwt中不能存储敏感数据。而session的信息是存在服务端的，相对来说更安全。

**2. 无法作废已颁布的令牌**

所有的认证信息都在JWT中，由于在服务端没有状态，即使你知道了某个JWT被盗取了，你也没有办法将其作废。在JWT过期之前（你绝对应该设置过期时间），你无能为力。

**3.不易应对数据过期**

与上一条类似，JWT有点类似缓存，由于无法作废已颁布的令牌，在其过期前，你只能忍受“过期”的数据。

**4.严重依赖于秘钥**

如果秘钥不小心泄露，系统的安全性将受到威胁。服务端可以定期更换密钥，但必须考虑使用旧密钥签发的有效token的处理。

**5.服务端无法主动推送消息**

服务端由于是无状态的，JWT将无法使用像 Session 那样的方式推送消息到客户端，例如过期时间将至，服务端无法主动为用户续约，需要客户端向服务端发起续约请求。

**6.冗余的数据开销**

一个 JWT 签名的大小要远比一个 Session ID 长很多，如果你对有效载荷(payload)中的数据不做有效控制，其长度会成几何倍数增长，且在每一次请求时都需要负担额外的网络开销。


#### 1.6 JWT vs SAML？

与**简单Web令牌（SWT）**和**安全性声明标记语言令牌（SAML）**相比，**JSON Web令牌（JWT）**有以下的好处。

**1.使用轻量级的JSON。**

由于JSON不如XML冗长，因此在编码时JSON的大小也较小，从而使JWT比SAML更轻量级。使用JWT是在HTML和HTTP环境中传递数据的不错的选择。

**2.更安全。**

SWT只能使用共享密钥的对称加密HMAC算法。而JWT和SAML令牌可以使用X.509证书形式的公/私钥进行签名。与JWT的简单性相比，对XML进行数字签名而不引入难以发现的安全漏洞似乎是一件很困难的事情。

**3.通用性**

JSON解析器在大多数编程语言中都很常见，因为它们直接映射到对象。相反，XML没有自然的文档到对象映射。与SAML断言相比，JWT使用起来更加容易。

**4.跨平台。**

考虑到JWT的用途，JWT常用于Internet。强调了在多个平台（尤其是移动端）上对JWT进行客户端处理的简便性。

**JWT VS SAML**
![legacy-app-auth-5 (1)](/images/jwt-go/comparing-jwt-vs-saml2.png)
与SAML对比，JWT的内容更加简短，http传输速度更快。



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

各个语言的jwt官方推荐库： https://jwt.io/#libraries-io

下面演示使用jwt-go生成jwt，验证jwt

```go
package main

import (
   "errors"
   "fmt"
   "github.com/dgrijalva/jwt-go"
   "os"
   "time"
)

var HmacKeySecret = `W42!@=-wqqk#pIn<A?130?eskfit]w\q'311mdasie./juwOW~w@9$MJhgPQ#&6e`

func main(){
   fmt.Println("发起JWT请求..")

   //模拟创建jwt
   newJWT,err := IssueJWT()
   if err !=nil {
      fmt.Println("生成jwt失败!err:",err)
      os.Exit(1)
   }
   fmt.Println("生成jwt成功：")
   fmt.Println(newJWT)

   fmt.Println("===================================")
   fmt.Println("测试jwt验证...")

   err = ValidJWT(newJWT)
   if err !=nil {
      fmt.Println("JWT验证失败!err:",err)
      os.Exit(1)
   }
}

//创建jwt示例
func IssueJWT() (string,error) {
   //设置claims
   mapClaims := make(jwt.MapClaims, 0)

   //设置过期时间
   mapClaims["exp"] = time.Now().Add(30 * time.Minute).Unix()
   //设置签发机构
   mapClaims["iss"] = "example.com"



   //设置自定义负载
   mapClaims["isAdmin"] = true
   mapClaims["csrf"] = time.Now().Nanosecond()
   fmt.Printf("playload:%v\n",mapClaims)

   var newToken *jwt.Token
   //可选择多种加密方式
   newToken = jwt.NewWithClaims(jwt.SigningMethodHS256, mapClaims)

   if newToken == nil {
      return "", errors.New("issue jwt failed, token is nil")
   }

   //生成完整的jwt
   tokenString, err := newToken.SignedString([]byte(HmacKeySecret))
   if err != nil {
      return "", err
   }
   return tokenString, nil
}

//ValidJWT 验证jwt令牌
func ValidJWT(jwtStr string) error {
   token, err := jwt.Parse(jwtStr, func(token *jwt.Token) (i interface{}, e error) {
      //jwt会根据token.Header["alg"]字段，自动匹配解密算法
      //返回约定的解密key
      return []byte(HmacKeySecret), nil
   })

   //jwt-go 默认校验exp，iss，nbf这三个claims。见标准claims校验方法 => func (c StandardClaims) Valid() error

   if err != nil {
      return err
   }

   if token == nil {
      return errors.New("token is nil")
   }

   fmt.Println("JWT校验成功!")
   if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
      fmt.Printf("Registered Claims => exp: %v, iss: %s\n", int64(claims["exp"].(float64)), claims["iss"])
      fmt.Printf("Private Claims => isAdmin: %v, csrf: %v\n", claims["isAdmin"], int64(claims["csrf"].(float64)))
   }

   return nil
}
```



运行结果：

```shell
发起JWT请求..
playload:map[csrf:126322000 exp:1573662686 isAdmin:true iss:example.com]
生成jwt成功：
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjc3JmIjoxMjYzMjIwMDAsImV4cCI6MTU3MzY2MjY4NiwiaXNBZG1pbiI6dHJ1ZSwiaXNzIjoiZXhhbXBsZS5jb20ifQ.IeoSUjfwXHrbLqdNVboHaTEoIwDOQI4XFMPZLW2Vypg
===================================
测试jwt验证...
JWT校验成功!
Registered Claims => exp: 1573662686, iss: example.com
Private Claims => isAdmin: true, csrf: 126322000

```





### 四、JWT最佳实践——防坑指南 

#### 4.1 如何提高JWT的安全性？

由于JWT是访问受限资源的凭据，获得JWT后需要防止安全的问题。

* 通常JWT会设置时效性，保留规定的一段时间后自动失效。

* 使用Https加密信道进行信息传输。

* 使用Http header 传递JWT
在请求过程中，所发送的JWT通常存放在使用**Bearer**模式的header中。格式如下：

  ``` javascript
  Authorization: Bearer <token>
  ```
  
* 如果JWT未进行加密，则不要将用户敏感信息放在playload。

  > 使用JWT会将令牌或者令牌中包含的所有信息都暴露给用户或者其他地方，即使他们无法篡改它，但也意味着不应该将机密信息放入令牌中（如身份证，电话，地址等用户敏感信息）。
  
  > 如有必要，为了能够减少数据库的查询，可以使用对称/非对称加密算法，加密后再存放到JWT，后端根据预设的解密流程解密即可。

* 一些关键资源的操作，进行二次验证
> 对于一些系统敏感、关键资源的修改，比如转账，删除账号等，需要做二次验证。比如手机短信验证，人脸识别，指纹识别，语音识别，邮箱验证码等。

#### 4.2 浏览器如何安全地存储JWT？
web浏览器在存储JWT，主要关注令牌是否容易被窃取，是否会受到CSRF、XSS攻击。

* 避免将JWT存放在local-storage中。如果您的JWT是存放在cookies内。cookies可以使用httpOnly标志，避免被JavaScript进行访问，减少被窃取的风险。将HttpOnly设置为true，可以避免XSS攻击，但不能防御CSRF攻击，需要加入包含随机值的 `csrf令牌` 才能保障安全。

  > `local-storage` 是一种被广泛使用的离线存储，web存储。如果当前用户对计算机有本地管理员权限，则很容易会被窃取到local-storage的信息。因此，建议不要在local-storage中存储任何敏感的信息。

* 【推荐的实践】在JWT的内容中加入一个随机值作为CSRF令牌，由服务端将该CSRF令牌也保存在cookie中，但设置HttpOnly=false，这样前端Javascript代码就可以取得该CSRF令牌，并在请求API时作为HTTP header传回。服务端在认证时，从JWT中取出CSRF令牌与header中获得CSRF令牌比较，从而实现对CSRF攻击的防护(利用同源策略)。为了防止XSS攻击，服务端接收用户输入数据或前端渲染时，需要对特殊字符进行转义。

  > 如果你的令牌是在`Authorization` 的header中发送的话，那么CORS（Cross-Origin Resource Sharing  跨域访问）就不再是问题了，因为它不使用cookies

#### 4.3 如何应对JWT泄露
不管是基于 Sessions 还是基于 JSON Web Token，一旦密令被盗取，都是一件棘手的事情。
为了防止用户 JWT 令牌泄露而威胁系统安全，你可以在以下几个方面完善系统功能：

* 清除已泄露的令牌：此方案最直接，也容易实现，但同时也引入了外部依赖及单点风险。你需将 JWT 令牌在服务端也存储一份，若发现有异常的令牌存在，则从服务端令牌列表中将此异常令牌清除。当用户发起请求时，强制用户重新进行身份验证，直至验证成功。对于服务端的令牌存储，可以借助 Redis 等缓存服务器进行管理，也可以使用 Ehcache 将令牌信息存储在内存中。
* 敏感操作保护：在涉及到诸如新增，修改，删除，上传，下载等敏感性操作时，定期(30分钟，15分钟甚至更短)检查用户身份，如手机验证码，扫描二维码等手段，确认操作者是用户本人。如果身份验证不通过，则终止请求，并要求重新验证用户身份信息。
* 地域检查：通常用户会在一个相对固定的地理范围内访问应用程序，可以将地理位置信息作为一个辅助来甄别用户的 JWT 令牌是否存在问题。如果发现用户A由经常所在的地区 1 变到了相对较远的地区 2 ，或者频繁在多个地区间切换，不管用户有没有可能在短时间内在多个地域活动(一般不可能)，都应当终止当前请求，强制用户重新进行验证身份，颁发新的 JWT 令牌，并提醒(或要求)用户重置密码。
* 监控请求频率：如果 JWT 密令被盗取，攻击者或通过某些工具伪造用户身份，高频次的对系统发送请求，以套取用户数据。针对这种情况，可以监控用户在单位时间内的请求次数，当单位时间内的请求次数超出预定阈值值，则判定该用户密令是有问题的。例如 1 秒内连续超过 5 次请求，则视为用户身份非法，服务端终止请求并强制将该用户的 JWT 密令清除，然后回跳到认证中心对用户身份进行验证。
* 客户端环境检查：对于一些移动端应用来说，可以将用户信息与设备(手机,平板)的机器码进行绑定，并存储于服务端中，当客户端发起请求时，可以先校验客户端的机器码与服务端的是否匹配，如果不匹配，则视为非法请求，并终止用户的后续请求。

### 参考资料：

[IETF文档#JWT —— ISSN：2070-1721](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#JWTClaimsReg)

https://www.cnblogs.com/yibutian/p/9507866.html

https://cloud.tencent.com/developer/article/1495531