# Basic Auth

在HTTP中，基本认证（Basic access authentication）是一种用来允许网页浏览器或其他客户端程序在请求时提供用户名和口令形式的身份凭证的一种登录验证方式。

**优点**

* 基本认证的一个优点是基本上所有流行的网页浏览器都支持基本认证。基本认证很少在可公开访问的互联网网站上使用，有时候会在小的私有系统中使用（如路由器网页管理接口）。后来的机制HTTP摘要认证是为替代基本认证而开发的，允许密钥以相对安全的方式在不安全的通道上传输。
* 程序员和系统管理员有时会在可信网络环境中使用基本认证，使用Telnet或其他明文网络协议工具手动地测试Web服务器。这是一个麻烦的过程，但是网络上传输的内容是人可读的，以便进行诊断。

**缺点**

* 虽然基本认证非常容易实现，但该方案创建在以下的假设的基础上，即：客户端和服务器主机之间的连接是安全可信的。特别是，如果没有使用SSL/TLS这样的传输层安全的协议，那么以明文传输的密钥和口令很容易被拦截。该方案也同样没有对服务器返回的信息提供保护。
* 现存的浏览器保存认证信息直到标签页或浏览器被关闭，或者用户清除历史记录。HTTP没有为服务器提供一种方法指示客户端丢弃这些被缓存的密钥。这意味着服务器端在用户不关闭浏览器的情况下，并没有一种有效的方法来让用户注销。


## 访问形式

**1、使用浏览器**

在使用浏览器访问设置了 HTTP Basic Auth 的服务器时，会弹出对话框，输入用户名和密码即可。

**2、使用 HTTP Client 工具**

```CLI
http://user:passwd@httpbin.org/basic-auth/user/passwd
```


## 原理

这一个典型的HTTP客户端和HTTP服务器的对话，服务器安装在同一台计算机上（localhost），包含以下步骤：

1. 客户端请求一个需要身份认证的页面，但是没有提供用户名和口令。这通常是用户在地址栏输入一个URL，或是打开了一个指向该页面的链接。
2. 服务端响应一个401应答码，并提供一个认证域。
3. 接到应答后，客户端显示该认证域（通常是所访问的计算机或系统的描述）给用户并提示输入用户名和口令。此时用户可以选择确定或取消。
4. 用户输入了用户名和口令后，客户端软件会在原先的请求上增加认证消息头，然后重新发送再次尝试。
   * 其名称与值的形式是这样的，该加密过程是可以反解出来的：
     ```CLI
     Authorization: Basic base64encode(username+":"+password)   
     ```

注意:客户端有可能不需要用户交互，在第一次请求中就发送认证消息头。

没有认证信息的请求响应如下：

```CLI
GET /private/index.html HTTP/1.0
Host: localhost

HTTP/1.0 401 Authorization Required
Server: HTTPd/1.0
Date: Sat, 27 Nov 2004 10:18:15 GMT
WWW-Authenticate: Basic realm="Secure Area"
Content-Type: text/html
Content-Length: 311

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
 "http://www.w3.org/TR/1999/REC-html401-19991224/loose.dtd">
<HTML>
  <HEAD>
    <TITLE>Error</TITLE>
    <META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=ISO-8859-1">
  </HEAD>
  <BODY><H1>401 Unauthorized.</H1></BODY>
</HTML>

# 输入用户名和密码后浏览器重新发送请求
GET /private/index.html HTTP/1.0
Host: localhost
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
```

Response 中的 WWW-Authenticate 字段会指示浏览器弹出询问用户名和密码的提示框。


# Bearer Auth




# JWT


# OAuth 2.0

OAuth是一个关于授权（authorization）的开放网络标准，在全世界得到广泛应用，目前的版本是2.0版。


## 应用场景

有一个"云冲印"的网站，可以将用户储存在Google的照片，冲印出来。用户为了使用该服务，必须让"云冲印"读取自己储存在Google上的照片。

问题是只有得到用户的授权，Google才会同意"云冲印"读取这些照片。那么，"云冲印"怎样获得用户的授权呢？

传统方法是，用户将自己的Google用户名和密码，告诉"云冲印"，后者就可以读取用户的照片了。这样的做法有以下几个严重的缺点：

