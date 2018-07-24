---
title: CORS
date: 2017-06-22 11:47:07
categories: [计算机网络]
tags: [计算机网络, 前端基础, 跨域]
---

## 概述

跨域资源共享标准新增了一组HTTP首部字段，允许服务器声明哪些源站有权限访问哪些资源。另外，规范要求，对那些可能对服务器数据产生副作用的HTTP请求方法（特别是GET以外的HTTP请求，或者搭配某些MINE类型的POST请求），浏览器必须首先使用OPTIONS方法发起一个预检请求（preflightrequest），从而获知服务器是否允许该跨域请求。服务器确认允许之后，才发起实际的HTTP请求。在预检请求的返回中，服务器端也可以通知客户端，是否需要携带身份凭证（包括Cookies和HTTP相关数据）。

### 简单请求

某些请求不会触发CORS预检请求，本文称这样的请求为“简单请求”。若请求满足所有下述条件，则该请求可视为“简单请求”：

- 使用下列方法之一：
- GET
- HEAD
- POST

- HTTP的头信息不超过以下几种字段：
- Accept
- Accept-Language
- Content-Language
- Content-Type为application/x-www-form-urlencoded、multipart/form-data、text/plain
- Last-Event-ID

客户端和服务器之间使用CORS首部字段来处理跨域权限：

```
// 请求报文
GET /resources/public-data HTTP/1.1
HOST: bar.other
User-Agent: Mozilla/5.0...
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip,deflate
Accept-Charset: ISO-8859-1,utf-8,q=0.7,*;q=0.7
Connection: keep-alive
Referer: http://foo.example/examples/access-control/simpleXSInvocation.html
Origin: http://foo.example

// 响应报文
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 00:23:53 GMT
Server: Apache/2.0.61
Access-Control-Allow-Origin: *
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Transfer-Encoding: chunked
Content-Type: application/xml
[XML Data]
```

请求首部字段`Origin`表明该请求来源于`http://foo.example`，响应中携带了响应首部字段`Access-Control-Allow-Origin`。使用`Origin`和`Access-Control-Allow-Origin`就可以完成最简单的访问控制。

服务器返回的`Access-Control-Allow-Origin: *`表明该资源可以被任意外域访问，如果服务端仅允许来自`http://foo.example`的访问，该首部字段的内容如下：`Access-Control-Allow-Origin: http://foo.example`。

### 预检请求

“需预检的请求”要求必须首先使用OPTIONS方法发起一个预检请求到服务器，以获知服务器是否允许该实际请求。“预检请求”的使用，可以避免跨域请求对服务器的用户数据产生未预期的影响。

当请求满足下述任一条件时，应首先发送预检请求：

- 使用了下面任一HTTP方法：
- PUT
- DELETE
- CONNECT
- OPTIONS
- TRACE
- PATCH

- 人为设置了对CORS安全的首部字段集合之外的其他首部字段，或`Content-Type`不属于简单请求的三个值之一。

首先发起预检请求：

```
// 请求报文
OPTIONS /resource/post-here/ HTTP/1.1
...
Origin: http://foo.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: X-PINGOTHER, Content-Type

// 响应报文
HTTP/1.1 200 OK
...
Access-Control-Allow-Origin: http://foo.example
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: X-PINGOTHER, Content-Type
Access-Control-Max-Age: 86400
```

预检请求完成之后，发送实际请求：

```
// 请求报文
POST /resources/post-here/ HTTP/1.1
...
X-PINGOTHER: pingpong
Content-Type: text/xml; charset=UTF-8
Origin: http://foo.example

// 响应报文
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://foo.example
```

浏览器检测到，从JavaScript中发起的请求需要被预检，首先发送了一个使用`OPTIONS`方法的“预检请求”。`OPTIONS`是HTTP/1.1协议中定义的方法，用以从服务器获取更多信息。该方法不会对服务器资源产生影响。预检请求中同时携带了下面两个首部字段：

```
Access-Control-Request-Method: POST
Access-Control-Request-Headers: X-PINGOTHER, Content-Type
```

- 首部字段`Access-Control-Request-Method`告知服务器，实际请求将使用`POST`方法。
- 首部字段`Access-Control-Request-Headers`告知服务器，实际请求将携带两个自定义请求首部字段：`X-PINGOTHER`与`Content-Type`。服务器根据此决定，该实际请求是否被允许。

预检请求的响应，表明服务器将接受后续的实际请求：

```
Access-Control-Allow-Origin: http://foo.example
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: X-PINGOHTER, Content-Type
Access-Control-Max-Age: 86400
```

