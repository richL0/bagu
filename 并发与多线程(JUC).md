场景：快速响应用户请求
描述：用户发起的实时请求，服务追求响应时间。比如说用户要查看一个商品的信息，那么我们需要将商品维度的一系列信息如商品的价格、优惠、库存、图片等等聚合起来，展示给用户。
分析：从用户体验角度看，这个结果响应的越快越好，如果一个页面半天都刷不出，用户可能就放弃查看这个商品了。而面向用户的功能聚合通常非常复杂，伴随着调用与调用之间的级联、多级级联等情况，业务开发同学往往会选择使用线程池这种简单的方式，将调用封装成任务并行的执行，缩短总体响应时间。另外，使用线程池也是有考量的，这种场景最重要的就是获取最大的响应速度去满足用户，所以应该不设置队列去缓冲并发任务，调高corePoolSize和maxPoolSize去尽可能创造多的线程快速执行任务。

![图12 并行执行任务提升任务响应速度](https://p0.meituan.net/travelcube/e9a363c8577f211577e4962e9110cb0226733.png)

# 多线程

## 线程

#### 线程的六种状态
![](https://segmentfault.com/img/remote/1460000023194699)
```java
// java中Thread类中的内部State枚举类
public enum State {
	NEW, // 新建，尚未启动
	RUNNABLE, // 可运行
	BLOCKED, // 阻塞，等待锁
	WAITING, // 等待，等待线程的
	TIMED_WAITING, // 限时等待
	TERMINATED; // 已终止
}
```

### 优先级
线程优先级可以指定，范围是 1~10，默认为 5。
Java 只是给操作系统一个优先级的**参考值**，线程最终**在操作系统中的优先级**还是由操作系统决定。
高优先级线程有更高的**概率**执行。

### 实现方法
#### 1. 继承 Thread 类
```java
public class MyThread extends Thread {
    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            System.out.println(getName() + ":" + i);
        }
    }
}

//创建MyThread对象
MyThread t1=new  MyThread();
MyThread t2=new  MyThread();
MyThread t3=new  MyThread();
//设置线程的名字
t1.setName("鲁班");
t2.setName("刘备");
t3.setName("亚瑟");
//启动线程
t1.start();
t2.start();
t3.start();
```
#### 2. 实现 Runnable 接口

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            try {
                Thread.sleep(20);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + ":" + i );
        }
    }
}

//创建MyRunnable类
MyRunnable mr = new MyRunnable();
//创建Thread类的有参构造,并设置线程名
Thread t1 = new Thread(mr, "张飞");
Thread t2 = new Thread(mr, "貂蝉");
Thread t3 = new Thread(mr, "吕布");
//启动线程
t1.start();
t2.start();
t3.start();
```
#### 3. 实现 Callable 接口
```java
public class CallerTask implements Callable<String> {
    public String call() throws Exception {
        return "Hello,i am running!";
    }

