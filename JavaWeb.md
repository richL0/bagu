# js
脚本语言  不需要编译

# Vue 
前端框架  免除了原生js的DOM操作
基于MVVM思想 实现数据的双向绑定
> [!Question] 什么是双向绑定？
双向绑定是指当*数据模型发生变化时，视图会自动更新，同时当视图发生变化时，数据模型也会相应更新*。这样可以减少手动操作DOM和数据的代码，简化开发过程。

# Ajax
*A*synchronous *J*avaScript *A*nd *X*ML  异步JavaScript 和 XML
用于：1. 搜索框的动态推荐热词

```js
var xhr = XMLHttpRequist();
xhr.open('GET', 'http://xxx.xxx/x/x');
```
## Axios
Ajax 写起来还是麻烦: 使用Axios
```xml
<script src = "js/axios-xxx/js"></script>
```

```js
function get(){
	axios({
		method:"get",
		url: "http://xxx.xxx/x"
	}).then( result =>{
		console.log(result.data);
	})
}

function post(){
	axios({
		method:"post",
		url: "http://xxx.xxx/x",
		data: "id=1"
	}).then( result =>{
		console.log(result.data);
	})
}
```

> [!info] 别名
```js
axios.get("https://xx.xxx/x").then(result=>{
	console.log(result.data)
})
axios.post("https://xx.xxx/x", "id=1").then(result=>{
	console.log(result.data)
})
```

## 前后端分离
混合开发: 后端开发需要考虑前端代码
分离开发: 使用接口 (定义一份**接口文档**)
![[接口文档(示例)]]

### YApi 接口管理平台
1. api管理
2. Mock服务 -> 实现模拟数据的测试


# 打包部署
打包: 使用NPM `build`脚本实现
部署: 使用nginx
> [!info] nginx 
> 是一个轻量级的Web服务器/反向代理服务器
特点是占用内存小  并发能力强







## 

# Maven 
1. 项目依赖管理的资源(jar包)
	1. 只需要修改`pom.xml`
2. 统一项目结构
	1. 目录结构相同
	2. 所有的ide通用(eclipse 和 idea)
3. 标准化的项目构建
	1. 快速完成编译  打包  发布
	2. 支持跨平台 (linux windows)

### 仓库
各种jar包存储仓库
顺序: 本地 <-> (远程) <-> 中央仓库

### 依赖管理
每个项目可以通过修改配置文件 设置项目依赖
#### 依赖传递
![[Pasted image 20240411233308.png]]

#### 排除依赖
通过配置实现  排除某个依赖中不想依赖的子包
```xml
<dependency>
	<groupId>xxx.x.x</groupId>
	<artifactId>x</artifactId>
	<version>xxx</version>
	<!--排除依赖-->
	<exclusions>
		<exclusion>
			<groupId>xxx</groupId>
			<artifactId>xxx</artifactId>
		</exclusion>
	</exclusions>
</dependency>

```

#### 依赖范围
只在某包里依赖
如 只在 test中依赖`junit`
```xml
<dependency>
	<groupId>xxx.x.x</groupId>
	<artifactId>x</artifactId>
	<version>xxx</version>
	<!--依赖范围-->
	<scope>test</scope>
</dependency>
```

#### 生命周期
maven 生命周期
clean > compile > test > package > install
1. clean - 清除输出目录
2. compile - 编译源代码
3. test - 运行测试用例
4. package - 打包构建产物
5. install - 安装到本地仓库

顺序执行, 如运行test 就会执行clean & compile 
##### test 
测试test类中的所有单元测试方法



# [[Spring]]

# [[网络#HTTP]]

# Tomcat
开源轻量级 Web服务器  支持servlet / JSP 
servlet容器： servlet 需要tomcat才能运行
SpringBoot就内嵌了tomcat

# 请求响应
## 请求
**Postman**
API开发工具，用于测试、开发和调试API接口。