- 首部字段`Access-Control-Allow-Methods`表明服务器允许客户端使用`POST`、`GET`和`OPTIONS`方法发起请求；
- 首部字段`Access-Control-Allow-Headers`表明服务器允许请求中携带字段`X-PINGOTHER`与`Content-Type`；
- 首部字段`Access-Control-Max-Age`表明该响应的有效时间为86400s，也就是24h。在有效时间内，浏览器无须为同一请求再次发送预检请求。请注意，浏览器自身维护了一个最大有效时间，如果该首部字段的值超过了最大有效时间，将不会生效。

### 附带身份凭证的请求

Fetch与CORS的一个有趣的特性是，可以基于HTTP Cookies和HTTP认证信息发送身份凭证。一般而言，对于跨域XMLHTTPRequest或Fetch请求，浏览器不会发送身份凭证信息。如果要发送凭证信息，需要设置`XMLHTTPRequest`的特殊标识位`withCredentials`：

```
var invocation = new XMLHTTPRequest()
var url = 'http://bar.other/resources/credentialed-content/'
function callOtherDomain () {
  if (invocation) {
    invocation.open('GET', url, true)
    invocation.widthCredentials = true
    invocation.onreadystatechange = handler
    invocation.send()
  }
}
```

将`XMLHTTPRequest`的`widthCredentials`标识设置为`true`，从而向服务器发送Cookies。因为这是一个简单GET请求，所以浏览器不会对其发起“预检请求”。但是，如果服务器端的响应中未携带`Access-Control-Allow-Credentials: true`，浏览器将不会把响应内容返回给请求的发送者。

对于附带身份凭证的请求，服务器不得设置`Access-Control-Allow-Origin`为`*`，请求将会失败；而将`Access-Control-Allow-Origin`设置为`http://foo.example`，请求将成功执行。

## HTTP首部字段

### 响应首部字段

#### Access-Control-Allow-Origin


```
Access-Control-Allow-Origin: <origin> | *
```

其中，`origin`参数的值指定了允许访问该资源的外域URI。对于不需要携带身份凭证的请求，服务器可以指定该字段的值为通配符，表示允许来自所有域的请求。

如果服务器端指定了具体的域名而非`*`，那么响应首部中的`Vary`字段的值必须包含`Origin`，这将告诉客户端：服务器对不同的源站返回不同的内容。

#### Access-Control-Expose-Headers

`Access-Control-Expose-Headers`首部字段指定了服务器允许的首部字段集合。

```
Access-Control-Expose-Headers: X-My-Custorm-Header, X-Another-Custom-Header
```

服务器允许请求中携带`X-My-Custom-Header`和`X-Another-Custom-Header`这两个字段。

#### Access-Control-Max-Age

`Access-Control-Max-Age`首部字段指明了预检请求的响应的有效时间。

```
Access-Control-Max-Age: <delta-seconds>
```

delta-seconds表示该响应在多少秒内有效。

#### Access-Control-Allow-Credentials

`Access-Control-Allow-Credentials`首部字段用于预检请求的响应，表明服务器是否允许credentials标识设置为`true`的请求。注意：简单GET请求不会被预检，如果此类请求的响应中不包含该字段，浏览器不会将响应返回给请求的调用者。

```
Access-Control-Allow-Credentials: true
```

#### Access-Control-Allow-Methods

`Access-Control-Allow-Methods`首部字段用于预检请求的响应，其指明了实际请求所允许使用的HTTP方法。

```
Access-Control-Allow-Methods: <method>[, <method>]*
```

#### Access-Control-Allow-Headers

`Access-Control-Allow-Headers`首部字段用于预检请求的响应，其指明了实际请求中允许携带的首部字段。

```
Access-Control-Allow-Headers: <field-name>[, <field-name>]*
```

### 请求首部字段

这些首部字段无需手动设置。当开发者使用XMLHTTPRequest对象发起跨域请求时，他们已经被准备就绪。

#### Origin

`Origin`首部字段表明预检请求或实际请求的源站。

```
Origin: <origin>
```

`origin`参数的值为源站URI。它不包含任何路径信息，只是服务器名称。注意：不管是否为跨域请求，`Origin`字段总是被发送。

#### Access-Control-Request-Method

`Access-Control-Request-Method`首部字段用于预检请求。其作用是，将实际请求所使用的HTTP方法告诉服务器。

```
Access-Control-Request-Method: <method>
```

#### Access-Control-Request-Headers

`Access-Control-Request-Headers`首部字段用于预检请求。其作用是，将实际请求所携带的首部字段告诉服务器。

```
Access-Control-Request-Headers: <field-name>[, <field-name>]*
```

## 参考文献

- MOZILLA文档，[HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)
