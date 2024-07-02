# Spring Boot
## 自动配置
Spring Boot 中，只需要添加相关依赖, 无需配置
通过 Spring Boot 的全局配置文件 `application.properties`或`application.yml`即可对项目进行设置比如更换端口号，配置 JPA 属性等等。

没有 Spring Boot 的情况下，如果我们需要引入**第三方依赖，需要手动配置**，非常麻烦。但是，Spring Boot 中，我们直接引入一个 *starter* 即可。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```


大概可以把 `@SpringBootApplication`看作是 `@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan` 注解的集合。根据 SpringBoot 官网，这三个注解的作用分别是：

- `@EnableAutoConfiguration`：启用 SpringBoot 的自动配置机制
- `@Configuration`：允许在上下文中注册额外的 bean 或导入其他配置类
- `@ComponentScan`：扫描被`@Component` (`@Service`,`@Controller`)注解的 bean，注解默认会扫描启动类所在的包下所有的类 ，可以自定义不扫描某些 bean。如下图所示，容器中将排除`TypeExcludeFilter`和`AutoConfigurationExcludeFilter`。



整个过程主要分成加载所有的配置类，这里用@ComponentScan来加载我们application路径下的包，用@EnableAutoConfiguration来用spring factory机制来加载第三方的jar包的配置类，所有加载好后，再去加载这些配置类用@Import，@Bean等注解去加载的别的配置类，此时所有需要加载的配置类都加载好了，再去实例化这些bean，将这些bean注册到IOC中



Spring Boot 通过`@EnableAutoConfiguration`开启自动装配，通过 SpringFactoriesLoader 最终加载`META-INF/spring.factories`中的自动配置类实现自动装配，自动配置类其实就是通过`@Conditional`按需加载的配置类，想要其生效必须引入`spring-boot-starter-xxx`包实现起步依赖


## 实现
SpringBoot的核心是 `@SpringBootApplication` 注解
可以把 `@SpringBootApplication`看作是 `@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan`


# IOC
交给IOC的专门的一个容器控制, 包括创建, 和外部资源的获取

有了IOC容器, 其进行注入组合对象, 使得对象间耦合性降低  易于复用&测试

## 三种配置方式
### xml配置
优点: 结构清晰
缺点: 配置繁琐, 不易维护, 扩展性差

### java配置
类的配置交给JavaConfig类实现, Spring只负责维护和管理, 本质就是把在XML上的配置声明转移到Java配置类中

优点: 适用与任何场景,  配置方便, 扩展性好(因为是Java代码)
缺点: 不是xml的结构化语言, 可读性差

### 注解配置
![[Pasted image 20240412031730.png]]

通过给类加注解, 实现Spring IOC, Spring会自动扫描有(@Component，@Controller，@Service，@Repository注解的类), **需要配置注解扫描器**
优点: 方便快捷, 易维护
缺点: 第三方的资源不能加注解
#### 组件扫描
`@SpringBootApplication` 有包扫描功能, 默认扫描当前包及其子包

**手动扫描**
添加从
```java
//@ComponentScan ({"扫描包","当前包"}) 
@Componentscan ({"dao","com.xxx"})


```

#  依赖注入(DI)
**组件之间依赖关系由容器在运行期决定**
应用程序依赖IOC容器提供的外部资源(对象, 资源, 常量等) 注入到程序的某个对象中

DI是IOC设计思想的一个实现

### 三种注入方式
> [!info] 常用的三种注入方式
1. 基于注解的注入（接口注入）
2. 构造方法注入（Construct注入）
3. setter注入

#### 注解(接口)注入

| 注解                         | 默认注入方式     | 来源     |
| -------------------------- | ---------- | ------ |
| `@Autowired`               | 按类型        | Spring |
| `@Resource`                | 按名称        | JDK    |
| `@RequiredArgsConstructor` | 给final字段注入 | Lombok |

##### Autowired 冲突解决方案
1. `@primary` 优先
```java
@Primary
@Repository(value = "userDaoB")
public class UserDaoB implements UserDAO {...}
```
2. `@Qualifier("bean名")`  
```java
@Qualifier("userDaoB")  
@Autowired  
private UserDAO userDAO;
```
3. `@Resource(name = "bean名")`  
```java
@Resource(name = "userDaoB")
private UserDAO userDAO;
```


#### 构造注入 (推荐)
XML中 `<constructor-arg>`是通过构造函数参数注入的
```xml
<bean id="userService" class="com.service.UserServiceImpl"> 
	<constructor-arg name="userDao" ref="userDao"/> 
	<!-- additional collaborators and configuration for this bean go here --> 
</bean>
```
JAVA 注解构造函数
```java
@Autowired // 注解模式中


public class UserServiceImpl {
	private UserDaoImpl userDao;