* "云冲印"为了后续的服务，会保存用户的密码，这样很不安全。
* Google不得不部署密码登录，而我们知道，单纯的密码登录并不安全。
* "云冲印"拥有了获取用户储存在Google所有资料的权力，用户没法限制"云冲印"获得授权的范围和有效期。
* 用户只有修改密码，才能收回赋予"云冲印"的权力。但是这样做，会使得其他所有获得用户授权的第三方应用程序全部失效。
* 只要有一个第三方应用程序被破解，就会导致用户密码泄漏，以及所有被密码保护的数据泄漏。

OAuth就是为了解决上面这些问题而诞生的。


## 名词定义

在详细讲解OAuth 2.0之前，需要了解几个专用名词。

* Third-party application：第三方应用程序，本文中又称"客户端"（client），即上一节例子中的"云冲印"。
* HTTP service：HTTP服务提供商，本文中简称"服务提供商"，即上一节例子中的Google。
* Resource Owner：资源所有者，本文中又称"用户"（user）。
* User Agent：用户代理，本文中就是指浏览器。
* Authorization server：认证服务器，即服务提供商专门用来处理认证的服务器。
* Resource server：资源服务器，即服务提供商存放用户生成的资源的服务器。它与认证服务器，可以是同一台服务器，也可以是不同的服务器。

OAuth的作用就是让"客户端"安全可控地获取"用户"的授权，与"服务商提供商"进行互动。


## OAuth的思路

OAuth在"客户端"与"服务提供商"之间，设置了一个授权层（authorization layer）。"客户端"不能直接登录"服务提供商"，只能登录授权层，以此将用户与客户端区分开来。"客户端"登录授权层所用的令牌（token），与用户的密码不同。用户可以在登录的时候，指定授权层令牌的权限范围和有效期。

"客户端"登录授权层以后，"服务提供商"根据令牌的权限范围和有效期，向"客户端"开放用户储存的资料。


## OAuth的授权流程

![oauth2 flow](images/oauth2_flow.png "oauth2 flow")

* （A）用户打开客户端以后，客户端要求用户给予授权。
* （B）用户同意给予客户端授权。
* （C）客户端使用上一步获得的授权，向认证服务器申请令牌。
* （D）认证服务器对客户端进行认证以后，确认无误，同意发放令牌。
* （E）客户端使用令牌，向资源服务器申请获取资源。
* （F）资源服务器确认令牌无误，同意向客户端开放资源。


## 客户端的授权模式

客户端必须得到用户的授权（authorization grant），才能获得令牌（access token）。OAuth 2.0定义了四种授权方式。

* 授权码模式(Authorization Code)，适用于一般服务器端应用
* 简化模式(Implicit)，适用于纯网页端应用，不过现在推荐使用 PKCE 作为替代
* 密码模式(Resource owner password credentials)
* 客户端模式(Client credentials)


### 授权码模式

授权码模式（authorization code）是功能最完整、流程最严密的授权模式。它的特点就是通过客户端的后台服务器，与"服务提供商"的认证服务器进行互动。

![oauth2 authorization code flow](images/oauth2_authorization_code_flow.png "oauth2 authorization code flow")

它的步骤如下：

* 首先微信用户点击豆瓣网微信授权登录按钮后，豆瓣网会将请求通过URL重定向的方式跳转至微信用户授权界面；
* 此时微信用户实际上是在微信上进行身份认证，与豆瓣网并无交互了，这一点非常类似于我们使用网银支付的场景；
* 用户使用微信客户端扫描二维码认证或者输入用户名密码后，微信会验证用户身份信息的正确性，如正确，则认为用户确认授权微信登录豆瓣网，此时会先生成一个临时凭证，并携带此凭证通过用户浏览器将请求重定向回豆瓣网在第一次重定向时携带的callBackUrl地址；
* 之后用户浏览器会携带临时凭证code访问豆瓣网服务，豆瓣网则通过此临时凭证再次调用微信授权接口，获取正式的访问凭据access_token；
* 在豆瓣网获取到微信授权访问凭据access_token后，此时用户的授权基本上就完成了，后续豆瓣网要做的只是通过此token再访问微信提供的相关接口，获取微信允许授权开发的用户信息，如头像，昵称等，并据此完成自身的用户逻辑及用户登录会话逻辑；


