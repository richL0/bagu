# 面向对象
## 访问控制修饰符
Java中，可以使用访问控制符来保护对类、变量、方法和构造方法的访问。Java 支持 4 种不同的访问权限。

`default` (即默认，什么也不写）: 在**同一包内**可见，不使用任何修饰符。
`private` : 在**同一类内**可见。使用对象：变量、方法。 注意：*不能修饰类*（外部类）
`public` : 对**所有类**可见。使用对象：类、接口、变量、方法
`protected` : 对**同一包内**的类和所有**子类**可见。使用对象：变量、方法。 注意：*不能修饰类*（外部类）。


## 传参
Java 的参数是**以值传递**的形式传入方法中，**而不是引用**传递。
> 对于输入为基础类型`int double boolean`等，传入的是值的副本，函数内的操作不影响外面的值
> 对于输入是一个对象的引用，如`ArrayList HashMap` 等，传入的是引用的副本，但是由于引用的副本和引用是*指向同一个对象*的，所以函数内的操作会影响外面的对象。


## 对象通用方法
### equals()
 equals() 和 `==` 的区别:
```java
Integer x = new Integer(1);
Integer y = new Integer(1);
System.out.println(x.equals(y)); // true
System.out.println(x == y);      // false
```
`equals()`  判断引用的对象是否等价, `==` 判断是否引用同一个对象

### HashCode()
**等价的对象Hash Value相同**

### HashCode 和 equalse

### clone()
Cloneable不包含任何方法, clone 方法在 object 类中定义


## 多态
分为：
- **编译时多态**主要指方法的**重载**
- **运行时多态**指程序中定义的对象**引用所指向的具体类型**在运行期间才确定

> 实现多态有三个必要条件：继承、重写、向上转型  
> （1） 继承：在多态中必须存在有继承关系的子类和父类。  
> （2） 重写：子类对父类中某些方法进行重新定义，在调用这些方法时就会调用子类的方法。  
> （3） 向上转型：将**子类的对象赋给父类引用**，只有这样该引用才能够具备调用父类的方法和子类的方法的能力。

### 向上转型

``` java
class Animal {
    public void sound() {
        System.out.println("Animal makes a sound");
    }
}

class Cat extends Animal {
    @Override
    public void sound() {
        System.out.println("Cat meows");
    }
}

public class Main {
    public static void main(String[] args) {
        Animal animal = new Cat(); // 向上转型
        animal.sound(); // 调用被子类重写的方法
    }
}
```
在上面的示例中，`Cat`类继承自`Animal`类，并重写了`sound()`方法。在`Main`类的`main()`方法中，通过将`Cat`对象向上转型为`Animal`类型的引用变量`animal`，我们可以调用`sound()`方法。尽管`animal`是`Animal`类型的引用，但实际上它引用的是一个`Cat`对象。当我们调用`animal.sound()`时，它会调用被`Cat`类重写的`sound()`方法，输出"Cat meows"。

## 包装类型
### int 和 Integer的区别
1. int 变量直接存储整数值,Integer 变量存储 Integer 对象的引用。 
2. int 的默认值是 `0`,Integer 的默认值是 `null`。
3. Integer 类有很多有用的方法,比如解析字符串为整数(`parseInt`),整数与字符串的转换(`toString`),等等。int 作为基本类型没有这些方法。
4. Integer 对象在内存中占用更多空间,因为它存储对象*头信息和整数值*。int 只存储值。
5. **装箱和拆箱**: 给 int 变量赋值 Integer 对象,会自动装箱,反之亦然,这需要消耗一定资源。


### 缓存池
`new Integer(123)` 与 `Integer.valueOf(123)` 的区别在于:

- `new Integer(123)` 每次都会新建一个对象
- `Integer.valueOf(123)`会使用缓存池中的对象，多次调用会取得同一个对象的引用。

**Java 8** 中，Integer缓存池的大小默认为 -128~127。  8bit

## 抽象类 和 接口
抽象类不能被实例化

|      | 抽象类                         | 接口                      |
| ---- | --------------------------- | ----------------------- |
| 定义方式 | 可以包含具体方法（有实现），还可以包含属性和构造函数。 | 只包含方法名和常量的声明            |
| 多继承  | 只能继承一个抽象类                   | 一个类可以实现多个接口,实现多继承的效果    |
| 变量   | 可以定义非静态和非常量字段               | 默认是public static final的 |

# String
**被声明为 final**, 不可被继承, 不可变。
## 不可变的好处

**1. 可以缓存 hash 值**
**2. String Pool 的需要**
**3. 安全性** 保证参数不可变
**4. 线程安全** String 不可变性天生具备线程安全，可以在多个线程中安全地使用

### 可以强行变
因为`private final char value[];` 
`final` 修饰的`value` 是引用，不能改引用，但可以改引用的对象


## StringBuffer & StringBuilder
都可变
安全性:
	StringBuilder **不线程安全**的
	**StringBuffer** 线程安全的，内部用 **synchronized** 同步, 非多线程环境下，**执行效率就会比较低**，因为加了没必要的锁。

### "+" 操作的本质
编译时把"+" 变成了`StringBuilder`的`append()` 方法
因此在循环体内，拼接最好用`StringBuilder` 的`append()` 方法  否则会创建大量的`StringBuilder` 对象;
### append() 源码
```java
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);
    str.getChars(0, len, value, count);
    count += len;
    return this;
}

private void ensureCapacityInternal(int minimumCapacity) {
    // overflow-conscious code
    if (minimumCapacity - value.length > 0) {
        value = Arrays.copyOf(value,
                newCapacity(minimumCapacity));
    }
}

```
1. 先判断是不是null'
2. 获取拼接的长度
3. 创建一个新数组  复制原String内容到新数组
4. 把需要拼接的String复制到新数组
5. 更新String的长度

### concat()
```java
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    char buf[] = Arrays.copyOf(value, len + otherLen);
    str.getChars(buf, len);
    return new String(buf, true);
}
```
1. 把原String复制到新数组中
2. 把需拼接的String复制到新数组
3. 创建一个String对象 并返回
大量concat需要**创建大量的String对象**

## 字符串常量池 String Pool
通过调用String.intern()可以显式的把字符串存入常量池
``` java
String s1 = new String("aaa");
String s2 = new String("aaa");
System.out.println(s1 == s2);           // false
String s3 = s1.intern();
System.out.println(s1.intern() == s3);  // true
```
用`new String("aaa");`创建的String是两个**不同的**对象
可以用`s1.intern();` 直接把String Pool的对象给s3,

而直接用 `"aaa"`会自动放入 String Pool 中。
``` java
String s4 = "bbb";
String s5 = "bbb";
System.out.println(s4 == s5);  // true
```

#### Java 6/7/8 版本
Java 6
- 常量池存在*永久代*
- 调用intern()，会复制到常量池中（两份，堆和永久代各一份）
- 所以可能
Java 7
- 常量池存在*堆*中
- 调用intern()，不会复制，而是直接存到常量池中
Java 8
- 常量池存在[[JVM#方法区 (元空间mataspace / 永久代PermSpace)|元空间]]中

#### 结构
是一个类似HashMap的结构，但是不支持扩展，可以通过`-XX:StringTableSize`配置大小
Java 7 是 1009个bin，后面改成了60013

#### 锁
如果用String做锁，因为是存放在常量池中的，所以只要是相同的String，锁都是一样的（和引用不一样）


# 关键字
## final
对于**基本类型**,  final 等价于 const in C++
对于**引用类型**, final让其不能引用其他对象,但是被引用的对象本身可以改.

对于**方法**,  声明其不能被子类重写(override)
对于**类**,  声明类不能被继承
## static
类所有的实例都共享静态变量, 可以用类名直接访问
### 静态方法不能调用非静态方法和变量
1. 静态上下文：静态方法在类级别上执行，而不依赖于类的实例。
2. 访问实例成员：非静态方法和变量是与类的实例相关联的。
3. 生命周期差异：静态方法和变量在类加载时就存在，并且在整个程序的执行期间都保持不变。
***本质是实例(非静态)和类的区别***
即与类加载顺序有关，**加载静态方法时，非静态的*未初始化***



# 异常处理
## Exception和Error
error 为错误，exception 为异常，**错误等级高**。
- **Error** ，意味着程序出现了严重的问题，而这些问题***不应该交给 Java 的异常处理***机制来处理，程序应该直接崩溃掉，比如说 OutOfMemoryError，内存溢出了
- **Exception**，意味着程序出现了一些在**可控范围**内的问题，我们应当采取措施进行挽救。
- ![[Pasted image 20240304001545.png]]
![[Pasted image 20240302211740.png]]

### Exception 和 Error 都继承了 Throwable 类
只有 Throwable 类（或者子类）的对象才能使用 throw 关键字抛出，或者作为 **catch 的参数**类型。

**catch 可以捕获所有Throwable的子类**  
###  unchecked 和  checked Exception
非检查异常(Unchecked Exception, Runtime Exception)：由于程序内部逻辑或数据异常导致的，可以不处理或者在需要时进行处理。
检查异常 (Checked Exception)：通常是由于外部因素导致的问题，*需要在代码中显式地处理*或声明抛出。

### try-catch-finally
***即便是 try 块中执行了 return、break、continue 这些跳转语句，finally 块也会被执行。***

```java
try { 
	return 112; 
}catch(IOExceprtion e1){
    // 异常处理
}catch(Exceprtion e){
    // 异常处理
}finally {
	System.out.println("即使 try 块有 return，finally 块也会执行");
}
```

不执行finally的情况
1. 死循环
2. `System.exit()`

#### finally 的作用
1. 清理资源 `file.close()`
2. 维护一致性
3. 释放锁或资源 ：在多线程编程中，可能使用了锁或其他共享资源来确保线程安全


### try with resources
把释放的资源写在 try 后的 `()` 中

```java
try (BufferedReader br = new BufferedReader(new FileReader(decodePath));
     PrintWriter writer = new PrintWriter(new File(writePath))) {
    String str = null;
    while ((str =br.readLine()) != null) {
        writer.print(str);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

处理必须关闭的资源时，始终有限考虑使用 try-with-resources





# IO/NIO
[[操作系统#I/O 多路复用]]
## 字节流和字符流
字节流 I/O 类：InputStream 和 OutputStream
字符流 I/O 类：Reader 和 Writer
磁盘的 I/O 类：File

|         | 字节流                             | 字符流                        |
| ------- | ------------------------------- | -------------------------- |
| 对象      | 字节                              | 字符（Unicode码元）              |
| 缓冲      | 无缓冲                             | 有缓冲，IO时会先将数据放入内存，后面直接从内存中读 |
| 存在位置    | 可存在于文件、内存中。 硬盘上的所有文件都是以字节形式存在的。 | 只存在于内存中。                   |
| 使用场景    | 适合操作文本文件之外的文件。 例：图片、音频、视频。      | 适合操作文本文件时使用。 （效率高。因为有缓存）   |
| Java相关类 | InputStream、OutputStream        | Reader、Writer              |

在硬盘上的所有文件都是以字节形式存在的（图片，声音，视频），而**字符值在内存中才会形成**。


## bufferedStream
传统的 Java IO 是阻塞模式的，它的工作状态就是“读/写，等待，读/写，等待。。。。。。”

字节缓冲流解决的就是这个问题：**一次多读点多写点，减少读写的频率，*用空间换时间***。

- 减少系统调用 & 磁盘读写次数：在使用字节缓冲流时，数据不是立即写入磁盘或输出流，而是先写入缓冲区，当缓冲区满时再一次性写入磁盘或输出流。这样可以减少系统调用的次数，从而提高 I/O 操作的效率。
- 提高数据传输效率：在使用字节缓冲流时，由于数据是以块的形式进行传输，因此可以减少数据传输的次数，从而提高数据传输的效率。

## 序列流
只有实现了 `Serializable` 接口的对象才能被序列化。



# JAVA8
JRE 和 JDK的区别
- **JRE is the JVM program**, Java application need to run on JRE.
- **JDK** is a superset of JRE, JRE + tools **for developing java programs**. e.g, it provides the compiler "javac"

## Lambda 表达式

## 方法引用

## 接口
JAVA8 前：
- ✔可以有常量变量
- ❌不能*实现方法*，只能提供抽象方法
- ❌不能定义*默认方法和静态方法*
JAVA8 后：
- ✔可以定义*默认方法和静态方法*
JAVA9：还支持*私有方法*

## Stream API 流操作



# 集合Collection
## ArrayList
**扩容机制**
1. 如果现在长度是0  则创建一个大小为10的array
2. 如果不是0  则扩到原来大小的$1/2$
```java
private Object[] grow(int minCapacity) {  
    int oldCapacity = elementData.length;  
    if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) { 
        int newCapacity = ArraysSupport.newLength(oldCapacity,  
                minCapacity - oldCapacity, /* minimum growth */  
                oldCapacity >> 1           /* preferred growth */); 
                // prefGrowth =  oldCapacity / 2
        return elementData = Arrays.copyOf(elementData, newCapacity);  
    } else {  
        return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];  
    }  
}


// ArraysSupport.newLength
public static int newLength(int oldLength, int minGrowth, int prefGrowth) {  
    // preconditions not checked because of inlining  
    // assert oldLength >= 0    // assert minGrowth > 0  
    int prefLength = oldLength + Math.max(minGrowth, prefGrowth); // might overflow  
    if (0 < prefLength && prefLength <= SOFT_MAX_ARRAY_LENGTH) {  
        return prefLength;  
    } else {  
        // put code cold in a separate method  
        return hugeLength(oldLength, minGrowth);  
    }  
}
```


## HashMap
存储是以数组+链表的方式存储的

对于加入的元素，
1. 先对Key计算hash
2. 如果现在长度是0  则创建一个大小为16的Node的Array
3. 判断hash在数组里面存不存在
	1. 不存在则创建一个链表的Node直接写
	2. 存在
		1. key相同则覆盖掉原来的
		2. key不同则在链表上遍历，如果有相同key的则覆盖这个Node，否则插在这个链表的后面
		3. 如果这个链表的长度大于建树阈值（8）且数组长度大于64  则把这个链表改成红黑数
4. 长度+=1
5. 判断当前长度是否大于阈值（负载因子（load factor）乘以当前容量）大于则数组扩大到原来的两倍

```java
static class Node<K,V> implements Map.Entry<K,V> {
	final int hash;  
	final K key;  
	V value;  
	Node<K,V> next;
}
```
### HashMap红黑树的阈值为什么是8？
[详解](https://blog.csdn.net/Liu_Wd/article/details/108052428)
1. HashCode碰撞次数服从泊松分布，单个hash槽内元素个数为8的概率小于百万分之一,  如果真到8个了,  说明hash函数不对劲,  可能后面还会碰撞
2. TreeNode是链表中的Node所占空间的2倍  空间占用大一点
3. 虽然时间复杂度低, 但是链表短的话影响不大
为啥小于等于6才改成链表
1. 如果设置成8 可能会导致链表和红黑树的频繁转换



| **项**       | **HashMap**                                                      | **TreeMap**                                                         | **LinkedHashMap**                                                |
| ----------- | ---------------------------------------------------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------- |
| **按插入顺序存放** | 不支持                                                              | 不支持。                                                                | **支持。** 遍历时，按插入的顺序出结果。                                           |
| **按key排序**  | 不支持。 按照HashCode进行输出。                                             | **支持**。 默认按key升序排序。可用Comparator自定义排序。 用Iterator 遍历TreeMap时，结果是排过序的。 | 不支持。                                                             |
| **数据结构**    | 数组 + 链表 + 红黑树 （put和get操作，基本可以达到常数时间的性能）                          | 红黑树。 （get或put操作的时间复杂度是O(log(n))）                                    | HashMap + 双向链表 此类是HashMap的子类。                                    |
| **null**    | key和value均允许为null。 只允许一条记录的key值为null(多条会覆盖)； 允许多条记录的Value为 null。 | 不允许key的值为null                                                       | key和value均允许为null。 只允许一条记录的key值为null(多条会覆盖)； 允许多条记录的Value为 null。 |
### LinkedHashMap
通过维护一个双向链表，实现有顺序的遍历，继承于HashMap
每一个节点有next指针，指向下一个插入的节点

```java
static class Entry<K,V> extends HashMap.Node<K,V> {  
    Entry<K,V> before, after;  
    Entry(int hash, K key, V value, Node<K,V> next) {  
        super(hash, key, value, next);  
    }  
}
```
通过`before` & `after` 节点记录插入顺序

### TreeMap
通过维护一个红黑树，没有数组，不继承于HashMap
```java
public class TreeMap<K,V>  
    extends AbstractMap<K,V>  
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable  
{ 
    private final Comparator<? super K> comparator; 
	// root 就是TreeMap的根
    private transient Entry<K,V> root;
    private transient int modCount = 0;  
    public TreeMap() { comparator = null;  }
```

put时比较树里面的key的大小，也不计算hash值

### JDK8

| **项**      | **JDK7**                                                           | **JDK8**                                                                                                   |
| ---------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| **数据结构**   | 数组 + 链表。 复杂度：O(n)                                                  | 数组 + 链表 + 红黑树  <br>（若链表长度大于8 且容量小于64 会进行扩容；若链表长度大于8 且数组长度大于等于64，会转化为红黑树（提高定位元素的速度）；若红黑树节点个数小于6，则将红黑树转为链表。） |
| **插入位置**   | 插入链表头部                                                             | 插入链表尾部                                                                                                     |
| **hash算法** | 复杂                                                                 | 简单。 红黑树效率高，提高查询效率的地方由红黑树实现，没必要像JDK7那么复杂。                                                                   |
| **多线程的结果** | 数据覆盖（场景：多线程put+put） 读出为null（场景：多线程扩容，put+get） 死循环 （场景：多线程扩容（头插法导致） | 数据覆盖（场景：多线程put） 读出为null（场景：多线程扩容，put+get） 不会发生：     死循环（因为是尾插法）                                            |
由于头插法会改变链表的顺序，多线程的时候可能第一个线程已经改变顺序了，第二个又继续头插，就会导致死循环。

JDK8改成尾插法，使得时间复杂度高了，需要从头找到尾，但是安全了。另一方面加入了红黑树，可以把时间复杂度降低。

## HashSet
HashSet的实现调用了HashMap，set的元素就是map的key。其value是一个空的Object()对象

## 线程安全（待补充）






# 反射
## Class.forName和ClassLoader

|           |                                                                              |                                                                |
| --------- | ---------------------------------------------------------------------------- | -------------------------------------------------------------- |
| **项**     | **Class.forName**                                                            | **ClassLoader**                                                |
| **灵活度**   | 灵活度低。 例如：加载的类*只能是classpath下的*                                                | *灵活度高*。 例如：可以自己编写加载类的方法：比如通过读取类文件的二进制数据，这个时候文件可以不存在ClassPath中。 |
| **是否初始化** | 是。 forName方法最后调用 forName0(本地方法)，第二个参数设置为了 true，代表对加载的类进行初始化（执行静态代码块、对静态变量赋值） | 否。                                                             |
|           | 有参构造                                                                         |                                                                |

## 作用
提供了在运行时**动态获取类的信息**、创建对象、调用方法和访问/修改字段的能力

用途:
1. 动态加载类 根据配置文件/用户输入加载不同的类
2. 获取类的信息 -> 类名、父类、接口、字段、方法等。
3. 创建对象 -> 运行时创建类的实例
	这对于实现通用的对象*创建框架或依赖注入容器*很有用。
4. 运行时动态调用类的方法，包括公共方法、私有方法和静态方法
5.  运行时动态访问&修改字段

由于反射使用了一些底层机制，*比直接调用方法或访问字段要慢*



