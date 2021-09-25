# 使用线程
- 继承 Thread 类
- 实现 Runnable 接口
- 实现 Callable 接口 (带返回值)
- 使用线程池

实现两个接口的类只能当作一个可在线程中执行的任务, 不是真正意义上的线程, 最后还是得通过 Thread 类来调用
## 实现 Callable 接口
与 Runnable 相比, Callable 可以有返回值, 线程池里, 返回值通过 FutureTask 进行封装

```java
public class MyCallable implements Callable<Integer> {
    public Integer call() {
        return 123;
    }
}
public static void main(String[] args) throws ExecutionException, InterruptedException {
    MyCallable mc = new MyCallable();
    FutureTask<Integer> ft = new FutureTask<>(mc);
    Thread thread = new Thread(ft);
    thread.start();
    System.out.println(ft.get());
}
```

Callable 与 Runnable 一样不能直接运行, 直接借助 Thread 类, 但是 Thread 类没有提供直接运行 Callable 接口的方法, 我们需要借助 FutureTask

```java
//创建FutureTask
FutureTask<String> futureTask = new FutureTask<>(callable);
```

FutureTask 是一个既实现 Future, 又实现 Runnable 接口的类, 所以本质还是一个 Runnable 
```java
//创建线程
Thread thread = new Thread(future);
//启动线程      
thread.start();
```

## 继承 Thread 类
同样也是需要实现 run 方法, 因为 Thread 类也实现了 Runnable 接口

## 实现接口 Vs 继承 Thread
实现接口更好一点
- Java 不允许多重继承, 因此继承 Thread 类无法继承其他类, 而可以实现多个接口
- 继承 Thread 类可能开销过大

# 线程池
利用池化思想来管理线程
**好处:**
1. 降低系统资源消耗: 避免反复创建和销毁线程带来的消耗
2. 提高响应时间: 任务到达时, 无需再等待线程创建
3. 统一管理线程: 线程属于稀缺资源, 无限制创建线程, 不仅消耗资源, 而且可能导致资源调度失衡
4. 提供额外功能: 比如任务延后或定期执行, 如延时定时线程池ScheduledThreadPoolExecutor