### 简化模式

简化模式是对授权码模式的简化，用于在浏览器中使用脚本语言如JS实现的客户端中，它的特点是不通过客户端应用程序的服务器，而是直接在浏览器中向认证服务器申请令牌，跳过了“授权码临时凭证”这个步骤。其所有的步骤都在浏览器中完成，令牌对访问者是可见的，且客户端不需要认证。如果我们使用此种授权方式来实现微信登录豆瓣网的过程的话，流程如下：

![oauth2 implicit grant flow](images/oauth2_implicit_grant_flow.png "oauth2 implicit grant flow")

从上面的流程中可以看到在第4步用户完成授权后，认证服务器是直接返回了access_token令牌至用户浏览器端，而并没有先返回授权码，然后由客户端的后端服务去通过授权码再去获取access_token令牌，从而省去了一个跳转步骤，提高了交互效率。

但是由于这种方式访问令牌access_token会在URL片段中进行传输，因此可能会导致访问令牌被其他未经授权的第三方截取，所以安全性上并不是那么的强壮。


### 密码模式

在密码模式中，用户需要向客户端提供自己的用户名和密码，客户端使用这些信息向“服务提供商”索要授权。这相当于在豆瓣网中使用微信登录，我们需要在豆瓣网输入微信的用户名和密码，然后由豆瓣网使用我们的微信用户名和密码去向微信服务器获取授权信息。

这种模式一般用在用户对客户端高度信任的情况下，因为虽然协议规定客户端不得存储用户密码，但是实际上这一点并不是特别好强制约束。就像在豆瓣网输入了微信的用户名和密码，豆瓣网存储不存储我们并不是很清楚，所以安全性是非常低的。因此一般情况下是不会考虑使用这种模式进行授权的。

假设微信登录豆瓣网的过程使用了此种客户端授权模式，其流程如下：

![oauth2 password credentials flow](images/oauth2_password_credentials_flow.png "oauth2 password credentials flow")

以上模式逻辑主要在服务器端实现，无法避免客户端应用程序对授权方身份信息的存储，所以一般情况下是不会采用这种授权模式的。


## 客户端模式

客户端模式是指客户端以自己的名义，而不是以用户的名义，向“服务提供方”进行认证。严格地说，客户端模式并不属于OAuth2.0协议所要解决的问题。在这种模式下，用户并不需要对客户端授权，用户直接向客户端注册，客户端以自己的名义要求“服务提供商”提供服务。

这种类似于之前共享单车比较火的情况下，出现的全能车的模式，即用户向全能车注册，全能车以自己平台设立的共享账户去开各种品牌的单车。

综上所述，虽然在OAuth2.0协议中定义了四种客户端授权认证模式，但是实际上大部分实际应用场景中使用的都是授权码（authorization code）的模式，如微信开放平台、微博开放平台等使用的基本都是授权码认证模式。


## KONG OAuth 2.0 Authentication

**1. 创建路由**

路由的path设置为/inner/apis/gateway/bin

**2. 添加插件**

```CLI
 curl -X POST \
   --url http://127.0.0.1:8001/routes/route_test/plugins/ \
   --data "name=oauth2" \
   --data "config.scopes=email, phone, address" \
   --data "config.mandatory_scope=true" \
   --data "config.enable_authorization_code=true"
```

response结果中包含 provision_key , provision_key 将被使用在web应用和Kong网关Z之间的API通讯,来确保通讯的安全.

```CLI
{
    "route_id": "2c0c8c84-cd7c-40b7-c0b8-41202e5ee50b",
    "value": {
        "scopes": [
            "email",
            "phone",
            "address"
        ],
        "mandatory_scope": true,
        "provision_key": "2ef290c575cc46eec61947aa9f1e67d3",
        "hide_credentials": false,
        "enable_authorization_code": true,
        "token_expiration": 7200
    },
    "created_at": 1435783325000,
    "enabled": true,
    "name": "oauth2",
    "id": "656954bd-2130-428f-c25c-8ec47227dafa"
}
```

**3. 创建消费者**