	public UserServiceImpl(UserDaoImpl userDaoImpl) {
	    this.userDao = userDaoImpl;
	}
}
```

为啥推荐?
1. 依赖不可变：其实说的就是final关键字。
2. 依赖不为空（省去了我们对其检查）：当要实例化UserServiceImpl的时候，**由于自己实现了有参数的构造函数**，所以不会调用默认构造函数，那么就需要Spring容器传入所需要的参数，所以就两种情况：
	1. 有该类型的参数->传入，OK 。
	2. 无该类型的参数->报错。
3. 完全初始化的状态：这个可以跟上面的依赖不为空结合起来，向构造器传参之前，要确保注入的内容不为空，那么肯定要调用依赖组件的构造方法完成实例化。而在Java类加载实例化的过程中，构造方法是最后一步（之前如果有父类先初始化父类，然后自己的成员变量，最后才是构造方法），所以返回来的都是初始化之后的状态。


#### setter注入
XML property都是setter方式注入
```xml
<bean id="userService" class="com.service.UserServiceImpl">
	<property name="userDao" ref="userDao"/> 
	<!-- additional collaborators and configuration for this bean go here -->
</bean>
```
JAVA 注解set函数
```java
public class UserServiceImpl {
    private UserDaoImpl userDao;

    public List<User> findUserList() {
        return this.userDao.findUserList();
    }

    public void setUserDao(UserDaoImpl userDao) {
        this.userDao = userDao;
    }
}
```


## 循环依赖
多个类互相引用对方
> [!example] 解决方法
> 1. 重新设计
> 2. `@Lazy`
> 3. Setter/Field 注入
> 4. `@PostConstruct`

#### @Lazy
延时加载: 在Bean还没有完全初始化玩完, 实际上注入的是一个代理, 只有被使用时才会完全初始化

#### Setter/Field 注入
使用setter注入, 而不是构造注入
这种方式构造时依赖没有注入  只有调用Set函数时才会注入进来

#### @PostConstruct
在要注入的属性上使用`@Autowired` 并在里一个依赖隔着属性的方法上使用`@PostConstruct` 标注
- 该注解的方法在整个Bean初始化中的执行顺序：
Constructor(构造方法) -> @Autowired(依赖注入) -> @PostConstruct(注释的初始化方法) 
即先注入B,  然后给B 注入 A
- 是JDK提供的注解, 不是Spring的

```java
@Service
public class ServiceAImpl implements ServiceA {
    @Autowired
    private ServiceB serviceB;

    @PostConstruct
    public void init() {
        System.out.println(serviceB);
        serviceB.setServiceA(this);
    }
}

@Service
public class ServiceBImpl implements ServiceB {
    private ServiceA serviceA;

