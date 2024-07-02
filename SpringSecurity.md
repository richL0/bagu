核心功能
1. 认证
2. 授权
3. 防护攻击：XSS （跨站脚本攻击 ） CSRF （跨站请求伪造）
4. 与Java Servlet API集成：


# JWT (JSON WebToken)
JWT（JSON Web Token）是一种基于Token的身份验证机制，JWT由三部分组成：头部（Header）、载荷（Payload）和签名（Signature）。
头部：算法信息和令牌类型
载荷：声明信息，例如用户ID、过期时间等
签名：验证令牌是否被篡改的一段数据
### 基于session的认证
服务器生成一个sessionID
客户端把sessionID保存到Cookie中
浏览器请求的header中自动带上sessionID到服务端
服务端校验sessionID是否合法
#### 缺点
服务端压力大：存大量的sessionID
扩展性差：不能扩展到分布式的其他服务器中-> 解决：用redis实现分布式共享
不支持跨域：session要配合cookie实现，cookie默认不支持跨域
CSRF攻击：cookie被拦截，会被中间人伪造请求

## 基于JWT的认证
用户登陆后服务器颁发一个token给客户端
客户端将Token保存在本地
此后的每个请求都在header中带上token
服务器验证Token的有效性
#### 优点
分布式
不需要服务器存储
不依赖cookie
#### 缺点
一旦签发，将很难收回，无法在服务端实现用户登出，只能在客户端删除token，或者等有效期结束

### Token续期
AccessToken & RefreshToken
AccessToken 实现访问数据
RefreshToken 实现AccessToken的续期：如果RefreshToken没过期就颁发一个新的AccessToken
```scss
  +--------+                                           +---------------+
  |        |--(A)------- Authorization Grant --------->|               |
  |        |                                           |               |
  |        |<-(B)----------- Access Token -------------|               |
  |        |               & Refresh Token             |               |
  |        |                                           |               |
  |        |                            +----------+   |               |
  |        |--(C)---- Access Token ---->|          |   |               |
  |        |                            |          |   |               |
  |        |<-(D)- Protected Resource --| Resource |   | Authorization |
  | Client |                            |  Server  |   |     Server    |
  |        |--(E)---- Access Token ---->|          |   |               |
  |        |                            |          |   |               |
  |        |<-(F)- Invalid Token Error -|          |   |               |
  |        |                            +----------+   |               |
  |        |                                           |               |
  |        |--(G)----------- Refresh Token ----------->|               |
  |        |                                           |               |
  |        |<-(H)----------- Access Token -------------|               |
  +--------+           & Optional Refresh Token        +---------------+
               Figure 2: Refreshing an Expired Access Token
```
通过RefreshToken的实现，可以管理Token签发难收回的情况，下次AccessToken失效就可以不颁发新的AccessToken

### Token的拉黑
应用在后台编写代码，标记某个用户的id为黑名单，在应用层实现拉黑


## 签名算法

# Oauth2

### SSO Single Sign-On
Single Sign-On 单点登录，一次登录可以在多个应用中使用，比如淘宝、天猫，公用一个认证服务器
鉴权的时候去访问同一个服务器，就不需要多次登录了

### 授权码模式
![](https://www.ruanyifeng.com/blogimg/asset/2014/bg2014051204.png)
授权服务器给client一个code（类似验证码，一次性的且很快就会过期）
client把code发给应用服务器，应用服务器通过code去访问授权服务器，返回用户的AccessToken和RefreshToken

### 简化模式
![](https://www.ruanyifeng.com/blogimg/asset/2014/bg2014051205.png)
授权服务器直接返回Token给用户，用户把Token给应用服务器
用于只有前端，没有后端的应用
### 其他模式
密码模式：直接把账号密码给应用服务器，应用服务器去请求授权
客户端模式：应用直接用自己的凭证请求授权
# CSRF 跨站请求伪造

[如何防止CSRF攻击？](https://tech.meituan.com/2018/10/11/fe-security-csrf.html)
Gmail邮件中，有一封邮件骗你点一个链接，这个链接里面伪造了一个向gmail的请求（把你所有的邮件都转发到其他邮箱里面），因为你从Gmail点进来的，保存了你的cookie信息，Gmail以为真的是你请求的。这样的过程就是**跨站请求伪造**（Cross-Site Request Forgery，CSRF）
## 解决方案
同源检测
CSRF Token
双重Cookie验证
验证码



# 授权
用户-权限-资源：
用户-组-权限-资源：

基于request的授权：在SecurityFilterChain中添加http的授权
基于注解的授权：在Controller的方法上加入`@PreAuthorize`注解


