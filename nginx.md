速度更快，并发更高（可以处理数以万计的并发请求
扩展性强，配置简单
高可靠性：用多进程主从模式运行，master可以在worker宕机后再拉起一个新的worker
支持热部署
内存消耗小：nginx处理静态文件好
异步非阻塞请求处理：epoll
# 原理

## 网络通信
基于事件模型的异步非阻塞，[[网络#Reactor模式|多Reactor多进程]]模式

# 用途
## 静态资源
为了加速前端页面的响应速度，可以将前端的相关资源，如html，js，css或者图片放到nginx指定目录下
```nginx
server {
	listen       8000;
	listen       somename:8080;
	server_name  somename  alias  another.alias;
	
	location / {     # 配置静态资源
		root   html;
		index  index.html index.htm;
	}
}
```

## Rewrite地址重写
## 反向代理
将请求转发到其他服务器或网络，以提高网站的性能和可靠性。

可以
1. 保护真实的web服务器
2. 缓存数据，请求相同数据时直接返回
3. 节约IP资源：Nginx和源服务器在一个局域网
4. 实现负载均衡
5. 解决Ajax[[JavaWeb#跨域|跨域]]问题
6. 

#### 工作流程
1. 用户通过域名发出访问Web服务器的请求，该域名被DNS服务器解析为反向代理服务器的IP地址；
2. 反向代理服务器接受用户的请求；
3. 反向代理服务器在本地**缓存中查找**请求的内容，找到后直接把内容发送给用户；
4. 如果本地缓存里没有用户所请求的信息内容，反向代理服务器会**代替用户向源服务器请求**同样的信息内容，并把信息内容发给用户，如果信息内容是缓存的还会把它保存到缓存中。

## 负载均衡
[[分布式#负载均衡]]

## web缓存

## 用户认证

## 核心组成
conf配置文件
error.log
access.log

## 请求处理
Nginx以 daemon 形式在后台运行，多进程 + 异步非阻塞 IO 事件模型来处理各种连接请求
进程中包括一个master和多个worker进程，父进程 fork生成多个 worker 子进程



## 配置文件
### 全局块
##### user
配置linux上运行Nginx服务器的用户
实现信息安全，运行Nginx的用户没有权限不能访问其他的文件

##### daemon
`daemon on/off;`
设定Nginx是否以守护进程的方式启动
守护进程是linux后台运行的服务进程，独立于terminal，关闭terminal不会关闭进程

### events块
##### accept_mutex
接收互斥锁，用于解决进程惊群现象
采用了「多 [[网络#Reactor模式|Reactor]] 多进程」，使用互斥锁防止[惊群现象](https://www.cnblogs.com/paul-617/p/15690810.html)

惊群效应指：多进程/线程在同时阻塞等待同一个事件的时候，事件发生就唤醒等待的所有进/线程，但是只能有一个进/线程抢到这个事件并处理，而其他进/线程重新进入休眠状态。

### HTTP块
#### Server块 & location块
```nginx
server{
	listen 80;
	server_name localhost;
	location / {
		root html;
		index index.html index.htm;
	}
	
	error_page 500 502 503 504 /50x.html;
	location = /50x.html{
		root html;
	}
}

```
#### alias与root的区别
*alias*（别名）是一个目录别名。
```nginx
location /123/abc/ {
	root /ABC;
}
# 当请求http://qingshan.com/123/abc/logo.png时
# 会返回 /ABC/123/abc/logo.png文件，即用/ABC 加上 /123/abc。
```

*root*（根目录）是最上层目录的定义。
```nginx
location /123/abc/ {
	alias /ABC;
}
# 当请求http://qingshan.com/123/abc/logo.png时
# 会返回 /ABC/logo.png文件，即用/ABC替换 /123/abc。
```