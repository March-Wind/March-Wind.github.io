---
title: JWT登录系统
# subtitle: 没有方法的阅读源码，会很痛苦
date: 2024-6-6 18:05:24
categories: nodejs
index_img: /img/node.js_logo..png
# math: true
# banner_img: /img/read-code-index.png
---

## 🌐 JWT 登录系统

### 📁 登录认证主要三个方式

1. Cookie-Session 方式：用户凭借用户名和密码登录，服务端生成一个 session 来记录用户状态，并存在数据库，然后生成一个 session id，服务器将这个 session id 通过 cookie 返回给浏览器，浏览器后续请求都带上 cookie，服务端提取 cookie 里的 session id，根据 session id 去数据库获取用户信息。
2. [JWT 认证](https://www.rfcreader.com/#rfc7519)：用户登录后，服务端生成 JWT(Json Web Token)，里面包含用户的信息，返回给客户端，客户端妥善保存，每次请求的时候带上 JWT，服务器读取 JWT 进行解码直接获取用户信息。
3. [OAuth 认证](https://www.rfcreader.com/#rfc6749)： 主要用户第三方应用认证，比如现在各个网站都提供的微信连登，就是典型的 ​​OAuth 2.0 授权协议 ​​。

### 优缺点比较

| 项目        | Cookie-Session                                                 | JWT                                                            | OAuth 2.0                                                      |
| ----------- | -------------------------------------------------------------- | -------------------------------------------------------------- | -------------------------------------------------------------- |
| 主要用途    | 认证                                                           | 认证                                                           | 第三方平台授权（如微信/Google 登录）                           |
| 状态管理 ​​ | ​​ 有状态 ​​。服务器端需要存储 Session。                       | ​​ 无状态 ​​。所有状态都存储在令牌本身。                       | ​​ 框架/协议 ​​，定义授权流程。其颁发的令牌可以是 JWT 格式。） |
| 通信方式    | `Set-Cookie` + `Cookie`                                        | `Authorization: Bearer <jwt>`                                  | `Authorization Code` / `Bearer Token`                          |
| 存储位置    | 客户端 ​​：Cookie。服务端 ​​：Session（内存、数据库、Redis）。 | ​​ 客户端 ​​。Token 通常存储在 localStorage 或内存中。         | 客户端 ​​。获取到的 Access Token 由客户端存储。                |
| 扩展性      | 较差 ​​。在分布式环境下需要做 Session 共享/粘滞。              | ​​ 极佳 ​​。天然支持分布式，任何服务节点只需密钥即可验证令牌。 | 极佳 ​​。为分布式和第三方应用授权设计。                        |
| 安全性      | 易受 ​​CSRF​​ 攻击，需额外防护。Session ID 泄露有风险。        | Token 泄露即拥有权限，需妥善存储。有效期管理是关键。           | 流程复杂，实现不当易引入漏洞。Refresh Token 机制提升了安全性   |
| RFC 依据    | RFC 6265                                                       | RFC 7519                                                       | RFC 6749 / 6750                                                |

现在企业一般都使用 JWT，因为兼容多端，减轻服务器压力。比如最近大火的 Open AI 的 API 就是使用 JWT，他的传送方式是`Authorization: Bearer <token>`，它的 token 是以 sk-开头的 token，是由 JWT 又做了一层安全编码。

### JWT 详解

JWT 是由三个 Base64Url 编码部分组成的字符串，以点号分隔：
`Header.Payload.Signature`
示例：
`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6IjEzNTgzMTA2NzcxQDE2My5jb20iLCJuYW1lIjp7ImZpcnN0TmFtZSI6IuWImCIsImxhc3ROYW1lIjoi5b-X5rSLIn0sInV1aWQiOiJiMWZlZTI5Ny0yMGEwLTQ4MzEtODhlYy0wY2FkM2M2ODYyN2UiLCJpZCI6IjY1NDBiOGY0ZDQ4OGNmZGQ5NWZhNGJkOSIsImlhdCI6MTcwMDQ1MDM3MywiZXhwIjoxNzAwNDUzOTczfQ.nUwSUzSEFUmaCuc9o2BsGbs6ldgFCplIM8RMUf0u1ys`。

Header 和 Payload 都是 base64 后的结果，Signature 是 Header + Payload 的签名。签名的目的是为了防止 token 被篡改，签名的算法是 Header 里的 alg 字段指定的算法，一般是 HMAC SHA256 或者 RSA，这里用的是 HMAC SHA256，签名的时候需要一个密钥，这个密钥只有服务端知道。 服务端接收的 JWT，然后根据密匙和 Header 和 Payload 数据生成签名，如果一致就说明数据没有被修改。

Header 示例：

    ```
    {
      "alg": "HS256",
      "typ": "JWT"
    }
    ```

Payload 示例：

    ```
    {
      "sub": "1234567890",
      "name": "John Doe",
      "admin": true,
      "iat": 1516239022
    }
    ```

Signature 示例：`6a5f6ac179d9ba9ccb2f1291f6285bba2913a842b2d212abcc5a8c0bfccb`任意字符串。

JWT 字符串的 Header 和 Payload 部分就是简单的 base64 编码，直接 base64 解码，就能获取原始信息。所以 JWT 还要做一层防护性的编码，并且生成 JWT 之前 Payload 部分也做一层防护性编码.

### 登录设计

一般使用双令牌认证，因为安全性比较高。一个叫访问令牌(Access Token)，一个叫刷新令牌(Refresh Token)。Access Token 过期后，可使用 Refresh Token 静默获取新 Token，无需频繁登录。

主要流程：登录 → 获取 Access Token 和 Refresh Token → 请求 API（Access Token 过期 → 用 Refresh Token 刷新 → 获取新 Access Token；如果 Refresh Token 过期就重新登录）。一般 Refresh Token 有效期是 30 天，也可以更久，但是不建议永久，Access Token 有效期一般几小时，高安全性要求的应用（如银行、支付系统）​，一般 5 分钟。
![image](/img2/jwt_1.png)

### JWT 研究资料

- [jwt 在线编解码网站](https://jwt.io/)
- [jwt 介绍](https://jwt.io/introduction)
