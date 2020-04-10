---
tags: [全局描述]
---

# Open API 介绍

连连API基于REST标准来设计的，为保证统一和安全，全局编码格式为UTF-8，全局使用https。我们的API具有可预测的面向资源的url，接受表单编码的请求主体，返回json编码的响应，并使用标准的HTTP响应代码、身份验证和请求动词。

为了数据准确性和生产环境数据安全，建议在沙盒环境测试这些接口.

# 版本控制

当我们对API进行向后不兼容的更改时，我们会发布新版本。要使用的版本在URL中指定。集合的当前版本是v1，比如:

    https://api.lianlianglobal.com/payments/v1/...

# 授权认证

在不同的对接场景下连连Open API存在两种认证方式（用户开发者和第三方应用开发者，通常情况下申请用户开发者就好了），都使用http头`Authorization`做认证：

**_ 用户对接模式 _**

### 用户开发者模式

当您在连连这边创建了用户开发者之后，您会收到连连给您返回的`developerId`、`masterToken`（`masterToken`能行使用户所有权限，请您务必安全保管）和`LLP_RSA_PUB_KEY.pem`，身份认证格式如下:

    Authorization: Basic &lt;&lt;Base64.encode(developerId:masterToken)&gt;&gt;

### 第三方应用开发者模式

当您在连连这边创建第三方应用开发者之后，您会收到连连给您返回的`clientId`、`clientSecret`和`LLP_RSA_PUB_KEY.pem`，至于`accessToken`则需要通过OAuth2.0模式向有资源的用户申请，身份认证格式如下:

    Authorization: Bearer &lt;&lt;accessToken&gt;&gt;

# 请求安全

为了请求安全防止重放攻击，连连要求所有请求都得有签名认证，连连在http头定义了`LLPAY-Signature`字段作为签名信息载体，`LLPAY-Signature`头文件中包含了请求包体和响应的epoch时间戳（是指格林威治时间1970年01月01日00时00分00秒起至现在的总秒数）例如：`LLPAY-Signature:t=&lt;&lt;epoch&gt;&gt;,v=&lt;&lt;signature&gt;&gt;`，一个请求的有效时间是5分钟。下面介绍下请求的签名格式：

### 请求签名

2. 对`HTTP请求方式`、`URI`、`请求epoch时间`（单位秒）、`请求包体`的数据按照一定顺序用字符串“&”做拼接后使用对接方的`RSA私钥`通过`SHA256WithRSA`算法做签名并用`Base64编码`，生成的签名字符串（`signature`）和`epoch`时间放入HTTP包头的`LLPAY-Signature`标签中，格式为：

<!---->


    LLPAY-Signature:t=&lt;&lt;epoch&gt;&gt;,v=&lt;&lt;signature&gt;&gt;


**第一步:** 确定签名`payload`

如下字段请用`&`一次连接

- `HTTP_METHOD`: 对应实际接口的方法（统一用大写），如`POST`、`PUT`、`GET`、`DELETE`等；
- `URI`: 请求的URI地址（除去host）.  例如`https://api.sandbox.lianlianglobal.com/collections/v1/merchants`中`/collections/v1/merchants`为URI
- `REQUEST_EPOCH`: 是指格林威治时间1970年01月01日00时00分00秒起至现在的总秒数,该值应与`t`值保持一致
- `REQUEST_PAYLOAD`: 请求包体  `{"currency":"USD"}`
- `QUERY_STRING`: 查询字段例如：`https://api.sandbox.lianlianglobal.com/collections/v1/merchants?attr1=value1&attr2=value2`,其中`QUERY_STRING`=`attr1=value1&attr2=value2`格式化为`attr1%3Dvalue1%26attr2%3Dvalue2`

 `payload`示例:

    POST&/collections/v1/merchants&19879234&{"currency":"USD"}&attr1%3Dvalue1%26attr2%3Dvalue2

**第二部:** 准备 `LLPAY-Signature` 签名头

你会用到以下内容:

- REQUEST_EPOCH (Seconds elapsed since 1970/1/1 00:00:00 GMT as a string)
- 连接字符串 `,`
- payload（第一步的结果）
- your_rsa_pri_key：你的RSA私钥

<!---->

    LLPAY-Signature: t=REQUEST_EPOCH,v=BASE64_ENCODE(SHA256WithRSA.sign(&lt;&lt;payload&gt;&gt;, &lt;&lt;your_rsa_pri_key&gt;&gt;))