```CLI
curl -X POST \
  --url "http://127.0.0.1:8001/consumers/" \
  --data "username=thefosk"
```

**4. 创建app**

为您的开发者账号添加一个app，app的名字为“Hello World App”。

```CLI
curl -X POST \
  --url "http://127.0.0.1:8001/consumers/thefosk/oauth2/" \
  --data "name=Hello World App" \
  --data "redirect_uris[]=http://konghq.com/"
```

response 结果如下:

```CLI
{
    "consumer_id": "a0977612-bd8c-4c6f-ccea-24743112847f",
    "client_id": "318f98be1453427bc2937fceab9811bd",        # 默认生成，可以作为name
    "id": "7ce2f90c-3ec5-4d93-cd62-3d42eb6f9b64",
    "name": "Hello World App",
    "created_at": 1435783376000,
    "redirect_uri": "http://konghq.com/",
    "client_secret": "efbc9e1f2bcc4968c988ef5b839dd5a4"     # 默认生成，可以作为password
}
```

**5. 客户端获取 Access token-客户端凭证模式**

```CLI
curl -X POST \
  --url "https://127.0.0.1:8443/inner/apis/gateway/bin/oauth2/token" \
  --header "Host: mockbin.org" \
  --data "grant_type=password" \                                # 采用basic Auth，请求中添加用户名和密码，对应的为app的client_id和client_secret
  --data "scope= xxxx " \                                       # auth2插件创建时配置
  --data "authenticated_useid= xxxx  " \
  --data "provision_key= 2ef290c575cc46eec61947aa9f1e67d3 " \   # auth2插件创建返回值
  --data "state = xxx " \
  --insecure

{
    "refresh_token": "N8YXZFNtx0onuuR7v465nVmnFN7vBKWk",
    "token_type": "bearer",
    "access_token": "njVmea9rlSbSUtZ2wDlHf62R7QKDgDhG",
    "expires_in": 7200
}
```

**6. 客户端获取 Access token-授权码模式**

6.1 获取CODE

```CLI
curl -X POST \
  --url "https://127.0.0.1:8443/inner/apis/gateway/bin/oauth2/authorize" \
  --header "Host: mockbin.org" \
  --data "client_id= xxxx " \                                   # 创建app时获取
  --data "scope= xxxx " \                                       # auth2插件创建时配置
  --data "authenticated_useid= xxxx  " \
  --data "provision_key= 2ef290c575cc46eec61947aa9f1e67d3 " \   # auth2插件创建返回值
  --data "response_type = code " \
  --data "redirect_uri = http://konghq.com/" \                  # 创建app时设置
  --data "state = xxx " \
  --insecure

{
	"redirect_uri = http://konghq.com/oauth2?code=xxxxxx&state=xxx
}
```

6.2 使用CODE获取ACCESSTOKEN

```CLI
curl -X POST \
  --url "https://127.0.0.1:8443/inner/apis/gateway/bin/oauth2/token" \
  --header "Host: mockbin.org" \
  --data "client_id= xxxx " \                                   # 创建app时获取
  --data "client_secret = xxxx " \                              # 创建app时获取
  --data "grant_type = authorization_code " \                  
  --data "scope= xxxx " \                                       # auth2插件创建时配置
  --data "code = xxxx  " \                                      # 上一步获取
  --data "redirect_uri = http://konghq.com/" \                  # 创建app时设置
  --data "state = xxx " \
  --insecure
  
{
    "refresh_token": "N8YXZFNtx0onuuR7v465nVmnFN7vBKWk",
    "token_type": "bearer",
    "access_token": "njVmea9rlSbSUtZ2wDlHf62R7QKDgDhG",
    "expires_in": 7200
}
```

**7. 调用API**

```CLI
curl -X GET \
  --url "http://127.0.0.1:8000/mock" \
  --header "Host: mockbin.org" \
  --header "Authorization: bearer njVmea9rlSbSUtZ2wDlHf62R7QKDgDhG"    # 携带access token访问api
```

kong在转发请求是，会添加如下header：

```CLI
    "x-consumer-id": "77e3f7ca-a969-48bb-a6d0-4a104ea7ad1e",
    "x-consumer-username": "thefosk",
    "x-authenticated-scope": "email address",
```