    public static void main(String[] args) {
        //创建异步任务
        FutureTask<String> task=new FutureTask<String>(new CallerTask());
        //启动线程
        new Thread(task).start();
        try {
            //等待执行完成，并获取返回结果
            String result=task.get();
            System.out.println(result);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}


```

#### run ()和 start ()的区别
`run()`：封装线程执行的代码，直接调用相当于调用普通方法。
`start()`：启动线程，然后*由 JVM 调用*此线程的 `run()`方法。

#### 继承`Tread`和实现`Ruannable`哪个好
实现`Ruannable`更好
- Java*单继承* -> 继承`Tread` 就不能继承其他的
- 解耦 -> 把代码中的*类和线程解耦*  更符合面向对象的设计思想


### Runnable vs Callable
`Runnable` 不会返回结果或抛出检查异常，但是 `Callable` 可以。


### [[内核线程和用户线程]]
#### 用户级线程


#### 



## 线程间协作
### setDaemon() 守护线程
所有非守护线程结束，则杀死所有守护线程。
`main() `属于非守护线程。

使用` setDaemon()` 方法将一个线程设置为守护线程。

```java
public static void main(String[] args) {
    Thread thread = new Thread(new MyRunnable());
    thread.setDaemon(true);
}
```

### yield 让步
让步/退出
声明当前线程已完成重要部分，可以切换给其它线程来执行。
该方法只是对线程调度器*建议*
```java
public void run() {
    Thread.yield();
}
```


### join
等待线程执行结束
```java
//启动线程
t1.start();
t1.join(); //等待t1执行完才会轮到t2，t3抢
t2.start();
t3.start();
```
##### join源码
通过native函数[[#wait]]实现等待，并不断轮询线程是不是isAlive的
轮询是因为join用wait实现，而wait会被notify错误唤醒
```java
public final synchronized void join(final long millis) {
	if (millis > 0) {
		if (isAlive()) {
			final long startTime = System.nanoTime();
			long delay = millis;
			do {
				wait(delay);
			} while (isAlive() && (delay = millis -
					TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)) > 0);
		}
	} else if (millis == 0) {
		while (isAlive()) {
			wait(0);
		}
	} else {
		throw new IllegalArgumentException("timeout value is negative");
	}
}

```




### wait/notify await/signal
![[#wait]]
![[#await]]
## 多线程的意义和存在的问题
CPU、内存、I/O 设备的速度是有极大差异

- CPU 增加了*缓存*，以*均衡内存*的速度差异；// 导致 `可见性`问题
- 操作系统增加了*进程、线程*，以分时复用 CPU，进而均衡 *CPU 与 I/O 设备*的速度差异；// 导致 `原子性`问题
- 编译程序优化*指令执行次序*，使得*缓存*能够得到更加合理地利用。// 导致 `有序性`问题

`可见性`：一个线程对共享变量的修改，*其他线程*能够立刻看到。
`原子性`：即一个操作或者多个操作，*要么全部执行*并且执行的过程不会被任何因素打断，*要么不执行*。
`有序性`：即程序执行的顺序*按代码顺序*执行。

![](https://img2018.cnblogs.com/blog/1158841/201906/1158841-20190604203226483-484132713.png)

| 体现方面 | 说明                        | 导致原因 | 解决方法                                         |
| ---- | ------------------------- | ---- | -------------------------------------------- |
| 原子性  | 一个或者多个操作在CPU执行的过程中不被中断的特性 | 线程切换 | synchronized LOCK JDK Atomic开头的原子类（非阻塞CAS算法） |
| 可见性  | 一个线程对共享变量的修改，另外一个线程能够立刻看到 | 缓存   | synchronized LOCK volatile                   |
| 有序性  | 程序执行的顺序按照代码的先后顺序执行        | 编译优化 | Happens-Before规则                             |

## ThreadLocal
解决线程隔离问题：每个线程应该只能访问自己的的变量, 但是在多线程中不能保证
`ThreadLocal`对象通过提供线程局部变量，每个线程`Thread`拥有一份自己的**副本变量**，多个线程互不干扰。

### ThreadLocal与`Synchronized`的区别
1. 没有锁, 而是通过复制变量的方法解决数据一致性;  通过空间换时间. 
2. 不需要共享数据, 每个线程的数据是自己的, 不要且不想访问其他线程的数据

### ThreadLocal原理与结构
![](https://javaguide.cn/assets/2-CFHd4NU8.png)
Thread维护一个`ThreadLocalMap`对象
- 其中每一个Entry的key是ThreadLocal对象的一个**弱引用**, val为对应的值
	- 弱引用实现了 ThreadLocal 和 线程生命周期的解绑
- 每个Thread向 ThreadLocal 中set值时, 都会向 `threadLocals` 中存一个,  get时也是直接用 `threadLocals` 找对应的值

#### Hash碰撞避免?
算了个魔数 `0x61c88647` 和斐波那契数列有关，可以使hash值均匀分布在$2^n$的数组中, 尽量避免hash冲突
如果冲突了, 直接覆盖原值

### 应用场景
*共享变量需要实现线程隔离*时使用
1. 跨层传递信息, 多层方法都需要访问一个共享变量,  为了避免线程不安全, 可以使用ThreadLocal
2. 隔离线程, 线程不安全的工具类
3. Spring的事务是使用ThreadLocal的
4. SpringMVC的HttpSession servlet 是使用ThreadLocal的

### 内存泄漏
[[JVM#内存溢出和内存泄漏]]
> [!danger] JAVA开发手册-强制
> 必须回收自定义的 ThreadLocal 变量记录的当前线程的值，尤其在线程池场景下，线程经常会被复用，如果不清理自定义的 ThreadLocal 变量，可能会影响后续业务逻辑和造成内存泄露等问题。尽量在代码中使用 try-finally 块进行回收。







# 线程池
如果*并发多*，并且每个线程都*执行短*任务就结束了，这样*频繁创建*线程就会大大*降低效率*
![[Pasted image 20240306004452.png]]

### 任务执行机制

![](https://oss.javaguide.cn/github/javaguide/java/concurrent/thread-pool-principle.png)

提交任务->核心线程运->核心线程用完了->加到队列->队列满了->非核心线程运行->所有线程都用完了->[抛弃策略](https://learn.skyofit.com/archives/490)


### 生命周期

| 状态         | 描述                            |
| ---------- | ----------------------------- |
| RUNNING    | 运行中：可以接受新任务，可以继续处理阻塞任务        |
| SHUTDOWN   | 关闭线程池（不接受新任务,提交报错）继续处理阻塞队列的任务 |
| STOP       | 关闭线程池并中断任务                    |
| TIDYING    | 所有任务都执行完了，且worker全都关了         |
| TERMINATED | 结束线程池                         |

![[Pasted image 20240306005112.png]]

##### 回收核心线程
不能直接回收核心线程，通过`setCorePoolSize(newCorepoolSize)`函数可以设置核心线程数量，但是不会直接删掉核心线程
系统不知道那个是核心线程，只知道有多少个线程`getPoolSize()`，删除线程是通过`keepAliveTime`的时间，如果超过了时间线程数大于核心线程数，就删掉一个线程

`allowCoreThreadTimeOut(boolean value)` 可以设置核心线程的销毁时间



### 参数
```java
public ThreadPoolExecutor(int corePoolSize,
						int maximumPoolSize,
						long keepAliveTime,
						TimeUnit unit,
						BlockingQueue<Runnable> workQueue,
						ThreadFactory threadFactory,
						RejectedExecutionHandler handler)
```

1. `int corePoolSize`
    1. 线程池的核心线程数
    2. 即便是线程池里没有任何任务，也会有corePoolSize个线程在候着等任务。
2. `int maximumPoolSize`
    1. 最大线程数。
    2. 超过此数量，会触发拒绝策略。
3. `long keepAliveTime`
    1. 线程的存活时间。
    2. 当线程池里的线程数大于corePoolSize时，如果等了keepAliveTime时长还没有任务可执行，则线程退出。
4. `TimeUnit unit`
    1. 指定keepAliveTime的单位
    2. 比如：秒：TimeUnit.SECONDS。
5. `BlockingQueue<Runnable> workQueue`
    1. 一个阻塞队列，提交的任务将会被放到这个队列里。
6. `ThreadFactory threadFactory`
    1. 线程工厂，用来创建线程
    2. 主要是为了给线程起名字，默认工厂的线程名字：pool-1-thread-3。
7. `RejectedExecutionHandler handler`
    1. 拒绝策略
    2. 当线程池里线程被耗尽，且队列也满了的时候会调用。
    3. 默认拒绝策略为AbortPolicy。即：不执行此任务，而且抛出一个运行时异常
#### keepAliveTime
怎么实现到时关闭线程？

```java
Runnable r = (allowCoreThreadTimeOut || workerCount > corePoolSize) ?  
    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :  
    workQueue.take();
if (r == null) timedOut = true;
```

1. 通过阻塞队列`poll(keepAliveTime)`获取任务，设置阻塞时间为`keepAliveTime`，如果时间到了获取出来的值是null，则说明没任务了，而且存活时间到了，就kill掉线程
2. 如果开启了allowCoreThreadTimeOut，则不判断当前线程数超没超核心线程数，直接用阻塞获取任务
3. 否则当前线程数超没超核心线程数，超了就用阻塞，没超就不阻塞获取


#### 参数选择
##### 核心线程数
**(理论的)合理设置**：N*(1+线程等待时间/线程运行时间)
但是还是不靠谱，因为系统还创建了其他线程，其也会占用CPU，因此算出来的值不合适
**用压测**试出来一个合理值


`CPU 密集型任务(N+1)`： 这种任务消耗的主要是 CPU 资源，可以将线程数设置为 N（CPU 核心数）+1。比 CPU 核心数多出来的一个线程是为了防止线程偶发的缺页中断，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。
`I/O 密集型任务(2N)`： 这种任务应用起来，系统会用大部分的时间来处理 I/O 交互，而线程在处理 I/O 的时间段内不会占用 CPU 来处理，这时就可以将 CPU 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 2N。

### 线程池拒绝策略
#### AbortPolicy 丢弃并抛出异常
默认策略
throw RejectedExecutionException
#### DiscardPolicy 直接丢弃
直接丢弃，不处理

#### DiscardOldestPolicy 抛弃最老的
移除队列最老任务，重新尝试执行当前任务

#### CallerRunsPolicy 
通过调用线程执行任务
就是用控制线程池的线程执行任务，所以可能阻塞后面的线程

#### 自定义线程
实现 `RejectedExecutionHandler`，自定义任务拒绝处理逻辑


### Executors 线程池类型
> [!error] 强制 - Java 开发手册
> 强制线程池不允许使用Executors创建， 而是通过ThreadPoolExecutor创建，明确线程池的运行规则，避免资源耗尽的风险

| 种类                            | 核心线程数 | 最大线程数               | 任务队列                  | 描述                                    |
| ----------------------------- | ----- | ------------------- | --------------------- | ------------------------------------- |
| FixedThreadPool               | n     | n                   | `LinkedBlockingQueue` | 创建一个固定大小的线程池，适用于负载较重的服务器。             |
| SingleThreadExecutor          | 1     | 1                   | `LinkedBlockingQueue` | 创建一个单线程的Executor，保证所有任务按顺序执行。         |
| ScheduledThreadPool           | n     | `Integer.MAX_VALUE` | `DelayedWorkQueue`    | 创建一个能延迟或定期执行任务的线程池。                   |
| CachedThreadPool              | 0     | `Integer.MAX_VALUE` | `SynchronousQueue`    | 创建一个可缓存线程的线程池，适用于执行很多短期异步任务。          |
| SingleThreadScheduledExecutor | 1     | 1                   | `DelayedWorkQueue`    | 创建一个单线程的ScheduledExecutor，支持定时和周期性任务。 |
`LinkedBlockingQueue`是无界队列，`Integer.MAX_VALUE`
`SynchronousQueue`同步队列，没有容量，不存储元素，在CachedThreadPool中保证提交的任务有线程来处理
`DelayedWorkQueue` 按照任务的执行时间排序，底层采用堆结构，如果满了就自动扩容到原来的1.5倍





### Ruannable 和 Callable
![[#Runnable vs Callable]]

### execute() vs submit()
`execute()`只能执行任务，不知道任务是否成功执行
`submit()`会创建一个`Future`对象，通过`Future` 的 `get()`方法可以阻塞到执行结束，并获取返回值，`get(long timeout，TimeUnit unit)`方法可以传入参数，如果超过时间会抛出异常

### 实践
https://javaguide.cn/java/concurrent/java-thread-pool-best-practices.html


#### 不同业务用不同的线程池
如果父任务子任务和公用一个线程池，线程池被父任务占满了，子任务没法执行，父任务就没法结束，就不能释放线程资源，造成**死锁**

# 锁

## ReentrantLock
ReentrantLock（可重入独占式锁）
AQS队列实现，性能高 & CPU占用高
**可以中断** 在等待过程中`lockInterruptibly()`中断
**可轮询**  在规定时间里，反复`Try Lock()`，如果在规定时间内，没获得锁，则`return`失败
**可以条件** 可以关联多个 `Condition` 对象，允许线程在特定条件下等待或通知其他线程
**可以公平锁** 默认非公平锁
**可重入**
**尝试获取锁状态**
### 步骤
1. state初始化为0，表示*未锁定状态*
2. A线程lock()时，会调用`tryAcquire()`获取锁并将*state+1*
3. *其他线程`tryAcquire`获取锁会失败*，直到A线程`unlock()` 到state=0，其他线程才有机会获取该锁。
4. A释放锁之前，自己可以**重复获取此锁**（state累加），这就是可重入的概念。

reentrantLock and  synchronized的区别

| ReentrantLock     | Synchronized  |
| ----------------- | ------------- |
| 可重入的互斥锁           | 非可重入的同步块和方法   |
| 必须*手动加锁和解锁*       | 进入和退出时自动加锁和解锁 |
| 可以设置公平性策略         | 没有公平性策略       |
| 可以使用尝试获取锁的方式      | 不能尝试获取锁       |
| *支持中断*和超时操作  避免死锁 | 不支持中断和超时操作    |
| 可以获取锁的等待时间和状态     | 不能设置等待时间      |
| 可以获取锁的持有次数        | 不能获取锁的持有次数    |
| 在JDK基于AQS实现       | 在JVM实现        |
|                   |               |

## synchronized
- `效率低`：锁的释放情况少，只有代码执行完毕或者异常结束才会释放锁；
	- 试图获取锁的时候*不能设定超时*
	- *不能中断*一个正在使用锁的线程
	- 没有获得锁的线程只能等待
- `不够灵活`：加锁和释放的时机单一，每个锁仅有一个单一的条件(某个对象)，相对而言，读写锁更加灵活
- `无法知道是否成功获得锁`，相对而言，Lock可以通过TryLock拿到状态
- `可见性`，synchronized解锁前会把线程本地内存的修改刷新到主存中
### 锁升级

无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁(**不可逆**)
**偏向锁**通过对比`Mark Word`解决加锁问题，避免`CAS`。(用于*不存在竞争，总由同一个线程获取*)
**轻量级锁**是通过`CAS`和*自旋*来解决加锁问题，避免阻塞和唤醒而影响性能。
**重量级锁**是将除了拥有锁的线程以外的*其他线程都阻塞*，底层用汇编的monitor实现

![锁升级](https://img2018.cnblogs.com/blog/720367/201909/720367-20190922214211591-1031710735.png)
#### mark-word
java每个对象的对象头中， 都有32或者64位的 mark-word。java的锁就是通过对象头的mark-word实现的
![](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/519/984/817/2850086000519984817.20220705154830.09462421367001285363814861368420:50001231000000:2800:80D289433B3763DBA15D5005AC2E8188FD6050EFFFFC1164D6955DD886BCF83C.png)
除了markword中的2位锁状态标志位， 其他62位都会随着锁状态标志位的变化而变化。
这里先列出各锁状态标志位代表的当前对象所用锁的情况。后面会详细解释各种锁的含义和运行原理。
- 锁状态标志位为01： 属于无锁或者偏向锁状态。因此还需要额外的偏向锁标记位1bit来确认是无锁还是偏向锁
- 锁状态标志位为00： 轻量级锁
- 锁状态标志位为10： 重量级锁
- 锁状态标志位为11： 已经被gc标记，即将释放

mark-word 偏向锁中存储了*线程ID*，只有不同的线程id才会触发升级
#### 升级机制
1. 线程1根据mark-word设置偏向锁
2. 线程2访问锁，发现线程ID不同，升级为轻量级锁
3. 线程2通过CAS自旋等待，当自旋次数超过阈值，升级为重量级锁
4. 线程2挂起，加到内核态的等待队列中，操作系统申请资源
由于重量级锁需要从内核态转向用户态，消耗时间，所以**重**

#### 锁降级
HotSpot JVM 锁是可以降级的 or 重新偏向锁
但是如果频繁升降级会对JVM性能会造成影响


### JDK1.6 锁优化
[[锁优化]]
1.5的原子类基本都是自旋锁实现的，有局限性
以下是JDK1.6的优化：
1. [[#2. 自旋锁 VS 适应性自旋锁|自适应自旋锁]]
2. [[#锁升级]]
3. 锁消除
4. 锁粗化

##### 锁消除
JIT运行时，会进行分析逃逸分析，如果逃逸分析能够确定一个变量不会逃逸出线程，无法被其他线程访问，那么这个变量的读写肯定就不会有竞争，对这个变量实施的同步措施也就可以安全地消除掉。

##### 锁粗化
原则上，推荐将同步块的作用范围限制得尽量小，只在共享数据的实际作用域上进行同步，这样是为了使得需要同步的操作数量尽可能变少，即使存在锁竞争，等待锁的线程也能尽可能快地拿到锁。
但如果一系列的连续操作都对同一个对象反复加、解锁，那即使没有线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。



### 对类加锁
实例方法：实际上是对调用该方法的*对象*加锁
静态方法：实际上是对该*类*进行加锁，而不属于某个对象

```java
public void func() {
    synchronized (SynchronizedExample.class) {
        // ...
    }
}
```
作用于整个类，也就是说两个线程调用*同一个类的不同对象*上的这种同步语句，也会进行同步。
```java
public class SynchronizedExample {