    public void setServiceA(ServiceA serviceA) {
        System.out.println(serviceA);
        this.serviceA = serviceA;
    }
}
```


# AOP
面向切面编程, 是一种抽象的面向对象编程, 是对面向编程的补充

通过动态代理对象实现，动态代理对象的创建有两种jdk和cglib

把重复的代码抽离出来, ***抽象为一个切面对象***  如logMsg任务
业务中只保留核心代码

### 动态代理
包括JDK和CGLib动态代理方法
https://www.bilibili.com/video/BV1tY411Z799

##### JDK动态代理
只提供接口实现类的代理，不支持普通类的代理
通过**实现接口类**、**反射**调用实现

> [!info]  注意
> 因为通过实现接口类实现
> 只能代理接口的实现类，且无法代理[[JAVA基础#JAVA8|静态方法和私有方法]]

###### 底层实现
运行时生成一个动态代理类
动态代理类实现了目标类的接口，其继承了Proxy类
- 因为继承了Proxy类，且Java不能多继承，所以不能代理普通类
- 只能通过实现接口，代理有接口的类（相当于重写了一个实现接口的类，增强了目标类的方法）
代理类通过Handler拦截方法，实现方法增强
调用Handler的invoke方法，并*反射*调用目标类方法

##### CGLib代理
没有实现接口时，无法通过JDK代理时使用
通过**继承**、**子类调用父类**实现

> [!info]  注意
因为通过继承实现，所以*final*修饰的类和方法不能代理
且无法代理私有方法和静态方法

###### 底层实现
在运行时继承目标类，生成目标类的子类，重写目标类方法
通过Interceptor拦截方法，对应的方法实现增强
在Interceptor中实现增强，再通过*子类调用父类*的方法
final 修饰的类会失效：因为使用*继承*实现动态代理

#### 两种方法的对比
1. 因为用了反射，JDK7之前，CGLib会快，但是随着JDK的升级，JDK7/8已经更快了
2. CGLib第一次调用要生成字节码文件，多次调用性能还行

### 使用场景
1. 日志记录
2. 权限验证
3. 事务处理

### 通知（Advice）对象
Before
After
After Return
After throw
around
### 切入点（PointCut）


# 设计模式
1. 工厂模式 
	1. BeanFactory
2. 单例模式
	1. IOC->DI
3. 装饰器模式
	1. 动态给对象添加额外职责
4. 代理模式
	1. AOP的底层: 
		1. 静态代理
		2. 动态代理
5. 策略模式 
	1. Resource接口
6. 观察者模式 
	1. 事件驱动模型
		1. xxxListener, xxxEventPublisher
		2. 也叫 发布/订阅模型


# Spring Cloud 微服务





# 事务

## 注解开发
`@Transactional`，可以修饰Service类，也可以修饰接口、类方法
是基于AOP实现的，在代码加上了代理，代码中只需要写业务，不用手动管理事务
### 参数
##### rollbackFor】
默认运行时出现[[JAVA基础#Exception和Error|RuntimeException]] 或者 Error 就会回滚
`@Transactional(rollbackFor = xxx.class)`
rollbackFor 可以定义哪些情况回滚/不回滚，也可以设置[[JAVA基础#Exception和Error]]检查异常回滚


即使是中 `finally`中的代码也会回滚
![[Pasted image 20240416230235.png]]


##### propagation
事务传播行为：指的就是当一个事务方法被另一个事务方法调用时，当前事务方法应该如何进行事务控制。

![[Pasted image 20240416225922.png]]
### 失效
https://www.bilibili.com/video/BV1M8411w7wo
- *非public方法*：事务是基于AOP实现的，AOP基于动态代理实现，而动态代理不能代理非public方法，所以事务失效
- *调用类内的 @Transactional 方法*：同样因为动态代理，类内调用@Transactional 方法会绕过动态代理对象，使事务失效
- *try/catch*：方法中自己实现的Try-catch会导致**回滚失效**，因为Spring是通过捕获异常实现回滚，如果自己捕获异常，Spring就感知不到异常了，所以会回滚失败





# Bean
## Bean的声明
### XML配置
![[Pasted image 20240427163529.png]]
#### Bean 名
XML 配置 Bean，Bean名为：
1. 有 id 为 id
2. name 作为别名
3. 没 id、name 则为 class 的全限定名 `com.project.dao.impl.UserDaoImpl`

#### 范围
`<bean scope="">`
- singleton 默认值
在Spring创建时实例化，存储到容器内部的**单例池**中，每次getBean都是从单例池中获取相同的Bean实例
- prototype
初始化时不会创建Bean，调用getBean才创建，每次创建一个新的Bean
- 特殊的scope，在spring-webmvc中才有的：
	- request
	- session



#### 初始化、销毁方法
```java
public void init(){}
public void destroy(){}

// xml中
<bean ... init-method="init", destroy-method="destroy"/>
```

方法二：
实现`InitializingBean` 接口，重写`afterPropertiesSet`方法
```java
class Xxx implements InitializingBean
{
	...
	@Override
	public void afterPropertiesSet () throws Exception{}
}
```


### 实例化
#### 构造方法
默认调用空参构造，没有就报错
可以配置有参构造：
```java
<bean id="userService"class="com.itheima.service.impl.UserServiceImpl">
<constructor-arg name="name"value="ZhangSan"></constructor-arg>
```

#### 工厂方法
**静态方法**
自己写一个工厂类，定义一个静态方法，生产实例
```java
public class MyBeanFactory1{
	public static UserDao userDao(){
		//Bean创建之前可以进行一些其他业务逻辑操作
		return new UserDaoImpl();
	}
}

// xml
<bean id="userDaol"class="com.itheima.factory.MyBeanFactory1" factory-method="userDao">
```

**实例方法**
自己写一个工厂类，实例化一个工厂对象，定义一个普通方法，通过工厂对象生产实例
```java
public class MyBeanFactory2{
	public UserDao userDao(){
		return new UserDaoImpl();
	}
}

// xml
<bean id="myBeanFactory2"class="com.itheima.factory.MyBeanFactory2"/>
<bean id="userDao2" factory-bean="myBeanFactory2" factory-method="userDao"/>
```

##### 有参工厂方法
```java
<bean id="userDao2" factory-bean="myBeanFactory2" factory-method="userDao">
	<constructor-arg name="name"value="ZhangSan"></constructor-arg>
</bean>
```

#### 实现FactoryBean接口类
Spring 提供的FactoryBean
**可以实现延迟产生Bean**
重写getObject()方法






## Bean的生命周期
![Bean的生命周期](https://segmentfault.com/img/remote/1460000040365134)
## BeanDefinition


# 入门
BeanFactory 与 ApplicationContext 的关系:
1. BeanFactory 是 Spring 的早期接口，是Bean工厂，ApplicationContext 是高级接口，是Spring 容器
2. ApplicationContext 继承了 BeanFactory，实现了更多方法，其中维护这一个 BeanFactory 的对象
4. ApplicationContext 在配置文件加载容器一创建就将 Bean 实例化并初始化好，BeanFactory 是在首次调用时才进行Bean的创建