## 线程池的实现
### 总体设计
![](https://p1.meituan.net/travelcube/912883e51327e0c7a9d753d11896326511272.png)
在 JUC 包下
顶层是 Executor 接口, 提供了一种思想: 将任务提交和任务执行解耦, 用户只需提供 Runnable 对象, 无需关心有关线程的事情

ThreadPoolExecutor
![](https://p0.meituan.net/travelcube/77441586f6b312a54264e3fcf5eebe2663494.png)

构造了一种生产者消费者的模型, 将任务与线程解耦
线程池主要分成: 1. 任务管理, 2. 线程管理两部分

### 线程池的生命周期
线程池的状态, 并不是由用户手动设置, 而是由内部来维护.
线程池使用一个变量维护了两个值: 运行状态 (runstate) 和线程数量 (workerCount)
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```
高 3 位保存 runstate, 低 29 位保存 workercount. 用一个变量维护两个值, 可避免做决策时, 出现不一致的现象, 即避免了加锁来保证两个值一致正确

ThreadPoolExecutor 的状态有
![](https://p0.meituan.net/travelcube/62853fa44bfa47d63143babe3b5a4c6e82532.png)

生命周期转换如下
![](https://p0.meituan.net/travelcube/582d1606d57ff99aa0e5f8fc59c7819329028.png)

### 任务执行
#### 任务调度
任务提交后 (submit / execute)
1. 检查线程池运行状态, 如果不是 Runnable, 则直接拒绝
2. 如果 workerCount < corePoolSize, 则创建一个新线程执行
3. 如果 workerCount >= corePoolSize 且阻塞队列没有满, 则添加到阻塞队列
4. 如果 workerCount >= corePoolSize 且阻塞队列已满, 但 workerCount < maximumPoolSize, 则新建线程执行
5. 如果都不满足, 则根据拒绝策略决定, ==默认直接抛出异常==

![](https://p0.meituan.net/travelcube/31bad766983e212431077ca8da92762050214.png)

##### submit 和 execute 的区别
1. execute()方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功与否；
2. submit()方法用于提交需要返回值的任务。线程池会返回一个 Future 类型的对象，通过这个 Future 对象可以判断任务是否执行成功 ，并且可以通过 Future 的 get()方法来获取返回值，get()方法会阻塞当前线程直到任务完成，而使用 get（long timeout，TimeUnit unit）方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

#### 任务缓冲
阻塞队列(BlockingQueue)是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用
![](https://p1.meituan.net/travelcube/f4d89c87acf102b45be8ccf3ed83352a9497.png)

可选的阻塞队列:
![](https://p0.meituan.net/travelcube/725a3db5114d95675f2098c12dc331c3316963.png)

[如何选择队列](https://blog.hufeifei.cn/2018/08/12/Java/ThreadPoolExecutor%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5--%E5%A6%82%E4%BD%95%E9%80%89%E6%8B%A9%E9%98%9F%E5%88%97/)
#### 任务申请
大多数时候, 执行完任务的线程会去阻塞队列获取新的任务, 这部分策略在 `getTask()` 函数里
![](https://p0.meituan.net/travelcube/49d8041f8480aba5ef59079fcc7143b996706.png)

#### 任务拒绝
如果 workerCount > maximumPoolSize 并且阻塞队列已满时, 再提交任务则会触发任务拒绝
![](https://p0.meituan.net/travelcube/9ffb64cc4c64c0cb8d38dac01c89c905178456.png)

#### worker 线程管理
```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable{
    final Thread thread;//Worker持有的线程
    Runnable firstTask;//初始化的任务，可以为null
}
```

worker 实现了 Runnable 接口, 并且持有一个线程和初始任务 firstTask
![](https://p0.meituan.net/travelcube/03268b9dc49bd30bb63064421bb036bf90315.png)

线程池持有一张 hashMap 来持有 worker 的引用, 这样可以通过添加, 移去引用来控制线程的生命周期, 此时判断线程的情况便显得重要

worker 继承 AQS, 并重写了 `tryAcquire` 方法, 实现不可重入的特性
1. 接到任务时, lock 方法获取独占锁, 表示当前线程正在执行任务
2. 如果当前 worker 没有锁, 则说明没有任务, 可以对其中断
3. 线程池在 shutdown 或 tryTerminate 时会调用 interruptIdleWorkers 来中断空闲线程, 其中便是通过 worker.lock 方法来判断是否可回收

![](https://p1.meituan.net/travelcube/9d8dc9cebe59122127460f81a98894bb34085.png)

##### worker 线程增加
`addworker(firstTask, core)` 方法里, core 为 true 会判断当前活动线程是否小于 corePoolSize, 为 false 会判断是否小于 maximumPoolSize
![](https://p0.meituan.net/travelcube/49527b1bb385f0f43529e57b614f59ae145454.png)

##### 线程回收
线程销毁依赖于 jvm 垃圾回收机制. 其中, 核心线程会无限等待任务, 而非核心任务会限时获取任务, 当无法获取时, 会主动消除引用
```java
try {
  while (task != null || (task = getTask()) != null) {
    //执行任务
  }
} finally {
  processWorkerExit(w, completedAbruptly);//获取不到任务时，主动回收自己
}
```

##### worker 线程执行任务
worker 类中的 run 方法调用了 ThreadPoolExecutor 的 runWorker 方法
1. while 不断地通过 getTask 获取任务
2. getTask 方法从阻塞队列获取任务, 获得 null, 到5
3. 如果线程池正在停止, 那么确保线程是中断状态, 否则, 确保线程是非中断状态
4. 执行任务, 到1
5. 销毁线程

![](https://p0.meituan.net/travelcube/879edb4f06043d76cea27a3ff358cb1d45243.png)

### 常见线程池

### 线程池核心参数配置
- CPU 密集型任务(N+1)： 这种任务消耗的主要是 CPU 资源，可以将线程数设置为 N（CPU 核心数）+1，比 CPU 核心数多出来的一个线程是为了防止线程偶发的缺页中断，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。
- I/O 密集型任务(2N)： 这种任务应用起来，系统会用大部分的时间来处理 I/O 交互，而线程在处理 I/O 的时间段内不会占用 CPU 来处理，这时就可以将 CPU 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 2N。

# 线程的生命周期和状态
Java 线程在运行的生命周期中的指定时刻只可能处于下面 6 种不同状态的其中一个状态
![](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/19-1-29/Java%E7%BA%BF%E7%A8%8B%E7%9A%84%E7%8A%B6%E6%80%81.png)
![](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/19-1-29/Java+%E7%BA%BF%E7%A8%8B%E7%8A%B6%E6%80%81%E5%8F%98%E8%BF%81.png)

## sleep
不释放锁, Thread 的方法
## wait 
释放锁, object 的方法, 必须取得该 object 的锁才能执行
## join
阻塞到某个线程执行完成, 释放锁 (底层使用 wait), 
```java
Thread a = new Thread(){
    public synchronized void join() {
        while (isAlive()) {
            wait(0);
        }
    }
    
};
Thread b = new Thread(){
    public void run() {
        //主要用于等待a线程运行结束
        //底层使用 wait
        //相当于 b 线程拿到 a 线程这个线程对象的锁
        a.join();
    }
};
```
## yield
让出当前 cpu, 可能又回到自己, 不释放锁
# 中断
一个线程执行完毕会自动退出, 如果执行过程发生异常也会提前结束

## InterruptedException
调用一个类的 interrupt() 方法来中断线程执行, 如果线程处于 blocked, timed_waiting, waiting 的状态, 就会抛出该异常, 中断执行. 但不能中断 i/o 阻塞或 synchronized 锁阻塞

# 互斥同步 (加锁)
- synchronized (jvm 实现)
- ReentrantLock (jdk 实现)

## synchronized
在 Java 早期版本中，synchronized 属于 重量级锁，效率低下。
因为监控器锁 (monitor) 依赖底层操作系统 mutex Lock 实现, Java 的线程映射到操作系统的原生线程上, 因此挂起和唤醒都需要操作系统实现, 而操作系统的线程切换需要从用户态转为内核态, 这个转换的时间成本较高
==注意: 对象的wait(), notify() 等方法需要在同步代码块里才能调用, 否则会抛异常==

### 使用
1. 修饰实例方法 (锁住该对象)
2. 修饰静态方法 (锁住类)
==注意: 允许两个线程, 在同一时间, 一个访问类的同步方法, 一个访问实例的同步方法, 因为这两个方法锁住的对象不同==
3. 修饰代码块 (锁住括号里面的对象)
==注意: 尽量不要锁住 String, Integer 这类有缓存池的对象==

### Java 对象头
对象头包含三部分
1. **markword**
存储 hashcode, gc分代年龄, 锁标志, 偏向线程
2. **类型指针**
指向对象的类元数据, jvm 通过此指针确定是哪个类的实例
3. **数组特有的储存长度**

对象头
 长度 | 内容 | 说明
 - | - | - 
 一个字(32 bits/ 64bits) | markword | 存储hashcode等信息
 同上 | Class metadata Addr | 存储对象类型指针
 同上 | array length | 数组长度

### synchronized 锁分类
级别低到高依次是
1. 无锁
2. 偏向锁
3. 轻量锁 (自旋)
4. 重量锁

锁只能升级, 不能降级

==各状态 **markword**==

**无锁**
25 bits | 4 bits | 1 bit (是否偏向) | 2 bits
 - | - | - | -
hashcode | gc 分代年龄 | 0 | 01

**偏向锁**
23 bits | 2 bits | 4 bits | 1 bit (是否偏向) | 2 bits
 - | - | - | - | -
线程id | epoch | gc 分代年龄 | 1 | 01

**轻量锁**
30 bits  | 2 bits
- | -
指向==线程栈中的锁记录== | 00

**重量锁**
30 bits  | 2 bits
- | -
指向==monitor== | 10

### 加锁过程

![](https://upload-images.jianshu.io/upload_images/4491294-e3bcefb2bacea224.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)
![](./java-imgs/synchronized.jpg)

1. 判断锁对象的 markword 最后两位, 来判断锁对象此时有没有上锁
2. 接着判断偏向锁标志位是否为 1 , 如果不是, 则进行 CAS 竞争轻量锁, 即到 5
3. (偏向锁) 此时判断 markword 的 threadId 是否为本线程, 如果是, 则添加一条 lock record, 表示重入次数. 如果否, 则判断锁对象的 epoch 和该类的 cEpoch 是否相同
    1. 如果 epoch < cEpoch, 说明发生过批量重偏向, 直接CAS加偏向锁
    2. 如果 epoch = cEpoch, 竞争锁
4. 竞争锁得等到全局安全点(safe point，代表了一个状态，在该状态下所有线程都是暂停的), 此时有两个判断
    1. 持有偏向锁的对象的线程已退出同步代码块, 或者持有锁对象的线程已不存活, 则撤销偏向锁, 恢复到无锁状态(且不能偏向, 最后三位 001)(==PS:仅对该对象而言, 当同一个类的多个实例变量发生了偏向锁撤销时, 会将类的 cepoch + 1==)
    2. 持有锁对象的线程仍存活且在同步代码块内, 则锁升级为轻量锁, 且该线程仍持有锁(优先把锁对象的对象头指向原持有锁的线程)
5. (轻量锁) 先在线程的栈帧里赋值当前锁对象的markword, 然后 CAS 替换, 成功则获取到锁, 失败则首先自旋, 当自旋次数到达一定次数进入锁膨胀(或此时再有其他线程争夺锁), 膨胀的重量锁的 monitor 的 owner 将指向持有锁的线程
6. (重量锁) 此时锁对象的markword将指向monitor对象, 获取不到锁的线程将进入 EntrySet 集合中, 并处于 blocked (阻塞)状态
7. (轻量锁的释放) 获得锁对象的线程进行 CAS 将自己拥有的 Markword 替换回此时锁对象的 markword, 如果成功, 则直接退出. 如果失败, 说明此时已升级为重量锁, 唤醒其他线程进行新一轮竞争

特别说明:
1. CAS记录owner时，expected == null，newValue == ownerThreadId，因此，只有第一个申请偏向锁的线程能够返回成功，后续线程都必然失败
2. 当一个对象已经计算过identity hash code，它就无法进入偏向锁状态
3. 当一个对象当前正处于偏向锁状态，并且需要计算其identity hash code的话，则它的偏向锁会被撤销，并且锁会膨胀为重量锁
4. 重量锁的实现中，ObjectMonitor类里有字段可以记录非加锁状态下的mark word，其中可以存储identity hash code的值。或者简单说就是重量锁可以存下identity hash code。
    这里讨论的hash code都只针对identity hash code。用户自定义的hashCode()方法所返回的值跟这里讨论的不是一回事。Identity hash code是未被覆写的 java.lang.Object.hashCode() 或者 java.lang.System.identityHashCode(Object) 所返回的值。
5. 重量锁还有一个 waitSet 集合, 用于线程调用锁对象的 wait() 方法时, 将线程加入此集合中(此时会释放锁), 等待其他线程调用 notify() 方法将其唤醒

### synchronized 与 ReentrantLock 的区别
1. 都是可重入锁
2. synchronized 依赖 jvm 而 ReentrantLock 依赖 API
    - synchronized 是虚拟机层面实现的，并没有直接暴露给我们。ReentrantLock 是 JDK 层面实现的（也就是 API 层面，需要 lock() 和 unlock() 方法配合 try/finally 语句块来完成），所以我们可以通过查看它的源代码，来看它是如何实现的。
3. ReentrantLock 增加了一些高级功能
    - 等待可中断: 提供一种能够中断等待锁的线程的机制, 通过 lock.lockInterruptibly() 来实现. 也就是正在等待锁的线程可以响应中断, 放弃抢夺锁, 处理其他业务
    - 可实现公平锁(synchronized 只能非公平锁)
    - 可实现选择性通知 (锁可绑定多个条件): synchronized 可与 wait() 和 notify() 结合实现等待通知机制. ReentrantLock 也可以实现, 需要借助 Condition 接口和 newCondition 方法

    > Condition是 JDK1.5 之后才有的，它具有很好的灵活性，比如可以实现多路通知功能也就是在一个Lock对象中可以创建多个Condition实例（即对象监视器），线程对象可以注册在指定的Condition中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。 在使用notify()/notifyAll()方法进行通知时，被通知的线程是由 JVM 选择的，用ReentrantLock类结合Condition实例可以实现“选择性通知” ，这个功能非常重要，而且是 Condition 接口默认提供的。而synchronized关键字就相当于整个 Lock 对象中只有一个Condition实例，所有的线程都注册在它一个身上。如果执行notifyAll()方法的话就会通知所有处于等待状态的线程这样会造成很大的效率问题，而Condition实例的signalAll()方法 只会唤醒注册在该Condition实例中的所有等待线程。

# volatile
通过==缓存一致性协议实现==

## 单例模式
一个类只有一个实例对象, 如 Spring 中的 Service, Controller
### 好处
1. 节省内存, 不会创建多个实例对象
2. 有些类就应该被设计成单例, 比如打印机, 我们可以有多个打印功能的应用程序, 但是真正向打印机传输数据的类应该是单例的, 以避免两个打印作业同时输出到打印机中
### 多种实现
1. 饿汉式(使用静态代码块或静态变量赋值的形式), 不会有线程安全问题, 因为在类加载阶段便已经创建对象
2. 懒汉式(在实际运行时才加载)

```java
class Singleton {
    private static volatile Singleton single;

    public static Singleton getSingle() {
        if (single == null) {
            synchronized (Singleton.class) {
                if (single == null) {
                    single = new Singleton();
                }
            }
        }
        return single;
    }
}
```
volatile 修饰的目的:
    1. 保证可见性, 保证每个线程得到的都是当前内存中的状态
    2. 保证有序性, 确保 jvm 不会指令重排, 以保证得到的 single 对象一定已经创建好了

3. 静态内部类模式
```java
public class SingleTon{
  private SingleTon(){}
 
  private static class SingleTonHoler{
     private static SingleTon INSTANCE = new SingleTon();
 }
 
  public static SingleTon getInstance(){
    return SingleTonHoler.INSTANCE;
  }
}
```
此时当 Singleton 被加载进虚拟机时并不会创建实例, 因为静态内部类并没有加载, 而只有当调用 getInstance() 时才会加载, 实现延迟加载
当getInstance()方法被调用时，SingleTonHoler才在SingleTon的运行时常量池里，把符号引用替换为直接引用，这时静态对象INSTANCE也真正被创建

## CPU 缓存模型
CPU 缓存是为了解决 CPU 处理速度和内存处理速度不对等的问题。

工作方式:
先从内存复制数据到 CPU 缓存, 读取直接从缓存读取, 运算结束再写回内存

# 缓存一致协议

在多核CPU中，内存中的数据会在多个核心中存在数据副本，某一个核心发生修改操作，就产生了数据不一致的问题。而==一致性协议正是用于保证多个CPU cache之间缓存共享数据的一致。==
cache的依据, 局部性原理
## cache 结构
![](https://img-blog.csdn.net/20160103043551288?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
在单核CPU结构中，为了缓解CPU指令流水中cycle冲突，L1分成了指令（L1P）和数据（L1D）两部分，而L2则是指令和数据共存。

![](https://img-blog.csdn.net/20160103044115119?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
多核CPU的结构与单核相似，但是多了所有CPU共享的L3三级缓存。在多核CPU的结构中，L1和L2是CPU私有的，L3则是所有CPU核心共享的。

## MESI (缓存一致性)
### chche 写方式
1. write throuth(写通) : 每次 cpu 更新 cache, 立即更新到内存 (==volatile 关键字使用==)
2. write back(写回): 延迟更新

无论是写通还是写回，在多核环境下都需要处理缓存cache一致性问题。为了保证缓存一致性，处理器又提供了写失效（write invalidate）和写更新（write update）两个操作来保证cache一致性。
1. 写失效: 将 cache 标记为失效
2. 写更新: 更新 cache, 通知其他核更新数据

### cache line
![](https://img-blog.csdn.net/20160103045621163?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
cache line 是 cache 与内存数据交换的最小的单位. MESI 协议中, 状态有四种, 地址则是内存地址.

### mesi 状态
|状态|说明|
|-|-|
|Modified|当前 cpu 拥有最新的数据, 其他为 invalid, 虽然与内存不同, 但以 cache 为准
|exclusive|独占, 其他 cpu 没有, 与内存一致
|shared|当前 cpu 与其他 cpu 共享数据, 与内存一致
|Invalid|失效, 应该从主存获取, 但是是写命中

### cache 操作
MESI协议中，每个cache的控制器不仅知道自己的操作（local read和local write），每个核心的缓存控制器通过监听也知道其他CPU中cache的操作（remote read和remote write），再确定自己cache中共享数据的状态是否需要调整。

### 状态转换和 cache 操作
初始场景：在最初的时候，所有CPU中都没有数据，某一个CPU发生读操作，此时必然发生cache miss，数据从主存中读取到当前CPU的cache，状态为E（独占，只有当前CPU有数据，且和主存一致），此时如果有其他CPU也读取数据，则状态修改为S（共享，多个CPU之间拥有相同数据，并且和主存保持一致），如果其中某一个CPU发生数据修改，那么该CPU中数据状态修改为M（拥有最新数据，和主存不一致，但是以当前CPU中的为准），其他拥有该数据的核心通过缓存控制器监听到remote write行文，然后将自己拥有的数据的cache line状态修改为I（失效，和主存中的数据被认为不一致，数据不可用应该重新获取）。

#### modified
LR: 当前缓存拥有最新数据, 直接读取缓存
LW: 同上, 直接修改缓存
RR: 监听到其他 CPU 需要读取该数据, 因此需要先将当前 cache 写入内存, 再将状态转为 S
RW: 同上, 状态转为 I

#### exclusive
LR: 当前缓存拥有最新数据, 直接读取缓存
LW: 同上, 直接修改缓存, 状态修改为 M
RR: 监听到其他 CPU 需要读取该数据, 将状态转为 S
RW: 同上, 状态转为 I

#### shared
LR: 当前缓存拥有最新数据, 直接读取缓存
LW: 同上, 直接修改缓存, 状态修改为 M
RR: 监听到其他 CPU 需要读取该数据, 状态不变
RW: 同上, 状态转为 I

#### invalid
LR: 当前缓存拥有的数据已过期, 需要从内存读取, 此时其他 cpu 将监听到一个 RR 事件. 读取完状态转为 E 或 S
LW: 同上, 此时相当于命中缓存, 直接修改缓存, 状态修改为 M, 其他 cpu 监听到一个 RW 事件, 将状态转为 I, (M 状态的需要先写入内存)
RR: 与此 cpu 无关,状态不变
RW: 同上, 状态不变

### store buffer和invalid queue
缓存的一致性消息传递是要时间的，这就使其切换时会产生延迟。当一个缓存被切换状态时其他缓存收到消息完成各自的切换并且发出回应消息这么一长串的时间中CPU都会等待所有缓存响应完成。可能出现的阻塞都会导致各种各样的性能问题和稳定性问题。
#### store buffer
为了避免这种CPU运算能力的浪费，Store Bufferes被引入使用。处理器把它想要写入到主存的值写到缓存，然后继续去处理其他事情。当所有失效确认（Invalidate Acknowledge）都接收到时，数据才会最终被提交。

但又会引发其他风险
1. 就是处理器会尝试从存储缓存（Store buffer）中读取值，但它还没有进行提交。这个的解决方案称为Store Forwarding，它使得加载的时候，如果存储缓存中存在，则进行返回。
2. 什么时候完成提交, 没有保证

```java
value = 3；

void exeToCPUA(){
  value = 10;
  isFinsh = true;
}
void exeToCPUB(){
  if(isFinsh){
    //value一定等于10？！
    assert value == 10;
  }
}
```
==指令重排序的原理==: 假如 CPUA 一开始没有 value 的缓存, 却有 isfinish 的状态为 E 的cache. 此时对 value 的修改可能由 store buffer 处理. 而此时可能 CPUB 读到isFinish 是最新值, 而value则不是 
==volatile 原理==: lock 前缀的汇编指令会强制写入主存，也可避免前后指令的CPU重排序，并及时让其他核中的相应缓存行失效，从而利用MESI达到符合预期的效果.lock前缀的指令在功能上可以等价于内存屏障，可以让其立即刷入主存。
#### invalid queue
执行失效也不是一个简单的操作，它需要处理器去处理。另外，存储缓存（Store Buffers）并不是无穷大的，所以处理器有时需要等待失效确认的返回。这两个操作都会使得性能大幅降低。为了应付这种情况，引入了失效队列。它们的约定如下：
- 对于收到的使失效信号, 应马上返回
- 使失效动作并不马上执行, 而是放入一个队列中
- 此 cpu 将不会对该缓存进行任何操作, 直到其执行了使失效动作

#### 内存屏障 (Memory Barriers)
写屏障 Store Memory Barrier(a.k.a. ST, SMB, smp_wmb), 告诉处理器, 马上应用存储在 store buffer 里的全部指令
读屏障 Load Memory Barrier (a.k.a. LD, RMB, smp_rmb) 告诉处理器, 马上应用存储在失效队列里的全部指令
```java
void executedOnCpu0() {
    value = 10;
    //在更新数据之前必须将所有存储缓存（store buffer）中的指令执行完毕。
    storeMemoryBarrier();
    finished = true;
}
void executedOnCpu1() {
    while(!finished);
    //在读取之前将所有失效队列中关于该数据的指令执行完毕。
    loadMemoryBarrier();
    assert value == 10;
}
```

## JMM (Java 内存模型)
在当前 Java 内存模型, 线程可以把变量保存本地内存(抽象概念)（比如机器的寄存器）中，而不是直接在主存中进行读写。这就可能造成一个线程在主存中修改了一个变量的值，而另外一个线程还继续使用它在寄存器中的变量值的拷贝，造成数据的不一致。

## 并发编程的三个重要特性
1. 原子性 : 一个的操作或者多次操作，要么所有的操作全部都得到执行并且不会收到任何因素的干扰而中断，要么所有的操作都执行，要么都不执行。synchronized 可以保证代码片段的原子性。
2. 可见性 ：当一个变量对共享变量进行了修改，那么另外的线程都是立即可以看到修改后的最新值。volatile 关键字可以保证共享变量的可见性。
3. 有序性 ：代码在执行的过程中的先后顺序，Java 在编译器以及运行期间的优化，代码的执行顺序未必就是编写代码时候的顺序。volatile 关键字可以禁止指令进行重排序优化。

# ThreadLocal
实例
```java
//使用泛型
private ThreadLocal<SimpleDateFormat> t;
t.set(new SimpleDateFormat());
t.get();
t.remove();
```

## 原理
```java
Thread 类里
//与此线程有关的ThreadLocal值。由ThreadLocal类维护
ThreadLocal.ThreadLocalMap threadLocals = null;
```
Thread 类中有一个 ThreadLocalMap 类型的变量, 可以理解为一个 HashMap , key 为 threadlocal 实例, 而value 则为 实际要保存的对象
==ThreadLocalmap 使用开放地址法(中的线性探测法)来解决哈希冲突, 因为在ThreadLocalMap中的散列值分散得十分均匀，很少会出现冲突。并且ThreadLocalMap经常需要清除无用的对象，使用纯数组更加方便。==

![](https://snailclimb.gitee.io/javaguide/docs/java/multi-thread/images/threadlocal%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png)

## threadlocal 内存泄漏问题
ThreadLocalMap 中使用的 key 为 ThreadLocal 的弱引用,而 value 是强引用。所以，如果 ThreadLocal 没有被外部强引用的情况下，在垃圾回收的时候，key 会被清理掉，而 value 不会被清理掉。这样一来，ThreadLocalMap 中就会出现 key 为 null 的 Entry。假如我们不做任何措施的话，value 永远无法被 GC 回收，这个时候就可能会产生内存泄露。ThreadLocalMap 实现中已经考虑了这种情况，在调用 set()、get()、remove() 方法的时候，会清理掉 key 为 null 的记录。使用完 ThreadLocal方法后 最好手动调用remove()方法

弱引用:
> 如果一个对象只具有弱引用，那就类似于可有可无的生活用品。弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它 所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。
弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中。

# ReentrantLock
## lock，tryLock，lockInterruptibly 的区别
这三者都是锁定的方法
lock() 方法与 synchronized 关键字比较类似, 获取不到锁则一直处于阻塞状态, 即使被打断都不会退出阻塞
tryLock() 方法则是马上试图获取锁(即使该锁是公平锁), 不管拿不拿到锁都立刻返回, 拿到返回 true. 而带时间参数的tryLock(time) 方法, 则会在该时间内重试, 该方法则可用于公平锁与非公平锁
lockInterruptibly() 则优先响应中断, 在请求锁时, 如果被interrupt, 则会要求处理异常

# AQS
AQS 全称是 AbstractQueuedSynchronizer，是一个用来构建锁和同步器的框架，它维护了==一个共享资源 state 和一个 FIFO 的等待队列==（即上文中管程的入口等待队列），底层利用了 CAS 机制来保证操作的原子性

![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1603524624017-016776a3-2d50-40bb-bd2c-705ae3b171a0.jpeg)

实现独占锁为例（即当前资源只能被一个线程占有），其实现原理如下：
1. state 初始化 0，在多线程条件下，线程要执行临界区的代码，必须首先获取 state，
2. 某个线程获取成功之后， state 加 1，其他线程再获取的话由于共享资源已被占用，所以会到 FIFO 等待队列去等待
3. 等占有 state 的线程执行完临界区的代码释放资源( state 减 1)后，会唤醒 FIFO 中的下一个等待线程（head 中的下一个结点）去获取 state

==state 由于是多线程共享变量，所以必须定义成 volatile==，以保证 state 的可见性, 同时虽然 volatile 能保证可见性，但不能保证原子性，所以 AQS 提供了对 state 的原子操作方法，保证了线程安全

另外 AQS 中实现的 FIFO 队列（CLH 队列）其实是双向链表实现的，由 head, tail 节点表示，head 结点代表当前占用的线程，其他节点由于暂时获取不到锁所以依次排队等待锁释放, ==并且每个线程的等待状态 (waitStatus) 是在前一个节点中==
 
同时, head 节点指向的==线程永远是 null==, 此时可以理解为当前线程正在工作无需记录, 并且此节点可以服务后续节点, 比如对于公平锁, 当前节点的前序节点是头节点, 则此节点有机会获取到资源, 则会自旋, 而如果前序节点非头节点, 则直接排队, 无须等待

节点的结构体
```c
static final class Node {
    static final Node SHARED = new Node();//标识等待节点处于共享模式
    static final Node EXCLUSIVE = null;//标识等待节点处于独占模式
    static final int CANCELLED = 1; //由于超时或中断，节点已被取消
    static final int SIGNAL = -1;  // 节点阻塞（park）必须在其前驱结点为 SIGNAL 的状态下才能进行，如果结点为 SIGNAL,则其释放锁或取消后，可以通过 unpark 唤醒下一个节点，
    static final int CONDITION = -2;//表示线程在等待条件变量（先获取锁，加入到条件等待队列，然后释放锁，等待条件变量满足条件；只有重新获取锁之后才能返回）
    static final int PROPAGATE = -3;//表示后续结点会传播唤醒的操作，共享模式下起作用
    //等待状态：对于condition节点，初始化为CONDITION；其它情况，默认为0，通过CAS操作原子更新
    volatile int waitStatus;
```
## 公平锁
**加锁**
1. t1 进入 tryacquire 判断是否获取到资源, 有两种可能可以获取到
    1. 对于公平锁来说, 此时先判断没有队列(head == tail == null), 并且 CAS 设置 state 为 1, CAS 设置当前锁的线程为 t1 都成功
    2. 当前锁的线程是自己 t1, 由于是可重入锁, 直接把 state + 1
    ![](https://pic4.zhimg.com/80/v2-83ecd54b32587f159b1fcd694b652eb3_1440w.jpg)
    ```java
    final void lock() {
        acquire(1);
    }

    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        //获取状态量
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&//判断是否需要排队
                compareAndSetState(0, acquires))//CAS判断 {
                setExclusiveOwnerThread(current);//设置当前锁的线程
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {//可重入锁
            int nextc = c + acquires;
            if (nextc < 0)//避免整数溢出
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    //判断自己要不要排队
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; //为空Read fields in reverse initialization order
        Node h = head;//为空
        Node s;
        return h != t && // null!=null，刚开始的时候，返回false
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
    ```
2. t2 进来 tryAcquire, 
    1. 由于 t1 已经获取到资源(state > 0 && currentThread != t2), 所以 t2 获取资源的两种条件都不满足, 
    2. 但由于此时 t2 的前序节点是 head, 因此会先自旋一次尝试获得资源, 
    3. 修改前节点的 waitStatus 为 signal , 再自旋一次, 
    4. 如果还拿不到锁, 则进入阻塞状态等待

    ![](https://pic2.zhimg.com/80/v2-b1fb164007442b1c57dd6ea6403731e1_1440w.jpg)
    ```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && //此时抢不到锁
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode); //新建node
        // Try the fast path of enq; backup to full enq on failure
       //提前直接插入，避免了检测创建新队列的操作
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        } 
        //入队
        enq(node);
        return node;
    }
    private Node enq(final Node node) {
        for (;;) {//死循环
            Node t = tail;
            if (t == null) { // 创建队列
                if (compareAndSetHead(new Node()))//CAS地去创建一个新头部
                    tail = head;
            } 
            else {
                node.prev = t;//插入末尾
                if (compareAndSetTail(t, node)) {//让tail指向t2
                    t.next = node;//插入队尾
                    return t;
                }
            }
        }
    }
    // 在队列中尝试获取锁acquireQueued
    final boolean acquireQueued(final Node node, int arg) {
            boolean failed = true;
            try {
                boolean interrupted = false;
                for (;;) { //死循环
                    final Node p = node.predecessor();//拿出上一个节点
                    //判断上一个节点是不是头部，如果是头部则会尝试获取锁==自旋锁
                    if (p == head && tryAcquire(arg)) {
                        setHead(node);
                        //此时前node已经没有引用，就会被GC。所以thread=null也是为了gc无引用 
                        p.next = null; // help GC 
                        failed = false;
                        return interrupted;
                    }
                    if (shouldParkAfterFailedAcquire(p, node) &&//第一次返回false,第二次返回true
                        parkAndCheckInterrupt())//第二次返回true之后才会执行
                        interrupted = true;
                }
            } finally {
                // 如果线程自旋中因为异常等原因获取锁最终失败，则调用此方法
                if (failed)
                    cancelAcquire(node);
            }
        }
    //为了可以二次自旋,这个函数的名字表示自旋失败后应该park
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;//默认是0
        if (ws == Node.SIGNAL)
            /*
            * This node has already set status asking a release
            * to signal it, so it can safely park.
            */
            return true;
        if (ws > 0) {
            /*
            * Predecessor was cancelled. Skip over predecessors and
            * indicate retry.
            */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
            * waitStatus must be 0 or PROPAGATE.  Indicate that we
            * need a signal, but don't park yet.  Caller will need to
            * retry to make sure it cannot acquire before parking.
            */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL); //把ws改为-1
        }
        return false;
    }
    ```
3. 此时由于前面有非头节点的 t2 在排队, 因此也不需要尝试获得锁, 直接进入排队并修改前节点的 waitStatus 为 signal 后进入阻塞
    ![](https://pic1.zhimg.com/80/v2-4b3af12734ae266ba3660b133a02060c_1440w.jpg)

**解锁**
1. 先将 state - 1, 因为可重入的特性, 只有当 state = 0 才能释放锁
2. 如果头节点有后续节点, 且 waitStatus != 0 (非初始状态), 则尝试唤醒后续节点
3. 首先检查后续节点是否已取消 (waitStatus = CANCEL = 1), 如果是, 则从队尾 tail 往前搜索没有取消的节点进行唤醒
4. 节点唤醒后, 继续 CAS 将头节点设置为当前节点, 此时可以帮助 gc, 把原头节点的 next 设置为 null, 本头节点的 Thread 也可以设置为 null

> 寻找队列的第一个非取消状态的节点为啥要从后往前找呢，因为节点入队并不是原子操作, 如果 unparkSuccessor 刚好在这两者之间执行，此时是找不到  head 的后继节点的，如下
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1603524623970-933c1be7-64cc-4334-81c2-226bd7856233.jpeg)
```java
Node pred = tail;
if (pred != null) {
    node.prev = pred;
    if (compareAndSetTail(pred, node)) {
        pred.next = node;
        return node;
```

![](https://pic2.zhimg.com/80/v2-a672344aecf480d4dbe9c8317ff949f5_1440w.jpg)
![](https://pic1.zhimg.com/80/v2-aec008761f78022454117cd181f031b8_1440w.jpg)

## 非公平锁
```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 少了一个判断是否需要排队的操作
        // 只要当前 state 为空就直接尝试上锁
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

## 公平锁与非公平锁的区别
1. 非公平锁会在一开始进锁方法进行一次 CAS 获得锁
2. 非公平锁后续线程进来后, 不需要判断是否需要排队, 但如果抢不到锁同样需要进入队列等待, 所以在队列里也是公平的

整体来说, 非公平锁的效率较高, 因为减少了线程阻塞的次数, 同时也不需要判断是否需要排队, 但可能会导致饥饿问题

# 乐观锁 (CAS)
## Unsafe 类
Java 无法直接使用底层内存, 这使得 Java 相对安全
1. 不能直接修改别的类的变量
2. 内存交由 jvm 管理, 可以不显式进行垃圾回收

然而, jvm 还是开了后门, Unsafe 类可以提供硬件级别的**原子操作**, 但是 jdk 并没有把该类暴露出来, 因此对其的使用是受限制

## CAS
当前的处理器基本都支持CAS，只不过不同的厂家的实现不一样罢了。

基本思想: CAS有三个操作数：内存值V、旧的预期值A、要修改的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则什么都不做并返回false。

## 原子类包
**AtomicInteger 的类变量**
```java
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        valueOffset = unsafe.objectFieldOffset
         (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}
 
private volatile int value;
```

1. valueOffset表示的是变量值在内存中的偏移地址，因为Unsafe就是根据内存偏移地址获取数据的原值的
2. value是用volatile修饰的，这是非常关键的