    public void func2() {
        synchronized (SynchronizedExample.class) {
            for (int i = 0; i < 10; i++) {
                System.out.print(i + " ");
            }
        }
    }
}
```

```java
public static void main(String[] args) {
    SynchronizedExample e1 = new SynchronizedExample();
    SynchronizedExample e2 = new SynchronizedExample();
    ExecutorService executorService = Executors.newCachedThreadPool();
    executorService.execute(() -> e1.func2());
    executorService.execute(() -> e2.func2());
}
```

```
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9
```



### synchronized 锁失效
https://blog.csdn.net/C_AJing/article/details/105992563
@Transactional 事务中可能出现锁失效
使用注解会通过动态代理实现事务，即：开启事务->加锁->执行业务代码->释放锁->提交事务
在线程1执行完业务，释放锁尚未提交时，线程2由于事务隔离级别，看不到未提交的数据，线程2再次执行业务代码，出现**锁失效**

解决方法：加锁和释放锁放在AOP外面

## CountDownLatch
CountDownLatch底层基于AQS实现，调用`countDown`时，减小state的值，如果state减小到0，则唤醒下一个等待的线程
##### countDown()源码
```java
public void countDown() {
	sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
	if (tryReleaseShared(arg)) {
		signalNext(head);
		return true;
	}
	return false;
}

// 重写AQS的tryReleaseShared函数
protected boolean tryReleaseShared(int releases) {
	// Decrement count; signal when transition to zero
	for (;;) {
		int c = getState();
		if (c == 0)
			return false;
		int nextc = c - 1;
		if (compareAndSetState(c, nextc))
			return nextc == 0;
	}
}
```


## volatile
轻量级锁
保证*可见和有序性*  不保证原子性
- 当写一个 volatile 变量时，JMM 会把该线程在本地内存中的变量*强制刷新到主存*；
- 这个写操作会*导致其他线程*中的 volatile 变量*缓存无效*
1.  可见性
	- 每个线程都可以访问`volatile` 变量
2. 有序性
	- 禁止指令重排序
3. 原子性
	- 不具备互斥
	- 使用锁机制 / 原子类
	- ***32-bit***系统不保证64-bit数据的原子性  因为32位系统的处理器无法原子地读写64位数据
	- 
符合[[#Happens-Before规则]]

## 死锁
### 四大条件
1. 互斥：资源不共享
2. 请求与保持：不主动释放资源
3. 不剥夺：不可以抢占资源
4. 循环等待：无限等待原则

### 死锁的避免：银行家算法
找到一种资源分配的顺序，使其可以执行完的序列
![银行家算法](https://img-blog.csdn.net/20180508210518867)

### 死锁的判断：资源图

## 锁分类
![[Pasted image 20240305204829.png]]
### 1. 乐观锁 VS 悲观锁
乐观锁：默认没锁，直接CAS，如果执行过程发现CAS不匹配了（被其他线程修改了），报错or重试
悲观锁：默认有锁，尝试获取锁，得到锁后执行操作

- **悲观锁适合*写操作多*的场景**，先加锁可以保证写操作时数据正确。
- **乐观锁适合*读操作多*的场景**，不加锁的特点能够使其读操作的性能大幅提升。
##### 乐观锁：CAS
步骤
1. 比较**访问的值和预期值是否一致**
2. 一致则交换值
3. 否则**自旋**
**原子性**，不同指令架构实现了CAS的底层实现保证其原子性


### 2. 自旋锁 VS 适应性自旋锁
适应性自旋锁可以限制自旋次数，超过次数就阻塞。

阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，这种状态转换需要*耗费CPU时间*。如果同步代码块中的内容过于简单，*状态转换消耗的时间有可能比用户代码执行的时间还要长*。

### 3. 无锁 VS 偏向锁 VS 轻量级锁 VS 重量级锁
![[#锁升级]]

### 4. 公平锁 VS 非公平锁
[[#ReentrantLock]]分别实现了这两种锁，new对象的时候配置参数，*默认非公平*
公平锁：直接排队，需要阻塞线程，让操作系统再唤醒一次
非公平锁：尝试插队，插队成功则可以避免线程的阻塞（即CPU的切换），失败则排队

### 5. 可重入锁 VS 非可重入锁
[[#ReentrantLock]]是可重入的，Synchronized不可重入
分布式锁中，setnx是不可重入的，redisson可以实现可重入锁
可重入锁的一个优点是可*一定程度避免死锁*

### 6. 独享锁(排他锁) VS 共享锁
如果线程T对数据A加上排它锁后，则其他线程不能再对A加任何类型的锁。
`ReentrantReadWriteLock ReadLock/WriteLock`



# AQS
很多并发类都是基于AQS(Abstract Quened Synchronizer)实现的，例如：[[#ReentrantLock / Lock|ReentrantLock]]、[[#CountDownLatch]]、Semaphore、ReadWriteLock，CyclicBarrier。

![[Pasted image 20240305233420.png]]
1. `volatile`修饰共享变量state
2. 线程通过`CAS`去改变状态
3. 成功则获取锁成功
4. 没拿到锁则加入队列
**AQS将每一条请求共享资源的线程封装成一个CLH锁队列的一个结点（Node），来实现锁的分配**

AQS的底层通过Unsafe类和LockSupport实现
Unsafe实现*CAS操作*
LockSupport通过park/unpark实现*阻塞和唤醒*
##### LockSupport
如果没有`LockSupport.unpark(thread)`，那么`thread`在执行`LockSupport.park()` 命令时，会阻塞，直到有其他线程`unpark(thread)`

不可重入，可中断
```java
Thread mainThread = Thread.currentThread();

Thread thread = new Thread(()->{
	for (int i = 0; i < 5; i++) {
		try {
			Thread.sleep(500);
		} catch (InterruptedException e) {
			throw new RuntimeException(e);
		}
		LockSupport.unpark(mainThread);//释放mainThread的许可  
		LockSupport.park();// 获取thread的许可  
		System.out.println("thread " + i);
	}
});
thread.start();

for (int i = 0; i < 6; i++) {
	LockSupport.park();// 获取mainThread的许可  
	LockSupport.unpark(thread); // 释放thread的许可  
	System.out.println("Main " + i);
}
  
thread.interrupt();  
System.out.println("thread is interrupted!");
```

# 并发数据结构
## ConcurrentHashMap
#### jdk1.8
key和value 均不能为空

有写锁，没有读锁，可见性是通过volatile实现的
- Node的元素中val和指针next用volatile修饰
- 另外table数组也用volatile修饰
	- 虽然volatile修饰数组是保证了数组地址的可见性，而不是数组元素的可见性
	- 但是这样保证了table数组扩容后对其他线程的可见性

##### 初始化
自旋判断是不是还没有初始化（数组为null），已初始化则return，否则：
1. 判断sizeCtl参数，如果当前有线程正在初始化，则yield让出CPU使用权，继续自旋
2. 否则CAS修改sizeCtl值，获取轻量级锁，失败则继续自旋
3. 成功获取锁（成功修改sizeCtl）则创建数组，并退出自旋

##### put步骤
for循环自旋，期间获取对应key的Node
1. 判断Node是否为空
	1. 如果为空则尝试CAS修改这个值
	2. 成功则return，失败则自旋
2. 数组在此期间如果被扩容了（当前节点变成了转发节点（forwardNode），其中记录了新数组的指针)，则获取新的数组，然后继续自旋
3. 如果是onlyIfAbsent，且key相同，则不获取锁，直接返回
4. 否则，对头节点加synchronized锁
5. 加锁后，再次检查Node是不是修改了，如果被修改，则释放synchronized锁，继续自旋
6. 未修改则正常put（和HashMap相同）
7. put后增加map的元素个数，在这个操作中实现数组扩容
8. 数组扩容通过synchronized锁原数组头节点实现
9. 扩容时原数组仍可以访问，通过forwardNode，将原数组节点请求的转发到新数组上
10. ps:1.8的ConcurrentHashMap扩容还是用的头插法，但是因为synchronized锁，不会有死循环问题

#### jdk1.7
把数组分成多个Segment，对Segment加[[#ReentrantLock]]锁
value 不能为空
##### Segment
Segment的数量是刚好>=并发级别的2的幂
Segment数组的长度的最小值是2，（可能比Map.size()/Segment.length()大

##### put步骤
1. 找到key对应的Segment
	1. 如果不存在，则通过多次if判断是否仍然不存在（避免其他线程成功创建Segment）
	2. 仍不存在则new一个Segment，并通过CAS替换数组中的Segment
	3. 否则直接返回这个已存在的Segment
2. 对Segment TryLock获取锁
	1. 不成功则进入自旋，不断TryLock，且在此期间创建新Entry的对象
	2. 自旋期间记录第几次TryLock
	3. 如果第一次，就新建一个对应Entry的Node
	4. 如果自旋次数超过60次，则结束自旋，直接lock()阻塞获取锁
	5. 否则每两次判断：当前节点是不是插入了新节点，即队头节点变了（jdk7是头插法），如果变了，则重置自旋次数
3. 成功加锁后，将新建的Entry插入到数组中
	1. 先遍历链表，查找有没有相同key，有则用新值覆盖
	2. 否则头插法插入新节点
4. 释放锁


### HashTable
value不能为null
为了实现并发安全，HashTable在put时直接对整个HashTable加锁（在put函数上加上synchronized）



# 阻塞
### wait
是Object类中的native方法，参数为0时，无限等待，否则等待参数设定的时间
`wait()` 会一直等待`notify() / notifyAll() / interrupt()`，否则一直阻塞，阻塞期间，**线程会释放锁**

> [!info ]   `wait / notify` 都是 native方法，没有源码，以下是wait方法的注解
>  Causes the current thread to wait until it is awakened, typically by being *notified or interrupted*, or until a certain *amount of real time has elapsed*.

### await
相当于wait，等待`signal() / signalAll()`
await可以通过`Condition`实例指定等待的条件，更灵活。
`Condition`底层用[[#AQS]]实现，每个Condition对象维护一个AQS（实际上是记录了AQS Node的头尾两个指针
调用await时，在AQS中加一个节点，signal时唤醒第一个节点，signalAll时循环唤醒所有的节点，但是唤醒的线程需要争抢锁，可能抢不到，如果没抢到，则加到AQS后面
```java
// signal -> doSignal(first, false);
// signalAll -> doSignal(first, true);

private void doSignal(ConditionNode first, boolean all) {
	while (first != null) { // 不断循环，用于唤醒所有线程
		ConditionNode next = first.nextWaiter;
		if ((firstWaiter = next) == null)
			lastWaiter = null;
		if ((first.getAndUnsetStatus(COND) & COND) != 0) {
			enqueue(first);  // 当前节点加到最后
			if (!all)  // 如果不是signalAll，直接退出，不继续唤醒线程
				break;
		}
		first = next;
	}
}

```

##### demo
```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();

public void before() {
	lock.lock();
	try {
		System.out.println("before");
		condition.signalAll();
	} finally {
		lock.unlock();
	}
}

public void after() {
	lock.lock();
	try {
		condition.await();
		System.out.println("after");
	} catch (InterruptedException e) {
		e.printStackTrace();
	} finally {
		lock.unlock();
	}
}
```

# 实践
### notifyAll 和 线程池
线程池参数 `corePoolSize = 2, maxPoolSize = 2, BlockingQueue.size = 5`
线程池中创建4个线程，每个线程都`condition.await()`，阻塞等待
调用`condition.signalAll();`，但只能唤醒运行的线程，而不能唤醒在阻塞队列中的线程，所以队列中的线程开始执行后，会阻塞等待唤醒
```java
for (int j = 0; j < 10; j++) {
	// 线程池中创建四个线程，每个线程阻塞等待condition.signal()
	for (int i = 0; i < 4; i++) {
		poolExecutor.execute(()->{
			reentrantLock.lock();
			try {
				condition.await();
				System.out.println(Thread.currentThread().getName());
			} catch (Exception e) {
				throw new RuntimeException(e);
			}
			reentrantLock.unlock();});
	}

	Thread.sleep(10);
	reentrantLock.lock();
	condition.signalAll();
	reentrantLock.unlock();

	System.out.println("done : " + poolExecutor.getQueue().size());
}
```

##### 解决：
❌ 尝试1：用`lock.hasWaiters(condition)`方法检测有多少等待的，用while循环不断通知其他线程
没效果，因为只有阻塞队列的线程开始执行才能加入到condition的waiter中，所以没用

❌ 尝试2：用`join()`方法，不用signal，因为join在线程结束/中断后都可以执行，不用主动唤醒了
但是如果并发量太大，还是会触发抛弃

❌ 尝试3：在`join`的基础上，扩大线程池的阻塞队列大小，并发量大情况下还是不能解决

✔ 尝试4：扩大线程池后再用消息队列，慢慢处理任务，消峰填谷