**请求示例**

    POST /api/mkt/balance HTTP/1.1 
    Host: gtest-open-api.lianlianpay-inc.com 
    Content-Type: application/json 
    Authorization: Basic WTgzcHNkcFdqY3J0Vml5eHVveTNyWGp2OWpzMjV3aUs6WTgzcHNkcFdqY3J0Vml5eHVveTNyWGp2OWpzMjV3aUs= 
    LLPAY-Signature: t=1574130344,v=cJKgD/EpqNVnITR7yZ8BIev5j1E0ub0VbG4uGA69gR4T1FFc7NzqbiBoDEOBvkQtJXytQd7dY+WDo0Qm0c6gCnRHqIEyBen6SnBk/PjhIn7H93sHMyUEbesJqB6NAzOHA4uVj+8aTfREQWxKaizkDTT1dnrBUZ7KPxz4KKzRXtZ6tEh48HKsA5xqviedc+kpilaFbFSaoJmFj760TV8FB+mKCkZSrvX1Y+4x0bqTVBXAt2kE2Z8vCH16BDtlWGLZRSlWtZWyvpz6F0a/VWYVhoBEmgNFevnYDeAMGB6VEDBE1pZLMnhxfLfz6yu/p1pv1c2N2Yk5YSahQw4lLLiqQQ== 
    Accept: */* 
    Cache-Control: no-cache 
    Content-Length: 18 
    Connection: keep-alive 
    
    {"currency":"USD"} 

### 请求结果签名验证

-   若请求成功返回200，包体格式查看具体接口，对响应包体使用连连支付的RSA私钥用SHA256WithRSA做签名并用Base64编码，生成的签名字符串放入HTTP包头LLPAY-Signature标签中，格式为LLPAY-Signature: t = response_epoch, v = signature。
    其中：
- t=响应时间戳(格林威治时间1970年01月01日00时00分00秒起至现在的总秒数)
- v=BASE64_ENCODE(SHA256WithRSA(RESPONSE_EPOCH&RESPONSE_BODY, LLPAY_RSA_PRIVATE_KEY))

**第一步:** 确定 `payload`

如下字段创建`payload`用 `&` 做连接

- Response Timestamp: 响应时间戳(格林威治时间1970年01月01日00时00分00秒起至现在的总秒数)
- Response Payload: 响应包体，指定为JSON字符串如： `{"currency":"USD"}`

`payload`示例:

    19879234&{"currency":"USD"}

**第二部:** 使用连连的RSA公钥校验签名的有效性

    SHA256WithRSA.verify(LLPAY-Signature, '19879234&{" currency":"USD"}',  LLPAY_RSA_PRIVATE_KEY)

# 响应结果

**成功返回结果示例**

连连通过http状态码来判断请求的结果，一个成功的请求的http状态码为2XX，请求结果为相应的objects对象，例如:

```
HTTP/1.1 200 
status: 200 
Content-Type: application/json 
Content-Length: 61
Connection: keep-alive 
LLPAY-Signature:t=1574130398,v=b0VbG4uGA69gR4T1FFc7NzqbiBoDEOBvkQtJXytQd7dY+WDo0QmgR4T1FFc7NzqbiBoDEOBvkQtJXytQpzMjV3aUs6R4T1FFc7NzqbiBoDEOBvWTgzcHNkcFdqY3J0Vml5eHVc6gCnRHqIEyBen6SnBk/PjhIn7H93sHMyUEbesJqB6NAzOHA4uVj+8aTfREQWxKaizkDTT1dnrBUZ7KPxz4KKzRXtZ6tEh48HKsA5xqWGLZRSlWtZWyvpz6F0a/VWYVhoBEmgNFevkE2Z8vCH16VEDBE1pZ6VEDBE1pZ6BDBE1pZ6VEDBE1DtlWGLnYviedc+kpilaFbFSaoJmFj76==

{"currency":"USD","balance":"12.25"}

```

#### Errors

一个失败的请求会收到4XX类的http状态码表示已知错误内容（具体错误码API文档给出），5XX的状态码表示未知的错误类型：

#### Attributes

**_code_** _number_
失败码类型，数字类型，用于快速定位错误类型

**_message_** _string_
失败描述

**失败返回结果示例**

    HTTP/1.1 400
    status: 400
    Date: Tue, 19 Nov 2019 02:26:38 GMT
    Content-Type: application/json
    Content-Length: 77
    Connection: keep-alive
    
    {"code":"999995","message":"[holderType] is invalid"}

### HTTP状态码一览表

| CODE               | DESCRIPTION                |
| ------------------ | -------------------------- |
| 400                | 请求错误，例如：参数错误               |
| 401                | 授权认证失败或者是签名认证失败            |
| 403                | 请求未授权                      |
| 404                | 资源未找到，这里的资源指的是实际的Objects对象 |
| 500, 502, 503, 504 | 系统错误                       |

# 请求幂等保证

实际运行场景中，由于网络原因或者其他原因导致的网络中断是不可避免的，所以连连这边特意设计了请求幂等保证操作，所有的POST、PUT、DELETE请求都可以做幂等校验，幂等请求认证成功之后，会返回最初的请求结果（5XX未知异常类型的错误除外）。

你需要在http头加入`Idempotency-Key`以便让系统失败你的幂等请求：

    Idempotency-Key:&lt;&lt;unique id for client &gt;&gt;

# Request IDs

每个API请求都有一个关联的请求标识符。您可以响应头找到`Request-Id`下这个键值。

# 字段命名规范

连连所有的字段命名规范为驼峰式:

    https://api...com/resource/?filterBy="filter"
    
    {
      "storeName": "My Store",
      "kycStatus": "success"
    }

# Webhook

你可以配置webhook地址来接收连连这边的回调信息（`event`），具体的回调信息（`event`）在相应的接口中定义