### 简单参数
```java
@RequestMapping("/simpleParam")  
public String simpleParam(@RequestParam(name = "name", required = false) String usrName, Integer age){  
    System.out.println(usrName + ":" + age);  
    return "OK";  
}
```
`@RequestParam(name = "name", required = false)` 
把变量名为`name`的变量映射到它修饰的变量上
**required**：
- 为true时请求时必须有这个参数否则 400错误
- false 则在解析时usrName为null
### 实体参数
大量的参数 如果用简单参数不方便

可以创建一个实体类，把参数传入到实体类中
![[Pasted image 20240412014916.png]]

### 数组集合参数
```java
@RequestMapping("/arrayParam")
public String simpleParam(String[] names){
	if(names != null){
		System.out.println(names.toString());
	}else{
		System.out.println("null");
	}
	return "OK";
}

@RequestMapping("/listParam")
public String simpleParam(@RequestParam List<String> names){
	if(names != null){
		System.out.println(names.toString());
	}else{
		System.out.println("null");
	}
	return "OK";
}

```

### 日期参数

需要指定对应的pattern
```java
@RequestMapping("/dateParam")
public String simpleParam(@DateTimeFormat(pattern = "yy-MM-dd HH:mm:ss") LocalDateTime time){
	System.out.println(time.toString());
	return "OK";
}
```
 如果不满足则400错误
> [!warning]   
> Failed to convert from type [java.lang.String] to type [java.time.LocalDateTime] for value [2024-04-12 02:03:16]

### json参数
![[Pasted image 20240412021331.png]]

### 路径参数
```java
@RequestMapping("/path/{id}/{name}")
public String jsonParam(@PathVariable Integer id, @PathVariable String name){
	System.out.println(name + ":" + id.toString());
	return "OK";
}
```
`http://localhost:8080/path/1/myName/`

## 响应
```java
@RequestMapping("/getListUser")
public List<User> getListUser(){
	List<User> userList = new ArrayList<>();
	User user1 = new User();
	User user2 = new User();
	user1.setName("user1");
	user1.setAge(12);
	Addr addr1 = new Addr();
	addr1.setCity("成都");
	addr1.setProvince("四川");
	user1.setAddress(addr1);
	userList.add(user1);
	userList.add(user1);
	return userList;
}
```


![[Pasted image 20240412021846.png]]
`return` 的内容就是响应内容
因为**每个响应的类型不同, 不易于管理**
统一响应结果:
![[Pasted image 20240412022653.png]]




# 分层解耦
三层架构:
1. 接受请求 / 响应数据 (Controller)
2. 逻辑处理 (Service)
3. 数据访问 (DAO)


![[Pasted image 20240412031024.png]]

![[Spring#IOC]]




# 登录校验
**cookie**
优点：HTTP协议中支持的技术
缺点：
- 移动端APP无法使用Cookie
- 不安全，用户可以自己禁用Cookie
- Cookie不能跨域
**session**
优点：存储在服务端，安全
缺点：
- 服务器集群环境下无法直接使用Session
- Cookie的缺点
 **令牌技术**
优点：
- 支持PC端、移动端
- 解决集群环境下的认证问题
- 减轻服务器端存储压力
缺点：需要自己实现

## JWT令牌
![[Pasted image 20240416011707.png]]


# 事务
![[Spring#事务]]


# AOP
![[Spring#AOP]]


# 跨域 
CORS ：Cross-Origin Resource Sharing  跨域资源共享

同源策略：协议、域名、端口3个都相同就是同源，否则就是跨域
是浏览器前端的一种配置安全机制，后端其实是响应了数据回去的
前端可以通过CORS中添加配置允许跨域即可

### 后端解决跨域
1. 在controller的目标方法上加上 `@CrossOrigin` 注解
2. 在配置类中加入一个Cors的过滤器，字段包括：源地址、header、请求方式
3. 实现一个WebMvcConfiguration接口，与配置类相似，重写addCorsMapping：源地址、请求方式
